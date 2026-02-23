---
layout: post
title: "Terraform으로 기존 AWS 인프라를 코드로 관리하기 (IaC)"
date: 2026-02-23
categories: [DevOps, Terraform]
tags: [Terraform, AWS, IaC, EC2, RDS, KMS, IAM, Spring Boot]
---

## 개요

기존에 AWS 콘솔에서 수동으로 생성해서 운영 중이던 AWS 인프라(EC2, RDS, KMS, Security Group, IAM)를 Terraform으로 코드화하는 작업을 진행했습니다.

이미 운영 중인 리소스를 Terraform `import`를 통해 코드로 가져오는 과정을 정리합니다.

---

## 프로젝트 구성

Spring Boot + AWS KMS 기반의 회원 정보 암호화 서비스입니다.

| 구성요소 | 설명 |
|----------|------|
| Spring Boot 3.2 | 회원 CRUD API |
| AWS RDS PostgreSQL | 회원 데이터 저장 |
| AWS KMS | 회원 이름 컬럼 암호화 |
| EC2 + Nginx | 앱 서버 + 리버스 프록시 |
| Jenkins | CI/CD 파이프라인 |

---

## Terraform이란?

**Infrastructure as Code (IaC)** 도구로, AWS 콘솔에서 마우스로 클릭해 만들던 인프라를 `.tf` 파일에 코드로 선언해서 자동으로 생성/수정/삭제할 수 있습니다.

```bash
terraform init    # 초기화 (플러그인 다운로드)
terraform plan    # 변경사항 미리보기
terraform apply   # 실제 AWS에 적용
terraform destroy # 모든 리소스 삭제
```

**핵심 장점:**
- **재현 가능** — 환경을 날려도 `terraform apply` 한 번으로 동일하게 복구
- **팀 협업** — 인프라 변경사항을 Git으로 관리 (코드 리뷰 가능)
- **실수 방지** — `plan`으로 적용 전에 변경사항 미리 확인

---

## Terraform 파일 구조

```
terraform/
├── provider.tf          # AWS 연결 설정 (리전, 인증)
├── variables.tf         # 변수 선언
├── vpc.tf               # VPC, 서브넷, 게이트웨이
├── security_groups.tf   # 보안 그룹
├── kms.tf               # KMS 키 및 별칭
├── iam.tf               # IAM Role, Policy, Instance Profile
├── rds.tf               # RDS PostgreSQL
├── ec2.tf               # EC2 인스턴스
├── import.tf            # 기존 리소스 import 블록
├── outputs.tf           # 출력값 (IP, 엔드포인트 등)
└── terraform.tfvars.example  # 변수 값 예시
```

### provider.tf

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "springboot-kms-demo"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

---

## 기존 리소스 Import

이미 AWS에 존재하는 리소스를 Terraform이 모르는 상태에서 "이 리소스는 내가 관리할게"라고 등록하는 작업입니다.

### Import 방식 (Terraform 1.5+)

```hcl
# import.tf
import {
  to = aws_db_instance.postgres  # Terraform 코드에서의 이름
  id = "database-1"              # AWS에서의 식별자
}
```

구버전 방식 (CLI):
```bash
terraform import aws_db_instance.postgres database-1
```

---

## 1. RDS Import

### 주의사항: import 전 반드시 plan 확인

처음 import를 시도했을 때 아래와 같은 위험한 메시지가 출력됐습니다.

```
# aws_db_instance.postgres must be replaced
# Warning: this will destroy the imported resource ← 기존 RDS 삭제됨!
```

**원인:**
1. `db_name = "postgres"` 추가 → 기존 RDS에 없던 값 → 교체 강제
2. `storage_encrypted = true` 누락 → 기존 RDS 설정과 불일치 → 교체 강제

**해결:** 실제 RDS 설정값과 코드를 일치시키고, `lifecycle.ignore_changes`로 불필요한 변경을 방지

```hcl
resource "aws_db_instance" "postgres" {
  identifier     = "database-1"
  engine         = "postgres"
  engine_version = "17.6"
  instance_class = "db.t4g.micro"

  allocated_storage     = 20
  max_allocated_storage = 1000
  storage_type          = "gp2"
  storage_encrypted     = true  # 반드시 명시! 없으면 RDS 재생성됨

  username = var.db_username
  password = var.db_password

  db_subnet_group_name = data.aws_db_subnet_group.existing.name
  publicly_accessible  = false

  copy_tags_to_snapshot        = true
  monitoring_interval          = 60
  performance_insights_enabled = true
  backup_retention_period      = 1

  skip_final_snapshot = true

  lifecycle {
    ignore_changes = [
      password,
      engine_version,
      backup_window,
      maintenance_window,
      monitoring_role_arn,
      performance_insights_kms_key_id,
      kms_key_id,
      vpc_security_group_ids,
    ]
  }
}
```

### plan 결과 확인 후 apply

```bash
# RDS만 타겟으로 apply
terraform apply -target=aws_db_instance.postgres -auto-approve
```

```
Apply complete! Resources: 1 imported, 0 added, 1 changed, 0 destroyed.

Outputs:
rds_endpoint = "database-1.xxxx.ap-southeast-2.rds.amazonaws.com:5432"
```

---

## 2. EC2 Import

AWS CLI로 기존 EC2 정보를 먼저 조회했습니다.

```bash
aws ec2 describe-instances \
  --region ap-southeast-2 \
  --filters "Name=ip-address,Values=3.25.254.235" \
  --query "Reservations[0].Instances[0].{InstanceId:InstanceId,InstanceType:InstanceType,...}"
```

기존 EC2에 맞게 코드를 작성합니다. 핵심은 **기존 서브넷, 보안 그룹, IAM 프로파일을 data source로 참조**하는 것입니다.

```hcl
# 기존 서브넷 조회 (새로 만들지 않고 기존 것 참조)
data "aws_subnet" "existing" {
  id = "subnet-05ca7c4ca412c4227"
}

resource "aws_instance" "app_server" {
  ami           = "ami-0ba8d27d35e9915fb"
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnet.existing.id
  key_name      = var.ec2_key_pair_name

  vpc_security_group_ids = [
    "sg-03b8290f6a97f486d",
    "sg-01cc0b236d6d4c00c",
  ]

  iam_instance_profile = "springboot-ec2-role"

  root_block_device {
    volume_type = "gp3"
    volume_size = 10
  }

  lifecycle {
    ignore_changes = [
      ami,
      user_data,
    ]
  }
}
```

```bash
terraform apply -target=aws_instance.app_server -auto-approve
```

```
Apply complete! Resources: 1 imported, 0 added, 1 changed, 0 destroyed.

Outputs:
ec2_public_ip = "3.25.254.235"
ssh_command   = "ssh -i ~/.ssh/nginx-key.pem ubuntu@3.25.254.235"
```

---

## 3. KMS Key + Alias Import

KMS 키 조회:

```bash
aws kms list-aliases --region ap-southeast-2
aws kms describe-key --key-id 0a723299-3439-4424-af47-9fc6c9e1374b
aws kms get-key-policy --key-id 0a723299-... --policy-name default
```

```hcl
resource "aws_kms_key" "demo_key" {
  description             = "멤버 이름 컬럼 암호화용 키"
  enable_key_rotation     = false
  deletion_window_in_days = 7

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      # ... EC2 Role에 Encrypt/Decrypt 권한 부여
    ]
  })
}

resource "aws_kms_alias" "demo_key_alias" {
  name          = "alias/alias/member-name-key"
  target_key_id = aws_kms_key.demo_key.key_id
}
```

```
Apply complete! Resources: 2 imported, 0 added, 1 changed, 0 destroyed.

Outputs:
kms_key_arn   = "arn:aws:kms:ap-southeast-2:...:key/0a723299-..."
kms_key_alias = "alias/alias/member-name-key"
```

---

## 4. Security Group Import

```hcl
# 기존 Default VPC 조회
data "aws_vpc" "default" {
  default = true
}

# EC2용 보안 그룹 (launch-wizard-1)
resource "aws_security_group" "launch_wizard" {
  name   = "launch-wizard-1"
  vpc_id = data.aws_vpc.default.id

  ingress { from_port = 80;   to_port = 80;   protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 443;  to_port = 443;  protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 8080; to_port = 8080; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 0;    to_port = 0;    protocol = "-1";  cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0;    to_port = 0;    protocol = "-1";  cidr_blocks = ["0.0.0.0/0"] }

  lifecycle {
    ignore_changes = [ingress, egress, description]
  }
}

# EC2 → RDS 연결 전용 보안 그룹 (ec2-rds-1)
# AWS가 RDS 콘솔 연결 설정 시 자동 생성한 그룹
resource "aws_security_group" "ec2_rds" {
  name   = "ec2-rds-1"
  vpc_id = data.aws_vpc.default.id

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = ["sg-021073697c9f1e0db"]  # RDS 보안 그룹
  }

  lifecycle {
    ignore_changes = [ingress, egress, description]
  }
}
```

---

## 5. IAM Role + Policy Import

```hcl
resource "aws_iam_role" "ec2_role" {
  name = "springboot-ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# KMS 사용 권한 (인라인 정책)
resource "aws_iam_role_policy" "ec2_kms_policy" {
  name = "ec2-kms"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"]
      Resource = aws_kms_key.demo_key.arn
    }]
  })
}

# SSM Session Manager 접속 권한 (관리형 정책)
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# EC2에 Role을 붙이는 컨테이너
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "springboot-ec2-role"
  role = aws_iam_role.ec2_role.name
}
```

IAM 리소스 import 형식:

| 리소스 | import id 형식 |
|--------|----------------|
| `aws_iam_role` | Role 이름 |
| `aws_iam_role_policy` | `ROLE_NAME:POLICY_NAME` |
| `aws_iam_role_policy_attachment` | `ROLE_NAME/POLICY_ARN` |
| `aws_iam_instance_profile` | Profile 이름 |

```bash
terraform apply \
  -target=aws_security_group.launch_wizard \
  -target=aws_security_group.ec2_rds \
  -target=aws_iam_role.ec2_role \
  -target=aws_iam_role_policy.ec2_kms_policy \
  -target=aws_iam_role_policy_attachment.ssm \
  -target=aws_iam_instance_profile.ec2_profile \
  -auto-approve
```

```
Apply complete! Resources: 6 imported, 0 added, 4 changed, 0 destroyed.
```

---

## 최종 결과

`terraform apply` 완료 후 출력되는 정보:

```
Outputs:

app_url        = "http://3.25.254.235"
ec2_public_ip  = "3.25.254.235"
ec2_public_dns = "ec2-3-25-254-235.ap-southeast-2.compute.amazonaws.com"
kms_key_alias  = "alias/alias/member-name-key"
kms_key_arn    = "arn:aws:kms:ap-southeast-2:...:key/0a723299-..."
rds_endpoint   = "database-1.xxxx.ap-southeast-2.rds.amazonaws.com:5432"
ssh_command    = "ssh -i ~/.ssh/nginx-key.pem ubuntu@3.25.254.235"
```

### Terraform으로 관리되는 전체 리소스

| 리소스 | AWS 이름 |
|--------|----------|
| EC2 인스턴스 | `i-025f7503ab032a929` |
| RDS PostgreSQL | `database-1` |
| KMS Key | `0a723299-3439-4424-af47-9fc6c9e1374b` |
| KMS Alias | `alias/alias/member-name-key` |
| Security Group (EC2용) | `launch-wizard-1` |
| Security Group (RDS 연결용) | `ec2-rds-1` |
| IAM Role | `springboot-ec2-role` |
| IAM Inline Policy | `ec2-kms` |
| IAM Policy Attachment | `AmazonSSMManagedInstanceCore` |
| IAM Instance Profile | `springboot-ec2-role` |

---

## import 작업 시 주의사항 정리

**1. apply 전 반드시 plan 결과 확인**

`-/+` (destroy and replace) 표시가 있으면 기존 리소스가 삭제되므로 절대 apply하면 안 됩니다.

```
-/+ resource "aws_db_instance" "postgres" {
    # Warning: this will destroy the imported resource
```

**2. 실제 AWS 설정과 코드를 정확히 일치시켜야 함**

콘솔에서 직접 확인한 값을 코드에 반영해야 합니다. 다르면 불필요한 변경이 발생합니다.

**3. lifecycle ignore_changes 적극 활용**

AWS가 자동으로 관리하거나 민감한 값들은 Terraform이 건드리지 않도록 설정합니다.

```hcl
lifecycle {
  ignore_changes = [
    password,        # 비밀번호
    engine_version,  # 마이너 버전 자동 업그레이드
    backup_window,   # AWS 자동 지정 값
  ]
}
```

**4. 민감 파일은 반드시 .gitignore 처리**

```
terraform/terraform.tfvars     # 비밀번호 포함
terraform/*.tfstate            # 계정 ID, ARN 등 포함
terraform/.terraform/          # 플러그인 바이너리
```

---

## 참고 링크

- [Terraform AWS Provider 공식 문서](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [GitHub 소스코드](https://github.com/par3k/springboot-kms-demo)
