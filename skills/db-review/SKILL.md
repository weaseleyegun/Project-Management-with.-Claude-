---
name: db-review
description: 고객사/현장 DB 작업 — 스키마 문서 검증, SQL 쿼리 작성·리뷰, 테이블·컬럼 명명 규칙, 스키마 변경 마이그레이션 절차. "이 쿼리 리뷰해줘", "이 테이블이 정말 대상 라인 데이터인지 검증해줘", "테이블명 바꿔야 해 절차 만들어줘", "인덱스 문제 있나 봐줘" 같은 DB·SQL 요청에 사용한다. MariaDB / MS-SQL / PostgreSQL+TimescaleDB / MongoDB 대상.
---

# Database · SQL

## 접근 원칙

```
- 고객사/현장 DB는 원칙적으로 읽기전용(SELECT)만.
- DDL/DML(스키마 변경, 데이터 수정·삭제)은 명시적 승인 없이 절대 실행 금지.
- 접속정보는 .env 또는 PROJECT.md 에만. 대화·산출물에 반복 노출 금지.
```

## 응답 전 필수 확인

불명확하면 **쿼리를 쓰기 전에 먼저 묻는다.**

```
1. DBMS 종류와 버전
2. 대상 테이블·컬렉션 스키마
3. 데이터 규모 (행 수, 일 증가량)
4. 운영 DB인가 개발 DB인가
```

> MariaDB / MS-SQL / PostgreSQL은 `CHARINDEX` vs `POSITION` vs `INSTR`, `TOP` vs `LIMIT`, 날짜 함수가 전부 다르다.
> **DBMS 미확인 상태로 쿼리를 작성하지 않는다.** 방언 차이는 `references/dbms-notes.md` 참조.

## 스키마 검증 (원칙 7 — 문서 불신)

공급업체 제공 문서(Data Definition Sheet 등)를 그대로 신뢰하지 않는다.

```sql
-- 최소 검증 세트
SELECT COUNT(*) FROM {테이블};
SELECT MIN({시간컬럼}), MAX({시간컬럼}) FROM {테이블};
SELECT DISTINCT {라인/설비 식별컬럼} FROM {테이블};   -- ← 대상 라인이 맞는지
SELECT * FROM {테이블} ORDER BY {시간컬럼} DESC LIMIT 10;
```

**대상 라인 식별 검증은 반드시 수행한다.** 문서상 대상 라인 테이블이 실제로는 다른 라인 데이터였던 사례가 있다.

## 명명 규칙 (프로젝트 규약이 있으면 그쪽 우선)

```
테이블/컬렉션 : 대문자 SNAKE_CASE — EL01_MAIN_CHILLER
컬럼/필드     : 대문자 SNAKE_CASE — PROC_QNTY, IN_DATE
설비 접두어   : MAIN_CHILLER01_TEMP
시계열        : 시간 컬럼 인덱스 필수 (TimescaleDB는 hypertable)
```

## 쿼리 리뷰 체크리스트

1. **정합성** — JOIN 키 유일성, 중복 증폭(fan-out)
2. **NULL** — `COALESCE` 누락, `NOT IN` + NULL 함정
3. **시간 경계** — `>= 시작 AND < 종료` (`BETWEEN`은 종료 포함 주의)
4. **집계 범위** — `GROUP BY` ↔ `SELECT` 불일치
5. **윈도우 함수** — `PARTITION BY` 기준이 의도와 맞는가
6. **성능** — 인덱스 활용, `함수(컬럼) = 값` 풀스캔 패턴
7. **가독성** — CTE 분리 제안

리뷰 결과는 항목 번호를 매겨 표로 낸다(원칙 3).

## 마이그레이션 절차 (필수 6단계)

```
1. 백업 명령 (mysqldump / pg_dump / BACKUP DATABASE / mongodump)
2. 영향 범위 조사 (참조 뷰·프로시저·수집 프로세스)
3. 수집/적재 프로세스 중지 필요 여부 명시
4. 트랜잭션으로 감싼 변경 스크립트
5. 롤백 스크립트
6. 검증 쿼리 (변경 전후 건수·값 비교)
```

이름 교환은 충돌 방지를 위해 **임시명 경유 3단계**:

```sql
ALTER TABLE A RENAME TO A_TMP;
ALTER TABLE B RENAME TO A;
ALTER TABLE A_TMP RENAME TO B;
```

## 실행 환경 함정 (Windows / 한글 경로)

```
- 대괄호([...]) 포함 경로는 PowerShell에서 반드시 -LiteralPath 사용
- 큰 인라인 스크립트를 python -c 로 실행하지 말 것 (따옴표 파싱으로 멈춤).
  .py 파일로 저장 후 실행
- 콘솔 출력이 cp949 로 깨질 수 있음 → print 대신 UTF-8 텍스트 파일로 덤프한 뒤 읽기
- 조회 스크립트(.py/.sql)와 결과는 _scratch/ 에 보관 (재검증 근거)
```
