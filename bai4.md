# BÀI TẬP 4: Lọc lỗi thông minh – Đừng ngắt mạch oan uổng

## Phần 1 – Thiết kế luồng

### Lưu đồ thuật toán

``` mermaid
flowchart TD
    A[Checkout-Service gọi Promo-Service] --> B{Kết quả gọi API?}
    B --> C[Có Exception]
    C --> D{Exception là loại nào?}

    D -- Lỗi nghiệp vụ --> E["VoucherNotFoundException\n(HTTP 404)"]
    E --> F["Không tính vào Failure Rate\n(ignoreExceptions)"]

    D -- Lỗi hệ thống --> G["TimeoutException\nConnectException"]
    G --> H["Đếm vào Failure Rate\n(recordExceptions)"]

    F --> I[Circuit Breaker quyết định\nOPEN hay CLOSED]
    H --> I
```

### Giải thích luồng

* Lỗi hệ thống như `TimeoutException` hoặc `ConnectException` cho thấy Promo-Service gặp sự cố thực sự, vì vậy phải tính vào Failure Rate để Circuit Breaker có thể chuyển sang OPEN khi vượt ngưỡng.

* Lỗi nghiệp vụ như `VoucherNotFoundException (HTTP 404)` chỉ có nghĩa là khách hàng nhập sai mã giảm giá, không phải Promo-Service bị sập. Vì vậy phải bỏ qua (ignore), tránh việc Circuit Breaker ngắt mạch oan.

## Phần 2 – `application.yml`

YAML

```
resilience4j:
  circuitbreaker:
    instances:
      promoClient:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true

        record-exceptions:
          - java.util.concurrent.TimeoutException
          - java.net.ConnectException
          - org.springframework.web.client.HttpServerErrorException

        ignore-exceptions:
          - com.storex.exception.VoucherNotFoundException
```

## Kết quả sau khi cấu hình

| Trường hợp | Circuit Breaker xử lý |
| --- | --- |
| `TimeoutException` | Đếm vào Failure Rate |
| `ConnectException` | Đếm vào Failure Rate |
| `HttpServerErrorException (5xx)` | Đếm vào Failure Rate |
| `VoucherNotFoundException (404)` | Bỏ qua, không làm tăng Failure Rate |

Với cấu hình này, hàng nghìn khách hàng nhập sai mã giảm giá sẽ không làm Circuit Breaker mở oan, trong khi các lỗi hệ thống thực sự vẫn được phát hiện và xử lý đúng.
