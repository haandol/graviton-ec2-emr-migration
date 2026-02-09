# EMR Graviton4 마이그레이션 프로젝트

## 프로젝트 개요

Zeppelin 기반 EMR 환경을 Intel(x86-64)에서 Graviton4(ARM64)로 전환하는 마이그레이션 플레이북 문서 모음이다. Python 2 → 3, EMR 5.x → 7.x, Spark 2.x → 3.x, Zeppelin 0.8 → 0.10+, Intel → Graviton4 전환을 포함한다.

## 문서 구조

```
emr/
├── 01-emr-migration-overview.md      # 메인 가이드: 마이그레이션 배경, 목표, 4대 축, 인벤토리 조사
├── 02-python2-to-python3-guide.md    # Phase 1: Python 2→3 전환 (패키지 호환성, 자동 변환, 노트북 변환)
├── 03-intel-to-graviton-guide.md     # Phase 2~5: EMR/Spark 업그레이드, Zeppelin, ARM64 전환, IaC 자동화
└── 04-migration-checklist.md         # 실행 체크리스트: Phase별 체크리스트, Go/No-Go, 롤백
```

## 현재 환경 → 목표 환경

| 항목     | AS-IS                | TO-BE                     |
| -------- | -------------------- | ------------------------- |
| 아키텍처 | Intel x86-64 (m5/r5) | Graviton4 ARM64 (m8g/r8g) |
| OS       | Amazon Linux 2 (x86) | Amazon Linux 2023 (ARM64) |
| Python   | 2.7                  | 3.9+                      |
| EMR      | 5.x                  | 7.x                       |
| Spark    | 2.x                  | 3.x                       |
| Zeppelin | 0.8~0.9              | 0.10+                     |
| Java     | 8                    | Amazon Corretto 11+       |

## 마이그레이션 Phase 순서

1. **Phase 0** — 현행 환경 보존 및 인벤토리 조사 (AMI 생성, EBS 스냅샷, 버전/패키지/노트북/인프라 기록)
2. **Phase 0.5** — 스테이징 환경 구성 (기존 AMI에서 테스트용 인스턴스 생성, 운영 무중단)
3. **Phase 1** — Python 2 → 3 전환 (스테이징에서 수행, 가장 리스크 크고 작업량 많음)
4. **Phase 2** — EMR 5.x → 7.x / Spark 2.x → 3.x 업그레이드 설정 준비 (스테이징에서 수행)
5. **Phase 3** — Zeppelin 0.8 → 0.10+ 업그레이드 설정 (스테이징에서 수행)
6. **Phase 3.5** — S3 이관 허브 구성 (변환된 노트북/설정/스크립트/데이터를 S3에 체계적으로 업로드)
7. **Phase 4** — Intel → Graviton4(ARM64) 인스턴스 전환 (새 EMR 7.x Graviton 클러스터 생성)
8. **Phase 5** — IaC 자동화 (Terraform / CloudFormation / CDK) + 새 클러스터 데이터/설정 복원
9. **Phase 5.7** — 네트워크 및 데이터 소스 연결 검증 (S3, RDS, Redshift, YARN 내부 통신 등)
10. **Phase 6** — 통합 테스트 및 성능 검증
11. **Phase 7** — 프로덕션 전환 (Blue-Green Cutover)

> 운영 환경 상태를 먼저 보존(AMI)하고, 스테이징에서 소프트웨어 호환성을 확보한 뒤, 새 Graviton 클러스터로 전환한다. EMR은 in-place 업그레이드가 불가능하므로 반드시 새 클러스터를 생성하여 Blue-Green 방식으로 전환한다.

## 핵심 주의사항

### Python 2 → 3 전환 시

- `2to3` 또는 `futurize` 자동 변환 후 반드시 수동 검토 필요
- 나눗셈 결과(정수 vs float), 문자열 인코딩, dict 메서드 변경에 주의
- Zeppelin 노트북은 JSON 형식이므로 Python 셀 추출 → 변환 → 재삽입 워크플로 사용
- ARM64 휠 미제공 패키지는 소스 빌드 필요 — Phase 1에서 미리 파악

### Spark 2.x → 3.x 전환 시

- `PandasUDFType` deprecated → 데코레이터 방식 사용
- 암시적 타입 캐스팅 제한됨 → 명시적 `cast()` 사용
- `spark.sql.legacy.*` 호환 설정은 임시 조치이며 최종적으로 네이티브 방식으로 수정

### 운영 중인 EC2/EMR 마이그레이션 시

- Phase 0에서 반드시 AMI 생성 (원본 상태 완전 백업, --no-reboot 옵션으로 운영 무중단)
- Phase 1~3 코드 변환은 반드시 스테이징 환경에서 수행 (운영에 직접 변경 금지)
- EMR은 in-place 업그레이드 불가 — 새 클러스터 생성 후 Blue-Green 전환 필수
- S3 이관 허브를 중심으로 기존 환경 ↔ 새 환경 간 데이터 이동
- 새 클러스터 설정 복원 시 interpreter.json 등은 머지 방식 (단순 덮어쓰기 금지)
- x86 AMI는 ARM64 인스턴스에서 부팅 불가 — 단독 EC2는 새 ARM64 OS AMI에서 환경 재구성 필요

### Graviton 전환 시

- 인스턴스 매핑: m5→m8g, r5→r8g, c5→c8g
- pip 19.3+ 필수 (manylinux2014/aarch64 휠 지원)
- VPC/Subnet/보안그룹/IAM은 기존과 동일하게 유지
- 새 클러스터에서 S3, RDS, Redshift 등 데이터 소스 연결 검증 필수

### 롤백

- 기존 Intel 클러스터를 7일간 유지하여 롤백 대비
- S3 버전 관리 활성화 필수
- Phase 0 AMI로 원본 환경 완전 복원 가능 (최후의 수단)
- Critical 트리거: Zeppelin UI 접속 불가(30분+), Spark 인터프리터 반복 실패, OOM/크래시

## 단계 실패 시 대응 규칙

각 Phase를 실행하는 도중 실패가 발생하여 다음 단계로 진행할 수 없는 경우, 다음 순서를 반드시 따른다:

1. **현재 상황 설명** — 어떤 단계에서, 어떤 작업이, 어떤 이유로 실패했는지 구체적으로 설명한다.
2. **대안 제시** — 실패를 우회하거나 해결할 수 있는 대안을 복수로 제시한다. (예: 설정 변경, 버전 조정, 수동 개입, 롤백 등)
3. **사용자에게 액션 확인** — 제시한 대안 중 어떤 방향으로 진행할지 사용자에게 질문하여 확인을 받은 뒤 다음 작업을 수행한다.

> 실패 상태에서 사용자 확인 없이 임의로 다음 단계로 넘어가거나, 자의적으로 우회 조치를 적용하지 않는다.

## 문서 편집 규칙

- 이 프로젝트는 코드 저장소가 아닌 **기술 문서(플레이북)** 모음이다
- 모든 문서는 한국어로 작성한다
- 마크다운 형식을 사용하며, 코드 블록에는 적절한 언어 태그를 지정한다
- 문서 간 상호 참조 링크를 유지한다 (상대 경로 사용)
- 체크리스트(04)의 체크박스 형식(`☐`)을 유지한다
