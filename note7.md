Để cho phép truyền `null` vào hàm 2 tham số trong Java, nguyên tắc vàng bạn cần nhớ là: **Không được sử dụng kiểu nguyên thủy (primitive types)** như `int`, `boolean`, `double`.

Thay vào đó, bạn phải sử dụng **Wrapper Classes** (Lớp bao) tương ứng (Ví dụ: `Integer`, `Boolean`, `Double`) hoặc các kiểu tham chiếu như `String`, `Object`.

Dưới đây là ví dụ chi tiết code thực hiện:

### 1. Khai báo hàm

Giả sử ta có hàm cập nhật thông tin User với 2 tham số tùy chọn: `age` (tuổi) và `status` (trạng thái).

```java
public class UserUpdater {

    /**
     * @param age: Sử dụng Integer thay vì int để có thể nhận null
     * @param status: Sử dụng Boolean thay vì boolean để có thể nhận null
     */
    public void updateUser(Integer age, Boolean status) {
        
        // 1. Xử lý tham số 'age'
        if (age != null) {
            System.out.println("Cập nhật tuổi thành: " + age);
            // Logic update DB ở đây
        } else {
            System.out.println("Không cập nhật tuổi (Giữ nguyên giá trị cũ)");
        }

        // 2. Xử lý tham số 'status'
        // Cách viết gọn dùng toán tử 3 ngôi (Ternary Operator)
        // Nếu status null -> mặc định là false, ngược lại lấy giá trị truyền vào
        boolean finalStatus = (status != null) ? status : false; 
        
        System.out.println("Trạng thái cuối cùng: " + finalStatus);
    }
}

```

### 2. Cách gọi hàm (Test case)

Bạn có thể gọi hàm theo các trường hợp sau:

```java
public static void main(String[] args) {
    UserUpdater updater = new UserUpdater();

    // Case 1: Truyền đầy đủ dữ liệu
    updater.updateUser(25, true);
    // Output: Cập nhật tuổi 25, Trạng thái true

    // Case 2: Truyền null cả hai (Chỉ muốn chạy logic mặc định)
    updater.updateUser(null, null);
    // Output: Giữ nguyên tuổi, Trạng thái false (giá trị default)

    // Case 3: Chỉ truyền 1 tham số
    updater.updateUser(30, null);
    // Output: Cập nhật tuổi 30, Trạng thái false
}

```

### 3. Cách viết hiện đại (Java 8+ Optional)

Nếu bạn muốn code trông "ngầu" hơn và chuẩn phong cách Java hiện đại (thường thấy trong các team senior), hãy dùng `Optional`. Nó giúp tránh lỗi `NullPointerException` một cách tường minh hơn.

```java
import java.util.Optional;

public void updateUserModern(Integer age, Boolean status) {
    // Nếu age có giá trị thì in ra, nếu null thì bỏ qua
    Optional.ofNullable(age)
            .ifPresent(a -> System.out.println("Updating age to: " + a));

    // Nếu status là null, trả về giá trị mặc định là true
    boolean effectiveStatus = Optional.ofNullable(status).orElse(true);
    System.out.println("Status to save: " + effectiveStatus);
}

```

---

### Lưu ý từ BrSE:

Trong các dự án thực tế, khi bạn thiết kế API hoặc hàm cho phép `null`:

1. **Validate:** Luôn phải kiểm tra `!= null` trước khi dùng (unboxing), nếu không sẽ bị lỗi "Chết người" là `NullPointerException` khi hệ thống cố chuyển `null` sang `int`.
2. **Documentation:** Cần comment rõ ràng tham số nào được phép null và nếu null thì hệ thống sẽ xử lý thế nào (lấy mặc định hay bỏ qua).

Bạn có muốn mình tích hợp ví dụ xử lý `null` này vào một **API `PUT` (Update)** trong Spring Boot không?
