# Báo cáo Lab 25B - Game Day

**Họ tên / nhóm:** Nguyễn Phát Thịnh - 2A202602645
**Commit:** `c6d82b9` (commit hoàn thiện code và artifact trước bản Việt hóa)
**Điểm tự động (`make grade`):** 84 / 88

## 1. Kết quả Game Day

| Băng | Client | Availability | P95 (s) | Amplification | Retry ratio | Cost/turn | Recovery (s) | PASS |
|---|---|---:|---:|---:|---:|---:|---:|---|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | Không |
| G1 | resilient | 0.9993 | 4.771 | 1.061 | 0.001 | 2.092 | 0.0 | Có |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | Không |
| G2 | resilient | 0.9992 | 6.637 | 1.027 | 0.004 | 2.093 | 0.0 | Có |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | Không |
| G3 | resilient | 1.0000 | 4.668 | 1.059 | 0.030 | 2.143 | 0.0 | Có |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | Không |
| G4 | resilient | 1.0000 | 5.626 | 1.014 | 0.002 | 2.072 | 0.0 | Có |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | - | Không |
| G5 | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | - | Có |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | Không |
| G6 | resilient | 0.9765 | 5.990 | 1.039 | 0.005 | 2.245 | 0.0 | Có |

Resilient client vượt qua cả 6 băng public. Client không có vi phạm
`retry-after`, không retry lỗi client, không route không tương thích và không
failover không cần thiết.

## 2. Cấu hình client

| Tham số | Giá trị | Lý do |
|---|---:|---|
| `attempt_timeout_s` | 8.0 | Vẫn còn thời gian failover trong deadline 30 giây; P95 của G4 là 5.626 giây |
| `max_attempts_per_provider` | 2 | Cho phép retry một lần trên cùng provider nhưng không làm amplification tăng mạnh; G2 là 1.027 |
| `backoff_base_s / cap_s` | 0.25 / 4.0 | Full jitter phân tán retry và vẫn giữ đủ thời gian cho deadline |
| `retry_budget_ratio` | 0.1 | Giới hạn retry; retry ratio của G3 là 0.030 |
| `breaker.window_size / min_calls` | 20 / 10 | Tránh mở breaker khi mẫu quá nhỏ nhưng vẫn phản ứng trong incident |
| `failure_rate_threshold` | 0.5 | Mở breaker khi provider lỗi kéo dài |
| `slow_call_s` | 5.0 | Phát hiện suy giảm ở G4 trước khi timeout tích lũy |
| `open_duration_s` | 10.0 | Cho provider suy giảm thời gian hồi phục |
| `half_open_max_probes` | 2 | Giới hạn thundering herd trong quá trình hồi phục |

### Bảng quyết định của `on_error`

| Loại lỗi | Breaker | Retry / failover |
|---|---|---|
| Lỗi client 400 | release | `GiveUp`, không retry và không failover |
| Hết quota 429 | release | Cooldown provider dài rồi failover |
| Rate limit 429 | release | Tôn trọng `retry-after` rồi failover |
| 529/503/5xx/timeout | failure | Retry cùng provider nếu còn budget và deadline, sau đó failover |

## 3. Ba thí nghiệm

1. Retry budget là lớp bảo vệ cuối ngay cả khi breaker cấu hình sai. Với cấu hình
   bình thường, amplification của G2 là 1.027 và retry ratio của G3 là 0.030.
   Nếu breaker không bao giờ mở, việc tắt budget sẽ cho phép mọi lỗi tạm thời tạo
   thêm attempt; budget giới hạn lưu lượng retry gần mức 10%. Breaker phản ứng
   theo health của provider, còn budget bảo vệ toàn hệ thống khỏi retry
   amplification, vì vậy cần cả hai.

2. Timeout 30 giây không an toàn trong G6 vì một request đang bay có thể chiếm
   toàn bộ deadline của user và ngăn fallback. G4 gần như không đổi vì đây là
   các response thành công nhưng chậm, không phải provider lỗi. Timeout nên dựa
   trên latency phần đuôi khi hệ thống bình thường, deadline của user và thời
   gian cần cho một lần fallback hữu ích, không chỉ dựa trên thời gian tối đa
   provider có thể chạy.

3. Nếu bỏ lọc capability của tool, `gamma-mini` sẽ nhận tool request dù không hỗ
   trợ tool calling. Kết quả là model output không tương thích thay vì fallback
   hợp lệ. Theo góc nhìn product, hệ thống đã tiêu tốn chi phí và latency cho một
   request vốn không thể hoàn thành đúng yêu cầu.

## 4. Workflow hoàn tiền

| Executor | Completion | Crashes | Duplicate refunds | Duplicate emails | Lost refunds | Overpaid | Wasted LLM/crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1496700 cents | 1.123 |
| yours | 1.000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Chỉ idempotency key chưa đủ vì nó ngăn payment service thực hiện side effect
trùng, nhưng không ngăn việc tính lại các bước LLM đã hoàn tất. Chỉ checkpoint
cũng chưa đủ vì nó giúp resume nhưng không khử trùng lặp side effect khi response
bị mất. Thứ tự đúng là thực thi step, sau đó chỉ ghi checkpoint khi step thành
công. Nếu ghi checkpoint trước khi gọi payment, worker có thể bỏ qua payment sau
crash và làm mất refund.

## 5. Alerting và error budget

| Sự cố | Kỳ vọng | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (phút) |
|---|---|---|---|---:|
| `checkout_bug` | page | page | page | 27 |
| `provider_outage` | page | page | page | 4 |
| `deploy_blip` | none | page | none | - |
| `slow_leak` | ticket | none | ticket | 1337 |
| `degraded_evening` | page | page | page | 291 |
| `night_spike` | none | page | none | - |
| `small_bump` | none | page | none | - |

Burn-rate alerter vượt qua dữ liệu SLI public với 3 page, 1 ticket và 0 false
page. Hệ thống phát hiện `checkout_bug` sau 27 phút, `provider_outage` sau 4
phút và `degraded_evening` sau 291 phút. Hệ thống tạo ticket cho `slow_leak` sau
1337 phút mà không page. Báo cáo 30 ngày có 25,901,755 request, 55,618 request
lỗi, availability 99.7853%, đã tiêu thụ 42.95% error budget và còn 57.05%, vì
vậy chưa cần freeze release.

Workflow bảo vệ tính đúng đắn của side effect; alerting bảo vệ khả năng phát hiện
và phản ứng vận hành. Cần cả hai vì workflow đúng vẫn có thể unavailable, còn
alert không thể tự bảo đảm payment exactly-once.

## 6. Postmortem - G2 với naive client

**Tóm tắt:** Tại t=120, alpha mất 75% năng lực GPU. Ba tầng retry lồng nhau
đã khuếch đại tải, trong khi các request đã timeout vẫn tiếp tục chiếm slot của
provider. Dù alpha hồi phục tại t=300, các request bị bỏ lại trong hàng đợi cùng
retry tiếp tục giữ hệ thống trong trạng thái quá tải metastable.

**Ảnh hưởng:** Naive client có availability 0.4560, amplification 9.904 và
recovery 155 giây sau khi provider hồi phục. Resilient client đạt availability
0.9992, amplification 1.027 và recovery đo được bằng 0.

| Thời gian | Sự kiện | Bằng chứng |
|---:|---|---|
| 120 | Alpha mất 75% năng lực | `attempts_alpha/s=5`, `success_ratio=7/7=1.00` |
| 150 | Hàng đợi và retry tăng | `attempts_alpha/s=163`, `success_ratio=0/3=0.00` |
| 180 | Timeout phía client bắt đầu xuất hiện | `attempts_alpha/s=138`, `success_ratio=0/5=0.00` |
| 240 | Retry storm duy trì | `attempts_alpha/s=105`, `success_ratio=0/2=0.00`; amplification là 9.904 |
| 300 | Alpha hồi phục hoàn toàn | `attempts_alpha/s=115`, `success_ratio=0/5=0.00` |
| Sau 300 | Hệ thống vẫn suy giảm | Recovery của naive là 155 giây, không phải 0 |

Trigger là việc mất năng lực GPU. Root cause là thiết kế retry ba tầng không
giới hạn, không jitter, không retry budget và không circuit breaker. Các request
đã timeout vẫn chiếm slot provider, trong khi retry mới tiếp tục đến nhanh hơn
khả năng xử lý sau khi provider hồi phục.

Các cơ chế giúp giảm ảnh hưởng là retry có giới hạn, breaker fail-fast,
fallback có kiểm tra capability và metric theo virtual time. Tăng timeout hoặc
thêm retry sẽ làm hàng đợi nặng hơn và che giấu vấn đề availability.

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---:|---|---|---|---|
| 1 | Duy trì một tầng retry với retry budget | prevent | Platform | Amplification của G2 <= 1.2 trong CI |
| 2 | Alert theo amplification và recovery time | detect | SRE | Synthetic G2 tạo page trong cửa sổ quy định |
| 3 | Dùng breaker và giới hạn half-open probe | mitigate | Runtime | Scenario ẩn hồi phục trong <= 30 giây |

## 7. Điểm yếu còn lại

Client vẫn tốn chi phí cao hơn khi beta phải gánh spillover, đặc biệt cost per
good turn của G5 tăng lên 2.504. Một cải tiến production có thể là routing thích
ứng theo capacity và thêm cost budget rõ ràng, nhưng vẫn phải giữ các ràng buộc
capability và deadline.

## 8. Cache

| Client | Availability | FAQ availability | Cache hit ratio | Stale served | Wrong answers | Outdated answers | Cost/turn | PASS |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | Không |
| cached | 0.9110 | 0.9628 | 0.578 | 150 | 0 | 4 | 0.965 | Có |

| Tham số cache | Giá trị | Lý do |
|---|---:|---|
| `similarity_threshold` | 0.97 | Ngăn 181 lần hit nhầm câu hỏi đã xảy ra ở ngưỡng 0.80 |
| `ttl_s` | 60 | Giới hạn thời gian phơi nhiễm với thay đổi tài liệu không có webhook; outdated answers còn 4 |
| `max_stale_s` | 180 | Cung cấp 150 câu trả lời stale trong outage nhưng không kéo dài stale vô hạn |

Câu hỏi cá nhân bị loại khỏi cache vì cache theo user vẫn tạo rủi ro về quyền
riêng tư, phân quyền và invalidation; câu trả lời cá nhân phải được tính lại
theo dữ liệu hiện tại của user. Khi cache vẫn trả lời được trong lúc provider
outage, alert nên dựa thêm trên provider health, tỷ lệ `GiveUp`, tỷ lệ stale
served và tuổi cache, không chỉ dựa vào availability của user.

Trong C1, `wrong_answers_by_reason` là `{}` và tổng `wrong_answers` bằng 0;
không có nhóm nguyên nhân trả lời sai nào phát sinh. Có 4 `outdated_answers`,
nhưng không có phân rã nguyên nhân chi tiết trong artifact benchmark; con số này
được dùng trực tiếp để kiểm tra ngưỡng outdated của rubric.
