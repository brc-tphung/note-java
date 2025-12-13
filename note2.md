Tuyệt vời. Việc truyền tham số vào Annotation sẽ giúp bộ lọc của bạn trở nên **linh hoạt (dynamic)** hơn rất nhiều. Thay vì chỉ kiểm tra "Có/Không", bạn có thể kiểm tra "Quyền gì", "Level bao nhiêu", v.v.

Dưới đây là ví dụ nâng cấp: Chúng ta sẽ tạo annotation `@RoleCheck` để kiểm tra xem user có đúng quyền (ví dụ: `ADMIN` hoặc `USER`) hay không.

### 1\. Cập nhật Interface Annotation

Thêm phương thức vào trong interface. Tên phương thức chính là tên tham số bạn sẽ dùng.

```java
package com.example.demo.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RoleCheck {
    
    // Tham số bắt buộc: Role mà API yêu cầu
    String requiredRole(); 
    
    // (Tùy chọn) Có thể thêm nhiều tham số khác, ví dụ:
    // String message() default "Bạn không có quyền truy cập";
}
```

### 2\. Cập nhật Aspect để đọc tham số

Trong hàm xử lý của Aspect, tham số thứ hai chính là instance của Annotation. Bạn có thể chấm (`.`) để lấy giá trị truyền vào.

```java
package com.example.demo.aspect;

import com.example.demo.annotation.RoleCheck;
import jakarta.servlet.http.HttpServletRequest;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

@Aspect
@Component
public class RoleCheckAspect {

    // Tham số 'roleCheck' ở đây sẽ hứng giá trị từ annotation trên Controller
    @Around("@annotation(roleCheck)") 
    public Object validateRole(ProceedingJoinPoint joinPoint, RoleCheck roleCheck) throws Throwable {
        
        // 1. Lấy giá trị tham số đã truyền vào Annotation
        String roleRequired = roleCheck.requiredRole(); // Ví dụ: "ADMIN"
        
        // 2. Lấy Role hiện tại của người dùng từ Request
        // (Trong thực tế, cái này thường lấy từ JWT Token hoặc SecurityContext)
        // Ở đây mình giả lập lấy từ Header "X-Current-Role"
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes == null) return joinPoint.proceed();
        
        HttpServletRequest request = attributes.getRequest();
        String currentUserRole = request.getHeader("X-Current-Role");

        System.out.println("Role yêu cầu: " + roleRequired);
        System.out.println("Role hiện tại: " + currentUserRole);

        // 3. So sánh Logic
        if (currentUserRole == null || !currentUserRole.equals(roleRequired)) {
            throw new RuntimeException("Chặn truy cập! API này yêu cầu quyền: " + roleRequired);
        }

        // 4. Hợp lệ -> Cho qua
        return joinPoint.proceed();
    }
}
```

### 3\. Sử dụng trong Controller với các tham số khác nhau

Bây giờ bạn có thể tái sử dụng annotation này cho nhiều API với các yêu cầu quyền hạn khác nhau.

```java
package com.example.demo.controller;

import com.example.demo.annotation.RoleCheck;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/users")
public class UserController {

    // API này yêu cầu Header: X-Current-Role = ADMIN
    @GetMapping("/delete")
    @RoleCheck(requiredRole = "ADMIN") 
    public String deleteUser() {
        return "Xóa user thành công (Chỉ ADMIN mới thấy dòng này)";
    }

    // API này yêu cầu Header: X-Current-Role = STAFF
    @GetMapping("/view")
    @RoleCheck(requiredRole = "STAFF")
    public String viewUser() {
        return "Xem thông tin user (STAFF xem được)";
    }
}
```

### Cách test (Sử dụng Postman hoặc cURL)

1.  **Test thành công (Quyền ADMIN):**

      * URL: `GET /api/users/delete`
      * Header: `X-Current-Role: ADMIN`
      * Kết quả: 200 OK -\> "Xóa user thành công..."

2.  **Test thất bại (Quyền STAFF thử vào trang ADMIN):**

      * URL: `GET /api/users/delete`
      * Header: `X-Current-Role: STAFF`
      * Kết quả: 403/500 -\> "Chặn truy cập\! API này yêu cầu quyền: ADMIN"

### Mẹo nhỏ (Best Practice)

Nếu bạn muốn dùng annotation gọn hơn, ví dụ `@RoleCheck("ADMIN")` thay vì `@RoleCheck(requiredRole = "ADMIN")`, bạn hãy đổi tên biến trong interface thành `value()`:

```java
public @interface RoleCheck {
    String value(); // Java mặc định hiểu 'value' là tham số chính
}
```

Khi đó dùng sẽ gọn hơn: `@RoleCheck("ADMIN")`.

**Bạn có muốn tôi chỉ cách lấy dữ liệu từ phần thân (Body) của request (ví dụ check field trong JSON gửi lên) để lọc không?**
