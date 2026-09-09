# 25장: 실시간 게임 리더보드

## 소개

온라인 모바일 게임의 **리더보드(leaderboard)**를 설계합니다.

<div style="margin-left:3rem">
    <img src="./images/leaderboard.png" alt="리더보드" width="500" />
</div>

---

## 1단계: 문제 이해 및 설계 범위 설정

- C: 리더보드 점수는 어떻게 계산하는가?
- I: 사용자가 경기에서 이길 때마다 1점을 얻는다.
- C: 모든 플레이어가 리더보드에 포함되는가?
- I: 그렇다.
- C: 리더보드는 특정 기간 단위로 초기화되는가?
- I: 매월 새 토너먼트가 시작되고 새 리더보드가 생성된다.
- C: 상위 10명만 중요하다고 가정해도 되는가?
- I: 상위 10명을 보여주고 특정 사용자의 순위도 표시해야 한다. 시간이 허용되면 해당 사용자 위아래의 플레이어도 보여주는 기능을 논의한다.
- C: 토너먼트 사용자는 몇 명인가?
- I: DAU 500만 명, MAU 2,500만 명이다.
- C: 토너먼트에서 평균적으로 몇 경기를 플레이하는가?
- I: 플레이어 한 명이 하루 평균 10경기를 한다.
- C: 두 플레이어의 점수가 같으면 순위는 어떻게 정하는가?
- I: 같은 점수면 같은 순위로 처리한다. 시간이 허용되면 동점 처리 방식도 논의한다.
- C: 리더보드는 실시간이어야 하는가?
- I: 그렇다. 실시간 또는 가능한 한 실시간에 가까운 결과를 보여줘야 하며 배치된 과거 결과만 보여주는 것은 허용하지 않는다.

### **기능 요구사항**

- 리더보드 상위 10명 표시
- 특정 사용자의 순위 표시
- 특정 사용자 기준 위 4명과 아래 4명 표시(추가 기능)

### **비기능 요구사항**

- 점수의 실시간 갱신
- 점수 변경이 리더보드에 가능한 한 즉시 반영되어야 함
- 일반적인 확장성, 가용성, 신뢰성 요구사항

### **개략적 규모 추정**

원문에서는 피크 부하를 설명하기 위해 초당 온라인 사용자 수와 경기 수를 추정합니다. 사용자 분포가 하루 동안 균등하지 않으므로 평균보다 높은 피크를 고려해야 합니다.

사용자가 하루 평균 10경기를 하고 승리 시 점수가 갱신된다고 가정하면 점수 갱신 QPS는 수백~수천 수준의 피크가 발생할 수 있습니다. 원문 예시에서는 평균 500 QPS, 피크 2,500 QPS를 사용합니다.

사용자가 하루 평균 한 번 리더보드를 조회한다고 가정하면 상위 10명 조회 QPS는 점수 갱신보다 훨씬 낮습니다.

---

## 2단계: 상위 수준 설계 제안 및 합의

### **API 설계**

사용자 점수를 갱신하는 API가 필요합니다.

```
POST /v1/scores
```

이 API는 `user_id`와 경기 승리로 획득한 `points`를 받습니다.

점수 조작을 방지하기 위해 최종 클라이언트가 아니라 신뢰할 수 있는 게임 서버만 호출할 수 있어야 합니다.

리더보드 상위 10명을 조회하는 API는 다음과 같습니다.

```
GET /v1/scores
```

응답 예시:

```
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
```

특정 사용자의 점수와 순위도 조회할 수 있습니다.

```
GET /v1/scores/{:user_id}
```

응답 예시:

```
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
```

### **상위 수준 아키텍처**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="상위 수준 아키텍처" width="500" />
</div>

- 플레이어가 경기에서 이기면 클라이언트가 게임 서비스에 요청을 보냅니다.
- 게임 서비스가 승리 결과가 유효한지 검증하고 리더보드 서비스에 점수 갱신을 요청합니다.
- 리더보드 서비스가 리더보드 저장소의 사용자 점수를 갱신합니다.
- 플레이어가 리더보드 서비스에 요청해 상위 10명이나 자신의 순위를 가져옵니다.

클라이언트가 리더보드 서비스에 직접 점수를 기록하는 대안도 생각할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/alternative-design.png" alt="대안 설계" width="500" />
</div>

하지만 클라이언트가 점수를 직접 제출하도록 하면 조작된 요청을 보내기 쉬워 보안상 적합하지 않습니다. 점수의 진실 공급원(source of truth)은 게임 서버의 검증된 경기 결과여야 합니다.

게임 로직을 서버에서 완전히 관리하는 게임이라면 클라이언트가 승리 기록 API를 명시적으로 호출할 필요도 없습니다. 서버가 게임 상태를 기반으로 결과를 판단하고 자동으로 리더보드를 갱신할 수 있습니다.

게임 서버와 리더보드 서비스 사이에 메시지 큐를 넣는 것도 고려할 수 있습니다. 경기 결과를 업적, 분석, 알림 등 다른 서비스도 사용해야 한다면 이벤트 기반 구조가 유용합니다. 현재 요구사항에는 필수적이지 않으므로 기본 설계에는 포함하지 않습니다.

<div style="margin-left:3rem">
    <img src="./images/message-queue-based-comm.png" alt="메시지 큐 기반 통신" width="500" />
</div>

### **데이터 모델**

리더보드 저장 방식으로 관계형 DB, Redis, NoSQL을 비교합니다.

NoSQL 방식은 상세 설계에서 다룹니다.

#### 관계형 데이터베이스 방식

사용자 수가 적고 규모 요구사항이 크지 않다면 관계형 데이터베이스로도 충분히 구현할 수 있습니다.

단순한 리더보드 테이블에서 시작할 수 있습니다. 원문은 월별 테이블을 예시로 들지만 실제 구현에서는 `month`나 `tournament_id` 같은 컬럼으로 여러 시즌을 하나의 스키마에서 관리할 수도 있습니다.

<div style="margin-left:3rem">
    <img src="./images/leaderboard-table.png" alt="리더보드 테이블" width="500" />
</div>

리더보드 핵심 쿼리와 관계없는 부가 데이터는 생략합니다.

사용자가 1점을 얻으면 다음처럼 처리할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/user-wins-point.png" alt="사용자 점수 획득" width="500" />
</div>

테이블에 사용자가 없다면 먼저 삽입합니다.

```
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

이후에는 점수를 증가시킵니다.

```
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

상위 플레이어를 찾는 기본 방식은 다음과 같습니다.

<div style="margin-left:3rem">
    <img src="./images/find-leaderboard-position.png" alt="리더보드 순위 조회" width="500" />
</div>

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

모든 레코드를 정렬해야 하므로 대규모 데이터에서는 비효율적입니다.

`score`에 인덱스를 만들고 `LIMIT`을 사용하면 상위 N명 조회는 개선할 수 있습니다.

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

하지만 사용자가 상위권이 아닐 때 특정 사용자의 전체 순위를 계산하는 것은 여전히 비용이 큽니다.

#### Redis 방식

수백만 명의 플레이어에서도 복잡한 SQL 없이 빠르게 동작하는 저장 구조가 필요합니다.

Redis는 인메모리 데이터 저장소이며 이 요구사항에 적합한 **Sorted Set** 자료구조를 제공합니다.

Sorted Set은 각 멤버와 점수(score)를 저장하고 점수 기준으로 정렬된 상태를 유지합니다.
내부적으로 멤버와 점수 매핑을 위한 해시 구조와 점수 순서를 위한 Skip List 등의 구조를 조합해 구현할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/sorted-set.png" alt="Sorted Set" width="500" />
</div>

Skip List는 어떻게 동작할까요?
- 정렬된 연결 리스트를 기반으로 빠른 검색을 위한 여러 수준의 인덱스를 둡니다.
- 상위 레벨 포인터를 이용해 많은 노드를 건너뛰면서 탐색할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/skip-list.png" alt="Skip List" width="500" />
</div>

데이터가 커질수록 기본 연결 리스트를 순차 탐색하는 것보다 훨씬 적은 노드 방문으로 값을 찾을 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/skip-list-performance.png" alt="Skip List 성능" width="500" />
</div>

Sorted Set은 데이터를 항상 정렬 상태로 유지하며 추가와 탐색에 일반적으로 `O(logN)` 수준의 비용을 사용합니다.

반대로 관계형 DB에서 특정 사용자의 순위를 단순 쿼리로 계산하면 다음처럼 더 높은 점수를 가진 행의 수를 세는 중첩 쿼리가 필요할 수 있습니다.

```
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
```

Redis 리더보드에 필요한 주요 연산은 다음과 같습니다.
- **ZADD:** 사용자가 없으면 추가하고 있으면 점수를 갱신합니다. 일반적으로 `O(logN)`.
- **ZINCRBY:** 사용자의 점수를 지정한 값만큼 증가시킵니다. 사용자가 없으면 0에서 시작합니다. 일반적으로 `O(logN)`.
- **ZRANGE/ZREVRANGE:** 점수 순서로 사용자 범위를 가져옵니다. 정렬 방향, 범위, 결과 개수를 지정할 수 있습니다.
- **ZRANK/ZREVRANK:** 특정 사용자의 오름차순/내림차순 순위를 조회합니다.

사용자가 1점을 획득하면 다음과 같이 실행할 수 있습니다.

```
ZINCRBY leaderboard_feb_2021 1 'mary1934'
```

매월 새 리더보드를 생성하고 지난 리더보드는 이력 저장소로 이동할 수 있습니다.

상위 10명을 조회하는 예:

```
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

응답 예:

```
[(user2,score2),(user1,score1),(user5,score5)...]
```

특정 사용자 주변 순위를 가져오는 경우:

<div style="margin-left:3rem">
    <img src="./images/leaderboard-position-of-user.png" alt="사용자 리더보드 위치" width="500" />
</div>

사용자의 순위를 알고 있다면 해당 순위 주변 범위를 조회할 수 있습니다.

```
ZREVRANGE leaderboard_feb_2021 357 365
```

사용자 순위는 `ZREVRANK <user-id>`로 가져올 수 있습니다.

저장 공간을 추정해 봅니다.
- 한 달 동안 최악의 경우 MAU 2,500만 명이 모두 참여한다고 가정합니다.
- 사용자 ID가 24바이트 문자열이고 점수가 16비트 정수라면 최소 데이터 크기는 대략 26바이트 × 2,500만 ≈ 650MB입니다.
- Sorted Set 내부 자료구조 오버헤드를 고려해 실제 메모리는 더 많이 필요하지만 현대적인 Redis 클러스터에서 충분히 관리 가능한 규모입니다.

초당 2,500건의 점수 갱신도 단일 Redis 서버의 일반적인 처리 능력 범위 안에 들어갈 수 있지만 실제 용량 계획은 명령 종류와 하드웨어를 기준으로 벤치마킹해야 합니다.

추가 고려 사항:
- Redis 서버 장애 시 데이터 손실을 줄이기 위해 복제본을 구성합니다.
- Redis Persistence(RDB/AOF 등)를 사용해 장애 복구에 활용할 수 있습니다.
- 사용자 이름, 표시 이름 등 프로필 정보와 경기 승리 이력을 저장하기 위해 MySQL 같은 별도 저장소를 사용할 수 있습니다.
- 경기 결과 이력은 대규모 장애 후 리더보드를 재구성하는 원본 데이터로 활용할 수 있습니다.
- 상위 10명 사용자 프로필은 자주 조회되므로 캐시할 수 있습니다.

---

## 3단계: 상세 설계

### **클라우드 사업자를 사용할 것인가?**

서비스를 직접 배포하고 운영할 수도 있고 관리형 클라우드 서비스를 사용할 수도 있습니다.

직접 운영한다면 Redis에 리더보드 데이터를 저장하고 MySQL에 사용자 프로필을 저장하며 필요하면 사용자 프로필 캐시를 추가할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/manage-services-ourselves.png" alt="직접 서비스 운영" width="500" />
</div>

대안으로 AWS 같은 클라우드의 관리형 서비스를 사용할 수 있습니다. 예를 들어 AWS API Gateway가 API 요청을 AWS Lambda 함수에 라우팅하도록 구성할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/api-gateway-mapping.png" alt="API Gateway 매핑" width="500" />
</div>

AWS Lambda 같은 서버리스 컴퓨팅은 서버를 직접 프로비저닝하거나 관리하지 않고 코드를 실행할 수 있으며 요청량에 따라 실행 인스턴스를 자동으로 확장할 수 있습니다.

사용자가 점수를 얻는 흐름 예시:

<div style="margin-left:3rem">
    <img src="./images/user-scoring-point-lambda.png" alt="Lambda 점수 갱신" width="500" />
</div>

리더보드 조회 예시:

<div style="margin-left:3rem">
    <img src="./images/user-retrieve-leaderboard.png" alt="리더보드 조회" width="500" />
</div>

서버리스 아키텍처를 사용하면 인프라 프로비저닝과 일부 확장 작업을 클라우드 사업자에게 맡길 수 있습니다. 새 게임을 처음부터 만든다면 운영 부담과 비용 모델을 비교해 선택할 수 있습니다.

### **Redis 확장**

DAU 500만 명 수준에서는 저장 공간과 QPS 측면에서 단일 Redis 인스턴스 또는 작은 클러스터로 시작할 수 있습니다.

사용자 수가 10배 이상 증가해 저장 공간과 QPS가 크게 늘어나면 샤딩이 필요할 수 있습니다.

한 가지 방법은 점수 범위를 기준으로 데이터를 나누는 범위 파티셔닝입니다.

<div style="margin-left:3rem">
    <img src="./images/range-partition.png" alt="범위 파티셔닝" width="500" />
</div>

이 예에서는 사용자의 점수에 따라 샤드를 결정합니다. `user_id`와 샤드 간 매핑은 애플리케이션이나 별도 MySQL/캐시에 저장할 수 있습니다.

상위 10명을 조회할 때 가장 높은 점수 범위 샤드부터 조회합니다.

특정 사용자의 전체 순위는 사용자 자신의 샤드 안의 순위를 계산하고, 그보다 높은 점수 범위 샤드에 있는 전체 사용자 수를 더해 계산할 수 있습니다.

대안은 Redis Cluster를 사용해 해시 기반으로 데이터를 분산하는 것입니다.

<div style="margin-left:3rem">
    <img src="./images/hash-partition.png" alt="해시 파티셔닝" width="500" />
</div>

이 구성에서는 상위 10명 계산이 더 복잡해집니다. 각 샤드에서 상위 후보를 가져온 뒤 애플리케이션에서 병합해야 합니다.

<div style="margin-left:3rem">
    <img src="./images/top-10-players-calculation.png" alt="상위 10명 계산" width="500" />
</div>

해시 파티셔닝의 한계는 다음과 같습니다.
- 상위 K에서 K가 크면 모든 샤드에서 많은 데이터를 가져와야 하므로 지연 시간이 커집니다.
- 파티션 수가 늘어날수록 scatter-gather 비용이 증가합니다.
- 특정 사용자의 전역 순위를 바로 계산하기 어렵습니다.

이 문제에서는 점수 범위가 비교적 명확하다면 범위 기반 고정 파티션을 고려할 수 있습니다.

추가 고려 사항:
- 쓰기 중심 Redis 노드는 스냅샷이나 fork 시점의 메모리 사용을 고려해 여유 메모리를 확보해야 합니다.
- `redis-benchmark` 같은 도구로 실제 워크로드와 유사한 조건에서 성능을 측정해 용량을 결정할 수 있습니다.

### **대안: NoSQL**

다음 특성에 최적화된 NoSQL 데이터베이스도 고려할 수 있습니다.
- 높은 쓰기 처리량
- 같은 파티션 안에서 점수 기준 정렬을 효율적으로 수행

DynamoDB, Cassandra, MongoDB 등이 후보가 될 수 있습니다.

이 장에서는 DynamoDB를 예로 사용합니다. DynamoDB는 관리형 NoSQL 데이터베이스이며 수평 확장과 글로벌 세컨더리 인덱스(GSI)를 지원합니다.

<div style="margin-left:3rem">
    <img src="./images/dynamo-db.png" alt="DynamoDB" width="500" />
</div>

체스 게임의 리더보드를 저장하는 테이블에서 시작합니다.

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-1.png" alt="체스 리더보드 테이블 1" width="500" />
</div>

기본 구조는 단순하지만 점수 기준 조회에는 적합하지 않을 수 있습니다. 점수를 Sort Key로 사용하는 방식으로 개선할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-2.png" alt="체스 리더보드 테이블 2" width="500" />
</div>

또 다른 문제는 월을 하나의 파티션 키로 사용하면 현재 월에 쓰기와 읽기가 집중되어 핫 파티션이 생길 수 있다는 점입니다.

이를 완화하기 위해 `user_id % num_partitions` 같은 방식으로 파티션 번호를 키에 추가하는 **Write Sharding**을 사용할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-3.png" alt="체스 리더보드 테이블 3" width="500" />
</div>

파티션 수에는 트레이드오프가 있습니다.
- 파티션이 많을수록 쓰기 처리량을 더 넓게 분산할 수 있습니다.
- 반면 집계 읽기에서는 더 많은 파티션을 조회해야 하므로 읽기 지연 시간이 증가합니다.

이 방식에서는 앞에서 본 **scatter-gather** 패턴을 사용해 각 파티션의 결과를 수집하고 병합합니다.

<div style="margin-left:3rem">
    <img src="./images/scatter-gather-2.png" alt="Scatter-Gather" width="500" />
</div>

적절한 파티션 수는 실제 데이터 분포와 QPS를 대상으로 벤치마킹해 결정해야 합니다.

NoSQL 방식의 큰 단점 중 하나는 특정 사용자의 정확한 전역 순위를 계산하기 어렵다는 점입니다.

정확한 순위 계산 비용이 지나치게 크다면 대규모 환경에서 사용자에게 백분위(percentile)를 보여주는 제품 설계도 고려할 수 있습니다.

CRON 작업으로 점수 분포를 주기적으로 계산한 뒤 사용자의 백분위를 결정할 수 있습니다.

```
10th percentile = score < 100
20th percentile = score < 500
...
90th percentile = score < 6500
```

---

## 4단계: 마무리

시간이 허용되면 추가로 논의할 수 있는 항목은 다음과 같습니다.
- **더 빠른 사용자 정보 조회:** Redis Hash에서 `user_id -> user object` 매핑을 캐시해 데이터베이스 조회를 줄일 수 있습니다.
- **동점 처리:** 두 사용자의 점수가 같다면 마지막 경기 시각 같은 보조 기준으로 순서를 정할 수 있습니다.
- **대규모 장애 복구:** Redis 리더보드가 손실되면 MySQL의 경기 결과나 WAL/이벤트 이력을 이용해 임시 복구 스크립트로 리더보드를 재생성할 수 있습니다.