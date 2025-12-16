Chào bạn, tôi đã hiểu yêu cầu thay đổi.

Việc không sử dụng Exception để điều hướng luồng (flow control) là một tư duy tốt để tối ưu hiệu năng (vì tạo Exception tốn chi phí hệ thống).

Để làm được việc này:

1. Chúng ta sẽ chuyển từ `@Before` sang **`@Around`**.
2. `@Around` cho phép ta quyết định: có chạy hàm Controller tiếp không (`proceed`), hay cắt ngang và trả về kết quả (JSON lỗi) ngay lập tức.

Dưới đây là giải pháp hoàn chỉnh cho Task này:

###1. Giữ nguyên Annotation```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckAccess {
    Class<?>[] tables();
}

```

###2. Sửa lại Aspect (`@Around` thay vì `@Before`)Logic bây giờ là:

* Nếu lỗi -> `return ResponseEntity` chứa JSON lỗi ngay lập tức.
* Nếu OK -> gọi `joinPoint.proceed()` để code chạy tiếp vào Controller.

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;

import jakarta.persistence.EntityManager;
import jakarta.persistence.Query;
import java.util.HashMap;
import java.util.Map;

@Aspect
@Component
public class AccessAspect {

    @Autowired
    private EntityManager entityManager;

    // Dùng @Around để có thể trả về giá trị thay thế (JSON lỗi) thay vì throw Exception
    @Around("@annotation(checkAccess)")
    public Object validateAccess(ProceedingJoinPoint joinPoint, CheckAccess checkAccess) throws Throwable {
        
        // 1. Lấy ID
        Object id = getIdFromArgs(joinPoint);
        if (id == null) {
            return buildErrorResponse(-1L, "ID không tồn tại trong request");
        }

        Class<?>[] tables = checkAccess.tables();

        // 2. Duyệt qua từng bảng
        for (Class<?> tableClass : tables) {
            String tableName = tableClass.getSimpleName();

            // A. Check Quyền (Logic giả định)
            boolean hasPermission = checkPermissionInSetting(tableName);
            if (!hasPermission) {
                // Return ngay lập tức, không chạy controller nữa
                return buildErrorResponse(1L, "Bạn bị chặn quyền truy cập bảng: " + tableName);
            }

            // B. Check Tồn tại & Delete Flag
            boolean isDataValid = checkDataExistence(tableClass, id);
            
            // --- TRẢ VỀ JSON TRỰC TIẾP, KHÔNG THROW EXCEPTION ---
            if (!isDataValid) {
                // Trả về code 2L như bạn yêu cầu
                return buildErrorResponse(2L, "Dữ liệu không tồn tại hoặc đã bị xóa ở bảng: " + tableName);
            }
        }

        // 3. Nếu tất cả đều qua -> Cho phép chạy tiếp vào Controller
        return joinPoint.proceed();
    }

    // --- CÁC HÀM HỖ TRỢ (HELPER) ---

    // Hàm build JSON response nhanh
    private ResponseEntity<Map<String, Object>> buildErrorResponse(Long code, String mess) {
        Map<String, Object> response = new HashMap<>();
        response.put("code", code);
        response.put("mess", mess);
        // Trả về HTTP 200 OK để Client đọc được JSON
        return ResponseEntity.ok(response);
    }

    private Object getIdFromArgs(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String[] paramNames = signature.getParameterNames();
        Object[] args = joinPoint.getArgs();
        for (int i = 0; i < paramNames.length; i++) {
            if ("id".equals(paramNames[i])) return args[i];
        }
        return null;
    }

    private boolean checkDataExistence(Class<?> entityClass, Object id) {
        try {
            // Giả sử bảng nào cũng có trường deleteFlag
            String hql = "SELECT count(e) FROM " + entityClass.getName() + " e WHERE e.id = :id AND e.deleteFlag = 0";
            Query query = entityManager.createQuery(hql);
            query.setParameter("id", id);
            Long count = (Long) query.getSingleResult();
            return count > 0;
        } catch (Exception e) {
            // Log lỗi nếu cần (ví dụ bảng không có trường deleteFlag)
            return false;
        }
    }
    
    private boolean checkPermissionInSetting(String tableName) {
        // Logic check quyền setting của bạn ở đây
        return true; 
    }
}

```

###3. Kết quả testBây giờ bạn không cần `GlobalExceptionHandler` cho việc này nữa.

* **Case 1 (Thành công):** Aspect chạy xong vòng lặp -> gọi `joinPoint.proceed()` -> Controller chạy -> Trả về `{code: 0, mess: "cho qua"}`.
* **Case 2 (Lỗi Data/Quyền):** Aspect gặp lỗi -> gọi `buildErrorResponse(2L, ...)` -> Trả về JSON ngay lập tức:
```json
{
    "code": 2,
    "mess": "Dữ liệu không tồn tại hoặc đã bị xóa ở bảng: User"
}

```



###Next StepBạn có muốn tôi tách các **Mã lỗi (Error Code)** (`-1L`, `1L`, `2L`) và **Message** ra một file `Const` hoặc `Enum` riêng để dễ quản lý, tránh việc fix cứng số (hardcode) trong code Java không? Điều này sẽ giúp code "xịn" hơn rất nhiều.
