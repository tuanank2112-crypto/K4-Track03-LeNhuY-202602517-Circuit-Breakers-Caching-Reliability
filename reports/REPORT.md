# Lab 25B — Report

**Họ tên / nhóm:** Lê Như Ý (MSV: 202602517)  **Commit:** aabb544  **Điểm tự động (`make grade`):** 84.0 / 88 (100% public score, 88/88 sau hidden SLO)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Median service time của provider là 1.2s - 1.8s. 8.0s đủ dài để không ngắt nhầm call bình thường, nhưng đủ ngắn để failover kịp trước khi chạm turn deadline 30s. Thí nghiệm G6 cho thấy nếu đặt 30s, availability tụt xuống 0.9488 với 107 deadline give up. |
| max_attempts_per_provider | 2 | Đúng 1 lần retry trên cùng provider khi gặp transient error (529, 500, timeout). Ngăn chặn dồn tải cục bộ lên provider đang quá tải (ở G2 naive thử 36 lần làm amplification tăng lên 9.904). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Base 0.25s giúp retry nhanh đối với lỗi thoáng qua; cap 4.0s cùng full jitter phân tán các retry đồng thời, triệt tiêu hiện tượng tháo lũ đồng pha (thundering herd). |
| retry_budget_ratio | 0.1 | Khống chế tổng số retry không vượt quá 10% lượng request gốc. Ở thí nghiệm 3.1 khi breaker bị tắt, budget giữ amplification ở 1.378 và recovery 0.0s (so với không budget: ampl 1.675, recovery 25.0s). |
| breaker.window_size / min_calls | 20 / 10 | Sliding window 20 kết quả gần nhất phản ánh nhanh biến động tải; min_calls = 10 đảm bảo có đủ mẫu thống kê, tránh ngắt mạch nhầm khi chỉ có 1–2 lỗi ngẫu nhiên trong giờ thấp điểm. |
| breaker.failure_rate_threshold | 0.5 | Ở G3 brownout (alpha lỗi 30%), ngưỡng 50% tránh trip nhầm không cần thiết; khi provider thực sự sập hoặc lỗi >= 50% (G2, G6), breaker lập tức ngắt trong vài giây. |
| breaker.slow_call_s | 5.0 | Service median là 1.2s; cuộc gọi kéo dài >= 5.0s là dấu hiệu cạn kiệt tài nguyên / nghẽn hàng đợi (G4). Giúp phát hiện suy thoái trước khi chuyển thành timeout hàng loạt. |
| breaker.open_duration_s | 10.0 | Cho provider đúng 10.0s (bằng max_queue_wait_s của server) để xả sạch hàng đợi tồn đọng và giải phóng GPU trước khi cho probe thăm dò. |
| breaker.half_open_max_probes | 2 | Chỉ cho phép đúng 2 probe đồng thời thăm dò provider ở trạng thái HALF-OPEN, ngăn không cho hàng trăm request ập vào làm sập lại provider vừa hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                     +---------------------------+
                     | Provider trả về lỗi (err) |
                     +---------------------------+
                                   |
                         classify(err) -> kind
                                   |
       +-------------------+-------+-------------------+---------------------+
       |                   |                           |                     |
  CLIENT_ERROR      QUOTA_EXHAUSTED              RATE_LIMITED         OVERLOADED /
 (400, 401, 422)      (429 quota)                (429 limit)        SERVER_ERROR (5xx)
       |                   |                           |             TIMEOUT (0, 408)
 release(permit)     release(permit)             release(permit)             |
       |                   |                           |             on_failure(permit)
  GiveUp("client_    cooldown = 300s             cooldown = retry_after      |
      error")              |                           |             tries < max_tries
 (Không retry,       Failover ngay               Failover ngay        AND budget OK
  không failover)    (idx++, delay=0)            (idx++, delay=0)     AND fits deadline?
                                                                      /               \
                                                                    YES                NO
                                                                     |                  |
                                                                Retry SAME          Failover
                                                                provider với       (idx++, delay=0)
                                                                Full Jitter backoff
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`:
   - Ở G2: Khi **có budget**, `avail = 0.9983`, `ampl = 1.378`, `retry_ratio = 0.058`, `recovery = 0.0s`. Khi **tắt budget**, `avail = 0.9666`, `ampl = 1.675`, `retry_ratio = 0.495` (gấp 8.5 lần!), và `recovery = 25.0s`.
   - Ở G3: Có budget cho `ampl = 1.136`, `retry_ratio = 0.068`; không budget thì `ampl = 1.162`, `retry_ratio = 0.159`.
   - *Kết luận:* Khi breaker hoạt động đúng, nó bảo vệ cục bộ từng provider bằng cách fail-fast. Nhưng nếu breaker bị lỗi cấu hình hoặc trễ phát hiện, retry budget là van an toàn toán học toàn cục khống chế tổng tải retry không bao giờ vượt quá 10%. Cần cả hai vì breaker bảo vệ provider suy yếu, còn budget khống chế toàn bộ tài nguyên client.
2. **`attempt_timeout_s: 30`.**
   - Ở G4 (slow region): 8s timeout cho `avail = 1.0000, p95 = 5.562s`; 30s timeout cho `avail = 1.0000, p95 = 5.608s`.
   - Ở G6 (provider outage): 8s timeout cho `avail = 1.0000, p95 = 5.961s, give_ups = 0`. Nhưng với 30s timeout, `availability` tụt mạnh xuống **0.9488** (vi phạm SLO >= 95%), xuất hiện **107 lượt give_up do hết deadline** và **15 lượt no_provider_available**.
   - *Kết luận:* Vì deadline của toàn bộ turn là 30s, nếu timeout của mỗi attempt là 30s thì khi gặp provider chết hẳn, request sẽ treo toàn bộ 30s ở provider đầu tiên và không còn một mili-giây nào để failover. Timeout "đúng" phải dựa trên phân phối độ trễ bình thường (3–4× median ~ 5–8s), tuyệt đối không đặt bằng turn deadline.
3. **Bỏ lọc tool calling khỏi `ladder()`.**
   - Chạy G6 khi không lọc tool calling: `availability` sụp đổ xuống **0.8927**, phát sinh **256 lượt `incompatible_routes`**.
   - *Góc nhìn Product Manager:* 256 người dùng yêu cầu hành động nghiệp vụ (như kiểm tra trạng thái đơn, tính tiền hoàn) đã bị chuyển nhầm sang model gamma-mini (vốn không có khả năng gọi tool). Với khách hàng, chatbot đột ngột không thể thao tác nghiệp vụ, sinh ra câu trả lời ảo tưởng (hallucination) hoặc từ chối phục vụ vô cớ, trực tiếp phá hủy niềm tin của khách hàng vào sản phẩm và gây thiệt hại doanh thu.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trả lời:
- Vì sao chỉ idempotency key là **chưa đủ**: Idempotency key bảo đảm một lệnh cụ thể không bị thực thi lặp lại ở phía server thứ ba. Tuy nhiên, nếu không có checkpoint, khi worker bị crash ở bước sau (ví dụ gửi email hoặc tóm tắt), worker mới sẽ phải chạy lại từ bước 1, tiêu tốn lãng phí token LLM và thời gian (`wasted_llm_per_crash` của naive lên tới 1.123).
- Vì sao chỉ checkpoint cũng **chưa đủ**: Nếu mạng bị ngắt (ServiceTimeout) ngay sau khi cổng thanh toán đã trừ/hoàn tiền nhưng response bị mất, retry bước thanh toán nếu không có idempotency key cố định sẽ bị cổng thanh toán hiểu là một giao dịch mới, gây duplicate refund (mất 1,574,200 xu tiền overpaid ở bản naive).
- Điều gì xảy ra nếu ghi checkpoint *trước* khi gọi payment: Nếu ghi checkpoint trước khi gọi payment, lỡ worker bị crash ngay trước khi lệnh gửi đi, worker mới thức dậy sẽ thấy checkpoint đã ghi nhận bước payment hoàn thành và bỏ qua nó. Hậu quả là khách hàng nhận được email thông báo hoàn tiền nhưng tài khoản thực tế không bao giờ nhận được tiền (**lost refund** - lỗi nghiêm trọng nhất).

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (delay 1m) | page | 27 min |
| provider_outage | page | page (delay 0m) | page | 4 min |
| deploy_blip | none | page (FALSE ALARM) | none (paged=False) | — |
| slow_leak | ticket | not detected | ticket | 1337 min |
| degraded_evening | page | page (delay 214m) | page | 291 min |
| night_spike | none | page (FALSE ALARM) | none (paged=False) | — |
| small_bump | none | page (FALSE ALARM) | none (paged=False) | — |

- Error budget 30 ngày: Tổng 25,901,755 requests; Allowed bad = 129,508.8; Actual bad = 55,618.0; Availability = 99.7853%.
- Budget đã tiêu **42.95%**, còn lại **57.05%**. **Không đóng băng release** (`freeze_releases: False`) vì ngân sách lỗi vẫn còn hơn 57%.
- Vì sao `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa: 20 phút ở 12% lỗi chỉ tiêu tốn chưa đầy 1% error budget của tháng. Nó tự hồi phục và không đe dọa đến mục tiêu SLO 99.5% 30 ngày. Đánh thức on-call với sự cố nhỏ này gây mỏi mệt cảnh báo (alert fatigue), làm giảm độ nhạy bén của đội ngũ khi có thảm họa thực sự xảy ra.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):** Lúc t=120, hạ tầng GPU của provider sơ cấp (alpha) gặp sự cố giảm 75% năng lực xử lý. Client v1 (naive) phản ứng bằng cơ chế 3 tầng retry không kiểm soát (36 attempts/call, không jitter, không breaker), khuếch đại lưu lượng gửi tới alpha lên gấp 10 lần (amplification 9.904). Tình trạng này khiến hệ thống rơi vào trạng thái hỏng hóc tự duy trì (metastable failure), kéo dài sự cố thêm 155 giây sau khi provider đã hoàn toàn bình phục, làm availability sụp xuống 45.6%.

**Ảnh hưởng:** 1,285 lượt người dùng thất bại (availability chỉ đạt 45.6%, 1077/2362 turns hoàn thành); sự cố kéo dài tổng cộng 335 giây (t=120 đến t=455); tiêu tốn 4,983 cost units (4.627 cost/turn, gấp đôi bình thường); thiêu rụi hơn 100% error budget của tháng nếu xét độc lập.

**Timeline** (giây ảo — lấy từ CSV):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực | `calls_ok = 7`, `calls_done = 7`, `attempts_alpha = 5` (bắt đầu nghẽn) |
| 130 | Hàng đợi server tích lũy độ trễ | `calls_ok = 3`, `calls_done = 3`, `attempts_alpha = 10` |
| 150 | Client bắt đầu timeout hàng loạt, bão retry bùng nổ | `calls_ok = 0`, `calls_done = 3`, `attempts_alpha = 163` (tải vọt gấp 16 lần) |
| 200 | Hệ thống tê liệt hoàn toàn dưới bão retry | `calls_ok = 0`, `calls_done = 1`, `attempts_alpha = 135`, 529 và 429 dồn dập |
| 250 | Đỉnh điểm bão retry | `calls_ok = 0`, `calls_done = 6`, `attempts_alpha = 184` |
| 300 | alpha hồi phục 100% GPU hạ tầng | `calls_ok = 0`, `calls_done = 5`, `attempts_alpha = 115` |
| 350 | Provider đã khỏe nhưng hệ thống vẫn sập | `calls_ok = 0`, `calls_done = 4`, `attempts_alpha = 17` (hàng đợi nghẹt request ma) |
| 450 | Client cạn dần lượt retry, hàng đợi rác dần cạn | `calls_ok = 0`, `calls_done = 3`, `attempts_alpha = 16` |
| 455 | Hệ thống bắt đầu phục hồi (155s sau khi alpha khỏe) | `calls_ok = 4`, `calls_done = 5` (tỷ lệ 80%), `attempts_alpha = 13` |
| 460 | Hệ thống trở lại bình thường | `calls_ok = 9`, `calls_done = 12` (tỷ lệ 75%), `attempts_alpha = 14` |

**Trigger vs root cause:**
- **Trigger:** Provider alpha mất 75% GPU tại t=120 trong 3 phút.
- **Root cause:** Client v1 có 3 tầng retry lồng nhau (app × gateway × SDK = tối đa 36 attempts/call), khoảng cách retry cố định không có full jitter, thiếu circuit breaker để ngắt mạch và không có retry budget.
- **Metastable failure:** Server xử lý theo hàng đợi FIFO và vẫn tiếp tục xử lý các request cũ ngay cả khi client đã timeout bỏ cuộc. Khi client timeout và gửi retry mới, các request retry tiếp tục chèn vào đuôi hàng đợi trong khi GPU đang bận tính toán các "request ma" vô ích ở đầu hàng đợi. Tải do chính client tạo ra vượt quá năng lực phục vụ của alpha, tạo ra vòng lặp tự duy trì (self-sustaining overload) khiến server không thể tự thoát ra dù đã có lại 100% GPU lúc t=300.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Retry mù quáng không jitter; chồng lớp retry; không failover sang alpha-eu hay beta; server xử lý request không kiểm tra xem client còn sống không.
- *Giúp giải quyết:* Circuit breaker lập tức OPEN để fail-fast và giải phóng tải cho alpha; Failover sang alpha-eu/beta; Retry budget khống chế lượng retry tối đa 10%; Full jitter backoff phân tán tải.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế NaiveClient bằng ResilientClient với sliding-window breaker và retry budget 10% | Prevent | Le Nhu Y (Backend/Platform) | Amplification <= 1.2, recovery_s = 0.0s trên kịch bản G2 trong CI |
| 2 | Thiết lập multi-window burn-rate alerting (Google SRE) cho SLO availability 99.5% | Detect | SRE On-call Team | Cảnh báo page kích hoạt trong vòng 4-5 phút khi outage, 0 false page trên blip |
| 3 | Triển khai deadline propagation và cancellation header từ client đến API gateway | Mitigate | Infra / Gateway Team | Gateway hủy ngay lập tức các request đã quá deadline trong hàng đợi |

## 7. Một điểm yếu còn lại

- **Điểm yếu còn lại:** Khi provider primary (alpha) bị sập, ResilientClient failover sang beta. Tuy nhiên beta có chi phí cao hơn 40% (1.4 cost unit so với 1.0 của alpha) và rate limit chỉ có 16 rps. Nếu sự cố alpha kéo dài trong đợt cao điểm traffic, toàn bộ traffic đổ dồn về beta sẽ làm tăng chi phí vận hành đột biến và có nguy cơ gây quá tải dây chuyền (cascading failure) cho beta.
- **Cách khắc phục:**
  1. Áp dụng cơ chế **load shedding / graceful degradation**: Khi failover sang beta trong thời gian dài, client có thể chủ động chuyển các tác vụ không quan trọng hoặc người dùng miễn phí sang model rẻ hơn (gamma-mini) hoặc tắt bớt tính năng phụ.
  2. Kết hợp chặt chẽ với **Semantic Cache (Part G)**: Phục vụ ngay từ cache cho các câu hỏi phổ biến, giảm 50–60% lượng truy vấn chạm tới provider trong thời gian failover.
  3. Cài đặt **Client-side Concurrency Limiting (Adaptive Throttling)**: Dựa trên TCP Vegas hoặc CoDel để tự động bóp lưu lượng gửi sang beta tương ứng với giới hạn 16 rps.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9120 | 0.9645 | 0.635 | 174 | 0 | 6 | 0.927 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở 0.80 phát sinh 138 lỗi `other_question` (trả lời nhầm câu hỏi khác). Ở 0.92 loại bỏ 100% lỗi sai (`wrong_answers: 0`) mà vẫn giữ cache hit ratio cao ở 63.5% (vượt mốc yêu cầu 50%). |
| ttl_s | 45 | Tại t=480 bảng phí ship thay đổi ngầm không có webhook. Nếu TTL = 300s thì có tới 95 câu trả lời lỗi thời (`outdated_answers`). TTL = 45s giữ outdated_answers ở mức 6 (thỏa mãn tiêu chí <= 15). |
| max_stale_s | 300 | Sự cố tất cả provider sập kéo dài 120s (t=240 đến t=360). max_stale_s = 300s (> 120s) cho phép stale-if-error phục vụ 174 câu stale, đưa FAQ availability đạt 96.45% (vượt tiêu chuẩn >= 93%). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.**
   - Ngưỡng 0.80: `hit = 0.647`, `wrong_answers = 138` (lý do: `other_question: 138`), `cost/turn = 0.971`.
   - Ngưỡng 0.92: `hit = 0.635`, `wrong_answers = 0`, `cost/turn = 0.927`.
   - Ngưỡng 0.97: `hit = 0.552`, `wrong_answers = 0`, `cost/turn = 0.984`.
   - *Nhận xét:* Ở 0.80, các câu hỏi có embedding gần nhau về mặt từ ngữ nhưng khác biệt về ngữ nghĩa nghiệp vụ (ví dụ: "chính sách hoàn tiền" vs "thời gian hoàn tiền", "phí vận chuyển" vs "thời gian giao hàng") bị nhận diện nhầm, dẫn đến 138 câu trả lời sai. Ngưỡng 0.92 là điểm cân bằng tối ưu: loại bỏ hoàn toàn câu trả lời sai mà vẫn giữ hit ratio 63.5%.
2. **TTL.**
   - TTL = 30s: `outdated = 2`, `hit = 0.592`, `cost/turn = 0.955`.
   - TTL = 45s: `outdated = 6`, `hit = 0.635`, `cost/turn = 0.927`.
   - TTL = 60s: `outdated = 3`, `hit = 0.652`, `cost/turn = 0.915`.
   - TTL = 300s: `outdated = 95`, `hit = 0.731`, `cost/turn = 0.863`.
   - *Nhận xét:* Webhook chỉ phát huy tác dụng khi hệ thống CMS chủ động bắn thông báo khi cập nhật nội dung. Khi bên thứ ba (hãng vận chuyển) âm thầm thay đổi bảng giá cước lúc t=480 mà không có webhook, hệ thống hoàn toàn không biết để invalidate. Do đó, TTL là cơ chế an toàn duy nhất tự động vô hiệu hóa các câu trả lời cũ sau một khoảng thời gian hữu hạn.
3. **Stale-if-error.**
   - Khi đặt `max_stale_s: 0`: `faq_availability` sụp từ **0.9645** xuống **0.8634** (vi phạm điều kiện >= 93%), `stale_served = 0`, `hit = 0.534`.
   - *Nhận xét:* Trong 120s mọi provider bị sập (t=240-360), nếu không cho phép trả câu trả lời cũ (stale), toàn bộ câu hỏi FAQ không còn tươi sẽ bị lỗi ngay lập tức khi provider từ chối, khiến độ khả dụng của FAQ rớt xuống dưới mức cam kết.

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Bảo mật & Rò rỉ dữ liệu (PII):* Ngay cả khi cache theo user, nếu có sai sót trong việc tạo key hoặc embedding collision, câu trả lời mang thông tin riêng tư (địa chỉ, số điện thoại, giá trị đơn) có thể bị phục vụ nhầm cho người khác.
  2. *Tính thời gian thực (Real-time data):* Trạng thái đơn hàng liên tục biến đổi (đang đóng gói -> đã xuất kho -> đang giao). Câu trả lời cache sẽ đưa thông tin sai lệch cho khách hàng, gây hiểu lầm và bức xúc.
  3. *Tỷ lệ cache hit cực thấp:* Khách hàng hiếm khi hỏi lặp đi lặp lại cùng một câu hỏi cá nhân với cùng một trạng thái, việc cache tốn bộ nhớ vô ích.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  - Không được chỉ dựa vào metric user-facing availability hay HTTP status trả về cho client.
  - Phải đặt alert dựa trên:
    1. **Tỷ lệ phục vụ stale (`cache.stale_served_rate` hoặc `cache.stale_ratio > 0.05`):** Khi tỷ lệ stale tăng đột biến, chứng tỏ tầng backend provider đang thất bại trong việc revalidate.
    2. **Tỷ lệ lỗi ở tầng provider (`upstream_provider.error_rate > 5%` hoặc `provider.circuit_breaker.state == OPEN`):** Giám sát trực tiếp sức khỏe kết nối từ client tới các LLM deployment.
    3. **Tỷ lệ cache miss nhưng provider trả về GiveUp:** Số lượng yêu cầu mới không thể xử lý được vì không có sẵn trong cache.
