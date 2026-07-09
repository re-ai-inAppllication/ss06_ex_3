# Bài 3: Thực hành Refactor & Nâng cấp Giao dịch (Refinement Process - Robustness & Logging)

## 1. Phân tích lỗ hổng nghiêm trọng của code thô ban đầu
- **Không Validate:** Không kiểm tra `order` hay `order.getItems()` có rỗng/null không. Dễ gây NullPointerException.
- **Trừ kho nguy hiểm:** Dùng `orElse(null)` khi lấy Product, sau đó thao tác trực tiếp sẽ bị văng exception nếu Product không tồn tại. Không kiểm tra số lượng tồn kho (nếu stock < quantity, kho sẽ bị âm).
- **Lỗi tính nhất quán (Thiếu ACID):** Hàm không có `@Transactional`. Nếu `paymentGateway.charge()` bị lỗi (do thẻ ảo, sai mã OTP, lỗi mạng cổng thanh toán), exception ném ra luồng kết thúc, nhưng số lượng hàng trong kho **đã bị trừ** từ trước và lưu vào database. Hàng bị giam vĩnh viễn dù khách chưa trả tiền.
- **Thiếu Logging:** Không ghi lại lịch sử các bước (Bắt đầu xử lý đơn, Kho được trừ, Thanh toán thành công, hoặc Lỗi ở đâu). Gây mù thông tin khi debug.

## 2. Chuỗi 3 lượt Prompt Cải tiến đầu ra

**Vòng 1 (Robustness):**
> "Hàm `placeOrder` hiện tại quá nguy hiểm. Hãy refactor nó để an toàn hơn: 
> 1. Bổ sung kiểm tra đầu vào (ném IllegalArgumentException nếu null/empty).
> 2. Khi duyệt qua giỏ hàng, nếu không tìm thấy Product hoặc số lượng tồn kho (stock) nhỏ hơn số lượng mua (quantity), hãy ném ra `OutOfStockException`.
> 3. Gọi `paymentGateway.charge()`, nếu thanh toán thất bại, cổng này sẽ ném ra lỗi. Hãy tạo và ném ra Custom Exception là `PaymentFailedException`."

**Vòng 2 (Maintainability & Clean Code):**
> "Rất tốt. Tiếp theo, hãy nâng cấp về kiến trúc và bảo trì:
> 1. Thêm `@Transactional` của Spring Boot cho phương thức `placeOrder` để đảm bảo tính nguyên tử (Atomicity). Bất cứ ngoại lệ nào ném ra (đặc biệt là `PaymentFailedException`) đều phải tự động rollback lại số lượng tồn kho đã trừ.
> 2. Dùng thư viện Lombok thêm `@Slf4j` vào class. Bổ sung log cấp độ INFO ở các bước: bắt đầu đặt hàng, trừ kho sản phẩm nào, gọi thanh toán, hoàn tất. Bổ sung log cấp độ ERROR khi bắt gặp exception (ngoại lệ)."

**Vòng 3 (Context-specific Tuning):**
> "Để hoàn thiện chuẩn doanh nghiệp:
> 1. Thay vì trả về `void`, hàm hãy trả về đối tượng `OrderPlacementResult` chứa (boolean success, String orderId, String message). Xử lý try-catch bên trong hàm để trả về Result lỗi nếu xảy ra ngoại lệ nghiệp vụ (nhưng vẫn phải cấu hình để transaction rollback).
> 2. Viết 1 class JUnit Test sử dụng thư viện Mockito để mock các Repository và Gateway. Viết test case giả lập gọi thanh toán thất bại, verify (kiểm chứng) rằng exception được ném, kết quả fail và kho được rollback (không gọi đến `orderRepository.save()`)."

## 3. Minh chứng chạy thực tế (Code Lượt cuối)

```java
// --- OrderPlacementResult.java ---
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class OrderPlacementResult {
    private boolean success;
    private String orderId;
    private String message;
}

// --- OrderPlacementService.java ---
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderPlacementService {

    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;

    @Transactional(rollbackFor = Exception.class)
    public OrderPlacementResult placeOrder(Order order) {
        log.info("Bắt đầu xử lý đơn hàng cho Khách hàng: {}", order.getCustomerId());

        if (order == null || order.getItems() == null || order.getItems().isEmpty()) {
            log.error("Dữ liệu đơn hàng không hợp lệ (null hoặc rỗng)");
            throw new IllegalArgumentException("Dữ liệu đơn hàng không hợp lệ");
        }

        try {
            // 1. Kiểm tra và Trừ kho
            for (OrderItem item : order.getItems()) {
                Product product = inventoryRepository.findById(item.getProductId())
                        .orElseThrow(() -> new OutOfStockException("Không tìm thấy sản phẩm ID: " + item.getProductId()));

                if (product.getStock() < item.getQuantity()) {
                    log.error("Sản phẩm {} hết hàng hoặc không đủ. Kho: {}, Yêu cầu: {}", 
                            product.getName(), product.getStock(), item.getQuantity());
                    throw new OutOfStockException("Sản phẩm " + product.getName() + " không đủ số lượng");
                }

                product.setStock(product.getStock() - item.getQuantity());
                inventoryRepository.save(product);
                log.info("Đã trừ {} sản phẩm ID: {}", item.getQuantity(), product.getId());
            }

            // 2. Gọi cổng thanh toán
            log.info("Đang gọi cổng thanh toán số tiền: {}", order.getTotalAmount());
            boolean paymentStatus = paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());
            if (!paymentStatus) {
                throw new PaymentFailedException("Cổng thanh toán từ chối giao dịch");
            }

            // 3. Lưu đơn hàng
            orderRepository.save(order);
            log.info("Xử lý thành công đơn hàng Khách hàng: {}", order.getCustomerId());

            return OrderPlacementResult.builder()
                    .success(true)
                    .orderId(order.getId())
                    .message("Đặt hàng thành công")
                    .build();

        } catch (OutOfStockException | PaymentFailedException e) {
            log.error("Thất bại khi xử lý đơn hàng: {}", e.getMessage());
            // Ném exception để Spring Boot tự động Rollback lại kho
            throw e; 
        }
    }
}

// --- OrderPlacementServiceTest.java ---
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
public class OrderPlacementServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;
    @Mock
    private PaymentGateway paymentGateway;
    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderPlacementService orderPlacementService;

    @Test
    void placeOrder_ShouldRollback_WhenPaymentFails() {
        // Arrange
        Order order = new Order();
        order.setCustomerId("CUST_01");
        order.setTotalAmount(500000.0);
        OrderItem item = new OrderItem("PROD_01", 2);
        order.setItems(List.of(item));

        Product product = new Product("PROD_01", "Laptop", 10);
        
        when(inventoryRepository.findById("PROD_01")).thenReturn(Optional.of(product));
        when(paymentGateway.charge(anyString(), anyDouble())).thenReturn(false); // Giả lập thanh toán lỗi

        // Act & Assert
        assertThrows(PaymentFailedException.class, () -> {
            orderPlacementService.placeOrder(order);
        });

        // Verify: Kho đã bị gọi save (lúc trừ) nhưng do exception văng ra, @Transactional sẽ rollback DB.
        verify(inventoryRepository, times(1)).save(any(Product.class));
        
        // Cực kỳ quan trọng: Order không bao giờ được lưu
        verify(orderRepository, never()).save(any(Order.class));
    }
}
```
