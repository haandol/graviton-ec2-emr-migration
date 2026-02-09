# Intel → Graviton 전환 가이드

> **EMR Graviton4 마이그레이션 — Phase 2~5 상세 (EMR 업그레이드, Zeppelin 설정, ARM64 전환, S3 이관, IaC 자동화, 데이터 복원, 네트워크 검증)**

---

## 목차

1. [Phase 2: EMR 버전 업그레이드](#1-phase-2-emr-버전-업그레이드)
2. [Phase 3: Zeppelin 업그레이드](#2-phase-3-zeppelin-업그레이드)
3. [Phase 4: Graviton4(ARM64) 인스턴스 전환](#3-phase-4-graviton4arm64-인스턴스-전환)
4. [Phase 3.5: S3 이관 허브 구성 및 데이터 이관](#4-phase-35-s3-이관-허브-구성-및-데이터-이관)
5. [Phase 5: IaC 자동화 및 클러스터 프로비저닝](#5-phase-5-iac-자동화-및-클러스터-프로비저닝)
6. [새 Graviton 클러스터 데이터/설정 복원](#6-새-graviton-클러스터-데이터설정-복원)
7. [네트워크 및 데이터 소스 연결 검증](#7-네트워크-및-데이터-소스-연결-검증)
8. [통합 테스트 및 성능 검증](#8-통합-테스트-및-성능-검증)
9. [프로덕션 전환 (Cutover)](#9-프로덕션-전환-cutover)
10. [롤백 절차](#10-롤백-절차)
11. [부록: 단독 EC2에서 Zeppelin을 운영하는 경우](#11-부록-단독-ec2에서-zeppelin을-운영하는-경우)

---

## 1. Phase 2: EMR 버전 업그레이드

### 1.1 목표 EMR 릴리스 선정

| EMR 릴리스 | Spark  | Zeppelin | Graviton 지원   | 권장     |
| ---------- | ------ | -------- | --------------- | -------- |
| EMR 6.10.0 | 3.3.1  | 0.10.1   | ✅ (5.30+/6.1+) | 안정적   |
| EMR 7.0.0+ | 3.5.0+ | 0.10.1   | ✅              | **권장** |

> **권장**: EMR 7.x를 선택한다. Graviton을 공식 지원하며 최신 Spark 3.5+ 및 Zeppelin 0.10.1이 번들로 포함된다.

### 1.2 Spark 2.x → 3.x 주요 변경점 및 코드 수정 가이드

#### API 변경 요약

| 항목                         | Spark 2.x              | Spark 3.x                  | 수정 방법            |
| ---------------------------- | ---------------------- | -------------------------- | -------------------- |
| `SparkSession.builder`       | 동일                   | 동일                       | 변경 없음            |
| `DataFrame.toDF()`           | 동일                   | 동일                       | 변경 없음            |
| `pandas_udf` (PandasUDFType) | `PandasUDFType.SCALAR` | `PandasUDFType` deprecated | 데코레이터 방식 사용 |
| CSV 파싱                     | 관대한 파싱            | 엄격한 파싱                | `mode` 옵션 확인     |
| 암시적 타입 캐스팅           | 허용                   | 제한됨                     | 명시적 `cast()` 사용 |
| `spark.sql.legacy.*`         | —                      | 호환 설정 제공             | 필요시 설정 추가     |

#### 주요 코드 수정 예시

```python
# Spark 2.x: createDataFrame에서 암시적 타입 변환
df = spark.createDataFrame([(1, "a"), (2, None)])

# Spark 3.x: 스키마 명시 권장
from pyspark.sql.types import StructType, StructField, IntegerType, StringType
schema = StructType([
    StructField("id", IntegerType(), False),
    StructField("value", StringType(), True)
])
df = spark.createDataFrame([(1, "a"), (2, None)], schema=schema)
```

```python
# Spark 2.x: pandas_udf 데코레이터
from pyspark.sql.functions import pandas_udf, PandasUDFType

@pandas_udf("double", PandasUDFType.SCALAR)
def multiply(a, b):
    return a * b

# Spark 3.x: 간소화된 데코레이터
from pyspark.sql.functions import pandas_udf

@pandas_udf("double")
def multiply(a: pd.Series, b: pd.Series) -> pd.Series:
    return a * b
```

#### 호환성 설정 (임시 사용)

마이그레이션 초기에 Spark 2.x 동작을 유지해야 할 경우 아래 설정을 사용할 수 있다:

```properties
# spark-defaults.conf (임시 호환 설정)
spark.sql.legacy.timeParserPolicy=LEGACY
# spark.sql.legacy.createHiveTableByDefault=true  # Spark 3.x에서는 이미 기본값이 Hive이므로 불필요. Spark 4.0+ 전환 시 필요.
spark.sql.legacy.allowNonEmptyLocationInCTAS=true
spark.sql.adaptive.enabled=true  # Spark 3.2+부터 기본값 true이므로 중복 설정이나 명시적 선언용
```

> **주의**: `spark.sql.legacy.*` 설정은 임시 조치이며, 최종적으로 Spark 3.x 네이티브 방식으로 코드를 수정해야 한다.

### 1.3 EMR Configurations 매핑

```json
[
  {
    "Classification": "spark-defaults",
    "Properties": {
      "spark.driver.memory": "4g",
      "spark.executor.memory": "8g",
      "spark.executor.cores": "4",
      "spark.dynamicAllocation.enabled": "true",
      "spark.sql.adaptive.enabled": "true"
    }
  },
  {
    "Classification": "spark-env",
    "Properties": {
      "PYSPARK_PYTHON": "/usr/bin/python3",
      "PYSPARK_DRIVER_PYTHON": "/usr/bin/python3"
    }
  },
  {
    "Classification": "zeppelin-env",
    "Properties": {
      "ZEPPELIN_NOTEBOOK_DIR": "s3://<버킷명>/zeppelin/notebook"
    }
  }
]
```

---

## 2. Phase 3: Zeppelin 업그레이드

### 2.1 설정 파일 마이그레이션

기존 Zeppelin에서 새 버전으로 옮겨야 할 주요 설정 파일:

| 파일                     | 용도                                      |
| ------------------------ | ----------------------------------------- |
| `conf/zeppelin-site.xml` | Zeppelin 서버 설정 (포트, 바인드 주소 등) |
| `conf/shiro.ini`         | 인증/인가 설정                            |
| `conf/zeppelin-env.sh`   | 환경 변수 (JAVA_HOME, SPARK_HOME 등)      |
| `conf/interpreter.json`  | 인터프리터 설정                           |
| `notebook/`              | 노트북 JSON 파일 디렉터리                 |

```bash
# 기존 설정 백업
tar czf zeppelin-conf-backup.tar.gz \
  /usr/lib/zeppelin/conf/zeppelin-site.xml \
  /usr/lib/zeppelin/conf/shiro.ini \
  /usr/lib/zeppelin/conf/zeppelin-env.sh \
  /usr/lib/zeppelin/conf/interpreter.json

# 새 Zeppelin에 설정 복원
tar xzf zeppelin-conf-backup.tar.gz -C /opt/zeppelin-new/
```

### 2.2 원격 접속 설정

```xml
<!-- conf/zeppelin-site.xml -->
<property>
  <name>zeppelin.server.addr</name>
  <value>0.0.0.0</value>
  <description>Server binding address (0.0.0.0 = 모든 인터페이스)</description>
</property>

<property>
  <name>zeppelin.server.port</name>
  <value>8080</value>
</property>
```

> **보안 주의**: `0.0.0.0`으로 바인딩할 경우 보안그룹에서 접근을 제한해야 한다.

### 2.3 Python 인터프리터 경로를 Python 3으로 변경

#### 환경 변수 설정

`conf/zeppelin-env.sh`에 다음을 추가한다:

```bash
# conf/zeppelin-env.sh
export PYSPARK_PYTHON=/usr/bin/python3
export PYSPARK_DRIVER_PYTHON=/usr/bin/python3
export SPARK_HOME=/usr/lib/spark
export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
```

#### 인터프리터 설정 변경

Zeppelin UI > Interpreter 설정에서 또는 `conf/interpreter.json`을 직접 편집:

```json
{
  "interpreterSettings": {
    "python": {
      "properties": {
        "zeppelin.python": {
          "value": "/usr/bin/python3"
        }
      }
    },
    "spark": {
      "properties": {
        "PYSPARK_PYTHON": {
          "value": "/usr/bin/python3"
        },
        "PYSPARK_DRIVER_PYTHON": {
          "value": "/usr/bin/python3"
        },
        "spark.pyspark.python": {
          "value": "/usr/bin/python3"
        },
        "spark.pyspark.driver.python": {
          "value": "/usr/bin/python3"
        }
      }
    }
  }
}
```

### 2.4 노트북 JSON 파일 복사 및 로드 확인

```bash
# S3에서 노트북 동기화
aws s3 sync s3://<버킷명>/backup/zeppelin-notebooks/ /usr/lib/zeppelin/notebook/

# 또는 기존 마스터에서 직접 복사
scp -i key.pem -r hadoop@old-master:/usr/lib/zeppelin/notebook/* \
  /usr/lib/zeppelin/notebook/

# Zeppelin 재시작하여 노트북 로드 확인
sudo systemctl restart zeppelin
# 또는
sudo /usr/lib/zeppelin/bin/zeppelin-daemon.sh restart
```

> 노트북 로드 후 Zeppelin UI에서 노트북 목록이 정상적으로 표시되는지 확인한다.

---

## 3. Phase 4: Graviton4(ARM64) 인스턴스 전환

### 3.1 ARM64 AMI 선택

```bash
# Amazon Linux 2023 ARM64 최신 AMI 조회
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-arm64" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].{ID:ImageId,Name:Name}'

# Amazon Linux 2 ARM64 AMI 조회 (대안)
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-arm64-gp2" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].{ID:ImageId,Name:Name}'
```

### 3.2 인스턴스 타입 매핑표

| 현재 (Intel x86-64) | 목표 (Graviton4 ARM64) | vCPU | 메모리 | 비고          |
| ------------------- | ---------------------- | ---- | ------ | ------------- |
| m5.xlarge           | **m8g.xlarge**         | 4    | 16 GiB | 범용          |
| m5.2xlarge          | **m8g.2xlarge**        | 8    | 32 GiB | 범용          |
| m5.4xlarge          | **m8g.4xlarge**        | 16   | 64 GiB | 범용          |
| r5.xlarge           | **r8g.xlarge**         | 4    | 32 GiB | 메모리 최적화 |
| r5.2xlarge          | **r8g.2xlarge**        | 8    | 64 GiB | 메모리 최적화 |
| c5.xlarge           | **c8g.xlarge**         | 4    | 8 GiB  | 컴퓨팅 최적화 |
| c5.2xlarge          | **c8g.2xlarge**        | 8    | 16 GiB | 컴퓨팅 최적화 |

> `m8g`/`r8g`/`c8g` 시리즈는 Graviton4 기반이다. 참고: `m7g`/`r7g`/`c7g`는 Graviton3, `m9g`/`r9g`/`c9g`(프리뷰)는 Graviton5 기반이다.

### 3.3 Java: Amazon Corretto 11 ARM64 설치

```bash
# Amazon Linux 2023 ARM64
sudo dnf install -y java-11-amazon-corretto-headless

# Amazon Linux 2 ARM64
sudo amazon-linux-extras enable corretto8
sudo yum install -y java-11-amazon-corretto-headless

# 설치 확인
java -version
# openjdk version "11.0.x" ...
# OpenJDK Runtime Environment Corretto-11.x.x...
```

### 3.4 pip 및 ARM64 패키지 확인

```bash
# pip 버전 확인 (19.3+ 필수 — manylinux2014/aarch64 휠 지원)
pip3 --version

# pip 업그레이드 (19.3 미만인 경우)
sudo python3 -m pip install --upgrade pip

# ARM64 휠 제공 여부 확인
pip3 install --dry-run numpy 2>&1 | grep -i "aarch64\|arm64"

# requirements.txt 일괄 설치
pip3 install -r requirements-py3.txt

# 설치 실패 패키지 소스 빌드
sudo yum install -y gcc gcc-c++ python3-devel
pip3 install --no-binary :all: <패키지명>
```

### 3.5 VPC/Subnet/보안그룹/IAM 동일성 유지

신규 Graviton 인스턴스(또는 EMR 클러스터)는 기존 Intel 환경과 **동일한 네트워크 및 보안 설정**을 사용해야 한다:

- **VPC / Subnet**: 기존과 동일한 VPC 및 Subnet에 배치
- **보안그룹**: 기존 Master/Core 보안그룹을 그대로 할당
- **IAM 역할**: `EMR_EC2_DefaultRole`, `EMR_DefaultRole` 등 기존 역할 재사용
- **키페어**: 기존과 동일한 EC2 키페어 지정
- **볼륨**: 기존과 동일한 EBS 볼륨 크기 및 타입

---

## 4. Phase 3.5: S3 이관 허브 구성 및 데이터 이관

스테이징 환경에서 코드 변환(Phase 1~3)이 완료되면, 모든 결과물을 S3에 체계적으로 업로드한다. 이 S3 이관 허브가 기존 환경과 새 Graviton 환경을 잇는 중간 저장소 역할을 한다.

### 4.1 S3 이관 허브 구조

```text
s3://my-emr-migration-hub/
├── notebooks/
│   ├── original/           # Python 2 원본 노트북
│   └── converted-py3/     # Python 3 변환 완료 노트북
├── config/
│   ├── zeppelin-conf/     # zeppelin-site.xml, shiro.ini, interpreter.json 등
│   ├── spark/             # spark-defaults.conf, spark-env.sh
│   └── crontab-backup.txt
├── packages/
│   ├── requirements-current-py2.txt
│   └── requirements-py3.txt
├── custom-scripts/        # 사용자 정의 스크립트
├── hdfs-data/             # HDFS 데이터 (필요시)
├── metastore/             # Hive Metastore 덤프 (필요시)
└── bootstrap/
    └── bootstrap-graviton.sh
```

### 4.2 기존 환경에서 S3 업로드

```bash
S3_BUCKET="s3://my-emr-migration-hub"

# 1) Zeppelin 노트북 (변환 전 원본 + 변환 후)
aws s3 sync /usr/lib/zeppelin/notebook/ \
  ${S3_BUCKET}/notebooks/original/
aws s3 sync /path/to/converted-notebooks/ \
  ${S3_BUCKET}/notebooks/converted-py3/

# 2) Zeppelin 설정 파일
aws s3 sync /usr/lib/zeppelin/conf/ \
  ${S3_BUCKET}/config/zeppelin-conf/

# 3) Spark 설정 (커스텀 설정이 있는 경우)
aws s3 cp /etc/spark/conf/spark-defaults.conf \
  ${S3_BUCKET}/config/spark/
aws s3 cp /etc/spark/conf/spark-env.sh \
  ${S3_BUCKET}/config/spark/

# 4) Python 패키지 목록
aws s3 cp requirements-current-py2.txt ${S3_BUCKET}/packages/
aws s3 cp requirements-py3.txt ${S3_BUCKET}/packages/

# 5) 커스텀 스크립트, cron jobs, 기타 설정
aws s3 sync /home/hadoop/scripts/ ${S3_BUCKET}/custom-scripts/
crontab -l > /tmp/crontab-backup.txt
aws s3 cp /tmp/crontab-backup.txt ${S3_BUCKET}/config/crontab-backup.txt

# 6) HDFS 데이터 (필요시)
hadoop distcp hdfs:///user/hadoop/data ${S3_BUCKET}/hdfs-data/

# 7) Hive Metastore 덤프 (Hive 사용시)
# EMR은 기본적으로 Glue Data Catalog를 사용하므로 별도 이관 불필요할 수 있음
# 로컬 Hive Metastore인 경우:
mysqldump -u root hive_metastore > /tmp/hive-metastore-dump.sql
aws s3 cp /tmp/hive-metastore-dump.sql ${S3_BUCKET}/metastore/
```

### 4.3 EBS 볼륨 데이터 이관 (데이터 볼륨이 있는 경우)

EC2에 부착된 EBS 데이터 볼륨이 있는 경우 스냅샷 또는 S3를 통해 이관한다:

```bash
# 방법 1: EBS 스냅샷 → 새 볼륨 생성 후 부착
aws ec2 create-snapshot \
  --volume-id vol-XXXXXXXXXXXX \
  --description "마이그레이션 전 데이터 볼륨 백업"

# 스냅샷에서 새 EBS 볼륨 생성 (새 인스턴스의 AZ에 맞춰야 함)
aws ec2 create-volume \
  --snapshot-id snap-XXXXXXXXXXXX \
  --availability-zone ap-northeast-2a \
  --volume-type gp3

# 새 Graviton 인스턴스에 볼륨 부착
aws ec2 attach-volume \
  --volume-id vol-NEWXXXXXXXXXX \
  --instance-id i-NEWGRAVITONIXX \
  --device /dev/xvdf

# 방법 2: S3를 통해 이관 (대용량 데이터)
# 기존 EC2에서:
aws s3 sync /data/ ${S3_BUCKET}/ebs-data/
# 새 EC2에서:
aws s3 sync ${S3_BUCKET}/ebs-data/ /data/
```

---

## 5. Phase 5: IaC 자동화 및 클러스터 프로비저닝

### 4.1 Terraform `aws_emr_cluster` 예시

```hcl
resource "aws_emr_cluster" "zeppelin_migration" {
  name          = "my-emr-graviton"
  release_label = "emr-7.0.0"
  applications  = ["Zeppelin", "Spark"]

  service_role  = aws_iam_role.emr_service_role.arn

  ec2_attributes {
    instance_profile = aws_iam_instance_profile.emr_profile.arn
    subnet_id        = var.subnet_id
    key_name         = var.key_name

    emr_managed_master_security_group = var.master_sg_id
    emr_managed_slave_security_group  = var.core_sg_id
  }

  master_instance_group {
    instance_type  = "m8g.xlarge"
    instance_count = 1
  }

  core_instance_group {
    instance_type  = "m8g.xlarge"
    instance_count = 2
  }

  configurations_json = jsonencode([
    {
      Classification = "spark-env"
      Configurations = [
        {
          Classification = "export"
          Properties = {
            PYSPARK_PYTHON        = "/usr/bin/python3"
            PYSPARK_DRIVER_PYTHON = "/usr/bin/python3"
          }
        }
      ]
    },
    {
      Classification = "zeppelin-site"
      Properties = {
        "zeppelin.server.port" = "8890"
        "zeppelin.server.addr" = "0.0.0.0"
      }
    },
    {
      Classification = "spark-defaults"
      Properties = {
        "spark.sql.adaptive.enabled" = "true"
        "spark.dynamicAllocation.enabled" = "true"
      }
    }
  ])

  tags = {
    Environment = "production"
    Migration   = "graviton4"
  }
}
```

### 4.2 CloudFormation YAML 예시

```yaml
Resources:
  ZeppelinCluster:
    Type: AWS::EMR::Cluster
    Properties:
      Name: ZeppelinCluster
      ReleaseLabel: emr-7.0.0
      Instances:
        MasterInstanceGroup:
          InstanceCount: 1
          InstanceType: m8g.xlarge
        CoreInstanceGroup:
          InstanceCount: 2
          InstanceType: m8g.xlarge
        Ec2SubnetId: !Ref SubnetId
        Ec2KeyName: !Ref KeyPairName
        EmrManagedMasterSecurityGroup: !Ref MasterSG
        EmrManagedSlaveSecurityGroup: !Ref CoreSG
      Applications:
        - Name: Zeppelin
        - Name: Spark
      Configurations:
        - Classification: spark-env
          Configurations:
            - Classification: export
              ConfigurationProperties:
                PYSPARK_PYTHON: /usr/bin/python3
                PYSPARK_DRIVER_PYTHON: /usr/bin/python3
        - Classification: zeppelin-site
          ConfigurationProperties:
            zeppelin.server.port: "8890"
            zeppelin.server.addr: "0.0.0.0"
      JobFlowRole: !Ref EMRInstanceProfile
      ServiceRole: !Ref EMRServiceRole
      Tags:
        - Key: Environment
          Value: production
        - Key: Migration
          Value: graviton4
```

### 4.3 AWS CDK `CfnCluster` 예시

```typescript
import * as emr from "aws-cdk-lib/aws-emr";

const cluster = new emr.CfnCluster(this, "ZeppelinCluster", {
  name: "ZeppelinCluster",
  releaseLabel: "emr-7.0.0",
  instances: {
    masterInstanceGroup: {
      instanceCount: 1,
      instanceType: "m8g.xlarge",
    },
    coreInstanceGroup: {
      instanceCount: 2,
      instanceType: "m8g.xlarge",
    },
    ec2SubnetId: vpc.selectSubnets({
      subnetType: SubnetType.PRIVATE_WITH_EGRESS,
    }).subnetIds[0],
  },
  applications: [{ name: "Zeppelin" }, { name: "Spark" }],
  configurations: [
    {
      classification: "zeppelin-site",
      configurationProperties: {
        "zeppelin.server.port": "8890",
        "zeppelin.server.addr": "0.0.0.0",
      },
    },
    {
      classification: "spark-env",
      configurations: [
        {
          classification: "export",
          configurationProperties: {
            PYSPARK_PYTHON: "/usr/bin/python3",
            PYSPARK_DRIVER_PYTHON: "/usr/bin/python3",
          },
        },
      ],
    },
  ],
  jobFlowRole: emrInstanceProfile.attrArn,
  serviceRole: emrServiceRole.roleArn,
});
```

### 4.4 부트스트랩 스크립트: Python 3 환경 셋업 및 Zeppelin 설치 자동화

```bash
#!/bin/bash
# bootstrap-graviton.sh — EMR 부트스트랩 액션 스크립트

set -euo pipefail

echo "=== [1/4] Python 3 환경 셋업 ==="
sudo python3 -m pip install --upgrade pip
sudo python3 -m pip install \
  numpy pandas scipy matplotlib \
  boto3 pyarrow

echo "=== [2/4] Python 3을 기본으로 설정 ==="
sudo alternatives --set python /usr/bin/python3 2>/dev/null || true

echo "=== [3/4] Zeppelin Python 인터프리터 설정 ==="
# EMR 관리형 Zeppelin의 경우 자동 설치됨
# EC2 직접 설치 시 아래 스크립트 사용
if [ ! -d "/usr/lib/zeppelin" ]; then
  echo "Zeppelin 수동 설치..."
  sudo yum install -y java-11-amazon-corretto-headless
  wget -q https://dlcdn.apache.org/zeppelin/zeppelin-0.12.0/zeppelin-0.12.0-bin-all.tgz -P /tmp
  sudo tar -xzf /tmp/zeppelin-0.12.0-bin-all.tgz -C /opt/
  sudo mv /opt/zeppelin-0.12.0-bin-all /opt/zeppelin
  sudo cp /opt/zeppelin/conf/zeppelin-site.xml.template /opt/zeppelin/conf/zeppelin-site.xml
fi

echo "=== [4/4] 환경 변수 설정 ==="
cat >> ~/.bashrc << 'ENVEOF'
export PYSPARK_PYTHON=/usr/bin/python3
export PYSPARK_DRIVER_PYTHON=/usr/bin/python3
export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
ENVEOF

echo "부트스트랩 완료"
```

### 4.5 노트북 S3 동기화 스크립트

```bash
#!/bin/bash
# sync-notebooks.sh — Zeppelin 노트북 S3 동기화

set -euo pipefail

S3_BUCKET="${1:?Usage: sync-notebooks.sh <s3-bucket-name>}"
ZEPPELIN_NOTEBOOK_DIR="/usr/lib/zeppelin/notebook"
BACKUP_DIR="/tmp/zeppelin-notebook-backup-$(date +%Y%m%d%H%M%S)"

echo "=== [1/3] 기존 노트북 로컬 백업 ==="
cp -r "$ZEPPELIN_NOTEBOOK_DIR" "$BACKUP_DIR"
echo "백업 완료: $BACKUP_DIR"

echo "=== [2/3] S3에서 노트북 동기화 ==="
aws s3 sync "s3://${S3_BUCKET}/zeppelin/notebook/" "$ZEPPELIN_NOTEBOOK_DIR/" \
  --delete

echo "=== [3/3] Zeppelin 재시작 ==="
if command -v systemctl &>/dev/null && systemctl is-active --quiet zeppelin; then
  sudo systemctl restart zeppelin
else
  sudo /usr/lib/zeppelin/bin/zeppelin-daemon.sh restart 2>/dev/null || \
  sudo /opt/zeppelin/bin/zeppelin-daemon.sh restart 2>/dev/null || \
  echo "WARNING: Zeppelin 재시작을 수동으로 수행하세요"
fi

echo "노트북 동기화 완료 ($(find "$ZEPPELIN_NOTEBOOK_DIR" -name 'note.json' | wc -l)개 노트북)"
```

---

## 6. 새 Graviton 클러스터 데이터/설정 복원

IaC로 새 EMR 7.x Graviton 클러스터를 프로비저닝한 후, S3 이관 허브에서 노트북, 설정, 스크립트를 복원한다.

### 6.1 변환된 노트북 복원

```bash
S3_BUCKET="s3://my-emr-migration-hub"

# 변환 완료된 Python 3 노트북 복원
aws s3 sync ${S3_BUCKET}/notebooks/converted-py3/ \
  /usr/lib/zeppelin/notebook/
```

### 6.2 Zeppelin 설정 복원 (머지 방식)

> **주의**: 새 EMR 버전의 기본 설정을 단순 덮어쓰기하면 안 된다. 새 EMR 7.x / Zeppelin 0.10+에서 설정 스키마가 변경되었을 수 있으므로, 기존 커스텀 설정값만 머지해야 한다.

```bash
# 1) 새 EMR 기본 설정 백업
cp /usr/lib/zeppelin/conf/zeppelin-site.xml \
   /usr/lib/zeppelin/conf/zeppelin-site.xml.emr-default
cp /usr/lib/zeppelin/conf/interpreter.json \
   /usr/lib/zeppelin/conf/interpreter.json.emr-default

# 2) 기존 설정을 임시 경로로 다운로드
aws s3 sync ${S3_BUCKET}/config/zeppelin-conf/ /tmp/old-zeppelin-conf/
```

#### interpreter.json 설정 머지 스크립트

```python
#!/usr/bin/env python3
"""기존 interpreter.json의 Python/Spark 설정을 새 EMR 기본 설정에 머지한다."""

import json

# 기존 설정 로드
with open('/tmp/old-zeppelin-conf/interpreter.json') as f:
    old_conf = json.load(f)

# 새 EMR 기본 설정 로드
with open('/usr/lib/zeppelin/conf/interpreter.json') as f:
    new_conf = json.load(f)

# Python/PySpark 인터프리터 경로만 머지
python_path_keys = [
    'zeppelin.python', 'PYSPARK_PYTHON', 'PYSPARK_DRIVER_PYTHON',
    'spark.pyspark.python', 'spark.pyspark.driver.python'
]

for key, settings in new_conf.get('interpreterSettings', {}).items():
    if 'python' in key.lower() or 'spark' in key.lower():
        props = settings.get('properties', {})
        for prop_key in python_path_keys:
            if prop_key in props:
                if isinstance(props[prop_key], dict):
                    props[prop_key]['value'] = '/usr/bin/python3'
                else:
                    props[prop_key] = '/usr/bin/python3'

with open('/usr/lib/zeppelin/conf/interpreter.json', 'w') as f:
    json.dump(new_conf, f, ensure_ascii=False, indent=2)

print("interpreter.json 머지 완료")
```

### 6.3 커스텀 스크립트 및 cron job 복원

```bash
# 커스텀 스크립트 복원
aws s3 sync ${S3_BUCKET}/custom-scripts/ /home/hadoop/scripts/

# cron job 복원 (수동 확인 후 적용)
aws s3 cp ${S3_BUCKET}/config/crontab-backup.txt /tmp/
echo "=== 기존 cron jobs (검토 후 적용) ==="
cat /tmp/crontab-backup.txt
# 확인 후: crontab /tmp/crontab-backup.txt
```

### 6.4 Zeppelin 재시작 및 확인

```bash
sudo systemctl restart zeppelin
sleep 10
curl -s -o /dev/null -w "Zeppelin HTTP Status: %{http_code}\n" http://localhost:8080/
```

---

## 7. 네트워크 및 데이터 소스 연결 검증

새 Graviton 인스턴스에서 기존 데이터 소스(S3, RDS, Redshift, 온프레미스 등)에 접근할 수 있는지 검증한다. VPC/Subnet/보안그룹을 동일하게 설정하더라도, 실제 통신이 되는지 반드시 확인해야 한다.

### 7.1 기본 연결 검증

```bash
# 1) S3 접근 확인
aws s3 ls s3://my-data-bucket/ --max-items 1

# 2) RDS/Aurora 접근 확인 (사용하는 경우)
nc -zv my-rds-endpoint.xxxx.ap-northeast-2.rds.amazonaws.com 3306
# Python으로 실제 연결 테스트
python3 -c "
import pymysql
conn = pymysql.connect(
    host='my-rds-endpoint.xxxx.ap-northeast-2.rds.amazonaws.com',
    port=3306, user='reader', password='xxx')
print('RDS 연결 OK')
conn.close()
"

# 3) Redshift 접근 확인 (사용하는 경우)
nc -zv my-redshift-cluster.xxxx.ap-northeast-2.redshift.amazonaws.com 5439

# 4) 온프레미스 VPN 경유 접근 확인 (해당 시)
ping -c 3 10.x.x.x

# 5) 외부 API 접근 확인 (해당 시)
curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health
```

### 7.2 EMR 클러스터 내부 통신 확인

```bash
# YARN 노드 목록 및 상태 확인
yarn node -list

# Master → Core 노드 SSH 접근 확인
ssh -i key.pem hadoop@<CORE-NODE-IP> "hostname && python3 --version"

# Spark 분산 잡 실행 테스트
spark-submit --master yarn --deploy-mode cluster \
  --conf spark.pyspark.python=/usr/bin/python3 \
  s3://<버킷명>/test-jobs/simple-test.py
```

### 7.3 IAM 역할 권한 확인

```bash
# 현재 인스턴스에 할당된 IAM 역할 확인
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# S3 버킷 접근 권한 확인
aws s3api head-bucket --bucket my-data-bucket 2>&1

# Glue Data Catalog 접근 확인 (EMR에서 Hive/Spark SQL 사용 시)
aws glue get-databases --query 'DatabaseList[*].Name' 2>&1
```

---

## 8. 통합 테스트 및 성능 검증

### 5.1 기능 테스트

#### Zeppelin UI 접속 확인

```bash
# Zeppelin 프로세스 상태 확인
ps aux | grep zeppelin
sudo systemctl status zeppelin

# 포트 리스닝 확인
ss -tlnp | grep 8080

# HTTP 응답 확인
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/
```

#### Spark 인터프리터 동작 확인

Zeppelin 노트북에서 다음 셀을 실행한다:

```scala
// %spark 셀
sc.parallelize(1 to 100).sum()
// 기대 결과: 5050

spark.sql("SELECT 1 + 1 AS result").show()
// 기대 결과: +------+
//            |result|
//            +------+
//            |     2|
//            +------+
```

#### Python 패키지 import 확인

```python
# %pyspark 또는 %python 셀
import sys
print(f"Python version: {sys.version}")
assert sys.version_info.major == 3, "Python 3이 아닙니다!"

import numpy as np
import pandas as pd
print(f"numpy: {np.__version__}")
print(f"pandas: {pd.__version__}")

# PySpark 동작 확인
df = spark.createDataFrame([(1, "a"), (2, "b"), (3, "c")], ["id", "value"])
df.show()
print(f"Row count: {df.count()}")
```

#### 기존 노트북 재실행

```bash
# Zeppelin REST API를 활용한 자동 실행
ZEPPELIN_HOST="http://localhost:8080"

# 노트북 목록 조회
curl -s "${ZEPPELIN_HOST}/api/notebook" | python3 -m json.tool

# 특정 노트북 모든 셀 실행
NOTEBOOK_ID="2XXXXXXXXXXXX"
curl -X POST "${ZEPPELIN_HOST}/api/notebook/job/${NOTEBOOK_ID}"

# 실행 결과 확인
curl -s "${ZEPPELIN_HOST}/api/notebook/${NOTEBOOK_ID}" | \
  python3 -c "
import json, sys
nb = json.load(sys.stdin)
for p in nb['body']['paragraphs']:
    status = p.get('status', 'UNKNOWN')
    title = p.get('title', '(untitled)')
    print(f'  [{status}] {title}')
"
```

### 5.2 EMR 스텝 자동 테스트

```bash
# EMR 클러스터에 테스트 Spark 잡 제출
aws emr add-steps --cluster-id j-XXXXXXXXXXXX --steps '[
  {
    "Name": "SparkTest-Python3",
    "Type": "CUSTOM_JAR",
    "Jar": "command-runner.jar",
    "ActionOnFailure": "CONTINUE",
    "Args": [
      "spark-submit",
      "--deploy-mode", "cluster",
      "--conf", "spark.pyspark.python=/usr/bin/python3",
      "s3://<버킷명>/test-jobs/test-job.py"
    ]
  }
]'

# 스텝 상태 확인
aws emr describe-step --cluster-id j-XXXXXXXXXXXX --step-id s-XXXXXXXXXXXX \
  --query 'Step.{Name:Name,State:Status.State}'

# 스텝 로그 확인
aws emr ssh --cluster-id j-XXXXXXXXXXXX --command \
  "cat /var/log/spark/spark-step*.log | tail -50"
```

### 5.3 성능 벤치마크

동일 워크로드를 Intel과 Graviton4 환경에서 실행하여 비교한다:

| 벤치마크 항목               | Intel (m5.xlarge) | Graviton4 (m8g.xlarge) | 차이 |
| --------------------------- | ----------------- | ---------------------- | ---- |
| Spark SQL 쿼리 (TPC-DS 1GB) | — sec             | — sec                   | — %  |
| PySpark DataFrame 집계      | — sec             | — sec                   | — %  |
| Python 패키지 import 시간   | — sec             | — sec                   | — %  |
| 노트북 전체 실행 시간       | — sec             | — sec                   | — %  |
| 시간당 비용 (On-Demand)     | $— /hr            | $— /hr                  | — %  |
| **비용 대비 성능**          | baseline          | —                       | — %  |

> **기대치**: Graviton 인스턴스에서 10~15% 성능 향상, 동일 사양 대비 최대 30% 비용 절감

---

## 9. 프로덕션 전환 (Cutover)

### 9.1 전환 시간순 절차

#### T-24h: 전환 하루 전

```bash
# 1) 최종 노트북 및 설정 백업
aws s3 sync /usr/lib/zeppelin/notebook/ s3://<버킷명>/final-backup/notebook/
aws s3 cp /usr/lib/zeppelin/conf/zeppelin-site.xml s3://<버킷명>/final-backup/conf/

# 2) 기존 클러스터 상태 스냅샷
aws emr describe-cluster --cluster-id j-OLD-CLUSTER > cluster-snapshot-old.json
```

#### T-2h: 전환 2시간 전

```bash
# 1) 사용자에게 서비스 중단 공지
# 2) 진행 중인 Spark 잡 완료 대기
aws emr list-steps --cluster-id j-OLD-CLUSTER --step-states RUNNING \
  --query 'Steps[*].{Name:Name,State:Status.State}'

# 3) Zeppelin 노트북 최종 동기화
aws s3 sync /usr/lib/zeppelin/notebook/ s3://<버킷명>/final-backup/notebook/ --delete
```

#### T-0: 전환 시점

```bash
# 1) 기존 Zeppelin 서비스 중지
sudo systemctl stop zeppelin

# 2) 신규 Graviton4 EMR 클러스터 활성화 확인
aws emr describe-cluster --cluster-id j-NEW-CLUSTER \
  --query 'Cluster.Status.State'
# 기대값: "WAITING"

# 3) 최종 노트북 동기화 (delta만)
aws s3 sync s3://<버킷명>/final-backup/notebook/ \
  s3://<새 클러스터 노트북 버킷>/notebook/

# 4) DNS 또는 로드밸런서 전환 (해당 시)
# aws route53 change-resource-record-sets ...

# 5) 신규 클러스터 Zeppelin 접속 확인
curl -s -o /dev/null -w "%{http_code}" http://<NEW-MASTER-IP>:8080/
```

#### T+1h: 전환 1시간 후

```bash
# 안정성 모니터링
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElasticMapReduce \
  --metric-name IsIdle \
  --dimensions Name=JobFlowId,Value=j-NEW-CLUSTER \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

#### T+24h ~ T+7d: 안정화 기간

- 일일 모니터링: CloudWatch 대시보드에서 CPU, 메모리, Spark 잡 실패율 확인
- 사용자 피드백 수집
- 기존 Intel 클러스터는 **롤백 대비로 7일간 유지** 후 종료

### 9.2 전환 후 모니터링 (CloudWatch)

주요 모니터링 지표:

| 지표                            | 네임스페이스         | 알람 기준             |
| ------------------------------- | -------------------- | --------------------- |
| `IsIdle`                        | AWS/ElasticMapReduce | 예상치 못한 Idle 상태 |
| `AppsRunning`                   | AWS/ElasticMapReduce | 0으로 떨어질 때       |
| `AppsFailed`                    | AWS/ElasticMapReduce | 1 이상                |
| `YARNMemoryAvailablePercentage` | AWS/ElasticMapReduce | < 10%                 |
| `HDFSUtilization`               | AWS/ElasticMapReduce | > 80%                 |
| `CPUUtilization`                | AWS/EC2              | > 90% (5분 지속)      |
| `MemoryUtilization`             | CWAgent              | > 90%                 |

---

## 10. 롤백 절차

### 10.1 롤백 트리거 조건

| #   | 트리거 조건                             | 심각도       |
| --- | --------------------------------------- | ------------ |
| 1   | Zeppelin UI 접속 불가 (30분 이상)       | **Critical** |
| 2   | Spark 인터프리터 반복 실패              | **Critical** |
| 3   | 핵심 노트북 실행 결과 불일치            | **High**     |
| 4   | Python 패키지 import 오류 (핵심 패키지) | **High**     |
| 5   | 성능 저하 30% 이상                      | **Medium**   |
| 6   | OOM/크래시 반복 발생                    | **Critical** |

### 10.2 롤백 실행 절차

```text
┌─────────────────────────────────────────────────────┐
│  1. 롤백 결정 (팀 리드 승인)                         │
│  2. 신규 클러스터 트래픽 차단                        │
│  3. DNS/LB를 기존 Intel 클러스터로 전환              │
│  4. 기존 Intel 클러스터 Zeppelin 재시작              │
│  5. 기존 클러스터 정상 동작 확인                     │
│  6. 신규 클러스터 데이터 보존 후 중지                │
│  7. 원인 분석 및 재시도 계획 수립                    │
└─────────────────────────────────────────────────────┘
```

```bash
# Step 3: DNS를 기존 Intel 클러스터로 전환 (Route 53 예시)
# aws route53 change-resource-record-sets --hosted-zone-id ZXXXXXX \
#   --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
#     "Name":"zeppelin.internal.example.com","Type":"A","TTL":60,
#     "ResourceRecords":[{"Value":"<OLD-MASTER-IP>"}]}}]}'

# Step 4: 기존 Intel 클러스터 Zeppelin 재시작
ssh -i key.pem hadoop@<OLD-MASTER-IP> \
  "sudo systemctl start zeppelin"

# Step 5: 정상 동작 확인
curl -s -o /dev/null -w "%{http_code}" http://<OLD-MASTER-IP>:8080/

# Step 6: 신규 클러스터 중지 (데이터 보존)
aws emr modify-cluster --cluster-id j-NEW-CLUSTER --no-auto-terminate
```

### 10.3 AMI 기반 완전 복원 (최후의 수단)

DNS/LB 전환만으로 롤백이 불가능한 경우(기존 클러스터가 이미 종료된 경우 등), Phase 0에서 생성한 AMI로 원본 환경을 완전히 복원할 수 있다:

```bash
# Phase 0에서 생성한 AMI로 기존 환경 복원
aws ec2 run-instances \
  --image-id ami-PREMIGRATION-XXXX \   # Phase 0에서 생성한 AMI
  --instance-type m5.xlarge \           # 기존과 동일한 Intel 타입
  --subnet-id subnet-XXXXXXXXXXXX \
  --security-group-ids sg-XXXXXXXXXXXX \
  --key-name my-key
```

### 10.4 데이터 정합성 보존

- 롤백 시 신규 클러스터에서 생성/수정된 노트북이 있을 수 있으므로 **S3 버전 관리**를 활성화해 둔다
- 롤백 전 신규 클러스터의 노트북을 별도 S3 경로에 백업한다:

```bash
# 신규 클러스터 노트북 백업 (롤백 전)
aws s3 sync s3://<새 클러스터 노트북 버킷>/notebook/ \
  s3://<버킷명>/rollback-backup/notebook-graviton-$(date +%Y%m%d)/
```

- 기존 Intel 클러스터의 데이터는 전환 기간 동안 변경하지 않았으므로 그대로 복원 가능

---

## 11. 부록: 단독 EC2에서 Zeppelin을 운영하는 경우

EMR 관리형 클러스터가 아닌, **단독 EC2 위에 Spark + Zeppelin을 직접 설치하여 운영하는 경우**의 마이그레이션 절차이다.

### 11.1 EMR vs 단독 EC2의 차이점

| 항목                       | EMR 관리형 클러스터                       | 단독 EC2                                          |
| -------------------------- | ----------------------------------------- | ------------------------------------------------- |
| 클러스터 생성              | `aws emr create-cluster` CLI              | `aws ec2 run-instances` + 수동 설치               |
| Spark/Zeppelin 설치        | EMR이 자동 설치                           | 직접 다운로드/설치                                |
| 설정 경로                  | `/usr/lib/zeppelin`, `/usr/lib/spark`     | `/opt/zeppelin`, `/opt/spark` 등 (설치에 따라 다름) |
| in-place 업그레이드        | 불가 (새 클러스터 생성)                   | 불가 (새 인스턴스 생성)                           |
| ARM64 전환 시 AMI 재사용성 | EMR AMI는 자동 선택됨                     | **x86 AMI는 ARM64에서 사용 불가** — 새 OS AMI 필요 |

### 11.2 단독 EC2 마이그레이션 절차

```bash
# Step 1: 기존 x86 EC2에서 AMI 생성 (상태 보존)
aws ec2 create-image --instance-id i-XXXXXXXXXXXX \
  --name "zeppelin-ec2-pre-migration-$(date +%Y%m%d)" --no-reboot

# Step 2: 새 Graviton4 인스턴스 생성
#   ※ 기존 x86 AMI는 ARM64 인스턴스에서 부팅 불가!
#   → Amazon Linux 2023 ARM64 공식 AMI로 새 인스턴스를 만든 후 환경을 재구성
aws ec2 run-instances \
  --image-id ami-ARM64-AL2023-XXXX \
  --instance-type m8g.xlarge \
  --subnet-id subnet-XXXXXXXXXXXX \
  --security-group-ids sg-XXXXXXXXXXXX \
  --key-name my-key \
  --user-data file://bootstrap-graviton-standalone.sh
```

### 11.3 단독 EC2용 부트스트랩 스크립트

```bash
#!/bin/bash
# bootstrap-graviton-standalone.sh — 단독 EC2 Graviton4 환경 구성

set -euo pipefail

echo "=== [1/6] Java 11 (Amazon Corretto ARM64) 설치 ==="
sudo dnf install -y java-11-amazon-corretto-headless

echo "=== [2/6] Python 3 환경 구성 ==="
sudo dnf install -y python3 python3-pip python3-devel gcc gcc-c++
sudo python3 -m pip install --upgrade pip

echo "=== [3/6] Spark 3.5.x 설치 (ARM64) ==="
SPARK_VERSION="3.5.3"
HADOOP_VERSION="3"
wget -q "https://dlcdn.apache.org/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" -P /tmp
sudo tar -xzf /tmp/spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz -C /opt/
sudo ln -s /opt/spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION} /opt/spark

echo "=== [4/6] Zeppelin 0.12.0 설치 ==="
wget -q "https://dlcdn.apache.org/zeppelin/zeppelin-0.12.0/zeppelin-0.12.0-bin-all.tgz" -P /tmp
sudo tar -xzf /tmp/zeppelin-0.12.0-bin-all.tgz -C /opt/
sudo ln -s /opt/zeppelin-0.12.0-bin-all /opt/zeppelin

echo "=== [5/6] Python 패키지 설치 ==="
sudo python3 -m pip install -r /tmp/requirements-py3.txt

echo "=== [6/6] 환경 변수 설정 ==="
cat >> /etc/profile.d/spark-zeppelin.sh << 'ENVEOF'
export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
export SPARK_HOME=/opt/spark
export ZEPPELIN_HOME=/opt/zeppelin
export PYSPARK_PYTHON=/usr/bin/python3
export PYSPARK_DRIVER_PYTHON=/usr/bin/python3
export PATH=$SPARK_HOME/bin:$ZEPPELIN_HOME/bin:$PATH
ENVEOF

source /etc/profile.d/spark-zeppelin.sh

echo "=== 부트스트랩 완료 ==="
echo "Zeppelin 시작: /opt/zeppelin/bin/zeppelin-daemon.sh start"
```

### 11.4 S3에서 데이터 복원

부트스트랩 완료 후, S3 이관 허브에서 노트북과 설정을 복원한다:

```bash
S3_BUCKET="s3://my-emr-migration-hub"

# 변환된 노트북 복원
aws s3 sync ${S3_BUCKET}/notebooks/converted-py3/ /opt/zeppelin/notebook/

# 커스텀 스크립트 복원
aws s3 sync ${S3_BUCKET}/custom-scripts/ /home/ec2-user/scripts/

# Zeppelin 시작
/opt/zeppelin/bin/zeppelin-daemon.sh start
```

> **주의**: 단독 EC2의 경우 설정 파일 경로가 EMR과 다르므로 (`/opt/zeppelin` vs `/usr/lib/zeppelin`), 설정 복원 시 경로를 확인해야 한다.
