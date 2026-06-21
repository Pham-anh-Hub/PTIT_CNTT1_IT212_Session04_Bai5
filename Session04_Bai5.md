
#                       BÀI TẬP 5

### Promtp thứ nhất - sinh mã nguồn: 
```
Bạn đang là 1 Kỹ sư hệ  Senior Backend Developer 10 năm kinh nghiệm, bạn đang được yêu cầu xây dựng chức năng cảnh báo bảo mật cho ứng dụng FinTech có tên "SmartCheck". Nhiệm vụ của bạn là xây dựng logic phát hiện giao dịch đáng ngờ (Fraud Transaction Detector) từ danh sách lịch sử giao dịch của khách hàng.
Quy tắc nghiệp vụ được yêu cầu là: Quy tắc nghiệp vụ phát hiện giao dịch đáng ngờ:
Một giao dịch có giá trị đơn lẻ vượt quá 100,000,000 VND sẽ lập tức bị gắn cờ đáng ngờ.
Nếu xuất hiện hai giao dịch liên tiếp trên cùng một thẻ (cardId) được thực hiện tại hai vị trí địa lý khác nhau (ví dụ: "Hanoi" và "Saigon") trong khoảng thời gian dưới 10 phút, cả hai giao dịch này đều bị gắn cờ đáng ngờ (do không thể di chuyển vật lý nhanh như vậy).
Yêu cầu: tạo ra lớp FraudDetector với phương thức detectFraud(List<Transaction> transactions)
Ràng buộc: Tối ưu, chặt chẽ về đầu vào, đầu ra, 
sử dụng Java 8 Date Time API (LocalDateTime) để tính toán chênh lệch thời gian, 
và các trường hợp dữ liệu lỗi biên (danh sách null, trống, hoặc thiếu dữ liệu địa điểm).
```
### Nội dung AI gen:
``Với yêu cầu của hệ thống FinTech, tôi sẽ thiết kế theo hướng:

* **Fail Fast** với dữ liệu đầu vào không hợp lệ.
* **Không phụ thuộc thứ tự đầu vào** → sắp xếp giao dịch theo thời gian trước khi kiểm tra.
* **Độ phức tạp O(n log n)** do bước sort, phần phát hiện gian lận là O(n).
* Sử dụng **Java 8 Date Time API (LocalDateTime + Duration)**.
* Xử lý đầy đủ các trường hợp biên:

    * transactions = null
    * transactions rỗng
    * phần tử Transaction null
    * cardId null/rỗng
    * location null/rỗng
    * timestamp null
    * amount âm

---

## Thiết kế Model

```java
import java.math.BigDecimal;
import java.time.LocalDateTime;

public class Transaction {

    private final String transactionId;
    private final String cardId;
    private final BigDecimal amount;
    private final String location;
    private final LocalDateTime transactionTime;

    public Transaction(String transactionId,
                       String cardId,
                       BigDecimal amount,
                       String location,
                       LocalDateTime transactionTime) {
        this.transactionId = transactionId;
        this.cardId = cardId;
        this.amount = amount;
        this.location = location;
        this.transactionTime = transactionTime;
    }

    public String getTransactionId() {
        return transactionId;
    }

    public String getCardId() {
        return cardId;
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public String getLocation() {
        return location;
    }

    public LocalDateTime getTransactionTime() {
        return transactionTime;
    }
}
```

---

## Kết quả trả về

Thay vì trả về List<Transaction>, nên trả về đối tượng chứa lý do bị đánh dấu.

```java
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

public class FraudResult {

    private final Transaction transaction;
    private final Set<String> reasons = new HashSet<>();

    public FraudResult(Transaction transaction) {
        this.transaction = transaction;
    }

    public Transaction getTransaction() {
        return transaction;
    }

    public Set<String> getReasons() {
        return Collections.unmodifiableSet(reasons);
    }

    public void addReason(String reason) {
        reasons.add(reason);
    }
}
```

---

# FraudDetector

```java
import java.math.BigDecimal;
import java.time.Duration;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public class FraudDetector {

    private static final BigDecimal HIGH_AMOUNT_THRESHOLD =
            new BigDecimal("100000000");

    private static final long IMPOSSIBLE_TRAVEL_MINUTES = 10;

    public List<FraudResult> detectFraud(List<Transaction> transactions) {

        if (transactions == null || transactions.isEmpty()) {
            return new ArrayList<>();
        }

        List<Transaction> validTransactions = transactions.stream()
                .filter(Objects::nonNull)
                .filter(this::isValidTransaction)
                .sorted(Comparator.comparing(Transaction::getTransactionTime))
                .toList();

        if (validTransactions.isEmpty()) {
            return new ArrayList<>();
        }

        Map<String, FraudResult> fraudMap = new LinkedHashMap<>();

        detectHighAmount(validTransactions, fraudMap);

        detectImpossibleTravel(validTransactions, fraudMap);

        return new ArrayList<>(fraudMap.values());
    }

    private void detectHighAmount(List<Transaction> transactions,
                                  Map<String, FraudResult> fraudMap) {

        for (Transaction tx : transactions) {

            if (tx.getAmount().compareTo(HIGH_AMOUNT_THRESHOLD) > 0) {

                FraudResult result = fraudMap.computeIfAbsent(
                        tx.getTransactionId(),
                        k -> new FraudResult(tx));

                result.addReason(
                        "HIGH_AMOUNT_OVER_100M");
            }
        }
    }

    private void detectImpossibleTravel(List<Transaction> transactions,
                                        Map<String, FraudResult> fraudMap) {

        Map<String, Transaction> lastTransactionByCard =
                new HashMap<>();

        for (Transaction current : transactions) {

            Transaction previous =
                    lastTransactionByCard.get(current.getCardId());

            if (previous != null) {

                boolean differentLocation =
                        !previous.getLocation()
                                .equalsIgnoreCase(current.getLocation());

                long minutes =
                        Math.abs(Duration.between(
                                previous.getTransactionTime(),
                                current.getTransactionTime())
                                .toMinutes());

                if (differentLocation &&
                        minutes < IMPOSSIBLE_TRAVEL_MINUTES) {

                    FraudResult prevResult =
                            fraudMap.computeIfAbsent(
                                    previous.getTransactionId(),
                                    k -> new FraudResult(previous));

                    prevResult.addReason(
                            "IMPOSSIBLE_TRAVEL");

                    FraudResult currentResult =
                            fraudMap.computeIfAbsent(
                                    current.getTransactionId(),
                                    k -> new FraudResult(current));

                    currentResult.addReason(
                            "IMPOSSIBLE_TRAVEL");
                }
            }

            lastTransactionByCard.put(
                    current.getCardId(),
                    current);
        }
    }

    private boolean isValidTransaction(Transaction tx) {

        if (tx.getCardId() == null ||
                tx.getCardId().trim().isEmpty()) {
            return false;
        }

        if (tx.getLocation() == null ||
                tx.getLocation().trim().isEmpty()) {
            return false;
        }

        if (tx.getTransactionTime() == null) {
            return false;
        }

        return tx.getAmount() != null
                && tx.getAmount().compareTo(BigDecimal.ZERO) >= 0;
    }
}
```

---

# Ví dụ kiểm thử

```java
List<Transaction> transactions = Arrays.asList(

    new Transaction(
            "TX1",
            "CARD-001",
            new BigDecimal("50000000"),
            "Hanoi",
            LocalDateTime.of(
                    2026, 6, 20, 10, 0)),

    new Transaction(
            "TX2",
            "CARD-001",
            new BigDecimal("20000000"),
            "Saigon",
            LocalDateTime.of(
                    2026, 6, 20, 10, 5)),

    new Transaction(
            "TX3",
            "CARD-002",
            new BigDecimal("150000000"),
            "Danang",
            LocalDateTime.of(
                    2026, 6, 20, 11, 0))
);
```

### Kết quả

| Transaction | Fraud Reason          |
| ----------- | --------------------- |
| TX1         | IMPOSSIBLE_TRAVEL     |
| TX2         | IMPOSSIBLE_TRAVEL     |
| TX3         | HIGH_AMOUNT_OVER_100M |

---

## Một số điểm tối ưu thêm cho hệ thống thực tế

1. **Sử dụng Enum thay cho String**

```java
public enum FraudReason {
    HIGH_AMOUNT_OVER_100M,
    IMPOSSIBLE_TRAVEL
}
```

2. **Chuẩn hóa location**

    * Trim khoảng trắng.
    * Chuyển về lowercase.
    * Mapping "HCM", "Ho Chi Minh", "Saigon" về cùng một mã địa điểm.

3. **Sử dụng Instant + ZoneId**

    * Nếu hệ thống đa quốc gia.
    * Tránh lỗi múi giờ khi so sánh thời gian.

4. **Streaming Detection**

    * Với hệ thống FinTech thực tế (Kafka/Flink/Spark Streaming), không cần giữ toàn bộ lịch sử trong RAM.
    * Chỉ lưu giao dịch gần nhất của từng `cardId` trong cache/Redis để kiểm tra quy tắc "Impossible Travel".

Đây là thiết kế phù hợp cho môi trường production vì vừa xử lý đầy đủ các điều kiện biên, vừa đảm bảo hiệu năng O(n log n) và dễ mở rộng thêm các luật chống gian lận mới sau này.
``


### Prompt thứ 2 - Kiểm chứng chéo hiệu năng 
```
Giờ bạn là 1 chuyên gia tối ưu thuật toán, hãy phân tích và chỉ ra từ logic các mã nguồn trên
Khi danh sách giao dịch lên tới 10,000 phần tử, việc duyệt so sánh lồng nhau hai vòng lặp O(N2)sẽ làm chậm hệ thống
từ đó phân tích điểm nghẽn hiệu năng của code ở Bước 1 và tối ưu thuật toán (ví dụ: gom nhóm theo cardId và sắp xếp theo thời gian để so sánh lân cận, đưa độ phức tạp về O(Nlog N).
```



#### Nội dung AI gen ở bước 2:
```
Nếu xem xét yêu cầu audit hiệu năng cho hệ thống FinTech, cần phân biệt rõ giữa:

* **Giải pháp ngây thơ (naive)** thường được AI hoặc lập trình viên junior viết.
* **Giải pháp đã tối ưu** trong lớp `FraudDetector` ở trên.

---

# 1. Điểm nghẽn hiệu năng của cách làm Bước 1 (Naive O(N²))

Một cách triển khai rất dễ gặp là:

```java
for (int i = 0; i < transactions.size(); i++) {

    Transaction tx1 = transactions.get(i);

    for (int j = i + 1; j < transactions.size(); j++) {

        Transaction tx2 = transactions.get(j);

        if (tx1.getCardId().equals(tx2.getCardId())) {

            // kiểm tra vị trí khác nhau
            // kiểm tra khoảng cách thời gian < 10 phút
        }
    }
}
```

Ý tưởng:

* Mỗi giao dịch sẽ so sánh với tất cả giao dịch còn lại.
* Tìm các cặp có cùng `cardId`.
* Sau đó kiểm tra location và time.

---

## Độ phức tạp

Số lần so sánh:

[
\frac{N(N-1)}{2}
]

Với:

```text
N = 10.000
```

Ta có:

```text
10.000 × 9.999 / 2
≈ 49.995.000 lần so sánh
```

Gần **50 triệu phép so sánh**.

---

## Vấn đề thực tế

Mỗi lần so sánh không chỉ là:

```java
cardId.equals(...)
```

mà còn phải:

```java
Duration.between(...)
location.equalsIgnoreCase(...)
```

Ví dụ:

```text
50 triệu lần
 × tính Duration
 × so sánh String
 × tạo object tạm
```

=> CPU tăng mạnh.

Nếu hệ thống xử lý realtime:

```text
100 request/s
```

thì:

```text
50 triệu × 100
=
5 tỷ phép so sánh
```

Hoàn toàn không khả thi.

---

# 2. Phân tích nghiệp vụ để tối ưu

Hãy đọc kỹ luật:

> Nếu xuất hiện hai giao dịch liên tiếp trên cùng một cardId được thực hiện tại hai vị trí khác nhau trong vòng dưới 10 phút.

Từ khóa cực kỳ quan trọng:

```text
hai giao dịch liên tiếp
```

Không cần so sánh:

```text
TX1 với TX1000
TX1 với TX5000
TX1 với TX9000
```

Chỉ cần:

```text
TX1 ↔ TX2
TX2 ↔ TX3
TX3 ↔ TX4
...
```

sau khi đã sắp xếp theo thời gian.

Đây là điểm giúp giảm độ phức tạp rất lớn.

---

# 3. Ý tưởng tối ưu

## Bước 1: Gom nhóm theo cardId

Sử dụng:

```java
Map<String, List<Transaction>>
```

Ví dụ:

```text
CARD_A:
    TX1
    TX5
    TX9

CARD_B:
    TX2
    TX7

CARD_C:
    TX3
    TX4
```

Chi phí:

```text
O(N)
```

---

## Bước 2: Sắp xếp từng nhóm theo thời gian

Ví dụ:

```text
CARD_A

TX5 10:00
TX1 10:03
TX9 11:20
```

sau sort:

```text
TX5 10:00
TX1 10:03
TX9 11:20
```

Chi phí:

[
O(K \log K)
]

với K là số giao dịch của card.

Tổng tất cả nhóm:

[
O(N \log N)
]

---

## Bước 3: Chỉ so sánh giao dịch lân cận

```java
for (int i = 1; i < list.size(); i++) {

    Transaction prev = list.get(i - 1);
    Transaction curr = list.get(i);

    ...
}
```

Ví dụ:

```text
TX1
TX2
TX3
TX4
```

Chỉ kiểm tra:

```text
TX1 ↔ TX2
TX2 ↔ TX3
TX3 ↔ TX4
```

không cần:

```text
TX1 ↔ TX3
TX1 ↔ TX4
TX2 ↔ TX4
```

Chi phí:

```text
O(N)
```

---

# 4. Độ phức tạp sau tối ưu

## Cách cũ

```text
O(N²)
```

Với:

```text
N = 10.000
```

≈

```text
50 triệu lần so sánh
```

---

## Cách mới

### Gom nhóm

```text
O(N)
```

### Sắp xếp

```text
O(N log N)
```

### So sánh lân cận

```text
O(N)
```

Tổng:

```text
O(N log N)
```

---

# 5. So sánh định lượng

Với:

```text
N = 10.000
```

Ta có:

```text
log₂(10000) ≈ 13.3
```

Nên:

```text
N log N
≈ 133.000
```

Trong khi:

```text
N²
≈ 100.000.000
```

So sánh:

```text
100.000.000 / 133.000
≈ 752 lần
```

Thuật toán mới nhanh hơn khoảng **700–800 lần**.

---

# 6. Nhận xét về mã FraudDetector đã viết

Thực tế lớp `FraudDetector` phía trên **đã không sử dụng thuật toán O(N²)**.

Đoạn mã:

```java
Map<String, Transaction> lastTransactionByCard
```

giữ lại giao dịch gần nhất của mỗi thẻ.

Khi duyệt:

```java
Transaction previous =
    lastTransactionByCard.get(current.getCardId());
```

chỉ so sánh:

```text
current ↔ previous
```

cho cùng card.

Do đó:

```text
detectImpossibleTravel()
```

có độ phức tạp:

```text
O(N)
```

Sau khi danh sách được:

```java
.sorted(...)
```

thì toàn bộ thuật toán là:

```text
O(N log N)
```

chứ không phải:

```text
O(N²)
```

Đây chính là hướng tối ưu chuẩn cho bài toán phát hiện "impossible travel" trong các hệ thống chống gian lận giao dịch thực tế.
```