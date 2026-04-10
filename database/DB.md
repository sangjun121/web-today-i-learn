## 문제 1: 테이블 생성하기 (CREATE TABLE)   
**1. attendance 테이블은 중복된 데이터가 쌓이는 구조이다. 중복된 데이터는 어떤 컬럼인가?**
- 크루 정보인 crew_id와 nickname이 중복된다.

**2. attendance 테이블에서 중복을 제거하기 위해 crew 테이블을 만들려고 한다. 어떻게 구성해 볼 수 있을까?**
- 이 중복 정보를 제거하기 위해, 크루 정보를 별도의 crew 테이블로 분리할 수 있다. crew 테이블은 crew_id를 기본키로 갖고 크루정보인 nickname을 별도로 저장하도록 구성한다.

**3. crew 테이블에 들어가야 할 크루들의 정보는 어떻게 추출할까? (hint: DISTINCT)**
- SELECT DISTINCT crew_id, nickname FROM attendance로 중복 없이 추출할 수 있다. 해당 정보가 중복이므로, 테이블 정규화를 위해 신규 테이블인 crew를 추가한다.

## 문제 2: 테이블 컬럼 삭제하기 (ALTER TABLE)
**1. crew 테이블을 만들고 중복을 제거했다. attendance에서 불필요해지는 컬럼은?**
- nickname을 제거하고, crew_id를 외래키로 crew테이블과 연관관계를 맺을 필요가 있다.

**2. 컬럼을 삭제하려면 어떻게 해야 하는가?**
```sql
ALTER TABLE attendance
DROP COLUMN nickname;
```
해당 쿼리문을 사용하여 attendance 내부의 컬럼을 제거한다.

## 문제 3: 외래키 설정하기
- 현 문제상황: 만약에 crew 테이블에는 crew_id가 12번인 크루가 존재하지 않지만, attendance 테이블에는 여전히 crew_id가 12번인 크루가 존재한다면? 
해당 문제는 crew 테이블과 attendance의 연관 관계가 없기 때매 발생한 무결성 문제이다. 출석 기록은 있는데 정작 crew에 없는, 이런 논리적으로 존재하면 안되는 상황이 발생한다.
이를 참조 무결성이 보장되지 않은 상황이라고 한다. 이에 따라 attendance.crew_id가 반드시 crew.id를 참조하도록 외래키를 설정해야한다.
```sql
ALTER TABLE attendance
ADD CONSTRAINT fk_attendance_crew
FOREIGN KEY (crew_id)
REFERENCES crew(crew_id);
```
이와 같이, 제약조건을 걸어 두개의 컬럼사이의 참조를 정의할 수 있다.

## 문제 4: 유니크 키 설정
- 우아한테크코스에서는 닉네임 중복이 금지된다. 이런 경우, 각 크루들의 닉네임을 저장하는 nickname 컬럼의 데이터가 고유하도록 보장해야한다. 이를 UNIQUE key를 해당 컬럼에 제약조건을 추가함을써 해결할 수 있다.
```sql
ALTER TABLE crew
ADD CONSTRAINT uk_crew_nickname UNIQUE (nickname);
```

## 문제 5: 크루 닉네임 검색하기 (LIKE)
이름의 패턴으로 데이터를 찾는 방식을 지원한다. 첫 글자가 디로 시작하는 경우 LIKE 키워드를 통해 패턴에 부합하는 데이터를 조회할 수 있다. 
```sql
SELECT nickname
FROM crew
WHERE nickname LIKE '디%';
```

## 문제 6: 출석 기록 확인하기 (SELECT + WHERE)
어셔의 출석 기록을 먼저 확인할 필요가 있다. 이때 crew 테이블에서 어셔의 nickname을 기반으로 attendance 테이블의 출석기록을 조회해야한다.
따라서, JOIN을 사용하여 해당 테이블을 합쳐 조회할 수 있다.
```sql
SELECT a.*
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE c.nickname = '어셔'
  AND a.attendance_date = '2026-03-06';
```
해당 join은 기본값인 inner join이다. 따라서, 각 테이블 모두에 있는 crew_id의 행만 합쳐질 것이다. 이 상태에서 어셔의 닉네임과 3/6일을 기반으로 데이터를 조회한다.

## 문제 7: 누락된 출석 기록 추가 (INSERT)
확인해 보니, 어셔는 그날 출석 체크를 하지 못한 것이 사실로 드러났다. 사후 처리를 위해 출석을 추가해야 하는데 어떻게 추가해야 할까?

- 해당 출석을 추가해주기 위해, 조회와 마찬가지로 어셔의 닉네임을 기반으로 출석 기록을 추가해줘야 한다. 따라서 먼저 어셔의 id를 먼저 조회하고 이를 기반으로 출석 테이블에 기록을 추가해주면 된다.
```sql
INSERT INTO attendance (crew_id, attendance_date, start_time, end_time)
SELECT crew_id, '2026-03-06', '09:31', '18:01'
FROM crew
WHERE nickname = '어셔';
```

## 문제 8: 잘못된 출석 기록 수정 (UPDATE)
주니는 3월 12일 10시 정각에 캠퍼스에 도착했지만, 등교 버튼을 누르는 것을 깜빡하고 데일리 미팅에 참여했다. 뒤늦게야 알게 됐는데 시각은 10시 5분... 지각 처리가 되는 시점이었다.
- 기존에 이미 추가된 행에 대한 수정은 update 메서드를 사용할 수 있다. 마찬가지로 닉네임을 기반으로 크루 id를 조회하고 해당 id를 기반으로 attendance 테이블을 업데이트한다. 이번에는 join을 사용하여 업데이트 한다.
```sql
UPDATE attendance a
    JOIN crew c ON a.crew_id = c.crew_id
    SET a.start_time = '10:00'
    WHERE c.nickname = '주니'
    AND a.attendance_date = '2025-03-12';
```

## 문제 9: 허위 출석 기록 삭제 (DELETE)
마찬가지로 join을 사용하여, 해당 행을 찾아내고 삭제를 진행할 수 있다.
```sql
DELETE a
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE c.nickname = '아론'
  AND a.attendance_date = '2025-03-12';
```

## 문제 10: 출석 정보 조회하기 (JOIN)
- join으로 두 테이블을 합친다면, 수동으로 id를 조회하고 입력할 필요가 없다.
```sql
SELECT c.nickname, a.attendance_date, a.start_time, a.end_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id;
```

## 문제 11: nickname으로 쿼리 처리하기 (서브 쿼리)
- 앞서 살펴본, join이 아닌 2개의 쿼리로 crew_id를 찾고, attendance 테이블을 조회하는 방식이다. 이때 서브쿼리를 활용하여 해결할 수 있다.
```sql
SELECT *
FROM attendance
WHERE crew_id = (
  SELECT crew_id
  FROM crew
  WHERE nickname = '검프'
);
```

## 문제 12: 가장 늦게 하교한 크루 찾기
- endtime이 제일 큰 사람을 찾기 위해 order by를 이용하여 정렬하고 원하는 값을 찾을 수 있다.
```sql
SELECT c.nickname, a.end_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE a.attendance_date = '2025-03-05'
ORDER BY a.end_time DESC
LIMIT 1;
```

## 문제 13: 크루별로 '기록된' 날짜 수 조회
- groupby로 크루들을 묶고, 각 그룹의 날짜 개수를 카운트한다.
```sql
SELECT c.nickname, COUNT(a.attendance_date) AS days
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY c.nickname;
```

## 문제 14: 크루별로 등교 기록이 있는(start_time IS NOT NULL) 날짜 수 조회
```sql
SELECT c.nickname, COUNT(a.start_time) AS days
FROM attendance a
    JOIN crew c ON a.crew_id = c.crew_id
GROUP BY c.nickname;
```
## 문제 15: 날짜별로 등교한 크루 수 조회
- 날짜별로 groupby를 진행하면 된다.
```sql
SELECT a.attendance_date, COUNT(a.start_time) AS crew_count
FROM attendance a
GROUP BY a.attendance_date;
```

## 문제 16: 크루별 가장 빠른 등교 시각(MIN)과 가장 늦은 등교 시각(MAX)
- 마찬가지로 크루별로 grouping하고 연산 결과 출력시 min, max함수를 사용하여 조회를 진행한다.
```sql
SELECT c.nickname,
       MIN(a.start_time) AS earliest_time,
       MAX(a.start_time) AS latest_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY c.nickname;
```
