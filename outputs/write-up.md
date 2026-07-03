# Báo cáo ngắn — Tối ưu chi phí GPU cho NimbusAI

## 1. Tổng quan kết quả

Phân tích sử dụng bộ dữ liệu tổng hợp tháng 6/2026 cho thấy NimbusAI có thể giảm chi phí GPU hằng tháng từ **27.133 USD** xuống **14.626 USD**. Mức tiết kiệm dự kiến là **12.507 USD/tháng**, tương đương **46,1%**.

Đối với inference, chi phí giảm từ **48,87 USD/ngày** xuống **8,48 USD/ngày**. Khi chuẩn hóa theo sản lượng, chi phí giảm từ **6,488 USD/1M-token** xuống **1,126 USD/1M-token**, tương đương **82,6%**. Đây là KPI phù hợp hơn giá thuê GPU theo giờ vì nó phản ánh lượng đầu ra hữu ích mà hệ thống tạo ra.

Pipeline đã được kiểm tra thành công:

- `python verify.py`: **11/11 checks passed**.
- `pytest -q`: **15/15 tests passed**.

## 2. Phân tích các đòn bẩy tiết kiệm

| Đòn bẩy | Tiết kiệm/tháng | Tỷ trọng |
|---|---:|---:|
| Purchasing: spot/reserved | 10.040 USD | 80,3% |
| Inference: cascade/cache/batch | 1.212 USD | 9,7% |
| Right-size GPU có util-lie | 655 USD | 5,2% |
| Tắt GPU idle | 600 USD | 4,8% |
| **Tổng** | **12.507 USD** | **100%** |

Purchasing là đòn bẩy lớn nhất. Với mức giảm giá reserved 45%, điểm hòa vốn là **55% utilization**. Các workload có thể gián đoạn được chuyển sang spot; các workload chạy ổn định trên ngưỡng hòa vốn được chuyển sang reserved. Nhờ đó, riêng chi phí purchasing giảm từ **25.667 USD** xuống **15.627 USD/tháng**, tiết kiệm **39,1%**.

Inference sử dụng ba kỹ thuật kết hợp. Cascade chuyển request đơn giản sang model nhỏ; prompt caching giảm giá cho input được tái sử dụng; batch giảm 50% chi phí đối với request không yêu cầu phản hồi thời gian thực. Các chiết khấu có tính nhân: input vừa batch vừa cache hoàn toàn chỉ còn 5% giá naive, do `50% × 10% = 5%`. Batch không nên áp dụng cho traffic có SLO latency chặt.

## 3. GPU-Util lie và tác động tài chính

Hai GPU được phát hiện có GPU-Util cao nhưng MFU thấp:

| GPU | Loại | GPU-Util | MFU | MBU |
|---|---|---:|---:|---:|
| `gpu-h100-4` | H100 | 98,2% | 19,4% | 20,7% |
| `gpu-a10g-1` | A10G | 96,9% | 26,8% | 30,2% |

GPU-Util chỉ cho biết GPU có bận trong khoảng lấy mẫu hay không; nó không đo tỷ lệ peak FLOPs thực sự tạo ra công việc hữu ích cho model. Memory stall, đồng bộ, I/O wait, kernel nhỏ hoặc overhead khi launch kernel đều có thể làm GPU-Util rất cao trong khi MFU thấp. Vì vậy, `gpu-h100-4` gần như luôn bận nhưng chỉ khai thác khoảng một phần năm năng lực tính toán của H100.

Việc tắt các khoảng GPU idle tiết kiệm khoảng **20 USD/ngày**, tương đương **600 USD/tháng**. Right-size hai GPU util-lie được mô hình ước tính tiết kiệm thêm **655 USD/tháng**. Trước khi thay GPU trong production cần profile workload để phân biệt compute-bound và memory-bound, đồng thời kiểm tra throughput và latency sau thay đổi.

## 4. Phân bổ chi phí và quản trị

Tag coverage đạt **92%**, cao hơn ngưỡng chargeback 80%. Chi phí inference tối ưu theo team là:

| Team | Chi phí/ngày | Tỷ trọng |
|---|---:|---:|
| assistant | 2,59 USD | 30,6% |
| search | 2,49 USD | 29,4% |
| eval | 1,79 USD | 21,1% |
| rag | 1,60 USD | 18,9% |

NimbusAI đã đủ điều kiện kỹ thuật để thực hiện chargeback, nhưng nên duy trì showback trong một chu kỳ trước khi thu phí thật nhằm xác minh tag và xử lý các resource chưa được gắn nhãn. File FOCUS export giúp chuẩn hóa dữ liệu billing và giữ quy trình phân bổ nhất quán giữa nhiều cloud provider.

## 5. Sustainability

Một query đại diện gồm 800 token tiêu thụ khoảng **0,24 Wh** và phát thải **0,091 gCO2e** nếu chạy tại `us-east-1`. Trong các vùng được mô hình hóa, `europe-north1` có carbon intensity thấp nhất. Tuy nhiên, quyết định chuyển vùng còn phải xét latency, data residency và network egress; vùng sạch nhất không mặc nhiên là vùng phù hợp nhất cho mọi workload.

## 6. Ba hành động ưu tiên

1. Áp dụng chính sách spot/reserved trước vì đóng góp khoảng 80% tổng tiết kiệm; bắt buộc bổ sung checkpoint, theo dõi interruption và xác nhận duty cycle trước khi ký reserved dài hạn.
2. Triển khai cascade, caching và batch routing, đồng thời theo dõi `$/1M-token`, chất lượng đầu ra và latency theo từng route.
3. Tắt GPU idle và profile hai GPU util-lie bằng MFU/MBU trước khi right-size; không ra quyết định chỉ dựa trên GPU-Util.

## 7. Giả định và giới hạn

Kết quả dùng dữ liệu tổng hợp và bảng giá snapshot tháng 6/2026. Các khoản tiết kiệm đang được cộng tuyến tính, do đó khi áp dụng thực tế cần loại bỏ phần giao nhau giữa purchasing, right-sizing và tắt idle. Mô hình cũng chưa tính đầy đủ ảnh hưởng đến chất lượng, latency, cache storage, interruption thực tế và chi phí migration.

Repo hiện chưa triển khai phần mở rộng “Your Turn”, vì vậy báo cáo không ghi nhận điểm hoặc savings từ extensions. Để hoàn thành toàn bộ rubric, cần cài đặt và đo lường ít nhất hai extensions trước khi nộp bài.
