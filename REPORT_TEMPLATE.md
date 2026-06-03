# CSC4005 Lab 6 Report – Export ONNX + Consistency Test + Benchmark

## 1. Thông tin

- Họ tên: Nguyễn Nam Cường
- Mã sinh viên: 1671040005
- Lớp: KHMT 16-01
- Link GitHub repo:
- Link checkpoint hoặc mô tả checkpoint sử dụng:
- Link file ONNX nếu không commit trực tiếp:

## 2. Mô tả mô hình đầu vào

| Nội dung | Giá trị |
|---|---|
| Bài toán | Smart Campus Scene Classification |
| Dataset | MIT Indoor Scenes 67 subset |
| Số lớp | 5 |
| Classes | classroom, computerroom, library, corridor, office |
| Model PyTorch | vit_b16 |
| Checkpoint | best_model.pt |
| Image size | 224 |
| Train mode từ lab trước | head_only |

## 3. Export ONNX

Điền thông tin:

| Thông số | Giá trị |
|---|---|
| ONNX path | outputs\\vit_smartcampus.onnx |
| Opset | 17 |
| Dynamic batch | yes |
| Input name | input |
| Output name | logits |
| Model size | 106 KB |

Lệnh đã chạy:

```bash
python -m src.export_onnx --onnx_path outputs/vit_smartcampus.onnx --model_name vit_b_16 --img_size 224 --opset 17 --dynamic_batch
```

## 4. Consistency Test

| Metric | Giá trị |
|---|---:|
| passed | true |
| num_samples | 32 |
| batch_size | 8 |
| max_abs_diff | 3.147125244140625e-05 |
| mean_abs_diff | 7.1803573291617795e-06 |
| pred_match_rate | 1.0 |
| atol | 0.0001 |
| rtol | 0.001 |


Nhận xét:

PyTorch và ONNX cho kết quả nhất quán với nhau. Kết quả kiểm thử cho thấy passed = true và pred_match_rate = 1.0, nghĩa là toàn bộ 32 mẫu kiểm tra đều cho cùng nhãn dự đoán giữa hai mô hình.

Sai khác giữa đầu ra của PyTorch và ONNX là rất nhỏ. Giá trị max_abs_diff = 3.147125244140625e-05 và mean_abs_diff = 7.1803573291617795e-06 đều thấp hơn đáng kể so với các ngưỡng kiểm tra (atol = 0.0001, rtol = 0.001). Đây là các sai khác số học bình thường do cách thực hiện phép tính của các framework khác nhau.

Các sai khác này không làm thay đổi kết quả phân loại vì tỷ lệ khớp nhãn dự đoán đạt 100%. Do đó có thể kết luận rằng mô hình ONNX sau khi export vẫn giữ nguyên hành vi dự đoán của mô hình PyTorch ban đầu.

## 5. Benchmark

| Runtime | Batch size | Mean latency (ms) | Median latency (ms) | P95 latency (ms) | Throughput (img/s) | Model size (MB) |
|---|---:|---:|---:|---:|---:|---:|
| PyTorch | 1 | 207.0051339999918 | 209.09015000052023 | 229.52808999998524 | 4.830798061269531 | 327.3680200576782 |
| ONNXRuntime | 1 | 204.46506800017232 | 198.64829999914946 | 242.3759950002932 | 4.890810981948062 | 0.10294151306152344 |
| PyTorch | 4 | 832.5324979998186 | 821.2153499998749 | 934.6044300004905 | 4.804617248708136 | 327.3680200576782 |
| ONNXRuntime | 4 | 840.750060000064 | 812.3839499994574 | 966.9403449996935 | 4.757656514469587 | 0.10294151306152344 |


## 6. Phân tích kết quả

Trả lời:

1. ONNXRuntime có nhanh hơn PyTorch không?

    Kết quả benchmark cho thấy ONNXRuntime nhanh hơn một chút so với PyTorch ở batch size bằng 1. Cụ thể, độ trễ trung bình của ONNXRuntime là 204.47 ms trong khi PyTorch là 207.01 ms. Tuy nhiên, ở batch size bằng 4, PyTorch lại có độ trễ trung bình thấp hơn một chút so với ONNXRuntime. Nhìn chung, hiệu năng của hai runtime khá tương đương trên môi trường thử nghiệm này.

2. Batch size ảnh hưởng thế nào đến latency và throughput?

    Khi tăng batch size từ 1 lên 4, độ trễ (latency) tăng đáng kể vì mỗi lần suy luận phải xử lý nhiều ảnh hơn. Trong khi đó, throughput gần như không thay đổi và duy trì ở mức khoảng 4.8–4.9 ảnh/giây. Điều này cho thấy việc tăng batch size chưa mang lại lợi ích rõ rệt về hiệu suất tổng thể trên hệ thống hiện tại.

3. Vì sao cần warm-up trước khi đo benchmark?

    Warm-up giúp loại bỏ ảnh hưởng của các chi phí khởi tạo ban đầu như nạp mô hình vào bộ nhớ, tối ưu đồ thị tính toán, tạo cache và chuẩn bị tài nguyên của runtime. Nếu không thực hiện warm-up, các lần chạy đầu tiên thường chậm hơn và có thể làm sai lệch kết quả benchmark.

4. Vì sao không nên chỉ đo một lần rồi kết luận?

    Một lần đo duy nhất có thể bị ảnh hưởng bởi nhiều yếu tố như tiến trình nền của hệ điều hành, trạng thái cache hoặc sự dao động của CPU. Do đó cần thực hiện nhiều lần đo và sử dụng các chỉ số thống kê như mean, median và P95 để có được kết quả khách quan và đáng tin cậy hơn.

5. Nếu triển khai lên CPU/edge device, bạn chọn batch size nào? Vì sao?

    Nếu triển khai trên CPU hoặc thiết bị edge, tôi sẽ chọn batch size bằng 1. Lý do là batch size nhỏ giúp giảm độ trễ, tiết kiệm bộ nhớ và phù hợp với các hệ thống xử lý theo thời gian thực, nơi mỗi ảnh cần được dự đoán ngay khi được đưa vào hệ thống.

## 7. Liên hệ triển khai thực tế

ONNX giúp chuẩn hóa mô hình và cho phép triển khai trên nhiều nền tảng khác nhau mà không phụ thuộc trực tiếp vào PyTorch. Việc sử dụng ONNX cũng giúp giảm kích thước mô hình đáng kể, từ khoảng 327 MB xuống còn khoảng 0.1 MB trong kết quả benchmark, giúp việc lưu trữ và triển khai trở nên thuận tiện hơn.

Consistency test giúp phát hiện các lỗi phát sinh trong quá trình chuyển đổi mô hình từ PyTorch sang ONNX, chẳng hạn như sai khác đầu ra hoặc lỗi của các toán tử không được hỗ trợ đầy đủ. Benchmark cung cấp các số liệu về độ trễ, throughput và kích thước mô hình để hỗ trợ việc lựa chọn runtime phù hợp cho từng môi trường triển khai.

Nếu cần triển khai mô hình vào hệ thống Smart Campus thực tế, ngoài benchmark còn cần kiểm thử độ chính xác trên dữ liệu thực tế, khả năng xử lý nhiều yêu cầu đồng thời, mức sử dụng CPU và RAM, cũng như độ ổn định của hệ thống khi vận hành liên tục trong thời gian dài.

## 8. Kết luận

Mô hình đã được export sang định dạng ONNX thành công với đường dẫn `outputs\vit_smartcampus.onnx`. Kết quả consistency test cho thấy quá trình chuyển đổi không làm thay đổi hành vi của mô hình, với tỷ lệ khớp nhãn dự đoán đạt 100%.

Kết quả benchmark cho thấy ONNXRuntime và PyTorch có hiệu năng tương đối tương đương trên môi trường thử nghiệm, trong đó ONNXRuntime nhanh hơn nhẹ ở batch size bằng 1. Thông qua bài lab này, tôi hiểu rõ hơn quy trình chuyển đổi mô hình từ PyTorch sang ONNX, cách kiểm tra tính nhất quán của mô hình sau khi export và phương pháp đánh giá hiệu năng trước khi triển khai trong thực tế.
