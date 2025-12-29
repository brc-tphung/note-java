Chào bạn, rất vui được hợp tác. Với vai trò là một **BrSE (Bridge System Engineer)** kiêm **Java Dev**, tôi hiểu rõ yêu cầu của bạn. Đây là một kịch bản rất phổ biến nhưng tiềm ẩn nhiều rủi ro nếu không xử lý đúng vòng đời của Transaction.

Vấn đề cốt lõi ở đây là: **Tính nhất quán (Consistency)**.

* Nếu gửi mail *trước* khi commit và commit thất bại  Khách hàng nhận được mail "thành công" nhưng dữ liệu không có (Phantom notification).
* Nếu gửi mail *trong* transaction  Việc gửi mail (I/O network) làm transaction kéo dài, gây lock database lâu, giảm hiệu năng.

Giải pháp chuẩn trong Spring Boot cho trường hợp này là sử dụng **`@TransactionalEventListener`**.

Dưới đây là giải pháp chi tiết mà tôi đề xuất để thực hiện task này.

---

### 1. Kiến trúc luồng xử lý (Logical Flow)

Chúng ta sẽ tách biệt việc lưu dữ liệu và gửi mail. Việc gửi mail chỉ được kích hoạt **SAU KHI** database đã commit thành công (`AFTER_COMMIT`).

### 2. Triển khai Code (Implementation)

Dưới đây là các bước coding cụ thể:

#### Bước 1: Tạo Event Class

Đây là object chứa thông tin cần thiết để gửi mail (ví dụ: email người nhận, nội dung, user ID).

```java
public class UserRegisteredEvent {
    private final String email;
    private final String username;

    public UserRegisteredEvent(String email, String username) {
        this.email = email;
        this.username = username;
    }

    // Getters...
    public String getEmail() { return email; }
    public String getUsername() { return username; }
}

```

#### Bước 2: Service xử lý nghiệp vụ (Publisher)

Tại đây, chúng ta thực hiện lưu DB và bắn event. Lưu ý là event được bắn ra nhưng Listener sẽ chưa chạy ngay lập tức nếu chúng ta cấu hình đúng.

```java
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void registerUser(UserDto userDto) {
        // 1. Thực hiện logic lưu vào Database
        User newUser = new User();
        newUser.setEmail(userDto.getEmail());
        newUser.setUsername(userDto.getUsername());
        userRepository.save(newUser);

        // 2. Publish event
        // Tại thời điểm này, transaction chưa commit xong.
        eventPublisher.publishEvent(new UserRegisteredEvent(newUser.getEmail(), newUser.getUsername()));
        
        // Transaction sẽ commit sau khi dòng này kết thúc
    }
}

```

#### Bước 3: Event Listener (Consumer)

Đây là phần quan trọng nhất. Chúng ta dùng `@TransactionalEventListener` thay vì `@EventListener` thường.

* `phase = TransactionPhase.AFTER_COMMIT`: Chỉ chạy khi Transaction cha đã commit thành công.
* `@Async`: Nên chạy việc gửi mail ở một thread khác để không block thread chính trả về response cho user.

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Component
public class UserNotificationListener {

    @Autowired
    private EmailService emailService; // Giả sử bạn đã có service gửi mail

    @Async // Chạy trên thread riêng biệt, không ảnh hưởng response time
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserRegisteredEvent(UserRegisteredEvent event) {
        // Code gửi mail ở đây
        System.out.println("Transaction committed. Bắt đầu gửi mail tới: " + event.getEmail());
        
        try {
            emailService.sendWelcomeMail(event.getEmail(), event.getUsername());
        } catch (Exception e) {
            // Log lỗi. Lưu ý: Vì transaction DB đã commit rồi, 
            // nếu gửi mail lỗi ở đây, DB sẽ KHÔNG rollback.
            System.err.println("Gửi mail thất bại: " + e.getMessage());
        }
    }
}

```

#### Bước 4: Cấu hình Async

Đừng quên bật tính năng Async trong main class hoặc config class.

```java
@EnableAsync
@SpringBootApplication
public class MyApplication { ... }

```

---

### 3. Phân tích rủi ro (Góc nhìn BrSE)

Là một BrSE, tôi cần cảnh báo bạn về các trường hợp biên (edge cases) để hệ thống vận hành ổn định:

| Kịch bản | Kết quả với giải pháp trên | Đánh giá |
| --- | --- | --- |
| **Lỗi khi lưu DB** | Transaction rollback. Event Listener **KHÔNG** được kích hoạt. Không có mail nào được gửi. | **Tốt (Correctness)** |
| **Lưu DB thành công, gửi mail lỗi** | DB đã commit. Mail không gửi được. Dữ liệu user đã có nhưng user không nhận được thông báo. | **Chấp nhận được** (nhưng cần cơ chế Retry) |
| **Server sập ngay sau khi commit DB nhưng trước khi gửi mail** | Event nằm trong bộ nhớ (in-memory) sẽ bị mất. Mail không bao giờ được gửi. | **Rủi ro** |

### 4. Đề xuất nâng cao (Nếu task yêu cầu độ tin cậy tuyệt đối)

Nếu nghiệp vụ yêu cầu **"Không bao giờ được mất mail"** (Reliability), giải pháp in-memory event phía trên là chưa đủ. Chúng ta cần áp dụng **Transactional Outbox Pattern**:

1. Trong cùng Transaction lưu User, lưu thêm một record vào bảng `outbox_event` (trạng thái `PENDING`).
2. Sau khi commit, một tiến trình background (hoặc Change Data Capture) sẽ đọc bảng `outbox_event` để gửi mail.
3. Gửi thành công thì cập nhật trạng thái `PROCESSED`.

---

### Tổng kết

Với task hiện tại, nếu hệ thống không quá khắt khe về việc *buộc phải gửi mail bằng mọi giá*, việc sử dụng **`@TransactionalEventListener(phase = AFTER_COMMIT)`** kết hợp **`@Async`** là giải pháp "ngon-bổ-rẻ", clean code và chuẩn Spring Boot nhất.

**Bước tiếp theo tôi có thể làm cho bạn:**
Bạn có muốn tôi viết thêm phần **Unit Test** để mô phỏng việc transaction rollback thì mail sẽ không được gửi (verify zero interactions) không?
