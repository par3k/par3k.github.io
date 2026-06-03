---
layout: post
title: "Jenkins + AWS ALB로 Canary 배포 구현하기"
date: 2026-03-25
categories: [테크]
tags: [Jenkins, AWS, ALB, Canary, 배포전략, Spring Boot, EC2]
---

## Canary 배포란?

Canary 배포란 전체 서버 중 **일부에만 먼저 새 버전을 배포**하여 검증한 후, 문제가 없으면 나머지 서버에 순차적으로 확대하는 배포 방식입니다.

이름의 유래는 과거 광부들이 탄광에 **카나리아 새**를 먼저 들여보내 유해 가스를 감지했던 것에서 왔습니다. 일부 서버가 카나리아 역할을 하여 문제를 조기에 발견하는 것이 핵심입니다.

---

## 다른 배포 방식과의 비교

| 방식 | 설명 | 롤백 속도 | 리소스 비용 |
|------|------|-----------|-------------|
| **Canary** | 일부 서버에 먼저 배포 → 검증 → 전체 확대 | 빠름 | 낮음 |
| **Blue/Green** | 구버전/신버전 환경을 통째로 교체 | 매우 빠름 | 높음 (2배) |
| **Rolling** | 서버를 순차적으로 하나씩 교체 | 느림 | 낮음 |
| **A/B Testing** | 사용자 그룹별로 다른 버전 제공 | 중간 | 중간 |

Canary 배포는 Blue/Green 대비 **추가 인프라 비용 없이** 안전하게 배포할 수 있다는 점이 큰 장점입니다. 특히 수동 승인 단계(Manual Gate)와 결합하면 운영 중 위험을 효과적으로 통제할 수 있습니다.

---

## 배포 흐름

```
[1] 빌드
    └── 새 버전 JAR 빌드 (Maven)

[2] Canary 배포 (1개 서버)
    ├── ALB Target Group에서 Canary 서버 제외
    ├── 새 버전 JAR 배포 및 서비스 재시작
    ├── 헬스체크 확인 (/actuator/health)
    └── ALB에 Canary 서버 재등록

[3] 수동 검증 (Manual Gate)
    ├── 담당자가 Canary 서버에서 기능 직접 확인
    └── 이상 없으면 전체 배포 승인 / 문제 발견 시 롤백

[4] 전체 배포 (Rolling)
    └── 나머지 서버에 순차적으로 동일하게 배포

[5] 완료
```

핵심은 **[3] 수동 검증 단계**입니다. Jenkins의 `input` 스텝을 활용하면, 파이프라인이 여기서 멈추고 담당자의 승인을 기다립니다. 이 사이에 Canary 서버에서 충분히 테스트한 뒤 전체 배포를 결정할 수 있습니다.

---

## Jenkins Pipeline 전체 코드

```groovy
pipeline {
    agent any

    environment {
        EC2_USER    = 'ubuntu'
        APP_PATH    = '/home/ubuntu/app'
        JAR_NAME    = 'app.jar'
        PEM_KEY     = '/var/jenkins_home/.ssh/key.pem'
        CANARY_HOST = '10.0.1.10'        // Canary 서버 (1번)
        FULL_HOSTS  = '10.0.1.11 10.0.1.12'  // 나머지 서버
        TG_ARN      = 'arn:aws:elasticloadbalancing:...'  // ALB Target Group ARN
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Canary 배포') {
            steps {
                script {
                    // 1. ALB에서 Canary 서버 제외 (드레이닝 대기)
                    sh """
                        aws elbv2 deregister-targets \
                            --target-group-arn ${TG_ARN} \
                            --targets Id=${CANARY_HOST}
                        sleep 10
                    """

                    // 2. JAR 배포 및 서비스 재시작
                    sh """
                        scp -i ${PEM_KEY} -o StrictHostKeyChecking=no \
                            target/${JAR_NAME} ${EC2_USER}@${CANARY_HOST}:${APP_PATH}/
                        ssh -i ${PEM_KEY} -o StrictHostKeyChecking=no \
                            ${EC2_USER}@${CANARY_HOST} \
                            'sudo systemctl restart app'
                    """

                    // 3. 헬스체크 통과 확인
                    sh "sleep 20 && curl -f http://${CANARY_HOST}/actuator/health || exit 1"

                    // 4. ALB에 재등록
                    sh """
                        aws elbv2 register-targets \
                            --target-group-arn ${TG_ARN} \
                            --targets Id=${CANARY_HOST}
                    """
                }
            }
        }

        stage('라이브 테스트 승인') {
            // 1시간 내 미응답 시 자동 중단 후 Canary 롤백
            timeout(time: 1, unit: 'HOURS') {
                steps {
                    input message: 'Canary 서버 테스트를 완료했나요? 전체 배포를 진행할까요?',
                          ok: '전체 배포 진행',
                          submitterParameter: 'APPROVER'
                }
            }
        }

        stage('전체 배포') {
            steps {
                script {
                    def hosts = env.FULL_HOSTS.split(' ')
                    for (host in hosts) {
                        // Rolling 방식으로 한 대씩 순차 배포
                        sh """
                            aws elbv2 deregister-targets \
                                --target-group-arn ${TG_ARN} \
                                --targets Id=${host}
                            sleep 10

                            scp -i ${PEM_KEY} -o StrictHostKeyChecking=no \
                                target/${JAR_NAME} ${EC2_USER}@${host}:${APP_PATH}/
                            ssh -i ${PEM_KEY} -o StrictHostKeyChecking=no \
                                ${EC2_USER}@${host} \
                                'sudo systemctl restart app'

                            sleep 20
                            curl -f http://${host}/actuator/health || exit 1

                            aws elbv2 register-targets \
                                --target-group-arn ${TG_ARN} \
                                --targets Id=${host}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo '배포 성공!'
        }
        aborted {
            // 승인 단계에서 중단 시 Canary 서버만 롤백
            echo 'Canary 롤백 진행'
            sh """
                scp -i ${PEM_KEY} -o StrictHostKeyChecking=no \
                    backup/${JAR_NAME} ${EC2_USER}@${CANARY_HOST}:${APP_PATH}/
                ssh -i ${PEM_KEY} -o StrictHostKeyChecking=no \
                    ${EC2_USER}@${CANARY_HOST} \
                    'sudo systemctl restart app'
            """
        }
        failure {
            echo '배포 실패 - 담당자 확인 필요'
        }
    }
}
```

---

## 롤백 시나리오

### 라이브 테스트 중 문제 발견 시

1. Jenkins에서 **Abort** 클릭
2. `post { aborted }` 블록이 자동 실행되어 Canary 서버만 이전 버전으로 복구
3. 나머지 서버는 기존 버전이 계속 운영 중이므로 **서비스 영향 없음**

### 전체 배포 중 문제 발견 시

1. Pipeline 실패 처리 후 이미 배포된 서버는 수동으로 이전 JAR 재배포 필요
2. 이를 대비해 **이전 버전 JAR를 항상 `backup/` 경로에 보관**해두는 것을 권장합니다

---

## ALB Weighted Target Group 활용 (심화)

수동 Deregister/Register 방식 대신, **ALB Weighted Target Group**을 사용하면 트래픽 비율로 세밀하게 제어할 수 있습니다.

```
초기:   Canary TG  10%  /  기존 TG  90%
검증 후: Canary TG 100%  /  기존 TG   0%
```

이 방식은 점진적 트래픽 이동이 가능해 더 정교한 Canary 배포를 구현할 수 있습니다. 다만 Target Group 구성이 추가로 필요합니다.

---

## 배포 전 체크리스트

- [ ] ALB Target Group ARN 확인 및 환경변수 설정
- [ ] 헬스체크 엔드포인트 구현 (`/actuator/health` 또는 `/health`)
- [ ] 이전 버전 JAR 백업 경로 (`backup/`) 구성
- [ ] Jenkins 서버에서 AWS CLI 및 IAM 권한 확인 (`elbv2:*` 등)
- [ ] PEM 키 경로 및 SSH 접속 확인
- [ ] 승인 담당자 및 타임아웃 정책 결정 (기본 1시간)

---

## 마무리

Canary 배포는 구성이 단순하면서도 **운영 리스크를 크게 줄여주는** 실용적인 배포 전략입니다.

Blue/Green처럼 인프라를 2배로 유지할 필요 없이, 기존 서버 구성 그대로 적용할 수 있습니다. Jenkins의 `input` 스텝과 조합하면 수동 게이트를 쉽게 추가할 수 있어, 팀의 배포 자신감을 높이는 데 효과적입니다.

실제 운영에 적용할 때는 헬스체크 엔드포인트 구현과 이전 버전 백업 관리를 반드시 챙기세요.
