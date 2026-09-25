# Vì sao chọn lô này?

Họ và tên: Thái Đức Cường

## 1. Năm frame sẽ ưu tiên nếu chỉ có ngân sách rà năm ảnh

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

1. `frame_0182.jpg` — hạng 1, điểm 0.9591, t = 72.8 s. Điểm cao nhất trong cả 268 ảnh pool. Nó
   còn đạt A = 1.0, tức 18 box mơ hồ (0.15 ≤ conf < 0.5), cao bằng mức lớn nhất của cả pool. Đây
   là ảnh mà model biết có xe nhưng không chắc xe nào, nên sửa nó bổ sung được nhiều thông tin
   nhất trên mỗi phút công.
2. `frame_0369.jpg` — hạng 2, điểm 0.9324, t = 147.6 s. U = 0.9315 (gần như bằng 1: năm box khó
   nhất đều có conf ≈ 0.5), 16 box mơ hồ và 43 box được dự đoán. Chọn vì nó ở đoạn giữa video
   (147.6 s), xa mọi ảnh đã gán trước đó, nên lấy thêm được cả thông tin về nhịp giao thông giữa
   video chứ không chỉ lặp lại cảnh đầu video.
3. `frame_0380.jpg` — hạng 3, điểm 0.9170, t = 152.0 s. Cách `frame_0369.jpg` 4.4 giây, tức vượt
   được MIN_GAP_S = 2.0 s, nên vẫn là một cảnh khác chứ không phải bản sao. Giữ nó thay vì
   `frame_0372.jpg` (hạng 6, 0.9101) là quyết định xét ảnh gần trùng: xem bên dưới.
4. `frame_0326.jpg` — hạng 4, điểm 0.9155, t = 130.4 s. 39 box, 15 box mơ hồ.
5. `frame_0331.jpg` — hạng 5, điểm 0.9154, t = 132.4 s. Nhiều box nhất trong lô (47 box), A = 1.0.

Trong năm ảnh này không có ảnh nào bị bỏ vì không có box: cột `empty` trong CSV là `False` ở cả 268
dòng, tức model luôn ra ít nhất một box trên mọi ảnh. Trường hợp "model không dự đoán được box"
vì vậy không xảy ra ở vòng 1; nó chỉ có thể xuất hiện từ vòng 2 trở đi, khi mô hình đã bị tinh
chỉnh. Sự kiện đáng chú ý hơn là hai ảnh gần trùng bị loại dù điểm cao.

## 2. Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet

- `frame_0182.jpg` (hạng 1, 0.9591, t = 72.8 s): CSV cho U = 0.9182, A = 1.0, 28 box được dự
  đoán. Sau khi tôi sửa, `outputs/round1_diff.md` cho biết model đề xuất 13 box, tôi giữ nguyên
  13, xóa 0, kéo lại 0 và thêm 11 — nhiều xe hơn hẳn mức model tự tin. Đây là bằng chứng mạnh
  nhất rằng điểm bất định cao tỉ lệ với số xe thật bị bỏ sót.
- `frame_0099.jpg` (hạng 8, 0.9063, t = 39.6 s): U = 0.9460 — năm box khó nhất gần như không
  chắc chắn — dù A chỉ 0.7778. Sau khi sửa: 13 box của model → 22 box, thêm 10, xóa 1, kéo 4.
  Đây cũng chính là ảnh tôi đã quét bằng mắt trước khi mở pre-label và ghi vào
  `reports/BLIND_SCAN.md`: tôi đếm được khoảng 18–20 xe, và tôi đã cảnh báo trước về các xe rất
  xa chỉ còn hai chấm đèn. Số 22 sau khi sửa khớp với con số mắt tôi đếm được, chứ không phải
  với 13 box của model.
- `frame_0369.jpg` (hạng 2, 0.9324, t = 147.6 s): nhiều box nhất trong lô (43 box, 16 box mơ hồ).
  Sau khi sửa: 14 box → 31 box, thêm 18 — mức thêm nhiều nhất trong 12 ảnh. Một lần nữa, điểm
  cao đi cùng lượng xe bị bỏ sót lớn.

Trên ảnh contact sheet `outputs/selection_round1.jpg` ba ảnh này nằm ở ba khoảng thời gian
khác nhau (khoảng 40 s, 73 s và 148 s) và không có ảnh nào trùng cảnh với nhau, nên lô 12 ảnh có
danh sách thời gian trải đều từ 39.6 s đến 156.8 s.

## 3. Một frame có điểm cao nhưng không chọn, hoặc một frame có điểm thấp vẫn nên xem, và lý do

- Điểm cao nhưng không được chọn: `frame_0372.jpg`, hạng 6, điểm 0.9101, t = 148.8 s. Nó cao
  hơn 7 ảnh đã được chọn (kể cả `frame_0312.jpg` hạng 7 và `frame_0099.jpg` hạng 8) nhưng vẫn bị
  bỏ. Lý do là luật MIN_GAP_S: nó nằm cách `frame_0369.jpg` (147.6 s) chỉ 1.2 giây. Camera đứng
  yên, 1.2 giây là 3 khung hình, hai ảnh gần như giống hệt nhau — sửa cả hai tốn gấp đôi công mà
  mô hình học thêm rất ít. Cùng lý do đó, `frame_0368.jpg` (hạng 9, 0.9003, t = 147.2 s, chỉ
  cách `frame_0369.jpg` 0.4 giây) cũng bị bỏ. Đây là lý do tôi thay `frame_0372.jpg` bằng
  `frame_0380.jpg` ở danh sách năm ảnh: 152.0 s cách 147.6 s tới 4.4 giây, vẫn là cảnh khác.
- Điểm thấp nhưng vẫn nên xem: `frame_0180.jpg`, hạng 24, điểm 0.8539, t = 72.0 s. Điểm tổng
  thấp chỉ vì A = 0.5556 (ít box mơ hồ), nhưng U = 0.9745 là cao thứ hai trong cả pool, tức năm
  box khó nhất của ảnh này gần như nằm đúng ngưỡng phân vân. Hơn nữa nó cách `frame_0182.jpg`
  (72.8 s) chỉ 0.8 giây. Nếu tôi rà lượt sau mà không còn ảnh nào ở khoảng 72–73 s, tôi sẽ đổi
  `frame_0180.jpg` vào thay một ảnh đã gán, vì ở đây sự bất định tập trung đúng vào một vài xe
  nhỏ chứ không phân tán khắp ảnh.

Một nhận xét thêm về công thức: ở vòng 1 cột D bằng 1.0 cho mọi ảnh, vì chưa có ảnh nào được
gán nhãn nên "khoảng cách tới ảnh đã gán gần nhất" không phân biệt được ai. Trọng số W_D = 0.2
vì vậy chỉ cộng thêm một hằng số cho mọi ảnh và không ảnh nào bị loại vì lý do thời gian; toàn bộ
khác biệt giữa 268 ảnh đến từ U và A. Từ vòng 2 trở đi mới phải xét khoảng cách thật.

## 4. Điều phép chọn này chưa chứng minh về chất lượng mô hình

Điểm trong `selection_round1.csv` chỉ đo một điều: model có chắc về box của nó hay không. Nó
không nói được sau khi tôi sửa ảnh đó thì mô hình có giỏi hơn hay không, vì chưa có thí nghiệm nào
đo mối liên hệ đó. Trong chính lô vòng 1 của tôi, tôi đã sửa 12 ảnh, thêm 137 box và xóa 18 box
mà AP50 vẫn giảm từ 0.771 xuống 0.413 (xem `reports/REPORT.md`). Đó là bằng chứng trực tiếp
rằng "AI phân vân" và "sửa ảnh này sẽ giúp AI" là hai câu chuyện khác nhau.

Ngoài ra còn ba lý do khiến điểm cao không đáng tin ngay: (1) U cao có thể do ảnh nhiều bóng
đèn lẫn nhau chứ không phải do xe, mà ảnh như vậy sửa xong cũng không dạy được gì; (2) A đo số
box mơ hồ, nên ảnh đông xe luôn dễ lọt; (3) nhãn chấm của tập kiểm thử do chính mô hình tạo ra
chứ chưa có người rà từng box, nên kể cả khi mô hình tốt lên thì điểm cũng có thể không nhích
vì tham chiếu sai. Muốn chứng minh phép chọn có giá trị thì phải chạy thí nghiệm có đối chứng:
cùng một lô, nhưng so sánh với một lô chọn ngẫu nhiên, và đo trên tập kiểm thử cố định. Lab này
chỉ làm một lô nên chưa đủ cơ sở để kết luận.
