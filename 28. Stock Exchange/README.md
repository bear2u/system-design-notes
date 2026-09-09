# 28장: 증권 거래소

## 소개
이 장에서는 **전자 증권 거래소(electronic stock exchange)**를 설계합니다.

기본 기능은 매수자와 매도자의 주문을 효율적으로 매칭하는 것입니다.

대표적인 증권 거래소에는 **NYSE**, **NASDAQ** 등이 있습니다.

<div style="margin-left:3rem">
    <img src="./images/world-stock-exchanges.png" alt="세계 증권 거래소" width="500" />
</div>

---

## 1단계: 문제 이해 및 설계 범위 설정
 * C: 어떤 증권을 거래하는가? 주식, 옵션, 선물 중 무엇인가?
 * I: 단순화를 위해 주식만 다룬다.
 * C: 어떤 주문 작업을 지원하는가? 신규, 취소, 정정은 어떤가? 지정가, 시장가, 조건부 주문 중 어떤 유형을 지원하는가?
 * I: 주문 생성과 취소를 지원한다. 주문 유형은 지정가 주문만 고려한다.
 * C: 정규장 이후 거래도 지원해야 하는가?
 * I: 아니다. 정규 거래 시간만 지원한다.
 * C: 거래소의 기본 기능을 설명해 줄 수 있는가?
 * I: 클라이언트가 지정가 주문을 생성하거나 취소하고 체결 결과를 실시간으로 받을 수 있어야 한다. 주문장(order book)도 실시간으로 볼 수 있어야 한다.
 * C: 거래소 규모는 어느 정도인가?
 * I: 동시에 거래하는 사용자는 수만 명, 종목은 약 100개다. 하루 수십억 건의 주문을 처리하며 규제 준수를 위한 리스크 검사도 지원해야 한다.
 * C: 어떤 종류의 리스크 검사를 수행하는가?
 * I: 단순한 규칙을 사용한다. 예를 들어 사용자가 하루에 Apple 주식 100만 주 이상 거래하지 못하도록 제한한다.
 * C: 사용자 지갑과의 연동은 어떻게 하는가?
 * I: 주문 전에 고객에게 충분한 자금이 있는지 확인해야 한다. 미체결 주문에 필요한 자금은 주문이 완료될 때까지 보류해야 한다.

### **비기능 요구사항**
면접관이 제시한 규모는 소규모~중간 규모 거래소를 설계하는 문제로 볼 수 있습니다.
향후 종목과 사용자 수를 더 늘릴 수 있도록 유연한 확장성도 고려합니다.

그 밖의 비기능 요구사항:
 * **가용성:** 최소 99.99%를 목표로 합니다. 거래소의 중단은 신뢰와 평판에 큰 영향을 줄 수 있습니다.
 * **장애 내성:** 운영 장애의 영향을 줄이기 위해 빠른 탐지와 복구 메커니즘이 필요합니다.
 * **지연 시간:** 왕복 지연 시간은 밀리초 수준을 목표로 하고 특히 99번째 백분위(99p)를 중요하게 봅니다. 일부 사용자에게 지속적으로 높은 꼬리 지연 시간이 발생하지 않도록 해야 합니다.
 * **보안:** 계정 관리 시스템이 필요하며 법적 규제 준수를 위해 KYC를 지원해야 합니다. 인터넷에 공개된 리소스는 DDoS 공격으로부터 보호해야 합니다.

### **개략적 규모 추정**
 * 종목 100개, 하루 주문 10억 건
 * 정규 거래 시간: 09:30~16:00, 총 6.5시간
 * QPS = 1bil / 6.5 / 3600 ≈ 43,000
 * 피크 QPS = 5 × QPS ≈ 215,000
 * 일반적으로 장 시작 직후 거래량이 더 높을 수 있으므로 피크 부하를 별도로 고려해야 합니다.

---

## 2단계: 상위 수준 설계 제안 및 합의

### **기초 비즈니스 지식**
거래소와 관련된 기본 개념을 정리합니다.

**브로커(broker)**는 최종 사용자와 거래소 사이의 거래를 중개합니다. 예: Robinhood, Fidelity.

기관 투자자는 전문 거래 소프트웨어를 사용해 대규모로 거래하며 별도의 낮은 지연 시간 인터페이스나 주문 처리 기능이 필요할 수 있습니다.
예를 들어 매우 큰 주문이 시장 가격에 미치는 영향을 줄이기 위해 주문 분할을 사용할 수 있습니다.

주문 유형:
 * **지정가 주문(Limit Order):** 지정한 가격 또는 더 유리한 가격에서 매수/매도합니다. 즉시 체결되지 않거나 일부만 체결될 수 있습니다.
 * **시장가 주문(Market Order):** 가격을 지정하지 않고 현재 시장의 가능한 가격으로 즉시 체결을 시도합니다.

가격 관련 용어:
 * **Bid:** 매수자가 주식을 사려고 제시한 최고 가격
 * **Ask:** 매도자가 주식을 팔려고 제시한 최저 가격

원문에서는 미국 시장 데이터 호가를 L1, L2, L3 세 수준으로 설명합니다.

L1 시장 데이터는 최우선 매수/매도 호가와 수량을 보여줍니다.

<div style="margin-left:3rem">
    <img src="./images/l1-price.png" alt="L1 가격" width="500" />
</div>

L2는 더 많은 가격 수준을 포함합니다.

<div style="margin-left:3rem">
    <img src="./images/l2-price.png" alt="L2 가격" width="500" />
</div>

L3는 각 가격 수준과 해당 수준의 주문 대기 수량을 더 상세하게 보여줍니다.

<div style="margin-left:3rem">
    <img src="./images/l3-price.png" alt="L3 가격" width="500" />
</div>

**캔들스틱(candlestick)**은 특정 시간 구간의 시가, 종가, 고가, 저가를 나타냅니다.

<div style="margin-left:3rem">
    <img src="./images/candlestick.png" alt="캔들스틱" width="500" />
</div>

**FIX(Financial Information eXchange)**는 증권 거래 정보를 교환하는 데 널리 사용되는 프로토콜입니다. 예시 메시지는 다음과 같습니다.
```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### **상위 수준 설계**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="상위 수준 설계" width="500" />
</div>

거래 흐름:
 * 클라이언트가 거래 인터페이스에서 주문을 생성합니다.
 * 브로커가 주문을 거래소로 전송합니다.
 * 주문은 Client Gateway를 통해 거래소에 들어옵니다. 이 계층에서 검증, 요청 제한, 인증 등을 수행한 뒤 Order Manager로 전달합니다.
 * Order Manager가 Risk Manager의 규칙에 따라 리스크 검사를 수행합니다.
 * 리스크 검사를 통과하면 주문에 필요한 자금이 지갑에 충분한지 확인합니다.
 * 주문을 Matching Engine으로 전달합니다. 매칭이 발생하면 Matching Engine은 매수와 매도 양쪽에 대해 두 개의 체결 결과(fill)를 생성합니다. 주문과 체결에는 순서를 부여해 결정적으로 처리할 수 있도록 합니다.
 * 체결 결과를 클라이언트에 반환합니다.

시장 데이터 흐름(M1~M3):
 * Matching Engine이 체결 이벤트 스트림을 생성해 Market Data Publisher로 보냅니다.
 * Market Data Publisher가 주문장과 캔들스틱 데이터를 구성해 Data Service에 전달합니다.
 * 시장 데이터는 실시간 분석에 적합한 저장소에도 저장합니다. 브로커는 Data Service를 통해 최신 시장 데이터를 구독합니다.

리포팅 흐름(R1~R2):
 * Reporter가 주문과 체결에서 필요한 보고 필드를 수집해 데이터베이스에 기록합니다.
 * 보고 필드 예: `client_id`, `price`, `quantity`, `order_type`, `filled_quantity`, `remaining_quantity`

거래 흐름은 핵심 경로(critical path)에 있고 시장 데이터/리포팅 흐름은 상대적으로 비핵심 경로이므로 지연 시간 요구사항이 다릅니다.

#### 거래 흐름
거래 흐름은 핵심 경로이므로 매우 낮은 지연 시간에 최적화해야 합니다.

중심 구성 요소는 **Matching Engine**이며 Cross Engine이라고도 합니다. 주요 책임은 다음과 같습니다.
 * 각 종목의 주문장 유지 - 종목별 매수/매도 주문 목록
 * 매수와 매도 주문 매칭 - 하나의 매칭은 매수/매도 양쪽에 대해 각각 하나의 체결(fill)을 생성합니다. 빠르고 정확해야 합니다.
 * 체결 스트림을 시장 데이터로 전달
 * 매칭 결과를 결정적인 순서로 생성. 고가용성과 복구에 중요합니다.

다음 핵심 구성 요소는 **Sequencer**입니다. 들어오는 주문과 나가는 체결 이벤트에 순차 ID를 부여해 Matching Engine의 처리를 결정적으로 만드는 역할을 합니다.

<div style="margin-left:3rem">
    <img src="./images/sequencer.png" alt="Sequencer" width="500" />
</div>

들어오는 주문과 나가는 체결에 순서를 부여하는 이유는 다음과 같습니다.
 * 시간적 공정성과 처리 순서 명확화
 * 빠른 복구와 재생(replay)
 * 중복 처리 방지 및 exactly-once 처리 효과 구현에 활용

개념적으로 Kafka를 입력/출력 메시지 로그이자 Sequencer로 사용할 수도 있습니다. 하지만 이 장에서는 더 낮은 지연 시간을 목표로 자체 구현한다고 가정합니다.

**Order Manager**는 주문 상태를 관리하고 Matching Engine과 상호작용합니다. 주문을 보내고 체결 결과를 받습니다.

주요 책임은 다음과 같습니다.
 * 주문을 리스크 검사에 전달합니다. 예: 사용자의 거래량이 하루 100만 주를 넘지 않는지 확인합니다.
 * 사용자 지갑의 가용 자금을 확인합니다.
 * 주문을 Sequencer를 거쳐 Matching Engine으로 전달합니다. 네트워크 대역폭을 줄이기 위해 Matching Engine에는 필요한 주문 정보만 전달합니다.
 * Sequencer를 통해 돌아온 체결 결과를 받아 Client Gateway를 거쳐 브로커에 전송합니다.

Order Manager의 주요 과제 중 하나는 복잡한 주문 상태 전이를 정확하게 관리하는 것입니다. 상세 설계에서 Event Sourcing을 하나의 해결책으로 살펴봅니다.

마지막으로 **Client Gateway**는 사용자의 주문을 받아 Order Manager에 전달합니다.

<div style="margin-left:3rem">
    <img src="./images/client-gateway.png" alt="Client Gateway" width="500" />
</div>

Client Gateway는 핵심 경로에 있으므로 가능한 한 가볍게 유지합니다.

클라이언트 유형마다 여러 Client Gateway를 둘 수 있습니다. 예를 들어 **colo engine**은 브로커가 거래소 데이터 센터에 임대한 서버에서 실행되는 거래 엔진입니다.

<div style="margin-left:3rem">
    <img src="./images/client-gateways.png" alt="Client Gateways" width="500" />
</div>

#### 시장 데이터 흐름
Market Data Publisher는 Matching Engine의 체결 결과를 받아 체결 스트림을 기반으로 주문장과 캔들스틱 차트를 재구성합니다.

이 데이터는 구독자에게 집계된 시장 정보를 제공하는 Data Service로 전달합니다.

<div style="margin-left:3rem">
    <img src="./images/market-data.png" alt="시장 데이터" width="500" />
</div>

#### 리포팅 흐름
Reporter는 핵심 거래 경로에는 없지만 중요한 구성 요소입니다.

<div style="margin-left:3rem">
    <img src="./images/reporting-flow.png" alt="리포팅 흐름" width="500" />
</div>

거래 이력, 세금 보고, 규제 준수 보고, 결제/정산 등에 필요한 데이터를 저장합니다.
이 흐름은 극단적으로 낮은 지연 시간보다 정확성과 규제 준수가 더 중요합니다.

### **API 설계**
클라이언트는 브로커를 통해 거래소와 상호작용하며 주문 생성, 체결 조회, 시장 데이터 조회, 분석용 과거 데이터 다운로드 등을 수행합니다.

Client Gateway와 일반 브로커 사이에는 RESTful API를 사용할 수 있습니다.

기관 고객처럼 더 낮은 지연 시간이 필요한 경우 전용 프로토콜을 사용할 수 있습니다.

주문 생성:
```
POST /v1/order
```

매개변수:
 * `symbol` - 주식 종목 코드. String
 * `side` - buy 또는 sell. String
 * `price` - 지정가 주문 가격. Long
 * `orderType` - limit 또는 market. 이 설계에서는 limit만 지원. String
 * `quantity` - 주문 수량. Long

응답:
 * `id` - 주문 ID. Long
 * `creationTime` - 시스템에서 주문을 생성한 시각. Long
 * `filledQuantity` - 체결 완료 수량. Long
 * `remainingQuantity` - 아직 체결되지 않은 수량. Long
 * `status` - new/canceled/filled. String
 * 나머지 속성은 입력 매개변수와 동일

체결 조회:
```
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
```

매개변수:
 * `symbol` - 주식 종목 코드. String
 * `orderId` - 주문 ID. 선택 사항. String
 * `startTime` - epoch 기준 조회 시작 시각. Long
 * `endTime` - epoch 기준 조회 종료 시각. Long

응답:
 * `executions` - 범위에 포함된 체결 목록. Array
 * `id` - 체결 ID. Long
 * `orderId` - 주문 ID. Long
 * `symbol` - 종목 코드. String
 * `side` - buy 또는 sell. String
 * `price` - 체결 가격. Long
 * `orderType` - limit 또는 market. String
 * `quantity` - 체결 수량. Long

주문장 조회:
```
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
```

매개변수:
 * `symbol` - 종목 코드. String
 * `depth` - 매수/매도 각 방향에서 반환할 주문장 깊이. Int

응답:
 * `bids` - 가격과 수량 배열. Array
 * `asks` - 가격과 수량 배열. Array

캔들스틱 조회:
```
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
```

매개변수:
 * `symbol` - 종목 코드. String
 * `resolution` - 캔들 하나의 시간 구간 길이(초). Long
 * `startTime` - epoch 기준 시작 시각. Long
 * `endTime` - epoch 기준 종료 시각. Long

응답:
 * `candles` - 각 캔들 데이터 배열. Array
 * `open` - 시가. Double
 * `close` - 종가. Double
 * `high` - 고가. Double
 * `low` - 저가. Double

### **데이터 모델**
거래소의 주요 데이터는 세 범주로 나눌 수 있습니다.
 * Product, Order, Execution
 * Order Book
 * Candlestick Chart

#### Product, Order, Execution
Product는 거래되는 종목의 속성을 설명합니다. 예: 상품 유형, 거래 심볼, UI 표시 심볼.

이 데이터는 자주 바뀌지 않으며 주로 UI 렌더링과 거래 규칙에 사용됩니다.

Order는 매수/매도 지시를 나타내고 Execution은 매칭 결과입니다.

데이터 모델 예시는 다음과 같습니다.

<div style="margin-left:3rem">
    <img src="./images/product-order-execution-data-model.png" alt="Product Order Execution 데이터 모델" width="500" />
</div>

Order와 Execution은 세 가지 흐름 모두에서 사용됩니다.
 * 핵심 거래 경로에서는 높은 성능을 위해 메모리에서 처리하고 Sequencer의 이벤트를 이용해 복구할 수 있습니다.
 * Reporter는 보고 용도로 주문과 체결을 데이터베이스에 기록합니다.
 * Execution은 Market Data 서비스에 전달되어 주문장과 캔들스틱을 재구성합니다.

#### 주문장(Order Book)
주문장은 한 종목의 매수/매도 주문을 가격 수준별로 정리한 목록입니다.

효율적인 자료구조는 다음 연산을 빠르게 지원해야 합니다.
 * 특정 가격 수준 또는 가격 범위의 거래량 조회
 * 주문 추가/체결/취소
 * 최우선 Bid/Ask 가격 조회
 * 가격 수준 순회

주문 체결 예시:

<div style="margin-left:3rem">
    <img src="./images/order-book-execution.png" alt="주문장 체결" width="500" />
</div>

큰 주문이 여러 가격 수준을 소진하면 주문장의 최우선 호가와 스프레드가 변경될 수 있습니다.

주문장 구현 의사코드:
```
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

일반 리스트 대신 이중 연결 리스트를 사용하면 다음 연산을 효율화할 수 있습니다.
 * 새로운 주문을 리스트의 끝에 추가하면 `O(1)`으로 처리할 수 있습니다.
 * 가장 먼저 들어온 주문을 리스트의 앞에서 제거하면 FIFO 체결을 `O(1)`로 처리할 수 있습니다.
 * 주문 취소 시 `orderMap`을 이용해 `O(1)`에 주문을 찾고, Order 객체가 이전/다음 노드를 참조하면 연결 리스트에서 `O(1)` 삭제를 구현할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/order-book-impl.png" alt="주문장 구현" width="500" />
</div>

같은 자료구조를 Market Data 서비스가 주문장을 재구성할 때도 사용할 수 있습니다.

#### 캔들스틱 차트
Market Data 서비스는 일정 시간 구간에 처리된 주문과 체결을 기반으로 캔들스틱 데이터를 계산합니다.

```
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

메모리 사용을 줄이는 최적화:
 * 미리 할당한 ring buffer를 사용해 객체 할당 횟수를 줄입니다.
 * 메모리에 유지하는 캔들 수를 제한하고 오래된 데이터는 디스크에 저장합니다.

실시간 분석에는 KDB 같은 인메모리 컬럼형 데이터베이스를 사용할 수 있습니다. 장 종료 후에는 과거 데이터 저장소에 영속화합니다.

---

## 3단계: 상세 설계
현대 거래소의 한 가지 특징은 일반적인 웹 서비스와 달리 핵심 거래 경로를 매우 강력한 단일 서버나 소수의 고성능 서버 안에 집중시키는 설계가 사용될 수 있다는 점입니다.

이유를 성능 관점에서 살펴봅니다.

### **성능**
거래소에서는 모든 백분위 구간에서 일관되게 낮은 지연 시간이 중요합니다.

지연 시간을 줄이는 방법은 다음과 같습니다.
 * 핵심 경로에 포함되는 작업 수를 줄입니다.
 * 네트워크와 디스크 접근을 줄이고 각 작업의 실행 시간을 단축합니다.

첫 번째 목표를 위해 핵심 경로에서 부수 기능을 제거합니다. 극단적인 저지연 시스템에서는 동기식 로깅조차 핵심 경로에서 제외할 수 있습니다.

초기 설계처럼 서비스를 여러 프로세스/서버로 분리하면 서비스 간 네트워크 지연과 Sequencer의 디스크 I/O가 병목이 될 수 있습니다.

원문에서는 일반적인 분산 설계의 종단간 지연을 수십 밀리초 수준으로 보고, 더 극단적으로는 수십 마이크로초 수준을 목표로 하는 저지연 구조를 설명합니다.

이를 위해 핵심 구성 요소를 한 서버에 배치하고 프로세스 간 통신에는 `mmap` 기반 이벤트 저장소를 사용할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/mmap-bus.png" alt="mmap 버스" width="500" />
</div>

또 다른 최적화는 핵심 작업을 반복 실행하는 **application loop**를 하나의 CPU 코어에 고정(pin)해 컨텍스트 스위칭을 줄이는 것입니다.

<div style="margin-left:3rem">
    <img src="./images/application-loop.png" alt="Application Loop" width="500" />
</div>

단일 스레드 또는 코어 소유권 모델을 사용하면 여러 스레드가 같은 자원을 두고 경쟁하는 잠금 경합도 줄일 수 있습니다.

`mmap`은 디스크 파일을 애플리케이션의 가상 메모리 주소 공간에 매핑하는 UNIX 계열 시스템 호출입니다.

파일을 `/dev/shm`처럼 메모리 기반 파일 시스템에 만들면 공유 메모리를 통해 프로세스 간 데이터를 전달하는 구조로 사용할 수도 있습니다.

### **Event Sourcing**
Event Sourcing은 [디지털 지갑 장](../chapter28)에서 자세히 다룹니다. 여기서는 핵심 개념만 사용합니다.

현재 상태만 저장하는 대신 불변 상태 변경 이벤트를 순서대로 기록합니다.

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="Event Sourcing" width="500" />
</div>

 * 왼쪽: 전통적인 현재 상태 중심 스키마
 * 오른쪽: 이벤트 소싱 기반 스키마

현재까지의 설계는 다음과 같습니다.

<div style="margin-left:3rem">
    <img src="./images/design-so-far.png" alt="현재까지의 설계" width="500" />
</div>

 * 외부 도메인은 FIX 프로토콜을 사용해 Client Gateway와 통신합니다.
 * Order Manager가 새 주문 이벤트를 받아 검증하고 내부 상태에 추가한 뒤 Matching Core로 전달합니다.
 * 주문이 매칭되면 `OrderFilledEvent`를 생성해 mmap 이벤트 버스로 보냅니다.
 * 다른 구성 요소가 이벤트 저장소를 구독하고 각자의 후속 처리를 수행합니다.

추가 최적화로 Order Manager의 주문 상태 처리 로직을 라이브러리로 패키징해 여러 구성 요소가 로컬에서 동일한 상태를 유지하도록 할 수 있습니다. 이렇게 하면 주문 상태 조회를 위한 추가 네트워크 호출을 줄일 수 있습니다.

이 설계에서 Sequencer는 이벤트 저장소 자체가 아니라 **single writer** 역할을 합니다. 이벤트에 순서를 부여한 뒤 이벤트 저장소로 전달합니다.

<div style="margin-left:3rem">
    <img src="./images/sequencer-deep-dive.png" alt="Sequencer 상세 설계" width="500" />
</div>

### **고가용성**
99.99% 가용성 목표를 위해 거래소 아키텍처의 단일 장애점을 식별해야 합니다.
 * Matching Engine 같은 핵심 서비스의 대기 복제본을 준비합니다.
 * 장애 감지와 백업 인스턴스로의 failover를 자동화합니다.

Client Gateway 같은 무상태 서비스는 서버를 추가해 수평 확장하기 쉽습니다.

상태 유지 구성 요소는 모든 입력 이벤트를 처리하되 리더가 아닌 복제본은 외부로 결과 이벤트를 발행하지 않는 구조를 사용할 수 있습니다.

<div style="margin-left:3rem">
    <img src="./images/leader-election.png" alt="리더 선출" width="500" />
</div>

프라이머리 복제본 장애는 하트비트로 감지할 수 있습니다.

같은 서버 내부의 프로세스 복제만으로는 서버 전체 장애를 견디지 못하므로 서버 단위의 hot/warm 복제본을 별도로 구성할 수 있습니다.

이벤트 저장소를 서버 간 복제할 때는 매우 낮은 지연 시간을 위해 신뢰성 계층을 추가한 UDP 기반 전송 같은 방식도 고려할 수 있습니다.

### **장애 내성**
Warm 인스턴스까지 모두 사용할 수 없는 저확률 장애도 고려해야 합니다.

대규모 시스템에서는 자연재해나 데이터 센터 전체 장애 등에 대비해 핵심 데이터를 서로 다른 도시나 리전의 데이터 센터로 복제할 수 있습니다.

고려할 질문:
 * 프라이머리가 중단되면 언제 어떤 조건으로 백업 인스턴스로 failover할 것인가?
 * 여러 백업 중 새 리더를 어떻게 선택할 것인가?
 * 필요한 복구 시간 목표(RTO)는 얼마인가?
 * 어떤 기능을 우선 복구해야 하는가? 일부 기능을 제한한 degraded mode로 운영할 수 있는가?

대응 방법:
 * 소프트웨어 버그가 프라이머리와 복제본에 동시에 영향을 줄 수 있으므로 chaos engineering과 장애 주입 테스트로 실패 모드를 사전에 탐색합니다.
 * 운영 경험이 부족한 초기에는 자동 failover를 무조건 적용하기보다 수동 절차와 검증을 병행할 수 있습니다.
 * 프라이머리 장애 시 Raft 같은 리더 선출 메커니즘으로 새 리더를 선택할 수 있습니다.

서버 간 복제 예시:

<div style="margin-left:3rem">
    <img src="./images/replication-across-servers.png" alt="서버 간 복제" width="500" />
</div>

리더 선출 term 예시:

<div style="margin-left:3rem">
    <img src="./images/leader-election-terms.png" alt="리더 선출 Term" width="500" />
</div>

Raft의 동작 방식은 [이 자료](https://thesecretlivesofdata.com/raft/)를 참고할 수 있습니다.

마지막으로 허용 가능한 데이터 손실량도 정의해야 합니다. 이는 백업과 복제 정책, RPO에 영향을 줍니다.

증권 거래소에서는 주문과 체결 데이터 손실 허용 범위를 매우 엄격하게 잡아야 하므로 짧은 주기의 백업과 합의 기반 복제를 고려합니다.

### **매칭 알고리즘**
매칭 로직의 의사코드 예시는 다음과 같습니다.
```
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);

}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }

    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

이 매칭 알고리즘은 같은 가격 수준 안에서 먼저 들어온 주문을 먼저 매칭하는 FIFO 원칙을 사용합니다.

### **결정성(Determinism)**
이 설계에서는 Sequencer가 이벤트 순서를 부여해 동일한 순서의 입력을 재생할 수 있도록 합니다.

실제 벽시계 시각 자체보다 이벤트의 논리적 처리 순서가 중요합니다.

<div style="margin-left:3rem">
    <img src="./images/determinism.png" alt="결정성" width="500" />
</div>

기능적 결정성과 별개로 **지연 시간의 안정성**도 측정해야 합니다. 99p, 99.99p 같은 꼬리 지연 시간을 지속적으로 모니터링할 수 있습니다.

Java 같은 런타임의 Garbage Collection 이벤트는 지연 시간 급증 원인이 될 수 있으므로 저지연 시스템에서는 메모리 할당 패턴과 런타임 튜닝도 중요합니다.

### **Market Data Publisher 최적화**
Market Data Publisher는 Matching Engine의 체결 결과를 받아 주문장과 캔들스틱 차트를 재구성합니다.

메모리는 유한하므로 모든 과거 캔들을 메모리에 유지하지 않습니다. 클라이언트는 필요한 시간 해상도를 선택할 수 있으며 더 세밀한 시장 데이터는 별도 상품으로 제공할 수도 있습니다.

<div style="margin-left:3rem">
    <img src="./images/market-data-publisher.png" alt="Market Data Publisher" width="500" />
</div>

**Ring Buffer(원형 버퍼)**는 고정 크기 큐의 끝과 시작이 연결된 자료구조입니다. 메모리를 미리 할당해 런타임 할당을 줄일 수 있고 구현 방식에 따라 lock-free 구조로 만들 수 있습니다.

캐시 라인 경합을 줄이기 위해 padding을 사용해 자주 갱신되는 sequence 값이 다른 공유 변수와 같은 cache line에 들어가지 않도록 하는 기법도 사용할 수 있습니다.

### **시장 데이터 배포 공정성과 멀티캐스트**
구독자 간 시장 데이터 수신 시간 차이가 거래상 이점으로 이어질 수 있으므로 시장 데이터 배포의 공정성을 중요하게 관리해야 합니다.

이를 위해 데이터 배포에 multicast와 신뢰성 보완 UDP 프로토콜을 사용할 수 있습니다.

네트워크 전송 방식은 개념적으로 다음과 같이 나눌 수 있습니다.
 * **Unicast:** 하나의 소스에서 하나의 목적지로 전송
 * **Broadcast:** 하나의 소스에서 동일 브로드캐스트 도메인의 전체 대상에 전송
 * **Multicast:** 하나의 소스에서 특정 멀티캐스트 그룹의 여러 호스트로 전송

Multicast는 동일 데이터의 중복 전송을 줄이고 여러 구독자에게 효율적으로 배포하는 데 사용할 수 있습니다.

UDP는 기본적으로 패킷 전달을 보장하지 않으므로 sequence number, gap detection, retransmission 같은 신뢰성 계층을 별도로 설계할 수 있습니다.

### **Colocation**
거래소는 브로커가 거래소와 같은 데이터 센터에 서버를 설치하는 **colocation** 서비스를 제공할 수 있습니다.

물리적 네트워크 거리를 줄여 지연 시간을 크게 낮출 수 있는 서비스입니다.

### **네트워크 보안**
거래소는 일부 서비스가 인터넷에 노출되므로 DDoS 대응이 중요합니다.

대응 방법:
 * 공개 서비스/데이터와 핵심 비공개 거래 서비스를 네트워크 수준에서 분리해 공격 영향 범위를 제한합니다.
 * 변경 빈도가 낮은 공개 데이터를 캐시 계층에 저장합니다.
 * 캐싱하기 쉬운 URL 구조를 사용합니다. 예: `https://my.website.com/data/recent`는 임의 범위 쿼리를 받는 `https://my.website.com/data?from=123&to=456`보다 CDN/WAF 캐시에 유리할 수 있습니다.
 * 효과적인 allowlist/blocklist 정책을 운영합니다.
 * 요청 제한(rate limiting)으로 과도한 트래픽을 완화합니다.

---

## 4단계: 마무리
추가로 논의할 수 있는 내용은 다음과 같습니다.
 * 모든 거래소가 핵심 기능 전체를 하나의 대형 서버에 배치하는 것은 아니지만, 극단적인 저지연 목표 때문에 일부 시스템에서는 강하게 통합된 단일 서버 또는 소수의 고성능 노드를 사용하기도 합니다.
 * 전통적인 중앙 지정가 주문장 거래소와 달리 일부 현대 금융 시스템에는 클라우드 인프라나 AMM(Automated Market Maker) 같은 다른 거래 모델도 사용됩니다.