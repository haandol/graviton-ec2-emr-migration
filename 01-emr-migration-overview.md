# EMR Graviton4 마이그레이션 플레이북 — 메인 가이드

> **Zeppelin 기반 EMR 환경을 Intel(x86-64)에서 Graviton4(ARM64)로 전환하기 위한 종합 개요**

---

## 관련 문서

| 문서                                                          | 설명                                                           |
| ------------------------------------------------------------- | -------------------------------------------------------------- |
| [Python 2 → 3 전환 가이드](02-python2-to-python3-guide.md)    | 코드 변환, 패키지 호환성, Zeppelin 노트북 변환 상세            |
| [Intel → Graviton 전환 가이드](03-intel-to-graviton-guide.md) | EMR 업그레이드, Zeppelin 설정, ARM64 인스턴스 전환, IaC 자동화 |
| [실행 체크리스트](04-migration-checklist.md)                  | Phase별 체크리스트, Go/No-Go 기준, 롤백 절차                   |
| [Kiro 활용 가이드](kiro-migration-guide.md)                   | Kiro를 활용한 Phase별 자동화 및 프롬프트 레시피                |

---

## 1. 마이그레이션 배경 및 목표

현재 운영 중인 Intel 기반 EMR + Zeppelin + Python 2 환경은 다음과 같은 한계에 직면해 있다:

- **Python 2 EOL**: Python 2는 2020년 1월 공식 지원이 종료되어 보안 패치 및 라이브러리 업데이트가 중단됨
- **EMR 5.x 지원 축소**: 최신 Spark 기능 및 보안 패치가 EMR 6.x/7.x에 집중됨
- **비용 및 성능 최적화 필요**: Graviton 인스턴스로 전환 시 동일 워크로드 대비 비용 절감 가능

## 2. 현재 환경 vs 목표 환경

| 항목                  | 현재 (AS-IS)            | 목표 (TO-BE)                 |
| --------------------- | ----------------------- | ---------------------------- |
| **인스턴스 아키텍처** | Intel x86-64 (m5/r5 등) | Graviton4 ARM64 (m8g/r8g 등) |
| **OS**                | Amazon Linux 2 (x86)    | Amazon Linux 2023 (ARM64)    |
| **Python**            | Python 2.7              | Python 3.9+                  |
| **EMR 버전**          | EMR 5.x                 | EMR 7.x                      |
| **Spark**             | Spark 2.x               | Spark 3.x                    |
| **Zeppelin**          | 0.8~0.9                 | 0.10+                        |
| **Java**              | Java 8                  | Amazon Corretto 11+          |

## 3. 기대 효과

- **성능 향상**: Graviton 인스턴스 사용 시 최대 **15% 성능 향상** (AWS EMR 문서 기준)
- **비용 절감**: 동일 사양 대비 최대 **30% 비용 절감** 효과
- **보안 강화**: Python 3, 최신 EMR/Spark로 전환하여 보안 패치 지속 수령
- **생태계 활용**: 최신 라이브러리 및 프레임워크 활용 가능

## 4. 마이그레이션 4대 축 및 권장 순서

```text
┌─────────────────────────────────────────────────────────────┐
│  축 1: Python 2 → 3    (가장 큰 작업, 코드 변환 필수)      │
│  축 2: EMR 5.x → 7.x   (Spark 2→3 포함)                   │
│  축 3: Zeppelin 0.8 → 0.10+                                │
│  축 4: Intel → Graviton4 (ARM64)                             │
└─────────────────────────────────────────────────────────────┘
```

> **권장 순서**: 현행 보존(AMI/EBS) → 스테이징 환경 구성 → Python 전환(축1) → EMR/Spark 업그레이드(축2) → Zeppelin 업그레이드(축3) → S3 이관 허브 업로드 → Graviton 전환(축4)
> 운영 환경의 상태를 먼저 보존하고, 스테이징에서 소프트웨어 호환성을 확보한 뒤, 새 Graviton 클러스터로 전환하는 것이 안전하다.

## 5. Phase 0: 현행 환경 보존 및 인벤토리 조사

### 5.0 기존 EC2/EMR 상태 완전 백업 (AMI & EBS 스냅샷)

운영 중인 환경의 상태를 변경하기 전에, 현재 상태를 완전히 보존해야 한다. `pip freeze`나 설정 파일 백업만으로는 수동 설치된 패키지, 로컬 데이터, cron job 등을 재현할 수 없다.

#### AMI 생성

```bash
# 기존 EC2 인스턴스에서 AMI 생성 (재부팅 없이, 운영 무중단)
aws ec2 create-image \
  --instance-id i-XXXXXXXXXXXX \
  --name "emr-pre-migration-$(date +%Y%m%d)" \
  --description "마이그레이션 전 원본 상태 백업" \
  --no-reboot \
  --tag-specifications 'ResourceType=image,Tags=[{Key=Purpose,Value=migration-backup},{Key=Phase,Value=pre-migration}]'

# AMI 생성 완료 대기
aws ec2 wait image-available --image-ids ami-XXXXXXXXXXXX

# AMI 상태 확인
aws ec2 describe-images --image-ids ami-XXXXXXXXXXXX \
  --query 'Images[0].{State:State,Name:Name,Created:CreationDate}'
```

> **EMR 클러스터의 경우**: Master 노드와 Core 노드의 인스턴스 ID를 각각 확인하여 AMI를 생성한다.

```bash
# EMR 클러스터 노드별 인스턴스 ID 확인
aws emr list-instances --cluster-id j-XXXXXXXXXXXX \
  --query 'Instances[*].{Id:Ec2InstanceId,Group:InstanceGroupId,Type:InstanceGroupId,State:Status.State}'
```

#### EBS 스냅샷 생성 (데이터 볼륨이 있는 경우)

```bash
# 인스턴스에 부착된 EBS 볼륨 확인
aws ec2 describe-volumes \
  --filters "Name=attachment.instance-id,Values=i-XXXXXXXXXXXX" \
  --query 'Volumes[*].{VolumeId:VolumeId,Size:Size,Type:VolumeType,Device:Attachments[0].Device}'

# 데이터 볼륨 스냅샷 생성
aws ec2 create-snapshot \
  --volume-id vol-XXXXXXXXXXXX \
  --description "마이그레이션 전 데이터 볼륨 백업" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Purpose,Value=migration-backup}]'
```

### 5.1 EMR / Spark / Zeppelin / Java / Python 버전 확인

```bash
# EMR 클러스터 정보 확인
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXX \
  --query 'Cluster.{Release:ReleaseLabel,State:Status.State,Apps:Applications[*].Name}'

# Spark 버전 확인
spark-submit --version

# Zeppelin 버전 확인
cat /usr/lib/zeppelin/bin/common.sh | grep ZEPPELIN_VERSION

# Java 버전 확인
java -version

# Python 버전 확인
python --version
python3 --version 2>/dev/null
```

### 5.2 Python 패키지 목록 추출

```bash
pip freeze > requirements-current-py2.txt
pip list --format=columns > packages-list.txt
pip show <패키지명>
```

### 5.3 Zeppelin 노트북 인벤토리

```bash
# 노트북 전체 목록 파악
find /usr/lib/zeppelin/notebook/ -name "note.json" | wc -l

# Python 셀이 포함된 노트북 식별
grep -rl '"%python\|"%pyspark' /usr/lib/zeppelin/notebook/ | sort -u

# Python 셀 총 개수 파악
grep -r '"%python\|"%pyspark' /usr/lib/zeppelin/notebook/ | wc -l

# 노트북 전체 백업
aws s3 sync /usr/lib/zeppelin/notebook/ s3://<버킷명>/backup/zeppelin-notebooks/
```

### 5.4 인프라 구성 기록

```bash
# 현재 인스턴스 타입 확인
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXX \
  --query 'Cluster.Ec2InstanceAttributes'

# 인스턴스 그룹 상세 정보
aws emr list-instance-groups --cluster-id j-XXXXXXXXXXXX

# VPC / Subnet / 보안그룹 기록
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXX \
  --query 'Cluster.Ec2InstanceAttributes.{VPC:Ec2SubnetId,SG:EmrManagedMasterSecurityGroup,IAM:IamInstanceProfile}'
```

### 5.5 인벤토리 조사 결과 템플릿

| 항목                 | 현재 값             | 비고 |
| -------------------- | ------------------- | ---- |
| EMR Release          | emr-5.x.x           |      |
| Spark Version        | 2.x.x               |      |
| Zeppelin Version     | 0.8.x / 0.9.x       |      |
| Java Version         | 1.8.x               |      |
| Python Version       | 2.7.x               |      |
| Master 인스턴스 타입 | m5.xlarge           |      |
| Core 인스턴스 타입   | m5.xlarge           |      |
| Core 인스턴스 수     | N                   |      |
| VPC ID               | vpc-xxxxxxxx        |      |
| Subnet ID            | subnet-xxxxxxxx     |      |
| 보안그룹 (Master)    | sg-xxxxxxxx         |      |
| 보안그룹 (Core)      | sg-xxxxxxxx         |      |
| IAM 역할 (EC2)       | EMR_EC2_DefaultRole |      |
| IAM 역할 (Service)   | EMR_DefaultRole     |      |
| Zeppelin 노트북 수   | N개                 |      |
| Python 셀 수         | N개                 |      |
| pip 패키지 수        | N개                 |      |

## 6. Phase 0.5: 스테이징 환경 구성 (운영 무중단)

Phase 1~3의 코드 변환 및 테스트를 운영 중인 EC2/EMR에서 직접 수행하면 서비스에 영향을 준다. **반드시 스테이징 환경을 별도로 구성**하여 작업한다.

### 6.1 스테이징 인스턴스 생성

```bash
# 방법 A: 기존 AMI에서 스테이징 인스턴스 생성 (x86 동일 환경)
aws ec2 run-instances \
  --image-id ami-XXXXXXXXXXXX \            # Phase 0에서 생성한 AMI
  --instance-type m5.xlarge \              # 기존과 동일한 Intel 타입
  --subnet-id subnet-XXXXXXXXXXXX \
  --security-group-ids sg-XXXXXXXXXXXX \
  --key-name my-key \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=emr-staging-py3},{Key=Purpose,Value=migration-staging}]'
```

```bash
# 방법 B: EMR 클러스터인 경우, 동일 설정으로 스테이징 클러스터 생성
aws emr create-cluster \
  --name "staging-migration-test" \
  --release-label emr-5.x.x \             # 현재와 동일한 버전으로 먼저 시작
  --applications Name=Zeppelin Name=Spark \
  --instance-groups '[
    {"InstanceGroupType":"MASTER","InstanceType":"m5.xlarge","InstanceCount":1},
    {"InstanceGroupType":"CORE","InstanceType":"m5.xlarge","InstanceCount":2}
  ]' \
  --ec2-attributes '{
    "SubnetId":"subnet-XXXXXXXXXXXX",
    "KeyName":"my-key",
    "EmrManagedMasterSecurityGroup":"sg-XXXXXXXXXXXX",
    "EmrManagedSlaveSecurityGroup":"sg-XXXXXXXXXXXX"
  }' \
  --service-role EMR_DefaultRole \
  --auto-scaling-role EMR_AutoScaling_DefaultRole
```

### 6.2 스테이징 환경에서 수행할 작업

| 순서 | 작업                                     | 비고                                  |
| ---- | ---------------------------------------- | ------------------------------------- |
| 1    | S3에서 노트북/설정 복원                  | Phase 0 백업 데이터 사용              |
| 2    | Python 2 → 3 코드 변환 및 테스트 (Phase 1) | 운영 환경에 영향 없음              |
| 3    | Spark 2.x → 3.x API 변경 테스트 (Phase 2)  | EMR 7.x 스테이징 클러스터 별도 생성 가능 |
| 4    | Zeppelin 설정 변경 테스트 (Phase 3)      | Python 3 인터프리터 경로 등           |
| 5    | Python 2 출력 vs Python 3 출력 결과값 비교  | 나눗셈, 인코딩, dict 순서 등       |

> **핵심 원칙**: 모든 변환 작업은 스테이징에서 완료 → 변환 결과물을 S3에 업로드 → 최종 Graviton 클러스터에서 복원

### 6.3 Blue-Green 전환 전략

EMR 클러스터는 **in-place 업그레이드가 불가능**하므로, 반드시 새 클러스터를 생성하여 전환하는 Blue-Green 방식을 사용한다:

```text
┌──────────────────────────────────────────────────────────────┐
│  [Blue] 기존 EMR 5.x (Intel)     ← 운영 트래픽 처리 중     │
│  ─ Python 2, Spark 2.x                                      │
│                                                              │
│  Phase 0: 현재 상태 AMI 생성 + 인벤토리                      │
│      ↓                                                       │
│  [Staging] 기존 AMI → 스테이징 인스턴스                       │
│  ─ Phase 1~3: 코드 변환 + 테스트 (기존 Intel 환경에서)       │
│      ↓ (코드 변환 완료, S3에 변환된 노트북/설정 업로드)       │
│                                                              │
│  [Green] 새 EMR 7.x (Graviton4) 클러스터 생성                │
│  ─ Phase 4~5: ARM64 인스턴스, Python 3, Spark 3.x            │
│  ─ 변환된 노트북/설정을 S3에서 로드                           │
│  ─ Phase 6: 통합 테스트                                       │
│      ↓ (Go/No-Go 판정)                                       │
│                                                              │
│  Phase 7: DNS/LB를 Green으로 전환                             │
│  ─ Blue는 7일간 대기 후 종료                                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. 참고 자료 및 출처

### PDF 문서 출처

1. **Zeppelin 기반 EMR 마이그레이션 자동화** (PDF1)
   - IaC 자동화 (Terraform, CloudFormation, AWS CDK)
   - 부트스트랩 스크립트 및 노트북 S3 동기화
   - EMR 스텝 테스트 자동화

2. **Zeppelin 기반 EMR 인스턴스 (Intel→Graviton4) 이전 플레이북** (PDF2)
   - 사전 준비 사항 (ARM64 호환성, Java/Python 환경)
   - 마이그레이션 단계별 가이드
   - 체크리스트 및 롤백 계획

### AWS 공식 문서

- [AWS EC2 Graviton Getting Started](https://aws.amazon.com/ec2/graviton/getting-started/)
- [Amazon EMR - Supported Instance Types](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-supported-instance-types.html)
- [Apache Zeppelin on Amazon EMR](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-zeppelin.html)
- [AWS Graviton Getting Started (GitHub)](https://github.com/aws/aws-graviton-getting-started)
- [AWS EMR Best Practices - Spark](https://aws.github.io/aws-emr-best-practices/docs/bestpractices/Applications/Spark/best_practices/)

### Apache Zeppelin

- [Apache Zeppelin 공식 사이트](https://zeppelin.apache.org/)
- [Zeppelin 0.12.0 설치 가이드](https://zeppelin.apache.org/docs/0.12.0/quickstart/install.html)

### Python 2 → 3 전환 참고

- [Python 3 공식 포팅 가이드](https://docs.python.org/3/howto/pyporting.html)
- [2to3 공식 문서](https://docs.python.org/3/library/2to3.html)
- [python-future (futurize)](https://python-future.org/)
- [python-modernize](https://python-modernize.readthedocs.io/)
- [caniusepython3](https://pypi.org/project/caniusepython3/)
- [Python 2→3 주요 변경 사항 요약](https://docs.python.org/3/whatsnew/3.0.html)
