Chào bạn, với vai trò là **BrSE (Bridge SE)** kiêm **Java Developer**, tôi hiểu rõ yêu cầu của bạn. Đây là một bài toán điển hình về **AOP (Aspect Oriented Programming)** trong Spring Boot để xử lý các vấn đề cắt ngang (cross-cutting concerns) như bảo mật và kiểm tra dữ liệu.

Tôi sẽ thiết kế giải pháp theo hướng sạch sẽ, dễ bảo trì và đáp ứng đúng logic bạn yêu cầu:

1. **Input:** Annotation nhận danh sách các Entity (đại diện cho bảng).
2. **Logic:** Kiểm tra quyền + Kiểm tra sự tồn tại của ID + Kiểm tra `delete_flag`.
3. **Output:** JSON trả về kết quả (thành công hoặc lỗi).

Dưới đây là các bước thực hiện chi tiết:

---

###Bước 1: Tạo Annotation tuỳ chỉnh (`@CheckAccess`)Đầu tiên, chúng ta định nghĩa Annotation để đánh dấu vào các hàm cần kiểm tra. Thay vì truyền tên bảng dạng String (dễ sai sót), ta sẽ truyền `Class` của Entity (an toàn hơn trong Java).

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD) // Áp dụng cho method
@Retention(RetentionPolicy.RUNTIME) // Tồn tại lúc chạy
public @interface CheckAccess {
    // Danh sách các Entity (bảng) cần kiểm tra
    Class<?>[] targets();
}

```

---

###Bước 2: Tạo Custom ExceptionĐể trả về JSON lỗi đúng format, chúng ta nên ném ra một Exception riêng và bắt nó ở Global Exception Handler.

```java
public class AccessDeniedException extends RuntimeException {
    public AccessDeniedException(String message) {
        super(message);
    }
}

```

---

###Bước 3: Xử lý Logic với AOP (`AccessAspect`)Đây là phần "xử lý chính". Aspect sẽ chặn request trước khi nó vào Controller.

**Giả định kỹ thuật:**

1. Entity của bạn đều có trường `id` và `deleteFlag`.
2. Hàm Controller của bạn có tham số tên là `id` để xác định bản ghi cần check.
3. Bạn có một `PermissionService` để check bảng phân quyền.

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import jakarta.persistence.EntityManager;
import jakarta.persistence.Query;
import java.util.Arrays;
import java.util.List;

@Aspect
@Component
public class AccessAspect {

    @Autowired
    private EntityManager entityManager;

    // Giả sử bạn có service này để check bảng phân quyền
    // @Autowired
    // private PermissionService permissionService; 

    @Before("@annotation(checkAccess)")
    public void validateAccess(JoinPoint joinPoint, CheckAccess checkAccess) {
        
        // 1. Lấy ID từ tham số của hàm Controller
        Object id = getIdFromArgs(joinPoint);
        if (id == null) {
            throw new AccessDeniedException("ID không tồn tại trong request");
        }

        // 2. Lấy danh sách bảng (Entity) từ Annotation
        Class<?>[] entities = checkAccess.targets();

        // 3. Giả lập check quyền user hiện tại (Logic phân quyền của bạn)
        // Ví dụ: Long currentUserId = UserContext.getUserId();
        // boolean hasPermission = permissionService.check(currentUserId, entities);
        boolean hasPermission = true; // Tạm set true để demo
        
        if (!hasPermission) {
            throw new AccessDeniedException("Bạn bị chặn quyền truy cập vào tài nguyên này");
        }

        // 4. Check từng bảng xem ID có tồn tại và delete_flag = 0 (false) hay không
        for (Class<?> entityClass : entities) {
            checkEntityExistence(entityClass, id);
        }
        
        // Nếu chạy hết mà không lỗi -> Cho qua (Controller sẽ return code: 0)
    }

    // Helper: Tìm tham số có tên là "id" trong method
    private Object getIdFromArgs(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String[] paramNames = signature.getParameterNames();
        Object[] args = joinPoint.getArgs();

        for (int i = 0; i < paramNames.length; i++) {
            if ("id".equals(paramNames[i])) {
                return args[i];
            }
        }
        return null;
    }

    // Helper: Query dynamic vào DB để check ID và Delete Flag
    private void checkEntityExistence(Class<?> entityClass, Object id) {
        // Tạo câu query động: SELECT count(*) FROM Entity WHERE id = :id AND deleteFlag = 0
        String hql = "SELECT count(e) FROM " + entityClass.getName() + " e WHERE e.id = :id AND e.deleteFlag = 0"; // Hoặc e.deleteFlag = false tuỳ kiểu dữ liệu
        
        Query query = entityManager.createQuery(hql);
        query.setParameter("id", id);
        
        Long count = (Long) query.getSingleResult();

        if (count == 0) {
            // Trường hợp này bao gồm cả: ID không tồn tại HOẶC đã bị xoá mềm (deleteFlag = 1)
            throw new AccessDeniedException("Dữ liệu không tồn tại hoặc đã bị xoá (Table: " + entityClass.getSimpleName() + ")");
        }
    }
}

```

---

###Bước 4: Xử lý trả về JSON (`ControllerAdvice`)Để đảm bảo trả về format JSON `{code: ..., mess: ...}` như bạn yêu cầu khi có lỗi hoặc thành công.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Bắt lỗi khi logic trong Annotation chặn lại
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<Object> handleAccessDenied(AccessDeniedException ex) {
        Map<String, Object> response = new HashMap<>();
        response.put("code", 1); // Code lỗi
        response.put("mess", ex.getMessage()); // Lý do lỗi
        return new ResponseEntity<>(response, HttpStatus.FORBIDDEN); // 403 Forbidden
    }

    // Các exception khác...
}

```

---

###Bước 5: Cách sử dụng (Demo)Giả sử bạn có Entity `User` và `Order`.

**Entity:**

```java
@Entity
public class User {
    @Id
    private Long id;
    private Integer deleteFlag; // 0: active, 1: deleted
    //...
}

```

**Controller:**

```java
@RestController
@RequestMapping("/api")
public class DemoController {

    // Gắn Annotation, truyền vào class User
    @CheckAccess(targets = {User.class}) 
    @GetMapping("/users/{id}")
    public ResponseEntity<Map<String, Object>> getUserDetail(@PathVariable Long id) {
        
        // Nếu code chạy vào đến đây, tức là AOP đã cho qua
        Map<String, Object> response = new HashMap<>();
        response.put("code", 0);
        response.put("mess", "cho qua");
        response.put("data", "Thông tin user..."); // Data thực tế
        
        return ResponseEntity.ok(response);
    }
}

```

---

###Tóm tắt luồng hoạt động (Cho Dev Level 1)1. User gọi API `/api/users/1`.
2. **`AccessAspect`** nhảy vào chặn trước:
* Lấy ID = 1.
* Kiểm tra bảng phân quyền (Setting table).
* Chạy câu Query dynamic vào DB bảng `User`: `SELECT count... WHERE id=1 AND deleteFlag=0`.


3. **Kịch bản 1 (Lỗi):** Nếu ID = 1 có `deleteFlag = 1` -> Ném `AccessDeniedException`. `GlobalExceptionHandler` bắt và trả về JSON: `{code: 1, mess: "Dữ liệu không tồn tại hoặc đã bị xoá..."}`.
4. **Kịch bản 2 (Thành công):** Nếu mọi thứ OK -> Aspect cho phép chạy vào hàm `getUserDetail`. Hàm này trả về JSON: `{code: 0, mess: "cho qua"}`.

###Next StepBạn có muốn tôi viết thêm phần `PermissionService` để kết nối cụ thể với "bảng setting quyền" trong database của bạn (sử dụng JPA Repository) không?
