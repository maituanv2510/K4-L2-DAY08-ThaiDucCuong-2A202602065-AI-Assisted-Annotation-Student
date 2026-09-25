# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg tên một ảnh trong `to_label/round1/images/train/`

Số xe nhìn thấy bằng mắt: 18 - 20 ảnh

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe: 
- Các xe rất nhỏ ở khu vực phía trên/trung tâm ảnh, đặc biệt những xe ở xa chỉ còn thấy đèn.
- Các xe ở phía bên phải và sát mép ảnh, một số xe chỉ hiện một phần thân xe hoặc chủ yếu nhìn thấy đèn hậu.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
