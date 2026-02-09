# Python 2 → Python 3 전환 가이드

> **EMR Graviton4 마이그레이션 — Phase 1 상세**

---

## 목차

1. [패키지 호환성 점검](#1-패키지-호환성-점검)
2. [자동 변환 도구](#2-자동-변환-도구)
3. [주요 변경점 레퍼런스](#3-주요-변경점-레퍼런스)
4. [Zeppelin 노트북 변환 워크플로](#4-zeppelin-노트북-변환-워크플로)
5. [외부 패키지 대응](#5-외부-패키지-대응)
6. [단위 테스트: 변환된 코드의 결과 검증](#6-단위-테스트-변환된-코드의-결과-검증)

---

## 1. 패키지 호환성 점검

### 1.1 자동 호환성 검사

```bash
# caniusepython3 도구로 패키지 호환성 일괄 점검
# 주의: caniusepython3은 2021년 이후 아카이브 상태(더 이상 업데이트 없음).
# 참고용으로만 사용하고, 각 패키지의 PyPI 페이지에서 Python 3 지원 여부를 직접 확인할 것.
pip install caniusepython3
caniusepython3 -r requirements-current-py2.txt
```

### 1.2 호환성 매트릭스 템플릿

| 패키지명      | 현재 버전 (Py2) | Py3 지원 여부 | Py3 호환 버전 | ARM64 휠 제공 | 비고           |
| ------------- | --------------- | ------------- | ------------- | ------------- | -------------- |
| numpy         | 1.16.x          | ✅            | 1.24+         | ✅            |                |
| pandas        | 0.24.x          | ✅            | 2.0+          | ✅            |                |
| scipy         | 1.2.x           | ✅            | 1.11+         | ✅            |                |
| matplotlib    | 2.2.x           | ✅            | 3.7+          | ✅            |                |
| (사내 패키지) | x.x.x           | ❓            | —             | ❓            | 소스 검토 필요 |

> **주의**: ARM64 휠이 제공되지 않는 패키지는 Graviton 전환 시 소스 빌드가 필요하다. 이 단계에서 미리 파악해 둔다.

---

## 2. 자동 변환 도구

### 2.1 2to3 (Python 표준 도구)

```bash
# 단일 파일 변환 (미리보기, 실제 수정 없음)
2to3 script.py                    # 변경 사항 미리보기만

# 단일 파일 변환 (바로 적용)
2to3 -w -n script.py

# 디렉터리 전체 일괄 변환
2to3 -w -n ./src/

# 특정 fixer만 적용
2to3 -f print -f has_key -w script.py
```

### 2.2 futurize (python-future)

```bash
# python-future 설치
pip install future

# Stage 1: 안전한 변환 (Py2/Py3 동시 호환 코드 생성)
futurize --stage1 -w ./src/

# Stage 2: Py3 전용 코드로 완전 전환
futurize --stage2 -w ./src/
```

### 2.3 python-modernize

```bash
pip install modernize

# 미리보기
python-modernize ./src/

# 적용
python-modernize -w ./src/
```

> **권장**: `futurize`의 Stage 1 → Stage 2 순서로 진행하면 단계적 전환이 가능하다.
> 자동 변환 후 반드시 수동 검토가 필요하다.

---

## 3. 주요 변경점 레퍼런스

### 3.1 print 문 → print() 함수

```python
# Python 2
print "Hello, World!"
print >> sys.stderr, "Error"

# Python 3
print("Hello, World!")
print("Error", file=sys.stderr)
```

### 3.2 문자열 (unicode 기본값)

```python
# Python 2: str = bytes, unicode = 텍스트
s = "hello"        # bytes
u = u"hello"       # unicode

# Python 3: str = 텍스트(unicode), bytes = 바이트
s = "hello"        # unicode (str)
b = b"hello"       # bytes

# 주의: encode/decode 처리가 달라짐
# Python 2
"hello".encode("utf-8")  # str → str (사실상 동일)

# Python 3
"hello".encode("utf-8")  # str → bytes
b"hello".decode("utf-8") # bytes → str
```

### 3.3 나눗셈 연산자

```python
# Python 2
5 / 2    # → 2 (정수 나눗셈)
5.0 / 2  # → 2.5

# Python 3
5 / 2    # → 2.5 (항상 float)
5 // 2   # → 2 (명시적 정수 나눗셈)
```

> **위험**: 데이터 처리 로직에서 정수 나눗셈이 사용된 경우 결과값이 달라질 수 있다. 반드시 확인할 것.

### 3.4 dict 메서드 변경

```python
# Python 2: dict.keys(), .values(), .items() → 리스트 반환
for k, v in d.items():    # 리스트 생성
for k, v in d.iteritems(): # 이터레이터 (메모리 효율)

# Python 3: dict.keys(), .values(), .items() → 뷰(view) 반환
for k, v in d.items():    # 뷰 객체 (이터레이터처럼 동작)
# iteritems(), iterkeys(), itervalues() 삭제됨

# 리스트가 필요한 경우 명시적으로 변환
keys_list = list(d.keys())
```

### 3.5 except 구문

```python
# Python 2
try:
    ...
except Exception, e:
    print e

# Python 3
try:
    ...
except Exception as e:
    print(e)
```

### 3.6 기타 주요 변경점

| 항목                  | Python 2                                    | Python 3                                     |
| --------------------- | ------------------------------------------- | -------------------------------------------- |
| `range` / `xrange`    | `range()` → 리스트, `xrange()` → 이터레이터 | `range()` → 이터레이터, `xrange` 삭제        |
| `map` / `filter`      | 리스트 반환                                 | 이터레이터 반환                              |
| `raw_input` / `input` | `raw_input()` → 문자열, `input()` → eval    | `input()` → 문자열, `raw_input` 삭제         |
| `long` 타입           | `int`와 `long` 별도                         | `long` 삭제, `int`로 통합                    |
| `has_key()`           | `d.has_key(k)`                              | `k in d`                                     |
| `reduce`              | 내장 함수                                   | `functools.reduce`                           |
| `StringIO`            | `StringIO.StringIO`                         | `io.StringIO`                                |
| `ConfigParser`        | `ConfigParser.ConfigParser`                 | `configparser.ConfigParser`                  |
| `urllib` / `urllib2`  | 별도 모듈                                   | `urllib.request`, `urllib.parse` 등으로 통합 |

---

## 4. Zeppelin 노트북 변환 워크플로

Zeppelin 노트북은 JSON 형식으로 저장되며, 각 셀(paragraph)에 코드가 포함된다.

### 4.1 변환 절차

```text
1. 노트북 JSON 백업
2. Python 셀 추출
3. 2to3 / futurize 자동 변환
4. 수동 검토 및 수정
5. 변환된 코드를 JSON에 재삽입
6. 변환된 노트북 로드 테스트
```

### 4.2 노트북 Python 셀 추출 및 변환 스크립트

```python
#!/usr/bin/env python3
"""Zeppelin 노트북에서 Python 셀을 추출하여 2to3 변환 후 재삽입하는 스크립트."""

import json
import os
import subprocess
import sys
import tempfile
import shutil

def extract_and_convert_notebook(notebook_path, output_path=None):
    """Zeppelin 노트북의 Python 셀을 Python 3로 변환한다."""
    if output_path is None:
        output_path = notebook_path  # in-place

    with open(notebook_path, 'r', encoding='utf-8') as f:
        notebook = json.load(f)

    paragraphs = notebook.get('paragraphs', [])
    converted_count = 0

    for para in paragraphs:
        text = para.get('text', '')
        # %python 또는 %pyspark로 시작하는 셀 식별
        if text.startswith('%python') or text.startswith('%pyspark'):
            lines = text.split('\n')
            header = lines[0]  # %python 또는 %pyspark 지시자
            code = '\n'.join(lines[1:])

            if not code.strip():
                continue

            # 임시 파일에 코드 저장 후 2to3 실행
            with tempfile.NamedTemporaryFile(
                mode='w', suffix='.py', delete=False, encoding='utf-8'
            ) as tmp:
                tmp.write(code)
                tmp_path = tmp.name

            try:
                subprocess.run(
                    ['2to3', '-w', '-n', tmp_path],
                    capture_output=True, text=True
                )
                with open(tmp_path, 'r', encoding='utf-8') as f:
                    converted_code = f.read()
                para['text'] = header + '\n' + converted_code
                converted_count += 1
            finally:
                os.unlink(tmp_path)

    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump(notebook, f, ensure_ascii=False, indent=2)

    return converted_count


if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python convert_notebooks.py <notebook_dir>")
        sys.exit(1)

    notebook_dir = sys.argv[1]
    total = 0

    for root, dirs, files in os.walk(notebook_dir):
        for fname in files:
            if fname == 'note.json':
                path = os.path.join(root, fname)
                backup_path = path + '.bak'
                shutil.copy2(path, backup_path)
                count = extract_and_convert_notebook(path)
                total += count
                if count > 0:
                    print(f"  변환됨: {path} ({count}개 셀)")

    print(f"\n총 {total}개 Python 셀 변환 완료")
```

### 4.3 변환 실행

```bash
# 1) 노트북 디렉터리 백업
cp -r /usr/lib/zeppelin/notebook/ /usr/lib/zeppelin/notebook-backup-py2/

# 2) 변환 스크립트 실행
python3 convert_notebooks.py /usr/lib/zeppelin/notebook/

# 3) 변환 결과 diff 확인
diff -r /usr/lib/zeppelin/notebook-backup-py2/ /usr/lib/zeppelin/notebook/
```

---

## 5. 외부 패키지 대응

### 5.1 ARM64 휠 미제공 시 소스 빌드

```bash
# 빌드 의존성 설치
sudo yum groupinstall -y "Development Tools"
sudo yum install -y python3-devel gcc-c++ cmake

# 소스에서 패키지 설치
pip3 install --no-binary :all: <패키지명>
```

### 5.2 대체 패키지 검토

| 기존 패키지 (Py2 전용) | 대체 패키지 (Py3 호환)             |
| ---------------------- | ---------------------------------- |
| `MySQL-python`         | `mysqlclient` 또는 `PyMySQL`       |
| `pycrypto`             | `pycryptodome`                     |
| `PIL` (Pillow 이전)    | `Pillow`                           |
| `ujson` (구버전)       | `ujson>=5.0` 또는 `orjson`         |
| `subprocess32`         | Python 3 내장 `subprocess`         |
| `futures`              | Python 3 내장 `concurrent.futures` |

---

## 6. 단위 테스트: 변환된 코드의 결과 검증

```bash
# Python 2 환경에서 기준 출력 생성
python2 test_script.py > expected_output_py2.txt

# Python 3 환경에서 변환된 코드 실행
python3 test_script.py > actual_output_py3.txt

# 결과 비교
diff expected_output_py2.txt actual_output_py3.txt
```

> **핵심 확인 항목**:
>
> - 나눗셈 결과값 (정수 vs float)
> - 문자열 인코딩/디코딩 결과
> - dict 순서 의존 로직 (Python 3.7+에서 삽입 순서 보장)
> - `sorted()`, `map()`, `filter()` 반환 타입 차이
