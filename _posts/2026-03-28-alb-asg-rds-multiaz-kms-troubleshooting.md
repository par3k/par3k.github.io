---
layout: post
title: "AWS ALB + ASG로 EC2 이중화, RDS Multi-AZ, 그리고 KMS 키 교체 후 발생한 암호화 장애 트러블슈팅"
date: 2026-03-28
categories: [테크]
tags: [AWS, ALB, ASG, RDS, MultiAZ, KMS, Terraform, Spring Boot, 이중화, 트러블슈팅]
---

이번 포스팅에서는 단일 EC2로 운영하던 Spring Boot 서비스를 **ALB + ASG 기반 이중화 구성**으로 전환하고, **RDS Multi-AZ**를 활성화한 과정을 정리합니다. 그리고 이 과정에서 마주친 **KMS 키 교체 후 SSM 미반영으로 인한 무증상 암호화 장애** 트러블슈팅 경험을 공유합니다.

---

## 전체 아키텍처

```
Internet
    │
    ▼
[ALB] kms-demo-alb (internet-facing, HTTP:80)
    │
    ├──── [EC2 #1] ap-southeast-2a (Spring Boot :8080)
    │
    └──── [EC2 #2] ap-southeast-2c (Spring Boot :8080)
              │
              ▼
    [RDS PostgreSQL] Multi-AZ
      Primary: ap-southeast-2a
      Standby: ap-southeast-2c (자동 페일오버)
```

모든 인프라는 **Terraform**으로 관리합니다.

---

## EC2 이중화: aws_instance → Launch Template + ASG

기존에는 `aws_instance` 하나로 운영했습니다. 이를 `aws_launch_template` + `aws_autoscaling_group`으로 교체했습니다.

```hcl
# ec2.tf
resource "aws_launch_template" "app" {
  name_prefix   = "kms-demo-app-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.ec2_instance_type
  key_name      = var.ec2_key_pair_name

  network_interfaces {
    associate_public_ip_address = true
    security_groups = [
      aws_security_group.app_server.id,
      aws_security_group.ec2_rds.id,
    ]
  }

  iam_instance_profile {
    arn = aws_iam_instance_profile.ec2_profile.arn
  }
}

resource "aws_autoscaling_group" "app_servers" {
  name                = "kms-demo-asg"
  min_size            = var.asg_min_size        # 2
  max_size            = var.asg_max_size         # 4
  desired_capacity    = var.asg_desired_capacity # 2

  vpc_zone_identifier = [
    aws_subnet.public.id,           # ap-southeast-2a
    aws_subnet.public_secondary.id, # ap-southeast-2c
  ]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
}
```

ASG는 각 AZ에 인스턴스를 균등하게 분산시켜 **AZ 장애에도 서비스가 유지**되도록 합니다.

---

## ALB 설정

ALB는 두 퍼블릭 서브넷에 걸쳐 생성하고, 헬스체크는 Spring Boot Actuator를 활용합니다.

```hcl
# alb.tf
resource "aws_lb" "main" {
  name               = "kms-demo-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets = [
    aws_subnet.public.id,
    aws_subnet.public_secondary.id,
  ]
}

resource "aws_lb_target_group" "app" {
  name                 = "kms-demo-app-tg"
  port                 = 8080
  protocol             = "HTTP"
  vpc_id               = aws_vpc.main.id
  deregistration_delay = 30  # Canary 배포 속도 최적화 (기본 300초 → 30초)

  health_check {
    path                = "/actuator/health"
    interval            = 30
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200"
  }
}
```

### 헬스체크: /actuator/health

처음에는 `/members` 엔드포인트를 헬스체크로 사용했으나, Spring Security에 의해 **302 리다이렉트**가 반환되어 ALB가 unhealthy로 판정했습니다.

해결책으로 Spring Boot Actuator를 도입하고 `/actuator/health`를 인증 없이 접근 가능하도록 설정했습니다.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: never
```

```java
// SecurityConfig.java
.requestMatchers("/api/auth/**", "/login", "/register", "/actuator/health").permitAll()
```

---

## RDS Multi-AZ 활성화

`multi_az = true` 한 줄로 Standby 인스턴스가 다른 AZ에 자동 생성됩니다.

```hcl
# rds.tf
resource "aws_db_instance" "postgres" {
  # ...
  multi_az                 = true
  backup_retention_period  = 1     # 프리 티어 최대값
  skip_final_snapshot      = false
  deletion_protection      = true
}
```

> **주의**: `backup_retention_period = 7`은 프리 티어를 초과합니다. 테스트 환경에서는 `1`로 설정해야 합니다.

Primary 인스턴스 장애 시 AWS가 자동으로 Standby로 페일오버하며, 엔드포인트 주소는 그대로 유지됩니다.

---

## DBeaver에서 프라이빗 서브넷 RDS 접속 (SSH 터널링)

RDS는 프라이빗 서브넷(`publicly_accessible = false`)에 위치하므로 로컬에서 직접 접속이 불가합니다. DBeaver의 SSH 터널 기능을 사용합니다.

**DBeaver 설정:**

| 탭 | 항목 | 값 |
|----|------|-----|
| Main | Host | RDS 엔드포인트 |
| Main | Port | 5432 |
| Main | Database | postgres |
| SSH | Host/IP | EC2 공인 IP |
| SSH | User Name | ubuntu |
| SSH | Authentication | Public Key (nginx-key.pem) |

EC2가 ASG로 관리되므로 재시작 시 IP가 변경될 수 있습니다. IP 변경 시 SSH 탭의 Host만 업데이트하면 됩니다.

---

## 트러블슈팅: KMS 키 교체 후 회원가입이 무증상으로 실패하는 문제

이 프로젝트는 회원 이름을 AWS KMS로 암호화하여 저장합니다. 인프라를 재구성한 후 회원가입을 시도했지만 **데이터가 저장되지 않는 문제**가 발생했습니다.

### 증상

- 회원가입 폼 제출 후 다시 로그인 페이지로 이동 (에러 메시지 없음)
- DB에 데이터 없음
- 서버 로그에 에러 없음

### 원인 추적

서버 로그를 확인했습니다.

```
POST /register | duration=484ms | status=302   ← 첫 번째 시도
POST /register | duration=172ms | status=302   ← 두 번째 시도
POST /register | duration=50ms  | status=302   ← 세 번째 시도
```

302는 성공(→ /members)과 실패(→ /register) 모두에서 발생합니다. 더 자세히 보니 `INSERT` 쿼리 로그가 전혀 없었습니다.

```
[DEBUG] MemberMapper.findByEmail → Total: 0   ← 중복 이메일 체크
# INSERT 쿼리 없음 → 예외 발생 후 catch로 이동
```

KMS 암호화 단계에서 예외가 발생하고 있었습니다. 원인을 파악하기 위해 SSM과 Terraform 출력을 비교했습니다.

```bash
# SSM에 저장된 KMS 키
/springboot-kms-demo/KMS_KEY_ID = arn:aws:kms:.../3ead33a9-ec1b-427f-9eaf-...

# Terraform이 생성한 현재 KMS 키
kms_key_arn = arn:aws:kms:.../f2dc2d96-8f22-4519-841c-...
```

**두 키가 달랐습니다.** 이전에 `terraform destroy`로 인프라를 삭제할 때 기존 KMS 키도 함께 삭제됐고, `terraform apply`로 새 키가 생성됐지만 **SSM 파라미터는 옛날 키를 그대로 가리키고 있었습니다.**

```bash
# 삭제 예정 상태 확인
aws kms describe-key --key-id "3ead33a9-..."
# → KeyState: PendingDeletion, DeletionDate: 2026-04-01
```

### 해결

```bash
# SSM KMS_KEY_ID를 현재 키로 업데이트
aws ssm put-parameter \
  --name "/springboot-kms-demo/KMS_KEY_ID" \
  --value "arn:aws:kms:ap-southeast-2:...:key/f2dc2d96-..." \
  --type "String" \
  --overwrite \
  --region ap-southeast-2

# 두 EC2 인스턴스 재시작 (새 환경변수 반영)
sudo systemctl restart springboot
```

### 교훈

`terraform destroy` + 재생성 패턴을 사용할 때는 **외부 시스템(SSM, 환경변수 등)이 Terraform 리소스를 참조하고 있는지** 반드시 확인해야 합니다. KMS 키처럼 보안에 관여하는 리소스는 교체 후 연관된 모든 참조를 함께 업데이트해야 합니다.

Terraform outputs를 SSM에 자동으로 동기화하거나, SSM 파라미터 자체를 Terraform으로 관리하는 방법으로 이런 불일치를 방지할 수 있습니다.

```hcl
# SSM을 Terraform으로 관리하면 항상 최신 키가 반영됨
resource "aws_ssm_parameter" "kms_key_id" {
  name  = "/springboot-kms-demo/KMS_KEY_ID"
  type  = "String"
  value = aws_kms_key.member_name.arn
}
```

---

## 마치며

단일 EC2 구성에서 ALB + ASG 이중화로 전환하면서 세 가지를 얻었습니다.

1. **가용성**: EC2 또는 AZ 장애 시에도 서비스 유지
2. **무중단 배포**: Jenkins Canary 배포 파이프라인과 결합하여 배포 중 다운타임 제로
3. **DB 고가용성**: RDS Multi-AZ로 Primary 장애 시 자동 페일오버

그리고 KMS 키 교체 후 SSM 미반영 문제는 **에러 로그도 없고 사용자에게도 명확한 피드백이 없어** 원인 파악이 쉽지 않았습니다. 보안 인프라 변경 시에는 연관된 외부 참조 목록을 체크리스트로 관리하는 습관이 중요합니다.
