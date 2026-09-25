# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Thái Đức Cường

Công cụ gán nhãn đã dùng: CVAT (Docker, định dạng Ultralytics YOLO Detection 1.0)

Mọi con số trong báo cáo này lấy từ `reports/rounds_table.md`, `outputs/metrics_round0.json`,
`outputs/metrics_round1.json`, `outputs/selection_round1.csv`, `outputs/round1_diff.md` và
`reports/REVIEW_LOG.csv`. Tôi không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Camera trong bài này đứng yên trên cầu, quay xuống đường cao tốc, 2.5 khung hình mỗi giây. Một
chiếc xe đi qua khung hình chỉ vài giây rồi biến mất, nên hai ảnh cách nhau 0.4 giây gần như
giống hệt nhau và cùng chứa các xe đó. Tập kiểm thử 20 ảnh được lấy từ bốn đoạn quanh giây 20,
60, 100 và 140; 112 ảnh trong khoảng 4 giây trước và sau mỗi đoạn bị loại làm vùng đệm; 268 ảnh
còn lại thành pool. Ảnh pool gần ảnh kiểm thử nhất vẫn cách nó 4.4 giây.

Nếu chia ngẫu nhiên, cùng một chiếc xe có thể vừa nằm trong ảnh tôi dùng để huấn luyện vừa nằm
trong ảnh dùng để chấm điểm. Khi mô hình đã học đúng những chiếc xe đó, nó sẽ phát hiện chúng
trên ảnh kiểm thử một cách dễ dàng — mặc bằng chứng rò rỉ dữ liệu (data leakage). Số đo lệch theo
hướng đẹp hơn sự thật, và độ lệch càng lớn khi số ảnh gán càng nhiều: vòng 1 của tôi chỉ có 12
ảnh nhưng nếu trộn ngẫu nhiên thì ảnh học và ảnh chấm sẽ chung nhiều xe. Chia theo thời gian loại
bỏ khả năng đó ngay từ đầu, đổi lại tôi phải chấp nhận rằng mô hình không được đánh giá trên
đúng những cảnh đã thấy.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 288 | 0.413 | -0.359 | 1.000 | 0.025 | 0.048 | 0.000 | 0.017 | 0.122 |

Vòng 0 là `yolov8n` có sẵn của COCO, chưa huấn luyện thêm gì trên video này, gộp ba lớp car, bus,
truck thành một lớn `car`. Chấm trên 20 ảnh kiểm thử, 417 box tham chiếu trong đó 14 box cao dưới
16 px bị bỏ qua, còn lại 403 box được tính. Ở ngưỡng conf 0.25, mô hình khớp 197 box, dự đoán
nhầm 16 và bỏ sót 206. Precision 0.925 rất cao còn recall chỉ 0.489, nghĩa là mô hình thiếu hẳn
chiều số xe, không phải thiếu chính xác.

Xét theo kích thước, độ phủ là 0.182 ở xe nhỏ (66 box), 0.547 ở xe vừa (296 box) và 0.561 ở xe
lớn (41 box). Xe nhỏ thấp hơn ba lần xe vừa, trong khi xe nhỏ chiếm tới 66/403 box tham chiếu.
Nghĩa là phần lớn lỗi của mô hình lạnh nằm ở xe ở xa, gần đường chân trời. Điều này khớp với
những gì tôi ghi trước khi xem pre-label: trong `reports/BLIND_SCAN.md` tôi đã đoán trước rằng
chỗ dễ bị bỏ sót là các xe rất xa chỉ còn hai chấm đèn, và các xe sát mép phải ảnh chỉ lộ một
phần thân.

Trong `outputs/compare_round0.jpg`, cột cold start lệch với cột nhãn tham chiếu chủ yếu ở đúng
dải đó: các box vàng (xe bị bỏ sót) dồn ở phần đường xa và mép phải khung hình, còn box xanh lá
(đúng) tập trung ở các xe gần, chiếm phần giữa và dưới ảnh. Một trường hợp cụ thể cần người rà
lại nhãn tham chiếu trước khi kết luận mô hình sai là các xe ở rất xa, chỉ còn hai chấm đèn: ở
độ phân giải 1280×720, chiều cao box của chúng chỉ khoảng 10–15 px, ranh giới trên–dưới mơ hồ
vài pixel, còn cạnh trái–phải thì dễ dịch vài pixel khi người vẽ. Ở mức IoU 0.5, chỉ cần lệch
nửa chiều rộng box là thành trượt. Tôi ghi nhận loại này ở đây thay vì sửa `data/test/labels/`,
đúng như `data/DATA.md` yêu cầu. Thêm một điểm: toàn bộ nhãn tham chiếu của tập kiểm thử do một
mô hình khác vẽ ra và chưa có người xem từng box, nên khi mô hình của tôi không khớp, khả năng
nhãn chấm sai còn lớn hơn khả năng mô hình của tôi sai ở loại xe này.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Điểm mỗi ảnh là `score = 0.5·U + 0.3·A + 0.2·D`. Với mỗi box, độ bất định là
`u = 1 − |2·conf − 1|`, lớn nhất khi conf = 0.5, tức khi mô hình đang lưỡng lự thật sự.
`U` là trung bình của 5 giá trị u khó nhất trong ảnh — lấy trung bình vì một ảnh có thể có
trăm xe nhưng chỉ vài xe mới là thứ mô hình chưa biết. `A` là số box mơ hồ (0.15 ≤ conf < 0.5)
chia cho giá trị lớn nhất trong pool, đo mức độ rối chung của ảnh. `D` là khoảng cách thời gian
từ ảnh đó tới ảnh đã gán gần nhất, chặn ở 10 giây, chia cho 10. Ở vòng 1 chưa có ảnh nào được
gán nhãn nên `D = 1.0` cho cả 268 ảnh — cột D trong `outputs/selection_round1.csv` đúng một giá
trị như vậy, và số 0.2 chỉ cộng thêm một hằng số cho mọi ảnh. Mọi khác biệt trong lô vòng 1 vì
vậy đến từ U và A.

`MIN_GAP_S = 2.0` giây là luật chặn cứng lúc chọn: hai ảnh trong cùng một lô phải cách nhau ít
nhất 2 giây, tức 5 khung hình. Camera đứng yên nên hai ảnh sát nhau gần như giống hệt, sửa cả hai
là tốn công gấp đôi mà mô hình học thêm rất ít.

Ba frame tôi dẫn ở `reports/SELECTION.md`:

- `frame_0182.jpg` (hạng 1, 0.9591, t = 72.8 s) — điểm cao nhất pool, A = 1.0 với 18 box mơ hồ.
  Sau khi sửa, `outputs/round1_diff.md` cho thấy model đưa 13 box, tôi giữ 13, xóa 0, kéo 0, thêm
  11. Bất định cao ở đây đi kèm thật nhiều xe thật bị bỏ sót.
- `frame_0099.jpg` (hạng 8, 0.9063, t = 39.6 s) — U = 0.9460, năm box khó nhất gần như không
  chắc. Sau khi sửa: 13 → 22 box, thêm 10, xóa 1, kéo 4. Đây cũng là ảnh tôi đã quét bằng mắt
  trước khi xem pre-label; `reports/BLIND_SCAN.md` ghi tôi đếm được 18–20 xe, khớp với 22 box sau
  khi sửa chứ không phải với 13 box của model.
- `frame_0369.jpg` (hạng 2, 0.9324, t = 147.6 s) — 43 box, 16 box mơ hồ; sau khi sửa 14 → 31 box,
  thêm 18, mức thêm nhiều nhất trong lô.

Frame thứ tư để minh họa luật ảnh gần trùng là `frame_0372.jpg`: hạng 6, điểm 0.9101, cao hơn 7
ảnh đã được chọn, nhưng t = 148.8 s, chỉ cách `frame_0369.jpg` (147.6 s) 1.2 giây nên bị loại.
Tương tự, `frame_0368.jpg` (hạng 9, 0.9003, t = 147.2 s) cách 0.4 giây cũng bị loại. Với số điểm
chênh nhau dưới 0.01 giữa ba ảnh này, chọn hai ảnh gần như trùng là công gán nhãn lãng phí rõ
ràng.

Điểm bất định không chứng minh ảnh đó sẽ cải thiện mô hình. Nó chỉ nói mô hình đang không chắc,
chứ không nói nhãn đúng sẽ giúp mô hình tiến gần đúng hơn. Vòng 1 của chính tôi là ví dụ rõ
nhất: tôi chọn 12 ảnh theo đúng công thức đó, thêm 137 box và xóa 18 box, và AP50 vẫn tụt từ
0.771 xuống 0.413. Để chứng minh phép chọn có giá trị thì phải có lô đối chứng chọn ngẫu nhiên
và đo trên cùng tập kiểm thử. Điểm cao cũng còn có thể do ảnh nhiều bóng đèn lẫn nhau chứ không
phải do xe, mà loại ảnh đó sửa xong cũng không dạy được gì.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hay xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 288 | 0.413 | -0.359 | 1.000 | 0.025 | 0.048 | 0.000 | 0.017 | 0.122 |

Tập kiểm thử: 20 ảnh, 403 box tham chiếu được tính (417 box, bỏ 14 box cao dưới 16 px), IoU 0.5,
P/R/F1 tại conf 0.25. Cả hai vòng đều chấm trên đúng 20 ảnh đó.

**Mức độ sửa nhãn gợi ý ở vòng 1** (`outputs/round1_diff.md`): trên 12 ảnh, model đưa 169 box
gợi ý. Sau khi tôi sửa, còn 288 box: 135 giữ nguyên, 16 kéo lại cho sát thân, 18 xóa, 137 thêm
mới. Tỉ lệ chấp nhận khung của model là 80%. Nói cách khác, cứ 5 khung model đưa ra thì 4 khung
tôi giữ nguyên, nhưng lượng xe tôi phải tự vẽ thêm lớn hơn lượng khung sai tôi xóa. Hai ảnh nổi
bật: `frame_0331.jpg` có model đưa 20 box và bị tôi xóa 6 khung — nhiều nhất trong lô, đây là chỗ
model bám vào vệt sáng và đèn pha hơn là vào xe; `frame_0369.jpg` tôi thêm 18 box, nhiều nhất
trong lô. Tôi đã kiểm tra phần lớn 137 box thêm mới và thấy chúng tập trung ở xe nhỏ và xe sát
mép ảnh: trong `labels/round1/` có 91 box cao dưới 32 px trên tổng 288, trong khi phần pre-label
gợi ý chỉ có 24 trên 169. Đây đúng là nhóm xe mà vòng 0 bỏ sót nhiều nhất (recall 0.182), nên
phần sửa khung của tôi có chất lượng, chỉ là không nâng được điểm.

**AP50:** vòng 1 đạt 0.413, tức giảm 0.359 so với khởi đầu lạnh 0.771. Vì đây là vòng đầu tiên
nên chưa có vòng trước để so. Mức giảm này lớn hơn nhiều lần ngưỡng 0.01 mà `data/DATA.md` nói
là chênh lệch chỉ do tập nhỏ gây ra, nên đây không phải nhiễu đo mà là thay đổi thật.

**Nhóm xe nào xấu đi:** không nhóm nào tốt lên. Recall nhỏ 0.182 → 0.000, vừa 0.547 → 0.017, lớn
0.561 → 0.122. Trong khi đó precision lên từ 0.925 lên đúng 1.000 (10 khớp, 0 nhầm). Mô hình
sau tinh chỉnh không phát hiện sai, mà gần như không phát hiện gì: ở ngưỡng 0.25 nó chỉ ra 10 box
trên 403 box tham chiếu. Đây là dấu hiệu mô hình quá bám vào nhãn của 12 ảnh mình vừa gán: đó là
12 ảnh ban đêm có nhiều xe nhỏ ở xa, chạy 50 epoch trên 12 ảnh với `val=False`, mô hình học
được "ban đêm thì phải rất chắc mới dám báo", nên độ tin cậy của mọi box bị kéo xuống dưới 0.25.

**Một ca đổi sau fine-tune.** Ở `outputs/compare_round0.jpg`, cột cold start bỏ sót lái xe
nhỏ chạy xa ở phía trên, phần giữa khung hình; ở `outputs/compare_round1.jpg`, cùng vùng đó
vẫn là box vàng (bị bỏ sót) nhưng mô hình không còn dự đoán dở ở đâu nữa. Lý do có thể kiểm: ở vòng
0, `frame_0099.jpg` có 10 xe bị bỏ sót, sau khi tôi tự vẽ thêm 10 box nhỏ đúng chỗ đó và huấn
luyện lại, mô hình đã học được đúng loại xe đó, nhưng lại hạ độ tin cậy xuống dưới ngưỡng chấm
nên đánh giá vẫn gọi là bỏ sót. Kiểm được bằng `metrics_round1.json`: TP = 10, FP = 0 — không
còn hộp nào nhầm, chỉ còn thiếu.

**Ba tầng thông tin phải tách bạch:**

- *Quan sát độc lập* (`reports/BLIND_SCAN.md`, đã khoá bằng `blind_lock.json` trước khi mở
  pre-label): với `frame_0099.jpg` tôi đếm bằng mắt khoảng 18–20 xe và chỉ ra hai vùng dễ sai là
  xe rất xa chỉ còn hai chấm đèn và xe sát mép phải ảnh.
- *Lỗi pre-label đã sửa* (`reports/REVIEW_LOG.csv` và `outputs/round1_diff.md`): model đưa 13 box
  cho `frame_0099.jpg`, tôi xóa 1 khung không phải xe, kéo 4 khung lệch, thêm 10 xe mà model bỏ
  sót; ở `frame_0270.jpg` tôi ghi lại một xe mà tôi cũng không khoanh vì ranh giới thân xe không
  rõ.
- *Kết quả mô hình sau train* (`metrics_round1.json`, `compare_round1.jpg`): recall xuống 0.025.
  Nói cách khác, mắt tôi thấy 18–20 xe và sửa được đúng 22, mô hình trả về 10 box. Ba tầng
  này không nói cùng một điều, và chỉ tầng cuối mới là thứ đo được.

**Một ca khó theo guideline.** Xe ở rất xa chỉ còn hai chấm đèn là ca khó nhất: guideline cho
phép khoanh hay bỏ đều được, nên mỗi người gán có thể chọn khác nhau mà vẫn đúng. Tôi chọn bỏ ở
những chỗ không thấy rõ thân xe, và chỉ khoanh khi còn thấy cạnh thân. Hệ quả là nhãn của tôi
nhất quán hơn nhiều so với phần pre-label của model ở nhóm này: tôi thêm 137 box, phần lớn là
xe nhỏ, trong khi model chỉ có 24 box nhỏ. Cùng guideline đó còn áp dụng cho xe bị cắt mép ảnh:
chỉ khoanh phần nhìn thấy, không kéo khung ra ngoài ảnh. Tôi dùng cùng quy tắc cho cả hai loại,
và đây chính là lý do con số 288 box sau sửa lớn hơn hẳn 169 box gợi ý — nhiều phần là do tôi
ghi nhận xe mà model bỏ qua hẳn, không phải vì tôi sai.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Vòng 1 cho AP50 0.413 so với 0.771 ở vòng 0: giảm 0.359. Precision tăng lên 1.000 còn recall
tụt từ 0.489 xuống 0.025. Mô hình không phát hiện sai, mà gần như im lặng. Tôi chọn dừng ở
đây, chưa làm vòng 2, vì hai lý do. Thứ nhất, 12 ảnh và 288 box là quá ít để tinh chỉnh mà không
làm mô hình hẹp: mô hình gần như chỉ học được "chỉ báo xe rõ ràng", nên kéo mọi xe nhỏ xuống
dưới ngướng 0.25. Thứ hai, thêm một vòng nữa lúc này sẽ không phân biệt được "chọn ảnh sai" với
"tinh chịn quá tay" — muốn tách hai nguyên nhân thì phải kiểm soát số epoch, mà `data/DATA.md`
cho biết chênh lệch dưới 0.01 AP50 giữa hai vòng là nhiễu, nên kết luận sớm cũng dễ sai.

**Hai ca còn yếu cho vòng sau**, đều là nơi mô hình yếu nhất và tôi cũng đã ghi trong
`REVIEW_LOG.csv`:

1. Xe rất xa chỉ còn hai chấm đèn, và xe bị cắt mép ảnh. Đây là nhóm recall 0.000 ở vòng 1. Chi
   phí rà nhãn cao nhất trong lô vì phải phóng to mới phân biệt được, và guideline cho phép khoanh
   hay bỏ nên dễ không nhất quán giữa các ảnh. Nguy cơ ảnh gần trùng cũng lớn: nhóm này nằm dọc
   đường chân trời, cả lô các ảnh sẽ gần như cùng một cảnh. Tôi sẽ lấy tối đa 2 ảnh, cách nhau
   trên 5 giây, ưu tiên 2 ảnh khác vùng (một ở dải trung, một ở mép phải bị cắt).
2. `frame_0372.jpg` (hạng 6, 0.9101) và `frame_0180.jpg` (hạng 24, 0.8539 nhưng U = 0.9745, cao
   thứ hai trong pool). Chi phí rà bình thường, nhưng `frame_0372.jpg` cách `frame_0369.jpg` 1.2
   giây nên gần như trùng cảnh; tôi sẽ đổi lấy `frame_0180.jpg` thay cho nó, và chỉ lấy `frame_0180`
   nếu vòng sau không còn ảnh nào quanh giây 72 — vì nó cách `frame_0182.jpg` chỉ 0.8 giây.

**Giới hạn ảnh hưởng thế nào tới kết luận.** Ba giới hạn cùng hướng làm cho cả hai vòng khó so
sánh. Tập kiểm thử chỉ 20 ảnh nên mỗi ảnh là 5% điểm số, và 403 box tham chiếu tập trung ở vài
đoạn trong video. Luật bỏ qua box cao dưới 16 px loại 14 box, tức gần như toàn bộ nhóm xe chỉ còn
hai chấm đèn mà lẽ ra là thứ quan trọng nhất cần đo — nhóm mà mô hình yếu nhất lại không được tính
điểm. Và nhãn tham chiếu do mô hình tạo chứ chưa người rà, nên "recall 0.025" chỉ có nghĩa là
mô hình không khớp với một bộ nhãn máy sinh, chưa chắc mô hình sai: nếu nhãn tham chiếu quá
nghiêm với xe nhỏ thì phần recall mất đi có thể nằm ở chỗ tham chiếu sai. Tôi không sửa được
`data/test/labels/` và cũng không nên sửa, nên cách duy nhất là ghi nhận điểm yếu này thay vì
kết luận.

**Nếu AP50 giảm, tôi sẽ kiểm trước khi train thêm, theo thứ tự này:**

1. Đọc lại chính `outputs/round1_diff.md` và `REVIEW_LOG.csv` xem 137 box tôi thêm có đúng không.
   Ở vòng 1, 91/288 box sau sửa nhỏ hơn 32 px, tức tỉ lệ xe nhỏ trong nhãn của tôi cao gấp đôi
   phần pre-label và cao hơn cả tập tham chiếu (36%). Nếu tôi đã khoanh cả vệt sáng hoặc đèn
   pha, mô hình sẽ học nhầm rằng đêm thì không được báo gì. Đây là nghi phạm số một.
2. Kiểm tra mô hình có bị "chết" vì độ tin cậy thấp chứ không phải vì không thấy: precision =
   1.000, FP = 0, TP = 10 là dấu hiệu đúng của tình trạng này. Cách kiểm là hạ ngưỡng conf xuống
   0.05 và xem số box có lên đủ không. Nếu lên, lỗi nằm ở cách huấn luyện (quá nhiều epoch trên
   ít ảnh, `val=False`, dùng `last.pt`), không nằm ở nhãn.
3. Sửa điểm 1 và 2 trước, rồi mới đổi chiến lược chọn mẫu. Đổi chiến lược trước khi tin được
   nhãn thì tôi sẽ không biết mình đang sửa mô hình hay đang sửa mình.
