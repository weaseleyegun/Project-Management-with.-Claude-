# DBMS 방언 차이 — 쿼리 작성 전 확인

> `db-review` skill의 보조 자료. DBMS가 확정된 뒤 해당 열만 보면 된다.

## 자주 틀리는 함수·구문

| 용도 | MariaDB/MySQL | MS-SQL | PostgreSQL |
|---|---|---|---|
| 행 수 제한 | `LIMIT 10` | `SELECT TOP 10` | `LIMIT 10` |
| 문자열 위치 | `INSTR(s, sub)` | `CHARINDEX(sub, s)` | `POSITION(sub IN s)` |
| 문자열 결합 | `CONCAT()` | `+` 또는 `CONCAT()` | `\|\|` 또는 `CONCAT()` |
| NULL 대체 | `IFNULL` / `COALESCE` | `ISNULL` / `COALESCE` | `COALESCE` |
| 현재시각 | `NOW()` | `GETDATE()` | `NOW()` / `CURRENT_TIMESTAMP` |
| 날짜 덧셈 | `DATE_ADD(d, INTERVAL 1 DAY)` | `DATEADD(day, 1, d)` | `d + INTERVAL '1 day'` |
| 날짜 차이 | `TIMESTAMPDIFF(SECOND, a, b)` | `DATEDIFF(second, a, b)` | `EXTRACT(EPOCH FROM (b-a))` |
| 날짜 절삭 | `DATE_FORMAT(d, '%Y-%m-%d %H:00')` | `DATEADD(hour, DATEDIFF(hour,0,d), 0)` | `DATE_TRUNC('hour', d)` |
| 식별자 인용 | `` `col` `` | `[col]` | `"col"` |
| UPSERT | `INSERT ... ON DUPLICATE KEY UPDATE` | `MERGE` | `INSERT ... ON CONFLICT DO UPDATE` |

## 시계열 특이사항

- **PostgreSQL + TimescaleDB**: 대용량 시계열 테이블은 `create_hypertable()`로 전환. 시간 컬럼 기준 청크 분할. 집계는 `time_bucket()` 사용. 연속 집계는 continuous aggregate.
- **MariaDB**: 파티셔닝(RANGE by 시간)을 쓰지 않으면 수천만 행부터 급격히 느려진다. 시간 컬럼 인덱스 필수.
- **MS-SQL**: 파티션 함수/스킴 구성 여부 확인. `DATETIME2` vs `DATETIME` 정밀도 차이 주의.
- **MongoDB**: 시계열 컬렉션(`timeseries` 옵션) 사용 여부 확인. 인덱스는 `{설비: 1, 시각: -1}` 복합 순서가 중요.

## 백업 명령

```bash
mysqldump -u USER -p --single-transaction DB TABLE > dump.sql      # MariaDB/MySQL
pg_dump -U USER -t TABLE DB > dump.sql                              # PostgreSQL
mongodump --db DB --collection COLL --out ./dump                    # MongoDB
```
```sql
BACKUP DATABASE [DB] TO DISK = 'D:\backup\db.bak' WITH INIT;        -- MS-SQL
```

## 운영 DB 조회 시 부하 주의

- `SELECT *` + 정렬 없는 전체 스캔은 수집 프로세스와 락 경합을 일으킬 수 있다.
- 장시간 쿼리는 먼저 `EXPLAIN`(MS-SQL은 실행 계획)으로 확인한다.
- 가능하면 읽기 전용 복제본(replica)이나 야간 시간대를 요청한다.
