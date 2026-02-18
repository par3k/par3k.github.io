---
title: "Spring Boot + AWS KMS + PostgreSQL RDS로 컬럼 암호화 서비스 구축하기"
date: 2026-02-18 21:39:00 +0900
categories: [AWS]
tags: [aws, spring-boot, kms, postgresql, rds, ec2, nginx, thymeleaf]
description: Spring Boot 3 + AWS KMS로 DB 컬럼을 암호화하고 EC2 + Nginx + RDS PostgreSQL 환경에 배포하는 과정을 정리했다.
---

## 만들게 된 계기

업무에서 개인정보가 포함된 데이터를 DB에 저장할 일이 생겼다. 단순히 암호화 알고리즘을 직접 구현하는 방법도 있지만, 키 관리와 보안을 AWS가 대신해주는 **KMS(Key Management Service)** 를 활용하면 훨씬 안전하게 운영할 수 있다고 판단했다.

이번 포스팅에서는 Spring Boot 3에서 AWS KMS로 DB 컬럼을 암호화하고, EC2 + Nginx + RDS PostgreSQL 환경에 배포하는 과정을 정리했다.

## 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| 백엔드 | Spring Boot 3.2, Java 21, MyBatis |
| DB | AWS RDS PostgreSQL 16 |
| 암호화 | AWS KMS (대칭키) |
| 서버 | AWS EC2 (Ubuntu), Nginx |
| 화면 | Thymeleaf |

## 전체 아키텍처

```
[사용자 브라우저]
       │ HTTP (80)
       ▼
[EC2 - Nginx]        ← 리버스 프록시
       │ HTTP (8080)
       ▼
[EC2 - Spring Boot]
       │
       ├── AWS KMS  ← 이름 컬럼 암/복호화
       │
       └── RDS PostgreSQL
```

## AWS KMS 설정

### 1. KMS 키 생성

AWS Console → KMS → 고객 관리형 키 → 키 생성

```
키 유형:  대칭
키 사용:  암호화 및 복호화
별칭:     alias/member-name-key
```

### 2. EC2 IAM Role 생성 및 연결

EC2가 KMS를 사용할 수 있도록 IAM Role에 아래 인라인 정책을 추가한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:ap-southeast-2:계정ID:key/키ID"
    }
  ]
}
```

IAM Role을 EC2 인스턴스에 연결하면 액세스 키 없이도 KMS를 사용할 수 있다.

### 3. KMS 동작 테스트

```bash
aws kms encrypt \
  --key-id "arn:aws:kms:ap-southeast-2:계정ID:key/키ID" \
  --plaintext "test" \
  --region ap-southeast-2
```

`CiphertextBlob` 값이 출력되면 정상이다.

## Spring Boot 구현

### 의존성 (pom.xml)

```xml
<!-- Thymeleaf -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- MyBatis -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>

<!-- PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- AWS SDK v2 KMS -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>kms</artifactId>
    <version>2.25.0</version>
</dependency>
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://database-1.xxxxxx.ap-southeast-2.rds.amazonaws.com:5432/postgres?sslmode=verify-full&sslrootcert=/certs/global-bundle.pem
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

aws:
  kms:
    key-id: ${KMS_KEY_ID}
    region: ap-southeast-2
```

민감정보는 환경변수로 관리하고, 실제 값은 `systemd` 서비스 파일에서 주입한다.

### KMS 암/복호화 서비스

```java
@Service
@RequiredArgsConstructor
public class KmsEncryptionService {

    private final KmsClient kmsClient;

    @Value("${aws.kms.key-id}")
    private String keyId;

    public String encrypt(String plainText) {
        EncryptRequest request = EncryptRequest.builder()
                .keyId(keyId)
                .plaintext(SdkBytes.fromUtf8String(plainText))
                .build();
        return Base64.getEncoder().encodeToString(
                kmsClient.encrypt(request).ciphertextBlob().asByteArray()
        );
    }

    public String decrypt(String encryptedBase64) {
        byte[] cipherBytes = Base64.getDecoder().decode(encryptedBase64);
        DecryptRequest request = DecryptRequest.builder()
                .ciphertextBlob(SdkBytes.fromByteArray(cipherBytes))
                .keyId(keyId)
                .build();
        return kmsClient.decrypt(request).plaintext().asUtf8String();
    }
}
```

### 핵심 흐름

```
[등록 요청] 평문 name → KmsEncryptionService.encrypt() → Base64 암호문 → DB 저장
[조회 요청] DB에서 Base64 암호문 → KmsEncryptionService.decrypt() → 평문 name → 화면 표시
```

DB에는 암호화된 Base64 문자열이 저장되고, 화면에는 복호화된 평문이 표시된다.

## RDS PostgreSQL 설정

### RDS 생성 시 주의사항

- **퍼블릭 액세스**: 반드시 비활성화
- **Security Group**: EC2 Security Group에서 5432 포트만 허용
- **초기 DB 이름**: 생성 시 반드시 입력 (안 하면 직접 CREATE DATABASE 해야 함)

### SSL 인증서 설정

RDS는 기본적으로 SSL 연결을 요구한다. EC2에서 아래 명령어로 인증서를 받아야 한다.

```bash
sudo mkdir -p /certs
sudo wget -O /certs/global-bundle.pem \
  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

### 테이블 생성

```sql
CREATE TABLE member (
    id    BIGSERIAL    PRIMARY KEY,
    name  TEXT         NOT NULL,   -- KMS 암호화된 Base64 문자열
    email VARCHAR(255) NOT NULL
);
```

## EC2 배포

### systemd 서비스 등록

```ini
# /etc/systemd/system/springboot.service
[Unit]
Description=Spring Boot Application
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/app

Environment="DB_PASSWORD=실제패스워드"
Environment="KMS_KEY_ID=arn:aws:kms:ap-southeast-2:계정ID:key/키ID"
Environment="AWS_REGION=ap-southeast-2"

ExecStart=/usr/bin/java -jar /home/ubuntu/app/demo-0.0.1-SNAPSHOT.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable springboot
sudo systemctl start springboot

# 로그 확인
sudo journalctl -u springboot -f
```

### Nginx 리버스 프록시 설정

```nginx
# /etc/nginx/sites-available/springboot
server {
    listen 80;
    server_name 서버IP또는도메인;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/springboot /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Nginx가 80 포트에서 요청을 받아 Spring Boot 8080으로 전달해준다. 덕분에 URL에 포트번호 없이 깔끔하게 접속할 수 있다.

## 느낀 점

- **KMS의 편리함**: 키 관리를 AWS가 해주기 때문에 직접 암호화 로직을 구현하는 것보다 훨씬 안전하고 관리가 편하다.
- **환경변수 관리**: 패스워드, KMS ARN 같은 민감정보를 코드에 하드코딩하지 않고 systemd 환경변수로 분리하는 것이 중요하다.
- **RDS SSL**: RDS 연결 시 SSL 설정을 빠뜨리면 연결 자체가 안 된다. 인증서 경로 설정을 꼭 확인하자.
- **IAM Role**: EC2에 IAM Role을 붙이면 액세스 키 없이도 AWS 서비스를 사용할 수 있어서 보안상 훨씬 낫다.
