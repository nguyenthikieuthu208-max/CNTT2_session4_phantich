

# So sánh Trade-off các giải pháp Prompting nâng cao (Logic & Chuỗi)

**Bài toán:** Xử lý đơn hàng nhắn tin của cửa hàng thời trang: trích xuất thông tin, kiểm tra tồn kho, tính tiền, soạn tin xác nhận.

**Dữ liệu cửa hàng:** Áo thun 150.000đ (còn 10), Quần jean 350.000đ (còn 5), Nón lưỡi trai 80.000đ (còn 0). Tổng hàng từ 500.000đ giảm 10%, ship cố định 30.000đ.

## Bước 1: Đề xuất 2 prompt mẫu

### Giải pháp 1: Đơn giản, ngắn gọn (1 prompt duy nhất)

```
Bạn là nhân viên bán hàng. Đọc tin nhắn của khách, tính tổng tiền đơn hàng
và trả về JSON gồm: name, phone, items, total.

Bảng giá: Áo thun 150000, Quần jean 350000, Nón lưỡi trai 80000.
Hàng trên 500000 giảm 10%, ship 30000.

Tin nhắn: {TIN_NHAN}
```

### Giải pháp 2: Chi tiết, nhiều ràng buộc (Prompt Chaining + Chain-of-Thought + xử lý lỗi)

**P1 - Trích xuất**
```
Bạn là nhân viên nhập liệu. Trích xuất đơn hàng từ tin nhắn, chỉ trả JSON thuần.
Quy tắc:
- name viết hoa chữ cái đầu; phone bỏ khoảng trắng, đổi sang +84.
- Thông tin thiếu: đặt null, không tự điền.
- Tin nhắn trống hoặc không đọc được: trả {"status":"Invalid"}.
Cấu trúc: {"name":"","phone":"","address":"","items":[{"product":"","qty":0}],"status":"ok|missing_info|Invalid"}

Tin nhắn: {TIN_NHAN}
```

**P2 - Kiểm tra và tính tiền (CoT)**
```
Bạn là kế toán cửa hàng. Dữ liệu đơn: {JSON_P1}
Nếu status là "Invalid" thì dừng, trả {"status":"Invalid"}.
Bảng giá/tồn kho: Áo thun 150000 (còn 10); Quần jean 350000 (còn 5); Nón lưỡi trai 80000 (còn 0).
Quy tắc: tổng hàng >= 500000 giảm 10%; ship 30000.
Suy luận TỪNG BƯỚC trong thẻ <reasoning>:
1. Món có trong bảng giá không?
2. qty <= tồn kho? Món không đạt đưa vào rejected_items kèm lý do.
3. Tính thành tiền, tổng hàng (chỉ món hợp lệ).
4. Xét giảm giá, cộng ship, tính tổng.
5. Có món bị loại hoặc status khác "ok": status = "needs_confirmation".
Trả JSON thuần trong thẻ <result>:
{"valid_items":[{"product":"","qty":0,"unit_price":0,"amount":0}],"rejected_items":[{"product":"","reason":""}],"subtotal":0,"discount":0,"shipping":0,"total":0,"status":""}
```

**P3 - Soạn tin xác nhận**
```
Bạn là nhân viên chăm sóc khách hàng, giọng lịch sự. Dựa vào đơn đã tính: {JSON_P2}.
Viết tin nhắn tiếng Việt dưới 80 từ: nêu món hợp lệ, tổng tiền; nếu có món bị loại
hoặc thiếu thông tin thì nêu rõ và hỏi khách. Không bịa thông tin ngoài dữ liệu.
```

## Bước 2: Bảng so sánh Trade-off

| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
|---|---|---|
| Độ dễ hiểu, dễ viết | Cao | Trung bình |
| Tốn token và chi phí | Ít | Nhiều hơn (3 lần gọi) |
| Thời gian phản hồi | Nhanh | Chậm hơn |
| Chặn lỗi biên (thiếu SĐT, hết hàng, vượt tồn kho, dữ liệu rác) | Thấp | Cực tốt |
| Độ chính xác khi tính tiền | Dễ sai | Cao (có CoT từng bước) |
| Dễ kiểm tra, gỡ lỗi | Khó (không biết sai ở đâu) | Dễ (xem output từng bước) |
| Khả năng mở rộng | Thấp | Cao (thêm bước mới dễ) |
| Rủi ro ảo giác | Cao | Thấp |

## Bước 3: Lựa chọn và chạy thử

**Chọn Giải pháp 2.** Lý do: đơn hàng liên quan đến tiền và tồn kho nên sai sót gây thiệt hại trực tiếp. Chi phí token tăng thêm là nhỏ so với rủi ro báo sai giá hoặc nhận đơn hết hàng. Chuỗi nhiều bước cũng giúp biết lỗi nằm ở bước nào.

**Tin nhắn thử (có nhiều bẫy):**
> Mình là hoa, lấy 3 áo thun, 6 quần jean và 1 nón lưỡi trai. Giao về Hà Nội.

Bẫy: thiếu SĐT, quần jean vượt tồn kho (6 > 5), nón hết hàng.

**Kết quả đúng (tính theo quy tắc):** áo thun 3 x 150.000 = 450.000; quần jean và nón bị loại; 450.000 < 500.000 nên không giảm giá; cộng ship 30.000, tổng 480.000đ.

**Giải pháp 1 (kết quả thường gặp):**
```json
{"name":"Hoa","phone":"0900000000","items":["3 áo thun","6 quần jean","1 nón"],"total":2397000}
```
Sai: tự điền số điện thoại, nhận quần jean vượt tồn kho, nhận nón hết hàng.

**Giải pháp 2, P1:**
```json
{"name":"Hoa","phone":null,"address":"Hà Nội","items":[{"product":"Áo thun","qty":3},{"product":"Quần jean","qty":6},{"product":"Nón lưỡi trai","qty":1}],"status":"missing_info"}
```

**Giải pháp 2, P2:**
```
<reasoning>
1. Cả 3 món đều có trong bảng giá.
2. Áo thun 3 <= 10 đạt; Quần jean 6 > 5 loại; Nón lưỡi trai còn 0 loại.
3. Áo thun 3 x 150000 = 450000; tổng hàng = 450000.
4. 450000 < 500000 nên không giảm; + ship 30000 = 480000.
5. Có món bị loại và thiếu SĐT nên status = needs_confirmation.
</reasoning>
<result>
{"valid_items":[{"product":"Áo thun","qty":3,"unit_price":150000,"amount":450000}],"rejected_items":[{"product":"Quần jean","reason":"Vượt tồn kho (còn 5)"},{"product":"Nón lưỡi trai","reason":"Hết hàng"}],"subtotal":450000,"discount":0,"shipping":30000,"total":480000,"status":"needs_confirmation"}
</result>
```

**Giải pháp 2, P3:**
> Chào chị Hoa, shop ghi nhận 3 áo thun, tổng 480.000đ gồm phí ship. Quần jean chỉ còn 5 cái và nón lưỡi trai đã hết hàng. Chị vui lòng cho shop xin số điện thoại và cho biết có đồng ý đặt phần còn lại không ạ?

**Kết luận:** Giải pháp 2 bắt được cả 3 bẫy dữ liệu, còn Giải pháp 1 bỏ sót tất cả. Với bài toán có tiền và tồn kho, nên chọn Giải pháp 2.
