---
layout: post
title: "JWT에서 Redis 서버사이드 세션으로 전환한 이유 — ALB 이중화 환경의 세션 공유와 보안 강화"
date: 2026-03-29
categories: [테크]
tags: [Spring Boot, JWT, Redis, ElastiCache, Session, Security, AWS, Terraform, OWASP]
---

이번 포스팅에서는 Spring Boot 서비스의 인증 방식을 **JWT에서 Redis 서버사이드 세션으로 전환**한 과정을 정리합니다. 단순한 기술 교체가 아니라, ALB + ASG 이중화 환경에서 발생한 구조적 문제를 해결하고, 보안 취약점을 수정하는 과정에서 내린 의사결정이었습니다.

---

## 왜 JWT를 버렸나?

JWT는 stateless 인증의 대표적인 방법입니다. 서버가 상태를 저장하지 않아도 되고, 수평 확장(Scale-out)이 쉽다는 장점이 있습니다. 그런데 ALB + ASG 구성으로 EC2를 이중화하면서 문제가 생겼습니다.

문제는 JWT 자체가 아니라 **로그아웃**이었습니다.

JWT는 발급 후 서버가 무효화할 수 없습니다. 사용자가 로그아웃해도 토큰의 만료 시간이 남아있으면 토큰은 여전히 유효합니다. 이를 해결하려면 블랙리스트나 토큰 버전 관리가 필요한데, 그 저장소를 어디에 둘 것인지의 문제가 됩니다.

그리고 ALB 이중화 환경에서는 또 다른 문제가 있었습니다. **두 EC2 인스턴스가 동일한 세션 상태를 공유해야 한다는 것**입니다.

결국 두 가지 선택지가 남았습니다.

1. JWT + Redis 블랙리스트: 로그아웃된 토큰을 Redis에 저장
2. Redis 서버사이드 세션: 세션 자체를 Redis에 저장, 즉시 무효화 가능

두 번째가 더 단순합니다. "JWT의 stateless 장점을 포기하지만, 그 장점을 이미 활용하지 못하고 있었다"는 판단이었습니다.

---

## 아키텍처 변경

### 기존 구조 (JWT)

```
클라이언트 → [ALB] → EC2 #1 또는 EC2 #2
                        ↓
                   JwtFilter: 토큰 검증 (서버 메모리, stateless)
                        ↓
                   JwtUtil: HS256 서명 검증
```

### 변경 후 구조 (Redis 세션)

```
클라이언트 → [ALB] → EC2 #1 또는 EC2 #2
                        ↓
                   SessionFilter: 쿠키 SESSION_ID 추출
                        ↓
                   SessionService → [ElastiCache Redis] ← 공유 세션 저장소
```

ALB가 어느 EC2로 트래픽을 보내도 Redis에서 동일한 세션을 읽습니다. 이중화 환경에서 세션 공유 문제가 해결됩니다.

---

## ElastiCache Redis 인프라 (Terraform)

```hcl
# elasticache.tf
resource "aws_elasticache_subnet_group" "redis" {
  name       = "kms-demo-redis-subnet-group"
  subnet_ids = aws_subnet.private[*].id  # 프라이빗 서브넷에만 배치
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "kms-demo-redis"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  engine_version       = "7.1"
  port                 = 6379

  subnet_group_name  = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.elasticache.id]
}
```

보안 그룹은 EC2 인스턴스에서만 6379 포트 인바운드를 허용합니다. Redis는 인터넷에 노출되지 않고, VPC 내부 프라이빗 서브넷에만 위치합니다.

---

## 세션 흐름 구현

### SessionService: UUID + TTL 24시간

```java
public String create(String email, String role) {
    String sessionId = UUID.randomUUID().toString();  // SecureRandom 기반, 128비트
    LocalDateTime now = LocalDateTime.now();
    SessionData sessionData = new SessionData(
            sessionId, email, role, now, now.plusSeconds(SESSION_TTL_SECONDS));

    sessionRedisTemplate.opsForValue()
            .set(sessionId, sessionData, SESSION_TTL_SECONDS, TimeUnit.SECONDS);

    return sessionId;
}
```

`UUID.randomUUID()`는 Java 내부적으로 `SecureRandom`을 사용합니다. 예측이 불가능한 128비트 무작위값이므로 세션 ID로 충분한 보안 강도를 가집니다.

### SessionFilter: 쿠키에서 세션 조회

```java
protected void doFilterInternal(...) {
    String sessionId = CookieUtil.resolveSessionId(request);
    if (sessionId != null && SecurityContextHolder.getContext().getAuthentication() == null) {
        SessionData sessionData = sessionService.get(sessionId);
        if (sessionData != null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(sessionData.getEmail());
            UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
    }
    chain.doFilter(request, response);
}
```

JWT 방식과 달리 로그아웃 시 `sessionRedisTemplate.delete(sessionId)` 한 줄로 세션이 즉시 무효화됩니다.

### 로그아웃이 진짜 로그아웃이 된다

JWT 방식의 핵심 한계는 "로그아웃해도 토큰이 살아있다"는 것이었습니다. Redis 세션 방식에서는 로그아웃 시 Redis에서 키를 삭제하므로, 이후 같은 세션 ID로 요청해도 인증이 실패합니다.

---

## 보안 취약점 발견 및 수정

전환 과정에서 코드 리뷰를 통해 두 가지 HIGH 수준 취약점을 발견하고 즉시 수정했습니다.

### [HIGH-1] 비밀번호 평문 로그 출력

기존 `LoggingFilter`는 모든 요청의 body를 그대로 로그에 출력했습니다.

```
INFO [REQUEST] POST /api/auth/login | body={"email":"user@example.com","password":"mypassword123"}
```

CloudWatch Logs에 연동되어 있었다면 비밀번호가 AWS 로그 스토리지에 영구 저장될 수 있었습니다. GDPR, PCI-DSS에서 인증정보를 로그에 저장하는 것은 명시적 위반입니다.

수정 방법은 단순합니다. 민감 경로에 대해서는 body를 마스킹합니다.

```java
private static final Set<String> SENSITIVE_PATHS = Set.of(
        "/login", "/register", "/api/auth/login", "/api/auth/register"
);

private String resolveRequestBody(HttpServletRequest request, ContentCachingRequestWrapper wrappedRequest) {
    if (SENSITIVE_PATHS.contains(request.getRequestURI())) {
        return "[MASKED]";  // 민감 경로는 body 출력 안 함
    }
    String body = new String(wrappedRequest.getContentAsByteArray(), StandardCharsets.UTF_8);
    return body.isBlank() ? "-" : body;
}
```

이제 로그인/회원가입 요청은 `body=[MASKED]`로 출력됩니다.

### [HIGH-2] Secure 쿠키 속성 누락

`AuthController`(REST API)에서는 쿠키에 `Secure` 속성을 올바르게 설정했지만, `AuthViewController`(폼 기반 로그인)의 헬퍼 메서드에서 `Secure` 속성이 누락되어 있었습니다.

```java
// 수정 전
private void setSessionCookie(...) {
    Cookie cookie = new Cookie(SESSION_COOKIE_NAME, value);
    cookie.setPath("/");
    cookie.setHttpOnly(true);
    cookie.setMaxAge(maxAge);
    response.addCookie(cookie);  // Secure 속성 없음
}
```

`Secure` 속성이 없으면 HTTP 연결에서도 세션 쿠키가 전송됩니다. 중간자 공격(MITM)으로 세션 ID가 탈취될 수 있는 취약점입니다.

한 가지 배운 점은 `application.yml`에 `session.cookie.secure: true`를 설정해도 **수동으로 생성한 `Cookie` 객체에는 자동 적용되지 않는다**는 것입니다. Spring의 자동 세션 관리를 사용하지 않고 직접 쿠키를 만들 때는 모든 보안 속성을 명시적으로 설정해야 합니다.

또한 `SameSite=Strict` 속성은 Jakarta Servlet의 `Cookie` API에서 직접 지원하지 않습니다. `X-Forwarded-Proto` 헤더를 기반으로 ALB 환경에서 HTTPS 여부를 판단하는 `CookieUtil.isSecure()` 유틸을 만들어 일관된 쿠키 설정을 보장했습니다.

---

## ASG 자동 배포: S3에서 JAR 다운로드

ALB + ASG 구성에서는 ASG가 자동으로 새 EC2를 띄웁니다. 이때 새 인스턴스가 최신 애플리케이션 JAR를 자동으로 내려받아 실행해야 합니다.

이를 위해 `user_data.sh`에 S3에서 JAR를 자동 다운로드하는 로직을 추가했습니다.

```bash
# AWS CLI v2 설치
curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
/tmp/aws/install

# S3에서 최신 JAR 다운로드 후 서비스 시작
aws s3 cp s3://kms-demo-app-artifacts/demo-0.0.1-SNAPSHOT.jar \
  /home/ubuntu/app/demo-0.0.1-SNAPSHOT.jar --region ${aws_region}

systemctl start springboot
```

Jenkins 파이프라인이 빌드 후 JAR를 S3에 업로드하면, ASG로 새로 생성되는 인스턴스는 user_data 실행 시 자동으로 최신 JAR를 가져옵니다. Jenkins가 모든 EC2에 직접 배포할 필요 없이, S3가 배포 아티팩트 저장소 역할을 합니다.

---

## JWT vs Redis 세션: 언제 무엇을 선택할까

| 항목 | JWT (Stateless) | Redis 세션 (Stateful) |
|------|-----------------|----------------------|
| 즉시 로그아웃 | 불가 (블랙리스트 필요) | 가능 (Redis DEL) |
| 수평 확장 | 쉬움 (공유 저장소 불필요) | Redis 공유 필요 |
| 세션 탈취 대응 | 어려움 (서버 무효화 불가) | 즉시 삭제 가능 |
| 인프라 비용 | 추가 없음 | Redis 서버 필요 |
| 마이크로서비스 | 적합 (서비스 간 토큰 전달) | 제한적 (Redis 공유 필요) |

단일 서비스에서 ALB 이중화를 사용하고, 즉시 로그아웃이 필요하다면 Redis 세션이 더 적합합니다. 마이크로서비스 환경에서 서비스 간 인증 전달이 필요하다면 JWT가 적합합니다.

---

## 마치며

JWT는 훌륭한 기술이지만 "stateless의 이점이 실제로 필요한 상황"인지를 먼저 확인해야 합니다. 이번 사례에서는 이중화 환경에서 세션 공유가 필요했고, 즉시 로그아웃이 요구사항이었습니다. JWT가 주는 stateless 이점보다 Redis 세션이 해결하는 문제가 더 컸습니다.

보안 취약점은 기능을 구현하면서 놓치기 쉽습니다. 비밀번호 로깅 문제는 LoggingFilter를 처음 만들 때부터 존재했지만, 코드 리뷰 없이는 발견하기 어려웠습니다. Secure 쿠키 누락도 "설정 파일에 secure: true가 있으니 괜찮겠지"라는 착각에서 비롯됩니다. 체계적인 보안 검토 프로세스의 필요성을 다시 한번 확인했습니다.
