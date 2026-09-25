# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg

Số xe nhìn thấy bằng mắt: 18 xe

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Nhóm các xe ở rất xa gần đường chân trời (phía cuối đường cao tốc): chỉ xuất hiện dưới dạng các chấm sáng đèn pha/đèn hậu nhỏ, thân xe chìm vào nền tối nên AI dễ bỏ sót (false negative) hoặc box nhỏ dưới 16px bị lệch toạ độ.
2. Xe tải/xe van tối màu di chuyển ở làn giữa có vệt sáng đèn phản chiếu mạnh trên mặt đường: AI dễ bị đánh lừa vẽ box bao trùm cả vệt phản chiếu trên mặt đường thay vì ôm sát ranh giới thân xe theo quy tắc của GUIDELINE_LABEL.md.
