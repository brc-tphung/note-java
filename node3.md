Bạn nói rất đúng\! **`HandlerInterceptor`** là một giải pháp cực kỳ phổ biến và chính thống trong Spring MVC để xử lý các vấn đề liên quan đến HTTP Request (như lọc, kiểm tra quyền, logging) trước khi nó đến Controller.

Thực tế, đối với việc xử lý **API web**, `HandlerInterceptor` thường được ưu tiên hơn AOP vì nó nhẹ hơn và gắn liền với vòng đời của Servlet.

Dưới đây là cách làm chức năng tương tự (Check Annotation) bằng **`HandlerInterceptor`**.

### Cách triển khai bằng HandlerInterceptor

#### 1\. Annotation (Giữ nguyên)

Bạn vẫn cần cái này để đánh dấu API.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RoleCheck {
    String value(); // Ví dụ: "ADMIN"
}
```

#### 2\. Tạo Interceptor

Điểm khác biệt lớn nhất so với AOP là bạn phải ép kiểu `handler` sang `HandlerMethod` để lấy được Annotation.

```java
package com.example.demo.interceptor;

import com.example.demo.annotation.RoleCheck;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class RoleCheckInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        
        // 1. Kiểm tra xem request này có trỏ đến một Controller method (HandlerMethod) không?
        // (Vì request có thể trỏ tới resource static css/js, lúc đó handler là ResourceHttpRequestHandler)
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }

        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // 2. Lấy annotation từ method đang được gọi
        RoleCheck roleCheck = handlerMethod.getMethodAnnotation(RoleCheck.class);

        // 3. Nếu method không có annotation này -> Cho qua (return true)
        if (roleCheck == null) {
            return true;
        }

        // 4. Có annotation -> Thực hiện logic kiểm tra
        String requiredRole = roleCheck.value();
        String currentRole = request.getHeader("X-Current-Role");

        if (currentRole == null || !currentRole.equals(requiredRole)) {
            // Trả về lỗi 403 và chặn request (return false)
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("Access Denied: You need role " + requiredRole);
            return false; 
        }

        return true; // Cho phép đi tiếp vào Controller
    }
}
```

#### 3\. Đăng ký Interceptor (Bước bắt buộc)

Khác với AOP (tự động scan), Interceptor phải được đăng ký vào cấu hình MVC.

```java
package com.example.demo.config;

import com.example.demo.interceptor.RoleCheckInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private RoleCheckInterceptor roleCheckInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Đăng ký interceptor áp dụng cho tất cả các đường dẫn
        registry.addInterceptor(roleCheckInterceptor).addPathPatterns("/api/**");
    }
}
```

-----

### Vậy nên dùng AOP hay Interceptor?

Dưới đây là bảng so sánh giúp bạn chọn đúng công cụ:

| Tiêu chí | HandlerInterceptor | Spring AOP |
| :--- | :--- | :--- |
| **Phạm vi** | Chỉ hoạt động trong lớp Web/Controller (Spring MVC). | Hoạt động ở mọi lớp (Controller, Service, Repository...). |
| **Mục đích chính** | Xử lý HTTP Request/Response (Header, Cookie, URL). | Xử lý logic nghiệp vụ cắt ngang (Transaction, Logging sâu, Caching). |
| **Truy cập tham số** | Khó truy cập giá trị tham số của hàm (Body JSON đã parse). | Rất dễ truy cập tham số hàm (`joinPoint.getArgs()`). |
| **Độ phức tạp** | Đơn giản, dễ hiểu với web developer. | Phức tạp hơn, cần hiểu về Proxy pattern. |
| **Hiệu năng** | Nhanh hơn một chút cho Web request. | Có overhead do tạo Proxy bao bọc method. |

**Lời khuyên:**

  * Nếu bạn chỉ muốn kiểm tra **Header, URL, Cookies** hoặc quyền truy cập cơ bản cho API: **Dùng `HandlerInterceptor`** (như cách bạn hỏi là rất chuẩn).
  * Nếu bạn muốn validate dữ liệu bên trong **Body (Object Java)** sau khi đã parse, hoặc muốn log lại xem method trả về dữ liệu gì, hoặc dùng cho cả Service layer: **Dùng AOP**.

Bạn đang muốn áp dụng cái này cho trường hợp cụ thể nào không? (Ví dụ: logging request hay authentication JWT?)
