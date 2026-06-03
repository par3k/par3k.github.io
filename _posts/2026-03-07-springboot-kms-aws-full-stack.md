---
layout: post
title: "Spring Boot KMS 프로젝트 - 보안 강화, AWS 인프라 재구성, CI/CD 전체 과정"
date: 2026-03-07
categories: [테크]
tags: [Spring Boot, AWS, KMS, Terraform, Jenkins, PostgreSQL, JWT, Security, Parameter Store]
---

## 개요

Spring Boot + AWS KMS 기반 회원 정보 암호화 프로젝트를 전면적으로 개선한 과정을 정리합니다.

주요 작업 내용은 아래와 같습니다.

- Java 패키지 구조 리팩토링 (`common` 패키지 도입)
- `application.yml` 보안 취약점 수정
- AWS Parameter Store 연동으로 설정값 중앙 관리
- Terraform으로 AWS 인프라 전체 재구성 (import → 신규 생성)
- Jenkins CI/CD 자동화 배포
- 신규 기능 추가 (로그인 이력, ADMIN 권한 제어)

---

## 1. 패키지 구조 리팩토링

### 기존 구조의 문제점

기존 코드는 공통으로 사용하는 클래스들이 각 패키지에 혼재되어 있었습니다.

- `controller/GlobalExceptionHandler` → 컨트롤러가 아닌 공통 예외 처리
- `service/KmsConfig`, `service/KmsEncryptionService` → 인프라 유틸인데 서비스 패키지에 위치
- `auth/JwtUtil`, `auth/JwtFilter` → 인증 도메인 DTO와 공통 보안 유틸이 혼재
- `filter/LoggingFilter` → 별도 패키지로 관리

### 변경 후 구조

```
common/
├── exception/   GlobalExceptionHandler
├── filter/      LoggingFilter
├── kms/         KmsConfig, KmsEncryptionService
└── jwt/         JwtUtil, JwtFilter

auth/            LoginRequest, RegisterRequest, TokenResponse (도메인 DTO만 유지)
```

`common` 패키지 기준:
- **여러 도메인에서 공통으로 사용**하는 것
- **횡단 관심사(cross-cutting concern)**에 해당하는 것

`auth` 패키지에는 인증 도메인 전용 DTO만 남겼습니다.

---

## 2. application.yml 보안 강화

### 하드코딩된 민감 정보 제거

```yaml
# 변경 전
datasource:
  url: jdbc:postgresql://database-1.cjq8oyy2ip7x.ap-southeast-2.rds.amazonaws.com:5432/postgres?sslmode=verify-full
  username: flyasiana
  password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET:dGhpcy1pcy1hLXNlY3JldC1rZXktZm9yLWRldmVsb3BtZW50LW9ubHk...}
```

```yaml
# 변경 후
datasource:
  url: ${DB_URL}          # RDS 엔드포인트 노출 제거
  username: ${DB_USERNAME} # 사용자명 하드코딩 제거
  password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET}   # 기본값(fallback) 제거
```

세 가지 문제를 수정했습니다.

1. **RDS 엔드포인트 하드코딩** → Git 히스토리에 AWS 인프라 정보 노출
2. **DB 사용자명 하드코딩** → 소스코드에 접근 정보 노출
3. **JWT secret 기본값** → 환경변수 미설정 시 취약한 키로 서비스 동작

---

## 3. AWS Parameter Store 연동

### 선택 이유

| 방법 | 비용 | 특징 |
|---|---|---|
| Parameter Store | 무료 | Key-Value, SecureString(KMS 암호화) |
| Secrets Manager | 월 $0.40/시크릿 | JSON 객체, 자동 rotation |

이미 KMS를 사용 중이고 소규모 프로젝트이므로 Parameter Store를 선택했습니다.

### Spring Cloud AWS 설정

**pom.xml**

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.awspring.cloud</groupId>
            <artifactId>spring-cloud-aws-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-parameter-store</artifactId>
</dependency>
```

**application.yml**

```yaml
spring:
  config:
    import: "optional:aws-parameterstore:/springboot-kms-demo/"
```

`optional:` 접두사 덕분에 로컬 개발 환경에서 AWS 미연결 시에도 정상 기동됩니다. AWS 환경에서는 `/springboot-kms-demo/` 하위 파라미터를 자동으로 로드합니다.

### 등록한 파라미터

| 경로 | 타입 | 설명 |
|---|---|---|
| `/springboot-kms-demo/DB_URL` | String | RDS 접속 URL |
| `/springboot-kms-demo/DB_USERNAME` | String | DB 사용자명 |
| `/springboot-kms-demo/DB_PASSWORD` | **SecureString** | DB 비밀번호 |
| `/springboot-kms-demo/KMS_KEY_ID` | String | KMS 키 ARN |
| `/springboot-kms-demo/JWT_SECRET` | **SecureString** | JWT 서명 키 |

### EC2 IAM 권한 (Terraform)

```hcl
resource "aws_iam_role_policy" "ec2_ssm_policy" {
  name = "ec2-ssm-parameterstore"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["ssm:GetParameter", "ssm:GetParametersByPath"]
        Resource = "arn:aws:ssm:${var.aws_region}:*:parameter/springboot-kms-demo/*"
      },
      {
        Effect = "Allow"
        Action = ["kms:Decrypt"]
        Resource = "arn:aws:kms:${var.aws_region}:${data.aws_caller_identity.current.account_id}:key/*"
        Condition = {
          StringLike = { "kms:ViaService" = "ssm.${var.aws_region}.amazonaws.com" }
        }
      }
    ]
  })
}
```

---

## 4. Terraform 인프라 재구성

### 기존 구조의 문제

기존 Terraform 코드는 이미 존재하는 리소스를 `import`로 가져온 구조였습니다. 리소스를 전부 삭제하고 나니 `import.tf`의 블록들이 모두 존재하지 않는 리소스를 참조하는 상태가 됐습니다.

또한 `ec2.tf`에 하드코딩된 서브넷 ID, 보안 그룹 ID 등 삭제된 리소스의 ID가 그대로 남아있었습니다.

### 변경 내용

**import.tf** - 모든 import 블록 제거 (신규 생성 구조로 전환)

**security_groups.tf** - 보안 그룹 간 순환 참조를 `aws_security_group_rule`로 분리

```hcl
# EC2 → RDS 방향 허용 (순환 참조 방지를 위해 인라인 규칙 대신 별도 리소스)
resource "aws_security_group_rule" "ec2_rds_egress" {
  type                     = "egress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.ec2_rds.id
  source_security_group_id = aws_security_group.rds.id
}

resource "aws_security_group_rule" "rds_ingress_from_ec2" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = aws_security_group.ec2_rds.id
}
```

**ec2.tf** - user_data로 EC2 초기화 자동화

```hcl
user_data = templatefile("${path.module}/user_data.sh", {
  aws_region = var.aws_region
})
```

**rds.tf** - 신규 DB 서브넷 그룹 생성

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "kms-demo-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}
```

### .gitignore 주의사항

`.gitignore`에서 인라인 주석은 패턴의 일부로 인식됩니다.

```bash
# 잘못된 방법 (# 이후가 패턴에 포함되어 무시 안 됨)
terraform/.terraform/          # 플러그인 캐시

# 올바른 방법 (주석을 별도 줄로 분리)
# 플러그인 캐시
terraform/.terraform/
```

이 때문에 `.terraform/` 디렉토리가 git에 포함돼 648MB 바이너리가 커밋되는 사고가 발생했습니다. GitHub에서 100MB 제한으로 push가 거부됐고, `git reset --soft`로 커밋을 되돌린 후 올바르게 처리했습니다.

### 생성된 리소스 (23개)

```
terraform apply 결과:
  EC2:  3.25.208.88
  RDS:  kms-demo-postgres.cjq8oyy2ip7x.ap-southeast-2.rds.amazonaws.com
  KMS:  arn:aws:kms:ap-southeast-2:656868887700:key/d22b1323-...
```

---

## 5. Jenkins CI/CD

### 파이프라인 구성

```groovy
pipeline {
    environment {
        EC2_HOST = '3.25.208.88'
        EC2_USER = 'ubuntu'
        APP_PATH = '/home/ubuntu/app'
    }
    stages {
        stage('Git Pull')     { ... }
        stage('Build')        { sh '/usr/bin/mvn clean package -DskipTests' }
        stage('Deploy')       { sh 'scp + ssh systemctl restart springboot' }
        stage('Health Check') { sh 'curl -f http://${EC2_HOST}/members' }
    }
}
```

### 배포 중 발생한 문제들

**문제 1: SSH Connection refused**

보안 그룹 SSH 인바운드 규칙에 `192.168.219.101/32`(사설 IP)가 등록되어 있었습니다. Jenkins가 로컬 Docker에서 실행 중이었으므로, 실제 공인 IP를 확인해서 보안 그룹에 추가했습니다.

```bash
# 현재 공인 IP 확인
curl ifconfig.me

# 보안 그룹에 추가
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp --port 22 \
  --cidr {공인IP}/32
```

**문제 2: 앱 기동 실패 (502 Bad Gateway)**

`logback-spring.xml`에 EFS 마운트 경로(`/mnt/efs/logs`)가 하드코딩되어 있었고, EFS가 없는 환경이라 앱 기동 자체가 실패했습니다.

```xml
<!-- 변경 전 -->
<springProperty name="LOG_PATH" source="logging.file.path" defaultValue="/mnt/efs/logs"/>

<!-- 변경 후 -->
<springProperty name="LOG_PATH" source="logging.file.path" defaultValue="/home/ubuntu/app/logs"/>
```

---

## 6. 신규 기능 추가

### 멀티 에이전트 워크플로우 활용

기능 추가는 CLAUDE.md에 정의한 5단계 에이전트 파이프라인으로 진행했습니다.

```
[1] 요건 분석 → [2] 개발 → [3] 코드 품질 + [4] 보안 검수 (병렬) → [5] 배포
```

### 기능 1: 마지막 로그인 이력 저장

**DB 변경**

```sql
ALTER TABLE member ADD COLUMN last_login_at TIMESTAMP NULL;
```

**MemberMapper.xml**

```xml
<update id="updateLastLoginAt">
    UPDATE member
    SET last_login_at = NOW()
    WHERE email = #{email}
</update>
```

**AuthService.java**

```java
public TokenResponse login(LoginRequest request) {
    UserDetails userDetails = loadUserByUsername(request.getEmail());
    if (!passwordEncoder.matches(request.getPassword(), userDetails.getPassword())) {
        throw new IllegalArgumentException("이메일 또는 비밀번호가 올바르지 않습니다.");
    }
    memberMapper.updateLastLoginAt(request.getEmail()); // 로그인 성공 시 기록
    return new TokenResponse(jwtUtil.generate(request.getEmail()));
}
```

### 기능 2: ADMIN 권한 체계 도입 및 기본 비밀번호 자동 설정

**DB 변경**

```sql
ALTER TABLE member ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'USER';
```

**SecurityConfig.java**

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/api/members/**").authenticated()
    .requestMatchers(HttpMethod.POST, "/api/members/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.PUT, "/api/members/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/api/members/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)
```

**MemberService.java** - ADMIN이 멤버 생성 시 기본 비밀번호 자동 설정

```java
private static final String DEFAULT_PASSWORD = "Asiana1!";

public MemberDto create(MemberDto dto) {
    Member member = toEntity(dto);

    boolean isAdmin = SecurityContextHolder.getContext()
            .getAuthentication().getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));

    if (isAdmin) {
        member.setPassword(passwordEncoder.encode(DEFAULT_PASSWORD));
    }
    member.setRole("USER");
    memberMapper.insert(member);
    return toDto(member);
}
```

**AuthService.java** - DB의 role 값을 Spring Security에 동적 반영

```java
@Override
public UserDetails loadUserByUsername(String email) {
    Member member = memberMapper.findByEmail(email);
    String role = member.getRole() != null ? member.getRole() : "USER";
    return User.withUsername(member.getEmail())
            .password(member.getPassword())
            .roles(role)  // DB role 동적 반영
            .build();
}
```

### 기능 3: ADMIN만 새 멤버 등록 버튼 표시

Thymeleaf Spring Security 통합 라이브러리를 추가하면 템플릿에서 바로 권한 확인이 가능합니다.

**pom.xml**

```xml
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>
```

**list.html**

```html
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">

<!-- ADMIN만 표시 -->
<a th:href="@{/members/new}" sec:authorize="hasRole('ADMIN')">+ 새 멤버 등록</a>
```

---

## 7. 서버 타임존 KST 설정

로그와 DB 시간값이 UTC로 찍히는 문제를 JVM 옵션으로 해결했습니다.

**systemd 서비스 수정**

```ini
[Service]
ExecStart=/usr/bin/java -Duser.timezone=Asia/Seoul -jar /home/ubuntu/app/demo-0.0.1-SNAPSHOT.jar
```

애플리케이션 레벨에서 타임존을 지정하면 DB 드라이버, 로그 등 모든 시간값이 KST 기준으로 처리됩니다.

---

## DBeaver로 RDS 연결 (SSH 터널)

RDS가 프라이빗 서브넷에 있어 직접 접근이 불가능합니다. EC2를 Bastion으로 활용한 SSH 터널링을 사용합니다.

```
DBeaver SSH 탭 설정
  Host/IP  : EC2 퍼블릭 IP
  Port     : 22
  Username : ubuntu
  Auth     : Public Key (nginx-key.pem)

DBeaver Main 탭 설정
  Host     : RDS 엔드포인트
  Port     : 5432
  Database : postgres
  Username : flyasiana
```

---

## 보안 검수 결과 요약

멀티 에이전트 워크플로우의 보안 검수 에이전트가 발견한 주요 항목입니다.

| 등급 | 내용 | 상태 |
|---|---|---|
| HIGH | 기본 비밀번호 소스코드 하드코딩 | 배포 후 개선 과제 |
| HIGH | JWT 토큰에 role 미포함 (매 요청 DB 재조회) | 배포 후 개선 과제 |
| MEDIUM | Bean Validation 미적용 | 배포 후 개선 과제 |
| MEDIUM | 운영 환경 MyBatis TRACE 로그 | 배포 후 개선 과제 |
| INFO | SQL Injection — MyBatis `#{}` 바인딩으로 안전 | 통과 |

CRITICAL 이슈가 없어 배포를 진행했고, HIGH 항목들은 다음 스프린트 개선 과제로 등록했습니다.

---

## 마무리

이번 작업을 통해 개발 → 보안 → 인프라 → 배포까지 전체 사이클을 Claude Code의 멀티 에이전트 워크플로우로 자동화했습니다. 특히 요건 분석, 코드 품질 검수, 보안 검수를 별도 에이전트로 분리하고 CRITICAL 이슈가 없을 때만 배포하는 게이트 조건이 실제 운영 환경에서도 충분히 활용 가능한 패턴임을 확인했습니다.
