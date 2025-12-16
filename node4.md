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

Đã rõ, bạn muốn nhấn mạnh việc **Annotation nhận vào một mảng (danh sách) các bảng** và logic phải xử lý duyệt qua **từng bảng** trong danh sách đó để kiểm tra.

Nếu user không có quyền ở bảng bất kỳ trong danh sách, hoặc ID không tồn tại/đã xóa ở bảng đó, thì đều chặn.

Dưới đây là code đã tinh chỉnh để xử lý **mảng table**.

###1. Annotation (`@CheckAccess`)Tôi đặt tên tham số là `tables` để bạn truyền vào danh sách dạng mảng `{TableA.class, TableB.class}`.

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckAccess {
    // Nhận vào một mảng các Class (đại diện cho các bảng)
    Class<?>[] tables();
}

```

---

###2. Aspect Xử lý Logic (`AccessAspect`) - Core LogicĐây là phần quan trọng nhất. Tôi sẽ dùng vòng lặp `for` để duyệt qua danh sách bảng bạn truyền vào.

Logic trong vòng lặp:

1. Check **Setting Quyền** (User có được truy cập bảng này không?).
2. Check **Data** (ID có tồn tại trong bảng này và `delete_flag == 0` không?).

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import jakarta.persistence.EntityManager;
import jakarta.persistence.Query;

@Aspect
@Component
public class AccessAspect {

    @Autowired
    private EntityManager entityManager;

    @Before("@annotation(checkAccess)")
    public void validateAccess(JoinPoint joinPoint, CheckAccess checkAccess) {
        
        // 1. Lấy ID từ tham số request
        Object id = getIdFromArgs(joinPoint);
        if (id == null) {
            throw new AccessDeniedException("ID không tồn tại trong request");
        }

        // 2. Lấy danh sách bảng từ Annotation
        Class<?>[] tables = checkAccess.tables();

        // 3. Duyệt qua từng bảng để kiểm tra
        for (Class<?> tableClass : tables) {
            String tableName = tableClass.getSimpleName(); // Lấy tên bảng (Entity)

            // --- BƯỚC 3.1: CHECK QUYỀN TRONG BẢNG SETTING ---
            // Giả sử bạn có hàm check quyền. Logic: User hiện tại + Tên bảng -> Có quyền không?
            boolean hasPermission = checkPermissionInSetting(tableName); 
            
            if (!hasPermission) {
                // Trả về lỗi nếu bị chặn quyền
                throw new AccessDeniedException("Bạn không có quyền truy cập bảng: " + tableName);
            }

            // --- BƯỚC 3.2: CHECK TỒN TẠI VÀ DELETE_FLAG ---
            boolean isDataValid = checkDataExistence(tableClass, id);
            
            if (!isDataValid) {
                // Trả về lỗi nếu ID không có hoặc đã bị xoá
                throw new AccessDeniedException("Dữ liệu không tồn tại hoặc đã bị xóa ở bảng: " + tableName);
            }
        }
        
        // Nếu chạy hết vòng for mà không Exception -> Controller sẽ trả về {code: 0, mess: "cho qua"}
    }

    // --- Các hàm phụ trợ ---

    // Giả lập check bảng Setting quyền
    private boolean checkPermissionInSetting(String tableName) {
        // TODO: Query vào bảng 'permission_setting' của bạn
        // Query: SELECT allow FROM permission_setting WHERE user_id = ? AND table_name = ?
        // Ở đây tôi return true để demo luồng chạy
        return true; 
    }

    // Query kiểm tra ID và Delete Flag
    private boolean checkDataExistence(Class<?> entityClass, Object id) {
        // Query: SELECT count FROM Table WHERE id = :id AND delete_flag = 0
        try {
            String hql = "SELECT count(e) FROM " + entityClass.getName() + " e WHERE e.id = :id AND e.deleteFlag = 0";
            Query query = entityManager.createQuery(hql);
            query.setParameter("id", id);
            Long count = (Long) query.getSingleResult();
            return count > 0;
        } catch (Exception e) {
            // Nếu bảng không có trường deleteFlag hoặc lỗi SQL -> coi như không hợp lệ
            return false;
        }
    }

    private Object getIdFromArgs(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String[] paramNames = signature.getParameterNames();
        Object[] args = joinPoint.getArgs();
        for (int i = 0; i < paramNames.length; i++) {
            if ("id".equals(paramNames[i])) return args[i];
        }
        return null;
    }
}

```

---

###3. ControllerAdvice (Trả về JSON đúng chuẩn)Phần này giữ nguyên để đảm bảo output đúng yêu cầu của bạn.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<Map<String, Object>> handleAccessDenied(AccessDeniedException ex) {
        Map<String, Object> response = new HashMap<>();
        response.put("code", 1); // Code lỗi
        response.put("mess", ex.getMessage()); // Message chi tiết lỗi
        return new ResponseEntity<>(response, HttpStatus.OK); // Trả về 200 OK nhưng body báo lỗi (hoặc 403 tuỳ bạn)
    }
}

```

---

###4. Cách sử dụng (Truyền danh sách mảng)Bây giờ tại Controller, bạn có thể truyền 1 hoặc nhiều bảng vào Annotation.

Ví dụ: API xoá một User, logic yêu cầu User đó phải tồn tại trong bảng `User` và cả bảng `UserProfile` (ví dụ logic ràng buộc chặt).

```java
@RestController
@RequestMapping("/api")
public class UserController {

    // TRUYỀN MẢNG TABLE VÀO ĐÂY
    @CheckAccess(tables = {User.class, UserProfile.class}) 
    @PostMapping("/users/delete/{id}")
    public ResponseEntity<Map<String, Object>> deleteUser(@PathVariable Long id) {
        
        // Nếu code vào được đây nghĩa là:
        // 1. User có quyền với cả bảng User và UserProfile.
        // 2. ID này tồn tại và chưa bị xoá ở CẢ 2 bảng.
        
        Map<String, Object> response = new HashMap<>();
        response.put("code", 0);
        response.put("mess", "cho qua");
        return ResponseEntity.ok(response);
    }
}

```

###Next StepBạn có muốn tôi viết chi tiết câu Query SQL trong hàm `checkPermissionInSetting` để map đúng với cấu trúc bảng phân quyền (ví dụ: bảng `role_permission`) trong database thực tế của bạn không?


3. **Kịch bản 1 (Lỗi):** Nếu ID = 1 có `deleteFlag = 1` -> Ném `AccessDeniedException`. `GlobalExceptionHandler` bắt và trả về JSON: `{code: 1, mess: "Dữ liệu không tồn tại hoặc đã bị xoá..."}`.
4. **Kịch bản 2 (Thành công):** Nếu mọi thứ OK -> Aspect cho phép chạy vào hàm `getUserDetail`. Hàm này trả về JSON: `{code: 0, mess: "cho qua"}`.

###Next StepBạn có muốn tôi viết thêm phần `PermissionService` để kết nối cụ thể với "bảng setting quyền" trong database của bạn (sử dụng JPA Repository) không?
