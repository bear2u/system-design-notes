# 18장: Google Maps

## 소개

이 장에서는 **Google Maps**의 단순화된 버전을 설계합니다.

Google Maps와 관련된 몇 가지 정보입니다.
 * 2005년에 시작했습니다.
 * 위성 이미지, 도로 지도, 실시간 교통 상황, 경로 계획 등 다양한 서비스를 제공합니다.
 * 2021년 기준 일간 활성 사용자 약 10억 명, 전 세계 약 99% 지역 커버리지, 실시간 위치 정보 하루 약 2,500만 건의 갱신 규모를 기록했습니다.

---

## 1단계: 문제 이해 및 설계 범위 설정

지원자와 면접관 사이의 예시 질의응답입니다.
 * C: 일간 활성 사용자는 몇 명인가?
 * I: DAU 10억 명이다.
 * C: 어떤 기능에 집중해야 하는가?
 * I: 위치 갱신, 내비게이션, ETA, 지도 렌더링이다.
 * C: 도로 데이터의 크기는 어느 정도이며 사용할 수 있는가?
 * I: 여러 출처에서 도로 데이터를 확보했으며 원시 데이터는 수 TB 규모다.
 * C: 교통 상황도 고려해야 하는가?
 * I: 정확한 시간 추정을 위해 고려해야 한다.
 * C: 도보, 자전거, 자동차 같은 서로 다른 이동 수단은 어떤가?
 * I: 모두 지원해야 한다.
 * C: 여러 경유지를 포함하는 길찾기도 필요한가?
 * I: 인터뷰 범위에서는 제외한다.
 * C: 사업체 장소와 사진은 어떤가?
 * I: 좋은 질문이지만 여기서는 고려하지 않아도 된다.

이 장에서는 사용자 위치 갱신, ETA를 포함한 내비게이션 서비스, 지도 렌더링의 세 가지 핵심 기능에 집중합니다.

### **비기능 요구사항**

- **정확성:** 사용자에게 잘못된 길을 안내해서는 안 됩니다.
- **부드러운 내비게이션:** 지도 렌더링이 끊기지 않고 자연스러워야 합니다.
- **데이터 및 배터리 사용량:** 특히 모바일 디바이스에서는 클라이언트의 데이터와 배터리 사용량을 최소화해야 합니다.
- 일반적인 가용성과 확장성 요구사항을 충족해야 합니다.

### **지도 기초**

설계에 들어가기 전에 몇 가지 지도 관련 개념을 이해해야 합니다.

#### 위치 좌표 체계

지구는 자전하는 구 형태이며 위치는 위도(남북 방향 위치)와 경도(동서 방향 위치)로 표현합니다.

<div style="margin-left:3rem">
    <img src="./images/partitioning-system.png" alt="위치 좌표 체계" width="500" />
</div>

#### 3차원에서 2차원으로 변환

3차원 공간의 지점을 2차원 평면으로 변환하는 과정을 **지도 투영(map projection)**이라고 합니다.

여러 투영 방식이 있으며 각각 장단점이 있습니다. 대부분 실제 지형의 기하학적 형태를 어느 정도 왜곡합니다.

<div style="margin-left:3rem">
    <img src="./images/map-projections.png" alt="지도 투영" width="500" />
</div>

Google Maps는 Mercator 투영을 변형한 **Web Mercator** 방식을 사용합니다.

#### 지오코딩

지오코딩(geocoding)은 주소를 지리적 좌표로 변환하는 과정입니다.

반대로 좌표를 주소나 장소 정보로 변환하는 과정을 **역지오코딩(reverse geocoding)**이라고 합니다.

한 가지 구현 방법은 도로망이 지리 좌표 공간에 매핑된 GIS 등의 여러 데이터 소스를 활용해 보간(interpolation)하는 것입니다.

#### Geohash

Geohash는 지리적 영역을 문자와 숫자로 이루어진 문자열로 인코딩하는 방식입니다.

지구 표면을 평면으로 표현하고 영역을 재귀적으로 네 개의 사분면으로 나눕니다.

<div style="margin-left:3rem">
    <img src="./images/geohashing.png" alt="Geohash" width="500" />
</div>

#### 지도 렌더링

지도 렌더링에는 타일링(tiling)을 사용합니다. 전체 지도를 하나의 거대한 이미지로 렌더링하는 대신 전 세계 지도를 작은 타일로 나눕니다.

클라이언트는 현재 필요한 타일만 다운로드한 뒤 모자이크를 맞추듯 이어 붙여 렌더링합니다.

확대 수준(zoom level)마다 서로 다른 타일이 있으며 클라이언트는 현재 확대 수준에 맞는 타일을 선택합니다.

예를 들어 전 세계가 모두 보이도록 가장 멀리 축소하면 전 세계를 나타내는 256x256 타일 하나만 다운로드할 수 있습니다.

#### 내비게이션 알고리즘을 위한 도로 데이터 처리

대부분의 경로 탐색 알고리즘에서 교차로는 노드, 도로는 간선으로 표현합니다.

<div style="margin-left:3rem">
    <img src="./images/road-representation.png" alt="도로 그래프 표현" width="500" />
</div>

대부분의 내비게이션 알고리즘은 Dijkstra 또는 A* 알고리즘의 변형을 사용합니다.

경로 탐색 성능은 그래프 크기에 매우 민감합니다. 대규모 시스템에서는 전 세계 도로를 하나의 거대한 그래프로 표현하고 그 전체에서 알고리즘을 실행할 수 없습니다.

대신 지도 타일링과 비슷하게 세계를 더 작은 그래프로 계속 분할합니다.

라우팅 타일은 인접 타일에 대한 참조를 가지며 알고리즘은 연결된 타일을 탐색하면서 더 큰 도로 그래프를 구성할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/routing-tiles.png" alt="라우팅 타일" width="500" />
</div>

이 기법을 사용하면 메모리 대역폭 사용량을 크게 줄이고 주어진 출발지/목적지에 필요한 타일만 메모리에 올릴 수 있습니다.

하지만 장거리 경로에서는 작고 상세한 라우팅 타일을 너무 많이 연결하면 시간과 메모리 비용이 다시 커집니다. 따라서 상세 수준이 서로 다른 라우팅 타일을 준비하고 목적지까지의 거리와 탐색 단계에 맞는 상세도를 사용합니다.

<div style="margin-left:3rem">
    <img src="./images/map-routing-hierarchical.png" alt="계층적 지도 라우팅" width="500" />
</div>

### **개략적 규모 추정**

저장해야 하는 데이터는 다음과 같습니다.
 * 전 세계 지도 - 저장해야 할 모든 타일을 고려하면 약 70PB로 추정할 수 있습니다. 광대한 사막처럼 매우 비슷한 타일의 압축 효과도 고려합니다.
 * 메타데이터 - 상대적으로 크기가 작으므로 계산에서 제외합니다.
 * 도로 정보 - 라우팅 타일 형태로 저장합니다.

내비게이션 요청 QPS를 추정합니다. DAU 10억 명이 주당 35분을 사용한다고 가정하면 하루 약 50억 분의 사용량이 됩니다.
GPS 갱신 요청을 배치해서 보낸다고 가정하면 평균 약 200k QPS, 피크 약 1M QPS 규모를 예상할 수 있습니다.

---

## 2단계: 상위 수준 설계 제안 및 합의

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="상위 수준 설계" width="500" />
</div>

### **위치 서비스**

<div style="margin-left:3rem">
    <img src="./images/location-service.png" alt="위치 서비스" width="500" />
</div>

사용자의 위치 갱신을 기록하는 역할을 합니다.
 * 위치 갱신은 `t`초마다 전송합니다.
 * 위치 데이터 스트림은 시간이 지나면서 서비스를 개선하는 데 활용할 수 있습니다. 예를 들어 더 정확한 ETA 계산, 교통 상황 모니터링, 폐쇄 도로 탐지, 사용자 행동 분석 등에 사용할 수 있습니다.

위치 갱신을 매번 서버에 보내는 대신 클라이언트에서 여러 갱신을 배치한 뒤 묶어서 전송할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/location-update-batches.png" alt="위치 갱신 배치" width="500" />
</div>

이렇게 최적화해도 Google Maps 규모에서는 쓰기 부하가 매우 큽니다. 따라서 Cassandra처럼 대량 쓰기에 적합한 데이터베이스를 활용할 수 있습니다.

추가 분석을 위해 위치 갱신을 효율적으로 스트림 처리하는 Kafka도 사용할 수 있습니다.

위치 갱신 요청 예시는 다음과 같습니다.

```
POST /v1/locations
Parameters
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples.
```

### **내비게이션 서비스**

이 구성 요소는 합리적인 시간 안에 A와 B 사이의 빠른 경로를 찾는 역할을 합니다. 약간의 지연 시간은 허용할 수 있으며 반드시 절대적으로 가장 빠른 경로일 필요는 없지만 정확성은 중요합니다.

요청 예시:

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

응답 예시:

```json
{
  "distance": {"text":"0.2 mi", "value": 259},
  "duration": {"text": "1 min", "value": 83},
  "end_location": {"lat": 37.4038943, "Ing": -121.9410454},
  "html_instructions": "Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style=\"font-size:0.9em\">Restricted usage road</div>",
  "polyline": {"points": "_fhcFjbhgVuAwDsCal"},
  "start_location": {"lat": 37.4027165, "lng": -121.9435809},
  "geocoded_waypoints": [
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJwZNMti1fawwRO2aVVVX2yKg",
       "types" : [ "locality", "political" ]
    },
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
       "types" : [ "locality", "political" ]
    }
  ],
  "travel_mode": "DRIVING"
}
```

아직 교통 상황 변화와 경로 재탐색은 고려하지 않았으며 상세 설계에서 다룹니다.

### **지도 렌더링**

지도 타일 전체 데이터셋은 PB 규모이므로 클라이언트에 모두 저장하는 것은 현실적이지 않습니다.

클라이언트의 현재 위치와 확대 수준을 기준으로 필요한 타일을 서버에서 필요할 때 가져와야 합니다.

새 타일은 사용자가 지도를 확대/축소할 때, 지도를 이동할 때, 내비게이션 중 새로운 타일 영역으로 진입할 때 가져옵니다.

지도 타일을 클라이언트에 제공하는 방법은 다음과 같이 생각할 수 있습니다.
 * 요청 시 동적으로 타일을 만들 수 있지만 서버 부하가 매우 크고 캐싱도 어렵습니다.
 * 각 타일의 Geohash 같은 식별 값을 기준으로 정적 타일을 만들어 저장하고 CDN에서 제공할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/static-map-tiles.png" alt="정적 지도 타일" width="500" />
</div>

CDN을 사용하면 사용자는 지연 시간을 줄이기 위해 자신과 가장 가까운 POP(Point of Presence) 서버에서 지도 타일을 가져올 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/cdn-vs-no-cdn.png" alt="CDN 사용 비교" width="500" />
</div>

지도 타일을 결정하는 방식에는 다음 선택지가 있습니다.
 * 지도 타일의 Geohash를 클라이언트에서 직접 계산할 수 있습니다. 이 방식을 사용한다면 이후 계산 방식을 바꾸기 위해 모든 클라이언트를 강제로 갱신하기 어렵다는 점을 고려해 장기 호환성을 신중히 설계해야 합니다.
 * 또는 간단한 API가 클라이언트 대신 지도 타일 URL을 계산해 줄 수 있습니다. 대신 API 호출이 하나 더 필요합니다.

<div style="margin-left:3rem">
    <img src="./images/map-tile-url-calculation.png" alt="지도 타일 URL 계산" width="500" />
</div>

---

## 3단계: 상세 설계

### **데이터 모델**

서로 다른 유형의 데이터를 어떻게 저장할지 살펴봅니다.

#### 라우팅 타일

초기 도로 데이터는 여러 출처에서 가져오며 사용자 위치 갱신 데이터 등을 활용해 시간이 지나면서 개선합니다.

도로 데이터는 비정형 원시 데이터입니다. 주기적으로 실행되는 오프라인 처리 파이프라인에서 원시 데이터를 애플리케이션이 사용하는 그래프 기반 라우팅 타일로 변환합니다.

이 타일에는 복잡한 데이터베이스 기능이 필요하지 않으므로 데이터베이스 대신 S3 객체 스토리지에 저장하고 적극적으로 캐시할 수 있습니다.

인접 리스트(adjacency list)를 바이너리 파일로 효율적으로 압축하는 라이브러리도 활용할 수 있습니다.

#### 사용자 위치 데이터

사용자 위치 데이터는 교통 상황 갱신과 여러 분석 작업에 매우 유용합니다.

쓰기 비중이 높은 데이터이므로 Cassandra를 사용할 수 있습니다.

행 예시:

<div style="margin-left:3rem">
    <img src="./images/user-location-data-torw.png" alt="사용자 위치 데이터 행" width="500" />
</div>

#### 지오코딩 데이터베이스

이 데이터베이스는 위도/경도 쌍과 장소 사이의 키-값 매핑을 저장합니다.

읽기는 매우 빈번하고 쓰기는 상대적으로 드물기 때문에 빠른 읽기 성능을 제공하는 Redis를 활용할 수 있습니다.

#### 미리 계산한 전 세계 지도 이미지

앞에서 설명한 것처럼 지도 타일 이미지를 사전에 생성해 CDN에 저장합니다.

<div style="margin-left:3rem">
    <img src="./images/precomputed-map-tile-image.png" alt="사전 생성 지도 타일 이미지" width="500" />
</div>

### **서비스**

#### 위치 서비스

이 서비스에서 사용자 위치를 어떻게 저장하는지 데이터베이스 설계를 더 자세히 살펴봅니다.

<div style="margin-left:3rem">
    <img src="./images/location-service-diagram.png" alt="위치 서비스 다이어그램" width="500" />
</div>

위치 갱신에는 매우 많은 쓰기가 발생하므로 NoSQL 데이터베이스를 사용할 수 있습니다. 사용자 위치는 자주 바뀌고 새 갱신이 도착하면 이전 데이터가 빠르게 오래된 데이터가 되므로 일관성보다 가용성을 우선할 수 있습니다.

이 요구사항에 잘 맞는 Cassandra를 데이터베이스로 선택할 수 있습니다.

저장할 행의 예시는 다음과 같습니다.

<div style="margin-left:3rem">
    <img src="./images/user-location-row-example.png" alt="사용자 위치 행 예시" width="500" />
</div>

 * `user_id`를 파티션 키로 사용해 특정 사용자의 모든 위치 갱신에 빠르게 접근합니다.
 * `timestamp`를 클러스터링 키로 사용해 위치 갱신이 들어온 시간순으로 데이터를 저장합니다.

또한 Kafka를 이용해 위치 갱신을 다양한 목적의 다른 서비스에 스트리밍할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/location-update-streaming.png" alt="위치 갱신 스트리밍" width="500" />
</div>

#### 지도 렌더링

지도 타일은 여러 확대 수준별로 저장합니다. 가장 낮은 확대 수준에서는 전 세계를 단 하나의 256x256 타일로 표현합니다.

확대 수준이 증가할 때마다 지도 타일 수는 약 네 배씩 증가합니다.

<div style="margin-left:3rem">
    <img src="./images/zoom-level-increases.png" alt="확대 수준 증가" width="500" />
</div>

네트워크로 전체 래스터 이미지 데이터를 전송하는 대신 타일을 경로와 폴리곤 같은 벡터 데이터로 표현하고 클라이언트가 동적으로 렌더링하도록 최적화할 수 있습니다.

이 방식은 대역폭을 크게 절약할 수 있습니다.

#### 내비게이션 서비스

이 서비스는 빠른 경로를 찾는 역할을 합니다.

<div style="margin-left:3rem">
    <img src="./images/navigation-service.png" alt="내비게이션 서비스" width="500" />
</div>

이 하위 시스템의 각 구성 요소를 살펴봅니다.

먼저 지오코딩 서비스는 주소를 위도/경도 좌표로 변환합니다.

요청 예시:

```
https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

응답 예시:

```json
{
   "results" : [
      {
         "formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
         "geometry" : {
            "location" : {
               "lat" : 37.4224764,
               "lng" : -122.0842499
            },
            "location_type" : "ROOFTOP",
            "viewport" : {
               "northeast" : {
                  "lat" : 37.4238253802915,
                  "lng" : -122.0829009197085
               },
               "southwest" : {
                  "lat" : 37.4211274197085,
                  "lng" : -122.0855988802915
               }
            }
         },
         "place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
         "plus_code": {
            "compound_code": "CWC8+W5 Mountain View, California, United States",
            "global_code": "849VCWC8+W5"
         },
         "types" : [ "street_address" ]
      }
   ],
   "status" : "OK"
}
```

경로 계획 서비스(route planner)는 현재 교통 상황을 반영해 이동 시간을 최적화한 추천 경로를 계산합니다.

최단 경로 서비스는 객체 스토리지의 라우팅 타일을 대상으로 A* 알고리즘의 변형을 실행해 적합한 경로를 계산합니다.
 * 출발지/목적지 쌍을 받아 위도/경도 좌표로 변환하고 좌표에서 Geohash를 계산해 필요한 라우팅 타일을 결정합니다.
 * 초기 라우팅 타일부터 탐색을 시작해 목적지 타일까지 충분히 좋은 경로를 찾을 때까지 탐색합니다.

<div style="margin-left:3rem">
    <img src="./images/shortest-path-service.png" alt="최단 경로 서비스" width="500" />
</div>

ETA 서비스는 경로 계획 서비스가 호출하며 교통 데이터를 입력으로 하는 머신러닝 알고리즘 등을 사용해 예상 도착 시간을 계산합니다.

랭커(ranker) 서비스는 사용자가 지정한 필터를 기준으로 여러 후보 경로의 순위를 계산합니다. 예를 들어 유료도로 또는 고속도로 회피 옵션을 반영합니다.

업데이터(updater) 서비스는 중요한 데이터베이스 일부를 비동기적으로 갱신해 최신 상태를 유지합니다.

#### 개선: 적응형 ETA와 경로 재탐색

새로운 교통 데이터를 반영해 진행 중인 경로를 동적으로 갱신할 수 있습니다.

한 가지 방법은 현재 내비게이션을 사용 중인 사용자가 통과할 예정인 모든 라우팅 타일을 데이터베이스에 저장하는 것입니다.

데이터는 다음과 같은 형태가 될 수 있습니다.

```
user_1: r_1, r_2, r_3, …, r_k
user_2: r_4, r_6, r_9, …, r_n
user_3: r_2, r_8, r_9, …, r_m
...
user_n: r_2, r_10, r21, ..., r_l
```

특정 타일에서 교통사고가 발생하면 해당 타일을 통과하는 경로를 가진 모든 사용자를 찾아 경로를 다시 계산할 수 있습니다.

데이터베이스에 저장해야 하는 타일 수를 줄이려면 출발 라우팅 타일과 서로 다른 해상도 수준의 상위 라우팅 타일을 목적지 타일까지 포함하도록 저장하는 방법을 사용할 수 있습니다.

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

<div style="margin-left:3rem">
    <img src="./images/adaptive-eta-data-storage.png" alt="적응형 ETA 데이터 저장" width="500" />
</div>

이 구조에서는 사용자가 영향받는지 판단하기 위해 사용자의 상위 영역 타일이 사고가 발생한 타일을 포함하는지 확인할 수 있습니다.

현재 이동 중인 사용자의 여러 대체 경로를 함께 추적하고 더 빠른 경로가 생기면 사용자에게 알려줄 수도 있습니다.

#### 전달 프로토콜

서버에서 클라이언트로 데이터를 능동적으로 푸시하는 방법에는 여러 선택지가 있습니다.
 * 모바일 푸시 알림은 페이로드 크기가 제한적이고 웹 앱에서는 동일한 방식으로 사용할 수 없으므로 내비게이션 데이터 스트림에는 적합하지 않습니다.
 * WebSocket은 일반적으로 롱 폴링보다 서버의 지속적인 요청 처리 오버헤드가 적어 더 적합한 선택입니다.
 * SSE(Server-Sent Events)도 사용할 수 있지만, 라스트마일 배송 기능처럼 양방향 통신이 필요할 수 있으므로 WebSocket을 선택할 수 있습니다.

---

## 4단계: 마무리

최종 설계는 다음과 같습니다.

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="최종 설계" width="500" />
</div>

추가 기능으로 여러 경유지를 포함하는 내비게이션을 제공할 수 있습니다. 이러한 기능은 Uber나 Lyft 같은 기업 고객이 여러 위치를 방문하는 최적 경로를 계산하는 데 활용할 수 있습니다.
