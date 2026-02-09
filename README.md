# Kiro를 활용한 EMR Graviton4 마이그레이션 가이드

> **Kiro(CLI)를 실무 도우미로 활용하여 각 Phase의 작업을 자동화하고 가속하는 방법**

---

## 목차

1. [사전 준비](#1-사전-준비)
2. [Phase 0: 현행 환경 보존 및 인벤토리 조사 자동화](#2-phase-0-현행-환경-보존-및-인벤토리-조사-자동화)
3. [Phase 0.5: 스테이징 환경 구성](#3-phase-05-스테이징-환경-구성)
4. [Phase 1: Python 2→3 전환 지원](#4-phase-1-python-23-전환-지원)
5. [Phase 2: Spark 2.x→3.x 코드 마이그레이션](#5-phase-2-spark-2x3x-코드-마이그레이션)
6. [Phase 3: Zeppelin 설정 마이그레이션](#6-phase-3-zeppelin-설정-마이그레이션)
7. [Phase 3.5: S3 이관 허브 구성](#7-phase-35-s3-이관-허브-구성)
8. [Phase 4: Graviton 인스턴스 전환 지원](#8-phase-4-graviton-인스턴스-전환-지원)
9. [Phase 5: IaC 템플릿 생성 및 클러스터 프로비저닝](#9-phase-5-iac-템플릿-생성-및-클러스터-프로비저닝)
10. [Phase 5.5: 새 클러스터 데이터/설정 복원](#10-phase-55-새-클러스터-데이터설정-복원)
11. [Phase 5.7: 네트워크 및 데이터 소스 연결 검증](#11-phase-57-네트워크-및-데이터-소스-연결-검증)
12. [Phase 6~7: 테스트·전환 체크리스트 관리](#12-phase-67-테스트전환-체크리스트-관리)
13. [AGENTS.md 활용 전략](#13-agentsmd-활용-전략)
14. [실전 프롬프트 레시피 모음](#14-실전-프롬프트-레시피-모음)

---

## 1. 사전 준비

### 1.1 프로젝트 디렉터리 구성

마이그레이션 작업 전용 디렉터리에서 Kiro를 실행한다. 기존 플레이북 문서들과 함께 작업 파일을 관리한다:

```text
emr/
├── AGENTS.md                          # Kiro 프로젝트 지침
├── kiro-migration-guide.md            # 이 문서
├── 01-emr-migration-overview.md
├── 02-python2-to-python3-guide.md
├── 03-intel-to-graviton-guide.md
├── 04-migration-checklist.md
├── notebooks/                         # 변환 대상 Zeppelin 노트북
├── scripts/                           # 부트스트랩·동기화 스크립트
├── terraform/                         # IaC 템플릿
└── test/                              # 테스트 스크립트
```

### 1.2 AGENTS.md의 역할

`AGENTS.md` 파일은 Kiro가 프로젝트 컨텍스트를 자동으로 이해하도록 돕는 핵심 파일이다. 이 프로젝트에는 이미 AGENTS.md가 설정되어 있으므로, Kiro를 실행하면 마이그레이션 Phase 순서, 현재/목표 환경, 핵심 주의사항을 자동으로 인지한다.

---

## 2. Phase 0: 현행 환경 보존 및 인벤토리 조사 자동화

### 2.1 기존 EC2/EMR 상태 백업 (AMI 생성)

운영 환경을 변경하기 전에 현재 상태를 완전히 보존해야 한다. Kiro에 AMI 생성 스크립트를 요청한다:

```
프롬프트: 현재 운영 중인 EC2 인스턴스(또는 EMR 클러스터의 Master/Core 노드)의
AMI를 생성하는 bash 스크립트를 작성해줘. 다음 요구사항을 따라줘:
- 인스턴스 ID를 인자로 받음
- --no-reboot 옵션으로 운영 무중단 AMI 생성
- Purpose=migration-backup 태그 자동 부여
- AMI 생성 완료까지 대기 후 상태 확인
- EMR 클러스터인 경우 aws emr list-instances로 노드별 인스턴스 ID를
  자동 추출하는 옵션도 포함
01-emr-migration-overview.md의 Phase 0 AMI 생성 절차를 참고해줘.
```

### 2.2 EBS 스냅샷 생성

```
프롬프트: EC2 인스턴스에 부착된 모든 EBS 볼륨을 조회하고,
각 볼륨의 스냅샷을 자동으로 생성하는 스크립트를 만들어줘.
인스턴스 ID를 인자로 받고, 스냅샷에 migration-backup 태그를 달아줘.
```

### 2.3 현재 환경 정보 수집 스크립트 생성

EMR 클러스터에 SSH 접속 후, Kiro에 다음과 같이 요청한다:

```
프롬프트: 현재 EMR 클러스터의 전체 인벤토리를 수집하는 bash 스크립트를
작성해줘. EMR/Spark/Zeppelin/Java/Python 버전, pip 패키지 목록,
Zeppelin 노트북 수, 인스턴스 타입, VPC/Subnet/보안그룹/IAM 정보를
모두 수집해서 하나의 JSON 파일로 출력해야 해.
클러스터 ID는 인자로 받도록 해줘.
```

Kiro가 생성하는 스크립트는 다음을 자동 수집한다:

- `aws emr describe-cluster`로 클러스터 메타데이터
- `pip freeze`로 Python 패키지 목록
- `find`로 Zeppelin 노트북 개수
- 모든 결과를 구조화된 JSON으로 출력

### 2.4 패키지 호환성 매트릭스 자동 생성

```
프롬프트: requirements-current-py2.txt 파일을 읽고, 각 패키지의
Python 3 호환 여부와 ARM64(aarch64) 휠 제공 여부를 조사해서
마크다운 테이블로 정리해줘. 02-python2-to-python3-guide.md의
호환성 매트릭스 템플릿 형식을 따라줘.
```

### 2.5 인벤토리 결과 검토

수집된 결과를 Kiro에 넘겨 자동 분석한다:

```
프롬프트: inventory.json을 분석해서 01-emr-migration-overview.md의
인벤토리 결과 템플릿을 채워줘. 마이그레이션 시 리스크가 될 수 있는
항목이 있으면 별도로 표시해줘.
```

---

## 3. Phase 0.5: 스테이징 환경 구성

Phase 1~3의 코드 변환 및 테스트를 운영 중인 환경에서 직접 수행하면 서비스에 영향을 준다. 스테이징 환경을 별도로 구성하여 안전하게 작업한다.

### 3.1 스테이징 인스턴스 생성 스크립트

```
프롬프트: Phase 0에서 생성한 AMI로 스테이징 인스턴스를 만드는
bash 스크립트를 작성해줘. 다음을 인자로 받아야 해:
- AMI ID (Phase 0에서 생성한 것)
- 인스턴스 타입 (기존과 동일한 Intel 타입)
- Subnet ID, Security Group ID, Key Name
Name=emr-staging-py3 태그를 자동 부여하고,
생성된 인스턴스의 퍼블릭 IP와 SSH 접속 명령어를 출력해줘.
01-emr-migration-overview.md의 Phase 0.5 섹션을 참고해줘.
```

### 3.2 EMR 스테이징 클러스터 생성

EMR 관리형 클러스터인 경우:

```
프롬프트: 현재 운영 EMR 클러스터와 동일한 설정으로 스테이징 EMR 클러스터를
생성하는 AWS CLI 명령어를 만들어줘. inventory.json에서 EMR 릴리스,
인스턴스 타입, VPC/Subnet/보안그룹 정보를 읽어서 반영해줘.
클러스터 이름은 "staging-migration-test"로 하고,
01-emr-migration-overview.md의 Phase 0.5 섹션을 참고해줘.
```

### 3.3 스테이징 환경 검증

```
프롬프트: 스테이징 인스턴스/클러스터가 기존 환경과 동일하게 동작하는지
검증하는 스크립트를 만들어줘.
- Zeppelin UI 접속 확인
- Spark 인터프리터 동작 확인
- Python 버전 및 pip 패키지 목록 비교
- 기존 inventory.json과의 차이점을 출력
```

---

## 4. Phase 1: Python 2→3 전환 지원

이 Phase가 전체 마이그레이션에서 가장 작업량이 많다. Kiro를 적극 활용하면 변환 품질을 높이고 수동 검토 시간을 줄일 수 있다.

> **주의**: Phase 1~3은 반드시 스테이징 환경에서 수행한다. 운영 환경에 직접 변경을 가하지 않는다.

### 4.1 Python 파일 자동 변환 + 리뷰

단일 파일 또는 디렉터리 단위로 Kiro에 변환을 요청한다:

```
프롬프트: scripts/etl_pipeline.py 파일을 Python 2에서 Python 3로
변환해줘. 다음 사항에 특히 주의해줘:
- 정수 나눗셈이 float로 바뀌는 곳 (// 로 변경 필요 여부)
- unicode/str 인코딩 처리
- dict.iteritems() 등 삭제된 메서드
- print 문 → print() 함수
변환 후 원본과 달라진 부분을 요약해줘.
```

### 4.2 Zeppelin 노트북 일괄 변환

Zeppelin 노트북은 JSON 형식이므로 Kiro가 직접 파싱하여 Python 셀만 추출·변환할 수 있다:

```
프롬프트: notebooks/ 디렉터리에 있는 Zeppelin 노트북(note.json) 파일들을
읽어서, %python 또는 %pyspark 셀의 코드를 Python 3로 변환해줘.
02-python2-to-python3-guide.md의 변환 워크플로를 따르고,
변환 전 반드시 원본을 .bak으로 백업해줘.
각 노트북별로 변환된 셀 수와 주요 변경 내용을 요약해줘.
```

### 4.3 나눗셈 연산자 전수 검사

Python 2→3에서 가장 위험한 변경인 나눗셈 동작 차이를 Kiro로 탐색한다:

```
프롬프트: notebooks/ 와 scripts/ 디렉터리에서 정수 나눗셈(/)이
사용된 모든 위치를 찾아줘. 각 위치에서 / 를 // 로 바꿔야 하는지,
아니면 float 결과가 의도된 것인지 분석해줘.
```

### 4.4 대체 패키지 결정 지원

Python 2 전용 패키지의 대체재를 찾을 때:

```
프롬프트: requirements-current-py2.txt에서 Python 3에서
사용할 수 없는 패키지를 찾아줘. 각 패키지에 대해
02-python2-to-python3-guide.md의 대체 패키지 표를 참고하여
대체 방안을 제시해줘. ARM64 휠 제공 여부도 확인해줘.
```

---

## 5. Phase 2: Spark 2.x→3.x 코드 마이그레이션

### 5.1 deprecated API 탐색 및 자동 변환

```
프롬프트: notebooks/ 디렉터리의 모든 %pyspark 셀에서
Spark 2.x에서 deprecated되거나 삭제된 API를 찾아줘.
03-intel-to-graviton-guide.md의 Spark API 변경 요약 테이블을
기준으로 검사하고, Spark 3.x 호환 코드로 변환 방안을 제시해줘.
특히 다음에 주의:
- PandasUDFType → 데코레이터 방식
- 암시적 타입 캐스팅 → 명시적 cast()
- CSV 파싱 모드 변경
```

### 5.2 spark-defaults.conf 생성

```
프롬프트: 03-intel-to-graviton-guide.md의 EMR Configurations 매핑 섹션을
참고해서, 우리 환경에 맞는 spark-defaults.conf를 생성해줘.
Spark 3.x의 adaptive execution을 활성화하고,
Python 3 경로를 PYSPARK_PYTHON으로 설정해줘.
초기에는 spark.sql.legacy.* 호환 설정도 포함해줘.
```

### 5.3 legacy 설정 제거 계획

```
프롬프트: spark-defaults.conf에 있는 spark.sql.legacy.* 설정들을
목록으로 정리하고, 각 설정을 제거하려면 코드에서 어떤 수정이
필요한지 분석해줘. 제거 우선순위도 제안해줘.
```

---

## 6. Phase 3: Zeppelin 설정 마이그레이션

### 6.1 설정 파일 변환

```
프롬프트: 기존 Zeppelin 0.8의 설정 파일들(zeppelin-site.xml,
zeppelin-env.sh, interpreter.json)을 읽고,
Zeppelin 0.10+ 호환 설정으로 변환해줘.
03-intel-to-graviton-guide.md의 Phase 3 섹션을 참고해서
Python 3 인터프리터 경로 설정도 포함해줘.
```

### 6.2 interpreter.json 마이그레이션

```
프롬프트: 기존 interpreter.json을 분석해서 Zeppelin 0.10+에서
변경된 인터프리터 설정 키를 찾아줘. PYSPARK_PYTHON,
spark.pyspark.python 등의 경로가 모두 /usr/bin/python3으로
설정되어 있는지 확인하고 수정해줘.
```

---

## 7. Phase 3.5: S3 이관 허브 구성

스테이징에서 코드 변환(Phase 1~3)이 완료되면, 모든 결과물을 S3에 체계적으로 업로드한다. 이 S3 이관 허브가 기존 환경과 새 Graviton 환경을 잇는 중간 저장소 역할을 한다.

### 7.1 S3 이관 허브 구성 스크립트

```
프롬프트: 스테이징 환경에서 변환 완료된 결과물을 S3 이관 허브에
업로드하는 bash 스크립트를 만들어줘.
03-intel-to-graviton-guide.md의 Phase 3.5 S3 이관 허브 구조를 참고해서,
다음을 체계적으로 업로드해야 해:
- notebooks/converted-py3/: Python 3 변환 완료 노트북
- config/zeppelin-conf/: Zeppelin 설정 파일
- config/spark/: Spark 설정 파일
- packages/requirements-py3.txt: Python 3 패키지 목록
- custom-scripts/: 커스텀 스크립트
- config/crontab-backup.txt: cron job 백업
S3 버킷명은 인자로 받도록 해줘.
```

### 7.2 HDFS 데이터 이관 (해당 시)

```
프롬프트: EMR 클러스터의 HDFS에 있는 데이터를 S3로 이관하는
스크립트를 만들어줘. hadoop distcp를 사용하고,
대상 HDFS 경로와 S3 버킷을 인자로 받도록 해줘.
이관 전후 파일 수와 총 크기를 비교하여 정합성을 검증해줘.
```

### 7.3 EBS 데이터 이관 (해당 시)

```
프롬프트: 기존 EC2에 부착된 EBS 데이터 볼륨의 내용을 S3로 이관하거나,
EBS 스냅샷에서 새 볼륨을 생성하여 새 Graviton 인스턴스에 부착하는
두 가지 방법의 스크립트를 각각 만들어줘.
03-intel-to-graviton-guide.md의 EBS 볼륨 데이터 이관 섹션을 참고해줘.
```

### 7.4 Hive Metastore 이관 확인

```
프롬프트: 현재 EMR 클러스터가 Glue Data Catalog을 사용하는지,
로컬 Hive Metastore를 사용하는지 확인하는 방법을 알려줘.
로컬 Metastore인 경우 덤프 및 이관 스크립트를 만들어줘.
Glue Data Catalog인 경우 별도 이관이 불필요함을 확인해줘.
```

---

## 8. Phase 4: Graviton 인스턴스 전환 지원

### 8.1 인스턴스 매핑 검증

```
프롬프트: inventory.json에서 현재 사용 중인 인스턴스 타입을 추출하고,
03-intel-to-graviton-guide.md의 인스턴스 타입 매핑표를 참고하여
Graviton 대응 인스턴스를 매핑해줘. vCPU와 메모리가 동일한지 검증하고,
매핑 결과를 테이블로 정리해줘.
```

### 8.2 ARM64 패키지 호환성 최종 점검

```
프롬프트: requirements-py3.txt의 모든 패키지에 대해
aarch64/arm64 휠이 PyPI에 제공되는지 확인해줘.
휠이 없는 패키지는 소스 빌드에 필요한 시스템 의존성(gcc,
python3-devel 등)과 함께 정리해줘.
```

### 8.3 부트스트랩 스크립트 커스터마이징

```
프롬프트: 03-intel-to-graviton-guide.md의 bootstrap-graviton.sh를
기반으로, 우리의 requirements-py3.txt에 있는 패키지를 모두 설치하는
커스텀 부트스트랩 스크립트를 만들어줘. ARM64 휠이 없는 패키지는
--no-binary 옵션으로 소스 빌드하도록 해줘.
```

### 8.4 단독 EC2 운영 시 부트스트랩 (EMR이 아닌 경우)

EMR 관리형 클러스터가 아닌 단독 EC2에서 Zeppelin을 운영하는 경우:

```
프롬프트: 03-intel-to-graviton-guide.md의 부록(§11 단독 EC2 절차)을
참고해서, Amazon Linux 2023 ARM64에 Spark 3.5.x와 Zeppelin 0.12.0을
수동 설치하는 부트스트랩 스크립트를 만들어줘.
requirements-py3.txt의 패키지도 모두 설치하고,
환경 변수(JAVA_HOME, SPARK_HOME, PYSPARK_PYTHON 등)도 설정해줘.
x86 AMI는 ARM64에서 부팅 불가하므로 새 OS AMI에서 시작해야 한다는 점에 주의해줘.
```

---

## 9. Phase 5: IaC 템플릿 생성 및 클러스터 프로비저닝

Kiro는 기존 플레이북 문서를 참조하여 IaC 템플릿을 정확하게 생성할 수 있다.

### 9.1 Terraform 모듈 생성

```
프롬프트: 03-intel-to-graviton-guide.md의 Terraform 예시를 기반으로,
실제 사용 가능한 Terraform 모듈을 만들어줘. 다음을 포함해야 해:
- variables.tf: 인스턴스 타입, 클러스터 이름, VPC/Subnet 등을 변수로
- main.tf: aws_emr_cluster 리소스 (m8g 인스턴스, EMR 7.x)
- outputs.tf: 클러스터 ID, 마스터 퍼블릭 DNS
- bootstrap.sh: S3에 업로드할 부트스트랩 스크립트
인벤토리 조사 결과의 VPC/Subnet/보안그룹 값을 terraform.tfvars
예시로 작성해줘.
```

### 9.2 CloudFormation 템플릿 생성

```
프롬프트: 03-intel-to-graviton-guide.md의 CloudFormation 예시를 기반으로,
파라미터화된 CloudFormation 템플릿을 만들어줘.
Parameters 섹션에 인스턴스 타입, Subnet, 보안그룹, S3 버킷을 받고,
Outputs에 클러스터 엔드포인트를 출력해줘.
```

### 9.3 기존 IaC 코드 마이그레이션

이미 Terraform이나 CloudFormation 코드가 있는 경우:

```
프롬프트: terraform/ 디렉터리의 기존 EMR 클러스터 Terraform 코드를
읽고, Graviton4 마이그레이션에 맞게 수정해줘.
- 인스턴스 타입을 m5→m8g, r5→r8g로 변경
- EMR 릴리스를 7.x로 변경
- Python 3 환경 설정 추가
- 변경 전후 diff를 보여줘
```

---

## 10. Phase 5.5: 새 클러스터 데이터/설정 복원

IaC로 새 Graviton 클러스터를 프로비저닝한 후, S3 이관 허브에서 노트북, 설정, 스크립트를 복원한다.

### 10.1 노트북 및 설정 복원 스크립트

```
프롬프트: 새 Graviton EMR 클러스터에서 S3 이관 허브의 데이터를
복원하는 bash 스크립트를 만들어줘.
03-intel-to-graviton-guide.md의 §6(새 클러스터 데이터/설정 복원)을 참고해서:
- S3에서 변환된 노트북 복원
- 커스텀 스크립트 복원
- cron job 백업 파일 다운로드 (수동 검토용)
- Zeppelin 재시작 및 HTTP 상태 확인
S3 버킷명을 인자로 받도록 해줘.
```

### 10.2 interpreter.json 설정 머지

> **핵심**: 새 EMR 7.x의 기본 interpreter.json을 단순 덮어쓰기하면 안 된다. 기존 커스텀 설정값만 머지해야 한다.

```
프롬프트: 03-intel-to-graviton-guide.md의 interpreter.json 머지 스크립트를
참고해서, 기존 interpreter.json과 새 EMR 기본 interpreter.json을
안전하게 머지하는 Python 스크립트를 만들어줘.
새 EMR 기본 설정의 구조를 유지하면서, Python/PySpark 인터프리터 경로만
/usr/bin/python3으로 변경해야 해.
기존 설정에서 추가로 옮겨야 할 커스텀 설정이 있으면 감지하여 보고해줘.
```

### 10.3 cron job 검토 및 적용

```
프롬프트: S3에서 다운로드한 crontab-backup.txt를 읽고,
각 cron job이 새 Graviton 환경에서 그대로 동작할 수 있는지 분석해줘.
Python 경로, 스크립트 경로 등이 변경되어야 하는 항목을 찾아서
수정된 crontab 파일을 생성해줘.
```

---

## 11. Phase 5.7: 네트워크 및 데이터 소스 연결 검증

새 Graviton 클러스터가 기존 데이터 소스에 접근할 수 있는지 검증한다.

### 11.1 연결 검증 스크립트 자동 생성

```
프롬프트: 새 Graviton EMR 클러스터에서 모든 데이터 소스 연결을
자동으로 검증하는 bash 스크립트를 만들어줘.
03-intel-to-graviton-guide.md의 §7(네트워크 및 데이터 소스 연결 검증)을
참고해서 다음을 순서대로 검증해야 해:
- S3 버킷 접근 (읽기/쓰기)
- RDS/Aurora 연결 (호스트, 포트를 인자로)
- Redshift 연결 (호스트, 포트를 인자로)
- IAM 역할 권한 (S3, Glue 접근)
- YARN 내부 통신 (Master ↔ Core)
- Spark 분산 잡 실행 (yarn cluster mode)
각 항목을 PASS/FAIL로 출력해줘.
```

### 11.2 보안그룹 규칙 비교

```
프롬프트: 기존 Intel 클러스터와 새 Graviton 클러스터에 할당된
보안그룹의 인바운드/아웃바운드 규칙을 비교해줘.
기존 클러스터 ID와 새 클러스터 ID를 인자로 받아서,
규칙 차이가 있으면 경고를 출력하는 스크립트를 만들어줘.
```

### 11.3 IAM 역할 권한 감사

```
프롬프트: 새 Graviton 클러스터에 할당된 IAM 역할이 기존 환경과
동일한 권한을 가지고 있는지 검증해줘.
S3 버킷 접근, Glue Data Catalog 접근, CloudWatch Logs 접근 등
핵심 권한이 모두 있는지 확인하고, 누락된 권한이 있으면 보고해줘.
```

---

## 12. Phase 6~7: 테스트·전환 체크리스트 관리

### 12.1 테스트 스크립트 자동 생성

```
프롬프트: 03-intel-to-graviton-guide.md의 통합 테스트 섹션을 참고해서,
신규 Graviton EMR 클러스터의 상태를 자동으로 검증하는
bash 스크립트를 만들어줘. 다음을 순서대로 검증해야 해:
1. Zeppelin UI HTTP 200 응답
2. Spark 인터프리터 동작 (간단한 Spark 잡 제출)
3. Python 3 버전 및 필수 패키지 import
4. PySpark DataFrame 기본 연산
결과를 PASS/FAIL로 출력하고, 실패 시 원인 정보도 포함해줘.
```

### 12.2 체크리스트 진행 상황 업데이트

작업이 진행되면 Kiro에 체크리스트 업데이트를 요청한다:

```
프롬프트: 04-migration-checklist.md에서 Phase 0의 0-1 ~ 0-7 항목을
완료(☐ → ☑)로 변경해줘. 날짜는 오늘, 담당자는 "홍길동"으로 기입해줘.
```

### 12.3 Go/No-Go 판정 보조

```
프롬프트: 04-migration-checklist.md의 Go/No-Go 판정 기준을 읽고,
test-results.json의 테스트 결과와 대조해서
각 항목의 합격/불합격 여부를 판정해줘.
No-Go 항목이 있으면 원인과 해결 방안도 제시해줘.
```

### 12.4 롤백 상황 대응

문제가 발생했을 때 Kiro에 즉시 지원을 요청한다:

```
프롬프트: Graviton 클러스터에서 다음 오류가 발생하고 있어:
[오류 메시지 붙여넣기]
03-intel-to-graviton-guide.md의 롤백 트리거 조건에 해당하는지 판단하고,
롤백 없이 해결 가능하면 해결 방법을,
롤백이 필요하면 롤백 실행 절차를 안내해줘.
AMI 기반 완전 복원이 필요한 경우도 고려해줘.
```

---

## 13. AGENTS.md 활용 전략

### 13.1 AGENTS.md가 하는 일

프로젝트 루트의 `AGENTS.md` 파일은 Kiro가 세션을 시작할 때 자동으로 읽는 프로젝트 지침서이다. 이 프로젝트의 AGENTS.md에는 다음이 포함되어 있다:

- 마이그레이션 Phase 순서 (Phase 0 ~ 0.5 ~ 1 ~ 2 ~ 3 ~ 3.5 ~ 4 ~ 5 ~ 5.7 ~ 6 ~ 7)
- 현재 환경 → 목표 환경 매핑표
- 각 Phase의 핵심 주의사항
- 운영 중인 EC2/EMR 마이그레이션 시 주의사항
- 문서 편집 규칙 (한국어, 마크다운, 체크박스 형식)

이를 통해 Kiro는 **별도의 설명 없이도** 다음을 자동으로 인지한다:

- Phase 0에서 AMI를 생성하여 원본 상태를 완전히 보존해야 한다는 것
- Phase 1~3은 스테이징 환경에서 수행해야 한다는 것 (운영 무중단)
- EMR은 in-place 업그레이드가 불가능하여 Blue-Green 전환이 필수라는 것
- S3 이관 허브를 중심으로 데이터를 이동한다는 것
- interpreter.json 등은 머지 방식으로 복원해야 한다는 것 (덮어쓰기 금지)
- Python 2→3 전환이 가장 리스크가 크다는 것
- 인스턴스 매핑 규칙 (m5→m8g, r5→r8g, c5→c8g)
- 롤백 대비로 기존 클러스터를 7일간 유지하고, AMI로 완전 복원이 가능하다는 것

### 13.2 AGENTS.md 커스터마이징 팁

마이그레이션이 진행됨에 따라 AGENTS.md를 업데이트하면 Kiro의 응답 정확도가 높아진다:

```
프롬프트: AGENTS.md에 다음 정보를 추가해줘:
- 우리 클러스터 ID: j-ABC123XYZ
- S3 이관 허브 버킷: s3://our-emr-migration-hub
- Phase 0 AMI ID: ami-XXXXXXXXXXXX
- 현재 Phase: Phase 2 진행 중
- Phase 1 완료 일자: 2025-01-15
```

### 13.3 Phase별 컨텍스트 전환

각 Phase를 시작할 때 Kiro에 현재 단계를 명시하면 더 정확한 지원을 받을 수 있다:

```
프롬프트: 지금부터 Phase 3.5(S3 이관 허브 구성) 작업을 시작한다.
03-intel-to-graviton-guide.md의 Phase 3.5 섹션을 참조해서 작업해줘.
스테이징에서 변환 완료된 결과물을 S3에 업로드해야 해.
```

---

## 14. 실전 프롬프트 레시피 모음

### Phase별 핵심 프롬프트

#### Phase 0 — 환경 보존 및 인벤토리

| 목적               | 프롬프트                                                            |
| ------------------ | ------------------------------------------------------------------- |
| AMI 생성           | `현재 EC2 인스턴스의 AMI를 생성하는 스크립트를 만들어줘 (--no-reboot)` |
| EBS 스냅샷         | `인스턴스에 부착된 EBS 볼륨의 스냅샷을 생성하는 스크립트를 만들어줘`  |
| 환경 수집 스크립트 | `EMR 클러스터 j-XXX의 전체 인벤토리를 수집하는 스크립트를 만들어줘` |
| 패키지 호환성 점검 | `requirements-current-py2.txt의 Py3/ARM64 호환성을 분석해줘`        |
| 인벤토리 문서화    | `수집 결과를 01 문서의 템플릿 형식으로 정리해줘`                    |

#### Phase 0.5 — 스테이징 환경

| 목적                 | 프롬프트                                                              |
| -------------------- | --------------------------------------------------------------------- |
| 스테이징 인스턴스    | `Phase 0 AMI로 스테이징 인스턴스를 생성하는 스크립트를 만들어줘`      |
| 스테이징 EMR         | `현재와 동일한 설정의 스테이징 EMR 클러스터 생성 명령어를 만들어줘`   |
| 스테이징 환경 검증   | `스테이징이 기존 환경과 동일하게 동작하는지 검증하는 스크립트를 만들어줘` |

#### Phase 1 — Python 전환

| 목적        | 프롬프트                                                           |
| ----------- | ------------------------------------------------------------------ |
| 파일 변환   | `이 Python 2 코드를 Python 3로 변환하고 위험 포인트를 표시해줘`    |
| 노트북 변환 | `notebooks/의 Zeppelin 노트북 Python 셀을 Py3로 변환해줘`          |
| 나눗셈 검사 | `/ 연산자가 사용된 모든 위치에서 정수 나눗셈 의도 여부를 분석해줘` |
| 인코딩 검사 | `encode/decode 호출을 모두 찾아서 Py3 호환성을 검토해줘`           |
| 테스트 비교 | `Py2와 Py3 실행 결과를 비교해서 차이점을 분석해줘`                 |

#### Phase 2 — EMR/Spark 업그레이드

| 목적             | 프롬프트                                                           |
| ---------------- | ------------------------------------------------------------------ |
| API 마이그레이션 | `%pyspark 셀에서 Spark 2.x deprecated API를 찾아서 3.x로 변환해줘` |
| 설정 생성        | `EMR 7.x용 spark-defaults.conf를 생성해줘`                         |
| legacy 제거 계획 | `spark.sql.legacy.* 설정 제거 로드맵을 작성해줘`                   |

#### Phase 3 — Zeppelin

| 목적            | 프롬프트                                                     |
| --------------- | ------------------------------------------------------------ |
| 설정 변환       | `Zeppelin 0.8 설정을 0.10+ 호환으로 변환해줘`                |
| 인터프리터 설정 | `interpreter.json에서 Python 경로를 모두 python3로 변경해줘` |

#### Phase 3.5 — S3 이관 허브

| 목적              | 프롬프트                                                              |
| ----------------- | --------------------------------------------------------------------- |
| S3 업로드 스크립트 | `변환 결과물을 S3 이관 허브에 업로드하는 스크립트를 만들어줘`         |
| HDFS 이관         | `HDFS 데이터를 S3로 이관하고 정합성을 검증하는 스크립트를 만들어줘`   |
| EBS 이관          | `EBS 볼륨 데이터를 스냅샷 또는 S3로 이관하는 스크립트를 만들어줘`    |
| Metastore 확인    | `Glue Catalog vs 로컬 Hive Metastore 사용 여부를 확인해줘`           |

#### Phase 4 — Graviton

| 목적              | 프롬프트                                                          |
| ----------------- | ----------------------------------------------------------------- |
| 인스턴스 매핑     | `현재 인스턴스 타입을 Graviton 대응 타입으로 매핑해줘`            |
| 부트스트랩        | `ARM64 환경용 커스텀 부트스트랩 스크립트를 만들어줘`              |
| 패키지 빌드       | `aarch64 휠이 없는 패키지의 소스 빌드 스크립트를 만들어줘`        |
| 단독 EC2 부트스트랩 | `단독 EC2용 Spark+Zeppelin 수동 설치 부트스트랩을 만들어줘`     |

#### Phase 5 — IaC

| 목적           | 프롬프트                                            |
| -------------- | --------------------------------------------------- |
| Terraform      | `EMR Graviton 클러스터용 Terraform 모듈을 만들어줘` |
| CloudFormation | `파라미터화된 CloudFormation 템플릿을 만들어줘`     |
| CDK            | `AWS CDK TypeScript로 EMR 클러스터 스택을 만들어줘` |

#### Phase 5.5 — 데이터/설정 복원

| 목적                    | 프롬프트                                                                |
| ----------------------- | ----------------------------------------------------------------------- |
| 노트북/설정 복원        | `S3 이관 허브에서 새 클러스터로 데이터를 복원하는 스크립트를 만들어줘`  |
| interpreter.json 머지   | `기존과 새 interpreter.json을 안전하게 머지하는 스크립트를 만들어줘`    |
| cron job 검토           | `crontab-backup.txt를 분석해서 새 환경에 맞게 수정해줘`                |

#### Phase 5.7 — 네트워크 검증

| 목적              | 프롬프트                                                               |
| ----------------- | ---------------------------------------------------------------------- |
| 연결 검증 스크립트 | `모든 데이터 소스 연결을 자동 검증하는 스크립트를 만들어줘`            |
| 보안그룹 비교     | `기존/신규 클러스터의 보안그룹 규칙을 비교하는 스크립트를 만들어줘`    |
| IAM 권한 감사     | `새 클러스터의 IAM 역할 권한이 충분한지 검증해줘`                      |

#### Phase 6~7 — 테스트·전환

| 목적                | 프롬프트                                                         |
| ------------------- | ---------------------------------------------------------------- |
| 검증 스크립트       | `Graviton 클러스터 상태를 자동 검증하는 스크립트를 만들어줘`     |
| 체크리스트 업데이트 | `04-migration-checklist.md의 Phase X 항목을 완료로 업데이트해줘` |
| Go/No-Go            | `테스트 결과와 Go/No-Go 기준을 대조해서 판정해줘`                |
| 트러블슈팅          | `[오류 메시지] 이 오류의 원인과 해결 방법을 알려줘`              |
| 롤백 판단           | `이 상황이 롤백 트리거 조건에 해당하는지 판단해줘`               |

---

## 요약

```text
┌──────────────────────────────────────────────────────────────────────┐
│                  Kiro 활용 마이그레이션 흐름 (보완 버전)             │
│                                                                      │
│  Phase 0    ──→  AMI 생성 + EBS 스냅샷 + 인벤토리 수집/분석         │
│       ↓                                                              │
│  Phase 0.5  ──→  스테이징 인스턴스/클러스터 생성 + 환경 검증         │
│       ↓          (기존 AMI 또는 동일 설정으로 생성)                   │
│       ↓                                                              │
│  Phase 1    ──→  Python 2→3 자동 변환 + 수동 검토 보조               │
│       ↓          나눗셈·인코딩 전수 검사                              │
│       ↓          Zeppelin 노트북 일괄 변환                            │
│       ↓          ※ 스테이징에서 수행                                  │
│       ↓                                                              │
│  Phase 2    ──→  Spark deprecated API 탐색 + 코드 변환               │
│       ↓          spark-defaults.conf 생성                             │
│       ↓                                                              │
│  Phase 3    ──→  Zeppelin 설정 파일 변환                              │
│       ↓          인터프리터 Python 3 경로 설정                        │
│       ↓                                                              │
│  Phase 3.5  ──→  S3 이관 허브에 변환 결과물 업로드                    │
│       ↓          HDFS 데이터, EBS 데이터, Metastore 이관              │
│       ↓                                                              │
│  Phase 4    ──→  인스턴스 매핑 검증 + ARM64 패키지 호환성 점검       │
│       ↓          부트스트랩 스크립트 커스터마이징                      │
│       ↓                                                              │
│  Phase 5    ──→  Terraform / CloudFormation / CDK 템플릿 생성         │
│       ↓          새 Graviton EMR 클러스터 프로비저닝                   │
│       ↓                                                              │
│  Phase 5.5  ──→  S3에서 노트북/설정 복원 + interpreter.json 머지     │
│       ↓          커스텀 스크립트/cron job 복원                         │
│       ↓                                                              │
│  Phase 5.7  ──→  네트워크 연결 검증 (S3, RDS, YARN 등)               │
│       ↓          보안그룹 비교 + IAM 권한 감사                        │
│       ↓                                                              │
│  Phase 6    ──→  자동 검증 스크립트 + Go/No-Go 판정 보조             │
│       ↓                                                              │
│  Phase 7    ──→  Blue-Green 전환 + 트러블슈팅 + 롤백 지원            │
│                  기존 클러스터 7일 유지 후 종료                        │
└──────────────────────────────────────────────────────────────────────┘
```

**핵심 원칙**: Kiro는 자동 변환과 스크립트 생성을 담당하지만, **최종 검토와 실행 판단은 반드시 사람이 수행한다**. 특히 Python 2→3 나눗셈 변경, Spark API 호환성, interpreter.json 머지 검토, 네트워크 연결 확인, 프로덕션 전환 Go/No-Go 판정은 수동 확인이 필수이다.
