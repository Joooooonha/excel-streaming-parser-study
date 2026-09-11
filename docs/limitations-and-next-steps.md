# 한계와 후속 검증

## 1. 문서의 주장 범위

이 저장소는 인턴십 중 수행한 로컬 파서 실험과 설계 검토를 공개 가능한 형태로 재구성한 기술 사례입니다.

### 말할 수 있는 내용

- 제한된 heap에서 Workbook 전체 로딩이 큰 live set을 만들 수 있음
- 합성 50,000행 입력이 Workbook 방식에서 `-Xmx512m`, `-Xmx768m`으로 OOM 발생
- 같은 입력의 SAX 스트리밍 실행이 `-Xmx512m`에서 완료
- 원본 `byte[]`보다 파싱 객체와 전체 결과가 더 큰 위험으로 관측됨
- 출력까지 스트리밍하고 전체 parser 동시성을 제한해야 한다는 설계 근거 확보

### 말할 수 없는 내용

- 운영 시스템에 SAX 구조를 배포했다
- 장애가 특정 비율로 감소했다
- 처리량이 특정 비율로 증가했다
- AWS 비용이 특정 비율로 절감됐다
- 50,000행이 안전한 운영 상한이다
- 모든 `.xlsx`에서 Workbook과 동일한 결과를 만든다

## 2. 실험 데이터 한계

### 합성 입력

실험은 약 20개 컬럼을 가진 합성 데이터에 기반합니다. 실제 파일은 다음 특성이 다를 수 있습니다.

```text
wide rows
long text
high-cardinality shared strings
many styles
formulas
merged regions
multiple sheets
sparse cells
embedded objects
```

따라서 행 수나 압축 크기 하나만으로 안전 범위를 일반화할 수 없습니다.

### 단일 실행

각 수치는 한 번의 실행 기록입니다.

```text
available: observed single-run value
not available: mean, median, p95, p99, confidence interval
```

JIT warm-up, OS page cache, GC 타이밍과 백그라운드 프로세스가 결과에 영향을 줄 수 있습니다.

### 측정 지점

Workbook의 메모리는 코드 체크포인트의 used heap 차이입니다. SAX 값은 GC 이벤트 전후 값입니다. 두 값 모두 실행 전체의 정확한 peak heap이나 max RSS가 아닙니다.

## 3. 비교 공정성

Workbook과 SAX는 출력 방식도 달랐습니다.

```text
Workbook: full object model + full result + JSON bytes
SAX:      row events + streaming output + temp file
```

현재 결과는 **전체 설계 대안의 비교**에는 의미가 있지만, 순수하게 parser API 하나만의 차이라고 말할 수 없습니다.

후속 실험은 세 조건을 분리해야 합니다.

1. Workbook + full result
2. SAX + full result
3. SAX + streaming result

## 4. 기능 정합성

SAX는 저수준 이벤트를 직접 처리하므로 Workbook API가 대신 처리하던 규칙을 구현해야 합니다.

| 항목 | 필요한 검증 |
|---|---|
| 빈 중간 셀 | column reference 기준 빈 문자열 보완 |
| shared strings | index lookup의 정확성 |
| 숫자·날짜 | style과 format 적용 |
| 수식 | cached value 또는 미지원 정책 |
| 병합 셀 | 대표값 복제 여부 결정 |
| 여러 sheet | 첫 sheet 또는 전체 sheet 정책 |
| header | 첫 행, 지정 행 또는 schema 정책 |
| 매우 긴 문자열 | 출력 크기와 제한 처리 |

출력 정합성을 다음 형태로 비교해야 합니다.

```java
String workbookHash = semanticHash(workbookResult);
String saxHash = semanticHash(saxResult);

assertThat(saxHash).isEqualTo(workbookHash);
```

단순 파일 byte hash는 JSON whitespace나 key order 차이에 영향을 받으므로 정규화한 논리 구조의 hash를 사용해야 합니다.

## 5. 운영 환경

로컬 CLI는 다음 비용을 제외했습니다.

- HTTP multipart buffering
- object storage upload와 download
- network bandwidth와 latency
- DB connection pool
- PostgreSQL JSONB 변환·저장
- 운영 서비스의 다른 요청
- 모니터링 agent
- container memory limit

운영 결론을 내리려면 end-to-end worker를 격리된 staging 환경에서 측정해야 합니다.

## 6. 장애 처리

아직 검증하지 않은 장애:

```text
object download timeout
corrupted xlsx
encrypted workbook
zip bomb
disk full
temp file write failure
worker process termination
DB timeout after parse success
duplicate message delivery
retry storm
```

특히 파싱은 성공했지만 최종 저장이 실패하면 `DONE`으로 전환하지 않아야 합니다.

## 7. 저장 방식

SAX로 parser heap을 낮춰도 최종 JSON을 JSONB로 저장하는 과정에서 driver가 전체 내용을 메모리에 올릴 수 있습니다. 다음을 비교해야 합니다.

| 후보 | 측정 내용 |
|---|---|
| JSONB | driver heap, insert time, row size, 조회 time |
| object storage JSON | upload time, download time, URL·권한 관리 |
| row batch | batch size, transaction time, index cost |

## 8. S3 비용 해석

inline 경로는 처리 한 건당 GET 요청 한 번을 줄일 수 있지만 비용 절감률은 다음 정보 없이는 계산할 수 없습니다.

```text
request pricing
region
actual upload count
inline hit ratio
GET/PUT request mix
data transfer
storage duration
other service usage
```

따라서 문서에서는 “GET 요청 한 번 생략 가능”까지만 말하고 “AWS 비용 50% 절감”처럼 표현하지 않습니다.

## 9. 운영 상한 결정

지원 범위는 서버가 간신히 버틴 최대값이 아닙니다.

```text
safe limit
= repeated success boundary
- concurrent request budget
- JVM non-heap budget
- OS/process reserve
- failure recovery reserve
```

압축 크기 외에도 rows, cells, uncompressed size와 output size를 함께 제한해야 합니다.

## 10. 후속 실험 계획

### 1단계: 재현 환경 고정

- [ ] JDK와 Apache POI 버전 고정
- [ ] 머신 또는 container 사양 기록
- [ ] `-Xms256m -Xmx512m` 기본 조건 고정
- [ ] 합성 샘플 생성 규칙 공개
- [ ] warm-up과 측정 실행 분리

### 2단계: 출력 정합성

- [ ] header·빈 셀·문자열 비교
- [ ] 숫자·날짜·수식 비교
- [ ] 병합 셀 정책 검증
- [ ] 여러 sheet 정책 검증
- [ ] semantic hash 일치 확인

### 3단계: 단일 worker 성능

- [ ] 조건별 최소 5회 이상 반복
- [ ] parse/write/total time 분리
- [ ] JFR allocation과 peak used heap
- [ ] max RSS
- [ ] GC count·total pause·max pause

### 4단계: 동시 처리

- [ ] worker 1 기준 확인
- [ ] 안전 후보만 worker 2 실행
- [ ] 완료·실패 건수
- [ ] throughput과 p50/p95 latency
- [ ] queue wait time과 backpressure

### 5단계: end-to-end

- [ ] object storage GET 포함
- [ ] final persistence 포함
- [ ] 결과 크기 상한 확인
- [ ] timeout·retry·duplicate 검증
- [ ] 프로세스 재시작 복구

### 6단계: 정책 확정

```yaml
supported_file_types: [.xlsx]
max_compressed_bytes: TBD
max_uncompressed_bytes: TBD
max_rows: TBD
max_cells: TBD
max_output_bytes: TBD
worker_count: TBD
queue_capacity: TBD
retry_limit: TBD
result_storage: TBD
```

## 11. 공개 재현 코드

현재 저장소의 수치는 인턴십 당시 기록입니다. 새 벤치마크를 구현할 경우 다음 디렉터리와 커밋에서 별도 관리합니다.

```text
benchmark/
├── sample-generator/
├── workbook-parser/
├── sax-streaming-parser/
└── runner/
```

새로 얻은 결과는 기존 결과를 덮어쓰지 않고 아래처럼 분리합니다.

```text
results/
├── internship-records/
└── public-reproduction-YYYY-MM-DD/
```

이를 통해 당시 수행한 업무와 이후 공개를 위해 추가한 검증을 혼동하지 않도록 합니다.
