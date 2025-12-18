Đây là một cách tiếp cận **cực kỳ thực tế**! Bạn hoàn toàn đúng, trong môi trường nhiều team làm việc (đặc biệt là outsourcing hoặc các team độc lập), việc "tin tưởng" họ đặt tên biến đúng chuẩn là rủi ro rất lớn. **"Trust but verify"** (Tin nhưng phải kiểm tra), hoặc tốt hơn là **"Explicit Configuration"** (Cấu hình rõ ràng).

Nếu bạn muốn Shared Domain nắm quyền kiểm soát nhưng vẫn mềm dẻo cho Business Domain tùy chỉnh tên trường, tôi đề xuất giải pháp sử dụng **Custom Annotation trên Entity**.

Thay vì bắt họ truyền tham số lằng nhằng vào hàm `check(...)` mỗi lần gọi (làm code Controller dài ngoằng và dễ sai), ta sẽ bắt họ **khai báo map field 1 lần duy nhất ngay trên Class Entity**.

Dưới đây là giải pháp:

### 1. Tạo Annotation Cấu hình (`@SecurityInfo`) trong Shared Domain

Annotation này sẽ đóng vai trò như bản "hướng dẫn sử dụng" cho Shared Domain biết phải chọc vào trường nào của Entity.

```java
// Nằm trong module: common-shared
package com.mycompany.shared.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE) // Đặt trên Class
@Retention(RetentionPolicy.RUNTIME)
public @interface SecurityInfo {
    
    // Tên trường ID (mặc định là "id" nếu không truyền)
    String idField() default "id";
    
    // Tên trường cờ xóa (mặc định "deleteFlag")
    String delField() default "deleteFlag";
    
    // Tên trường ID tổ chức (mặc định "organizationId")
    String orgField() default "organizationId";
}

```

---

### 2. Cập nhật `AccessControlService` (Shared Domain)

Logic bây giờ sẽ là:

1. Nhận Class vào.
2. Đọc Annotation `@SecurityInfo` trên Class đó.
3. Nếu có: Lấy tên field từ Annotation.
4. Nếu không có: Ném lỗi (hoặc dùng mặc định tùy bạn - ở đây tôi chọn ném lỗi để ép họ phải cấu hình).
5. Build câu Query dựa trên tên field lấy được.

**LƯU Ý QUAN TRỌNG:** Trong JPA (JPQL/HQL), bạn phải dùng **Tên biến Java (Field Name)** chứ không phải tên cột Database (Column Name).

```java
// Nằm trong module: common-shared
@Service("accessGuard")
public class AccessControlService {

    @PersistenceContext
    private EntityManager entityManager;

    public boolean check(Long id, Class<?>... tables) {
        Long currentOrgId = SecurityUtils.getCurrentUserOrgId();

        for (Class<?> tableClass : tables) {
            // 1. Lấy thông tin cấu hình từ Annotation
            SecurityInfo info = tableClass.getAnnotation(SecurityInfo.class);

            if (info == null) {
                // Chửi thẳng mặt nếu dev lười không gắn cấu hình :D
                throw new RuntimeException("Entity " + tableClass.getSimpleName() + " chưa gắn @SecurityInfo. Vui lòng cấu hình map field bảo mật.");
            }

            // 2. Lấy tên field dynamic từ config
            String idField = info.idField();
            String delField = info.delField();
            String orgField = info.orgField();

            // 3. Gọi hàm check với tên field động
            if (!checkDynamic(tableClass, id, currentOrgId, idField, delField, orgField)) {
                throw new CustomAccessDeniedException(1L, "Truy cập bị từ chối vào: " + tableClass.getSimpleName());
            }
        }
        return true;
    }

    private boolean checkDynamic(Class<?> clazz, Long id, Long orgId, 
                                 String idFieldName, String delFieldName, String orgFieldName) {
        try {
            // Build Query: SELECT count(e) FROM Entity e WHERE e.customId = :id ...
            StringBuilder hql = new StringBuilder();
            hql.append("SELECT count(e) FROM ").append(clazz.getName()).append(" e");
            
            // Dùng tên field được truyền vào
            hql.append(" WHERE e.").append(idFieldName).append(" = :id");
            hql.append(" AND e.").append(delFieldName).append(" = 0"); // Giả sử xoá là 1, chưa xoá là 0
            hql.append(" AND e.").append(orgFieldName).append(" = :orgId");

            Query query = entityManager.createQuery(hql.toString());
            query.setParameter("id", id);
            query.setParameter("orgId", orgId);

            return (Long) query.getSingleResult() > 0;

        } catch (Exception e) {
            // Log lỗi: Có thể do truyền tên field sai so với biến trong Java
            System.err.println("Lỗi mapping field: " + e.getMessage());
            throw new RuntimeException("Cấu hình @SecurityInfo sai tên biến Java cho entity: " + clazz.getSimpleName());
        }
    }
}

```

---

### 3. Cách Business Domain sử dụng (Rất linh hoạt)

Bây giờ team Business có thể đặt tên biến kiểu gì cũng được (tiếng Anh, tiếng Việt, viết tắt...), miễn là họ khai báo mapping.

**Ví dụ 1: Team làm chuẩn**

```java
@Entity
@SecurityInfo // Dùng mặc định (id, deleteFlag, organizationId)
public class StandardOrder {
    @Id
    private Long id;
    private Integer deleteFlag;
    private Long organizationId;
}

```

**Ví dụ 2: Team thích "phá cách" (Legacy Code)**

```java
@Entity
// Map thủ công vào đây:
@SecurityInfo(
    idField = "orderKey",       // Trong code java biến là orderKey
    delField = "isDeleted",     // Trong code java biến là isDeleted
    orgField = "departmentRef"  // Trong code java biến là departmentRef
)
public class LegacyOrder {
    
    @Id
    private Long orderKey; // Tương ứng DB: order_pk

    @Column(name = "is_del")
    private Integer isDeleted; // Tên biến Java là isDeleted

    @Column(name = "dept_id")
    private Long departmentRef;
}

```

**Ví dụ 3: Controller gọi hàm (Gọn gàng)**
Controller không cần quan tâm tên cột là gì nữa, chỉ cần truyền Class.

```java
@RestController
public class OrderController {

    // Vẫn gọi ngắn gọn thế này thôi
    @PreAuthorize("@accessGuard.check(#id, T(com.app.LegacyOrder))")
    public void deleteOrder(@PathVariable Long id) { ... }
}

```

### Tại sao cách này tối ưu nhất cho bạn?

1. **Quyền lực thuộc về Shared Domain:** Bạn ép buộc logic check (ID + Del + Org). Không ai có thể "lách" luật.
2. **Trách nhiệm thuộc về Business Domain:** Họ đặt tên biến sao thì họ tự config annotation. Nếu họ config sai -> Code họ chết -> Họ tự sửa.
3. **Code Controller sạch sẽ:** Không cần truyền chuỗi string tên cột dài dòng trong `@PreAuthorize` (tránh việc gõ sai chuỗi string "del_flag" thành "del_falg" mỗi lần gọi hàm).
4. **Hiệu năng:** Java đọc Annotation rất nhanh, và JPA Parser cũng xử lý tốt query string động này.

Bạn có muốn tôi bổ sung thêm phần check xem **Kiểu dữ liệu** của các field đó có khớp không (ví dụ field xóa cờ là `Boolean` thay vì `Integer`) để xử lý triệt để các tình huống "lệch pha" không?
