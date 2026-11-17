# Cẩm nang ôn bảo vệ: Kiến trúc, AI và Data Freshness

> Mục tiêu: đọc trong khoảng 10–15 phút để hiểu đúng luồng đang có trong code.  
> Phạm vi: kiến trúc hệ thống, dịch/TTS, AI Recognition, Recommend/Trip Planner, AI Chat và Data Freshness.

## 1. Bức tranh tổng thể

HelloVietnam được tách thành bốn khối trách nhiệm:

1. **Client:** ứng dụng người dùng Flutter trên Android và trang quản trị Flutter Web.
2. **Supabase:** Auth, Edge Functions, PostgREST/RPC, PostgreSQL và phân quyền.
3. **FastAPI:** dịch vụ Python chạy thuật toán Recommend và Trip Planner.
4. **Dịch vụ ngoài:** DeepSeek, Gemini, VBee, OSM, Goong, Firebase FCM, Cloudflare R2, Stripe Sandbox và API tỷ giá.

Luồng tổng quát:

```text
Flutter Android / Flutter Web
        |
        | JWT của người dùng
        v
Supabase Auth + Edge Functions
        |                         |
        | HTTP                    | API có secret phía server
        v                         v
Python FastAPI              DeepSeek / Gemini / VBee /
        |                   Stripe / R2 / Firebase...
        v
Supabase PostgREST/RPC + PostgreSQL
```

Điểm cần nhớ: ứng dụng **không giữ service-role key hoặc API key của nhà cung cấp**. Những secret này nằm ở Edge Function hoặc FastAPI. Client chỉ giữ cấu hình công khai cần thiết và gửi JWT để backend xác thực.

### Vì sao vừa dùng Supabase vừa dùng FastAPI?

- **Supabase** phù hợp với xác thực, phân quyền, CRUD, RPC, database và các API điều phối ngắn.
- **FastAPI/Python** phù hợp với Collaborative Filtering, phân cụm, tối ưu lịch trình và các thư viện tính toán Python.
- Edge Function là lớp gateway: kiểm tra người dùng rồi chuyển request Trip Planner/Recommend sang FastAPI.

Ví dụ luồng tạo lịch trình:

```text
Người dùng nhập tỉnh, số ngày, sở thích, ngày bắt đầu
  -> Flutter gọi Edge Function trip-planner
  -> Edge Function xác thực JWT và lấy đúng id_user
  -> Edge Function gọi FastAPI
  -> FastAPI đọc hồ sơ, tag, địa điểm và CF factors từ Supabase
  -> xếp hạng + chia địa điểm theo ngày + tối ưu tuyến
  -> gọi Goong để lấy thời gian/khoảng cách thực tế
  -> có thể lưu plan vào PostgreSQL
  -> trả lịch trình về Flutter
```

## 2. Các tính năng AI và thuật toán

### 2.1. Dịch và đọc văn bản

Hệ thống có hai cơ chế dịch và hai cơ chế đọc:

| Cơ chế | Nơi xử lý | Nhà cung cấp | Điểm mạnh | Điều kiện/giới hạn |
|---|---|---|---|---|
| Dịch offline | Thiết bị Android | Google ML Kit | Nhanh, riêng tư, không tốn API mỗi lần | Phải tải model ngôn ngữ trước; chất lượng ngữ cảnh hạn chế hơn |
| Dịch online | Supabase Edge Function | DeepSeek | Bản dịch có ngữ cảnh và tự nhiên hơn | Cần mạng, đăng nhập, chịu quota/rate limit |
| Đọc offline | Thiết bị Android | Flutter TTS/Android TTS | Phản hồi nhanh, không gọi server | Thiết bị phải có engine và voice tương ứng |
| Đọc online | Supabase Edge Function | VBee | Giọng tổng hợp trực tuyến | Cần mạng và cấu hình voice/API |

Luồng dịch offline:

```text
Văn bản -> model ML Kit đã tải trên máy -> bản dịch -> hiển thị
```

Luồng dịch online:

```text
Văn bản -> Edge Function translate -> xác thực + rate limit
        -> DeepSeek -> kiểm tra JSON -> trả bản dịch
```

Backend dịch online đang giới hạn tối đa **200 request/ngày** và **30 request/phút** cho mỗi người dùng. Prompt yêu cầu giữ nguyên ý nghĩa, giọng điệu, định dạng, số, tên riêng và trả JSON có cấu trúc.

Điểm nên nói chính xác: offline không có nghĩa là cài app xong sẽ dịch mọi ngôn ngữ ngay khi mất mạng; model cần được tải xuống trước.

### 2.2. AI Recognition

AI Recognition dùng **Gemini** để phân tích ảnh trong ngữ cảnh du lịch Việt Nam.

Luồng xử lý:

```text
Camera/Gallery
  -> Flutter đọc ảnh, xác định MIME và mã hóa Base64
  -> Edge Function ai-search
  -> Gemini trả JSON theo schema cố định
  -> backend chuẩn hóa loại kết quả và confidence
  -> nếu là food: đối chiếu thêm với food catalog trong database
  -> Flutter chọn UI theo result_kind
  -> kết quả hợp lệ được lưu vào lịch sử cục bộ
```

Sáu kết quả cấp cao:

| `result_kind` | Ý nghĩa | Hành vi chính |
|---|---|---|
| `food` | Món ăn | Hiển thị thông tin; thử đối chiếu food catalog; có thể mở chi tiết món |
| `landmark` | Địa danh/điểm tham quan | Hiển thị ngữ cảnh; có thể mở Maps nếu có query hợp lệ |
| `cultural_object` | Đồ vật/vật phẩm văn hóa | Hiển thị vật liệu, cách sử dụng, ý nghĩa; Maps có điều kiện |
| `sign_text` | Biển có chữ | OCR, dịch chữ; có thể đọc và mở Maps tùy loại biển |
| `unclear` | Ảnh không đủ rõ/chắc chắn | Không lưu lịch sử; yêu cầu chụp hoặc chọn ảnh khác |
| `unsupported` | Ảnh ngoài phạm vi/nhạy cảm | Không đưa ra phán đoán; không lưu lịch sử |

Quy tắc confidence:

- `>= 0.80`: `confident`.
- Từ `0.55` đến dưới `0.80`: `tentative`.
- Dưới `0.55`: kết quả được hỗ trợ bị hạ thành `unclear`.

Các guardrail quan trọng:

- Gemini phải trả JSON theo contract; backend không render thẳng câu trả lời tự do.
- Backend chuẩn hóa confidence và xóa các claim không đủ bằng chứng.
- Ảnh người, selfie, giấy tờ, thẻ thanh toán hoặc nội dung không an toàn được đưa về `unsupported`.
- Chỉ món ăn được đối chiếu với catalog nội bộ; hệ thống không khẳng định mọi kết quả Gemini đều có trong database.
- Có cơ chế thử lần lượt nhiều Gemini key/model đã cấu hình khi gặp lỗi credential, quota, model hoặc network. Đây là fallback **trong Gemini**, không phải tự động đổi sang DeepSeek để nhận diện ảnh.
- Lịch sử nhận diện và thumbnail hiện lưu bằng `SharedPreferences`, tách theo user; đây không phải lịch sử đồng bộ cloud.

### 2.3. Recommend và Trip Planner có phải Generative AI không?

Không nên gọi toàn bộ phần này là AI tạo sinh. Đây chủ yếu là hệ gợi ý và tối ưu thuật toán:

1. **Content-Based:** so khớp tag địa điểm với sở thích onboarding, lựa chọn chuyến đi và hành vi người dùng.
2. **Collaborative Filtering:** dùng `cf_user_factors` và `cf_place_factors`; điểm CF được tính bằng tích vô hướng tại thời điểm request.
3. **Hybrid score:** trộn điểm tag và điểm CF bằng hệ số `alpha` động.
4. **Diversity selection:** tránh chọn quá nhiều địa điểm cùng một loại.
5. **K-Means + Greedy Repair:** phân chia địa điểm vào từng ngày và sửa cụm để phù hợp số lượng/nhịp độ.
6. **Simulated Annealing:** tối ưu thứ tự ghé thăm, có xét lịch mở cửa, khung giờ và thời gian nghỉ trưa.
7. **Goong Distance Matrix:** sau khi đã có tuyến tốt, lấy thời gian/khoảng cách ô tô và xe máy. Nếu dữ liệu Goong của một ngày không đầy đủ, ngày đó giữ lịch tính bằng Haversine, không trộn kết quả nửa đúng nửa thiếu.

Công thức khái quát:

```text
final_score = alpha * tag_match + (1 - alpha) * cf_score
```

Với người dùng mới chưa có đủ dữ liệu CF, hệ thống vẫn có thể dựa nhiều hơn vào hồ sơ sở thích và tag; đây là cách giảm bài toán cold start.

### 2.4. AI Chat

AI Chat dùng DeepSeek, model mặc định trong code là `deepseek-v4-flash`. Edge Function chịu trách nhiệm xác thực, giới hạn lượt dùng, chuẩn bị lịch sử hội thoại, gọi model và ghi nhận token/chi phí ước tính. Model có thể trả thêm action có cấu trúc để ứng dụng mở một màn hình phù hợp; VBee được dùng khi người dùng yêu cầu đọc câu trả lời.

## 3. Data Freshness và cập nhật dữ liệu

### 3.1. Mục tiêu

Data Freshness không nhằm tự động viết lại toàn bộ nội dung biên tập. Nó theo dõi dữ liệu nguồn, phát hiện bản ghi đến hạn, nguồn biến mất hoặc trường vận hành thay đổi, sau đó tạo hàng đợi để admin kiểm tra.

Năm loại nội dung được mô hình hỗ trợ:

- `place`
- `activity`
- `culture`
- `food`
- `local_product` tương ứng bảng `local_products`

Tuy nhiên, checker hiện chỉ xử lý các row có nguồn được hỗ trợ và đủ định danh. Adapter hiện có là **OSM** và **Wikipedia**. Các row `legacy_import` không có nguồn thật sẽ không được checker claim.

### 3.2. Crawler và checker là hai luồng khác nhau

| Thành phần | Mục đích | Cách chạy hiện tại | Có giao diện admin điều khiển? |
|---|---|---|---|
| Python crawler | Thu thập/nhập dữ liệu ban đầu, xuất JSON/CSV và có thể upsert | Chạy thủ công bằng script/CLI | Chưa |
| Freshness checker | Kiểm tra lại nguồn của dữ liệu đã nhập và tạo proposal/audit | Cron hằng ngày hoặc trigger thủ công | Admin xem kết quả; nút kiểm tra lại chỉ đưa row về `due` |

Điểm dễ nói sai: checker **không đọc cache của crawler** và **không chạy lại toàn bộ Python crawler**. Khi chạy, checker trực tiếp gọi adapter OSM/Wikipedia cho từng row đến hạn, chuẩn hóa dữ liệu nguồn, tạo hash và so sánh các trường được phép.

### 3.3. Luồng checker

```text
Supabase cron lúc 19:15 UTC (02:15 Việt Nam)
  -> gọi Edge Function data-freshness-check bằng secret nội bộ
  -> tạo content_update_run
  -> claim tối đa 50 row đến hạn, có nguồn được hỗ trợ
  -> tối đa 5 worker kiểm tra song song
  -> adapter gọi OSM/Wikipedia, retry có giới hạn và timeout 30 giây/lần
  -> so sánh với dữ liệu hiện tại
  -> cập nhật freshness / tạo proposal / auto-expire sự kiện
  -> ghi số liệu và trạng thái run
  -> admin xem tại Data Freshness
```

Cron được cấu hình trong migration với lịch `15 19 * * *`. Muốn chứng minh nó đang chạy trên môi trường thật phải kiểm tra `cron.job` và tab **Run history**; việc có code cron trong repository chưa tự nó chứng minh deployment đang active.

### 3.4. Bao lâu thì kiểm tra lại?

Sau một lần kiểm tra, `next_check_at` được đặt theo loại dữ liệu:

| `availability_type` | Chu kỳ tiếp theo |
|---|---:|
| `scheduled_event` | 1 ngày |
| `business` | 7 ngày |
| `seasonal` | 14 ngày |
| `evergreen` | 30 ngày |

Cron chạy mỗi ngày nhưng **chỉ claim những row có `next_check_at <= now()`**, không kiểm tra lại mọi bản ghi mỗi ngày.

### 3.5. Ý nghĩa trạng thái

| Trạng thái | Ý nghĩa |
|---|---|
| `fresh` | Lần kiểm tra gần nhất hợp lệ và không có thay đổi cần duyệt |
| `due` | Đã đến hạn hoặc admin yêu cầu kiểm tra lại |
| `stale` | Nguồn hợp lệ báo thiếu lần thứ nhất; chưa sửa dữ liệu chính |
| `needs_review` | Có thay đổi nguồn hoặc nguồn thiếu lần thứ hai; đã có proposal chờ admin |
| `expired` | Sự kiện có `valid_until` đã qua và được tự hết hạn |

Tab **Stale data** trên admin hiện hiển thị cả `due` và `stale`, nên con số tại đây không đồng nghĩa tất cả đều đã được xác nhận là sai.

Quy tắc xử lý kết quả:

- Nguồn tồn tại, hash và các trường không đổi → `fresh`.
- Nguồn tồn tại nhưng trường vận hành thay đổi → tạo proposal và chuyển `needs_review`.
- Nguồn thiếu lần thứ nhất → `stale`, bản ghi chính vẫn active.
- Nguồn thiếu lần thứ hai liên tiếp → tạo proposal `possibly_closed`, đề xuất `status = archived`.
- Nguồn timeout, HTTP 429/5xx hoặc parse lỗi → ghi `last_error`, không tăng bộ đếm missing và không tự kết luận đóng cửa.
- Nguồn xuất hiện lại → đặt lại bộ đếm missing; proposal cũ có thể được đóng nếu không còn phù hợp.
- Sự kiện đã qua `valid_until` → tự động đặt `expired`; đây là đường auto-update nội dung rõ ràng duy nhất hiện nay.

### 3.6. Checker được phép theo dõi trường nào?

Các trường nguồn vận hành được allowlist:

```text
name, address, latitude, longitude,
timespan, timeclose, phone, website, status
```

Checker không tự ghi đè mô tả biên tập, ảnh tuyển chọn, tag, rating hoặc nội dung marketing. Thay đổi thông thường chỉ được áp dụng sau khi admin duyệt proposal. Khi duyệt, database còn kiểm tra dữ liệu hiện tại có bị người khác sửa sau lúc proposal được tạo hay không để tránh ghi đè nhầm.

Trong hàm apply hiện tại, bảng `place` hỗ trợ toàn bộ allowlist trên; với `activity`, `culture`, `food` và `local_products`, phần cập nhật trực tiếp hiện chỉ áp dụng `name` và `status`. Đây là giới hạn triển khai cần nói rõ nếu hội đồng hỏi sâu theo từng bảng.

### 3.7. Bốn tab trên admin

1. **Pending review:** so sánh dữ liệu trước/sau; admin duyệt hoặc từ chối proposal.
2. **Reports:** báo cáo của người dùng như đóng cửa, sai giờ, sai vị trí, sự kiện đã kết thúc hoặc lỗi khác.
3. **Stale data:** các row `due`/`stale`, lỗi gần nhất và nút kiểm tra lại.
4. **Run history:** nguồn trigger, số row được chọn/đã kiểm tra, proposal, auto-applied, lỗi và trạng thái run.

Hai nút dễ hiểu nhầm:

- **Refresh data:** chỉ reload dữ liệu trên giao diện.
- **Check again:** đặt `next_check_at = now()` và trạng thái `due`; nó không tự chạy checker ngay trong cùng request. Row sẽ được xử lý ở lượt checker tiếp theo.

Người dùng có thể gửi tối đa ba báo cáo mỗi ngày UTC; báo cáo trùng lý do trên cùng nội dung sẽ không được tạo thêm khi một báo cáo giống vậy còn mở.

### 3.8. Vì sao giới hạn 50 bản ghi mỗi lượt?

Giới hạn này bảo vệ Edge Function và nguồn ngoài khỏi timeout/rate limit. Checker còn giới hạn tối đa năm worker song song, mỗi request có timeout và retry hữu hạn. Đây là batch an toàn, không phải giới hạn dữ liệu toàn hệ thống.

Nếu quy mô tăng, có thể:

- tăng tần suất cron;
- chia shard theo `content_type` hoặc nguồn;
- dùng queue/worker riêng;
- thêm adapter nhà cung cấp chính thống;
- điều chỉnh batch sau khi đo rate limit và thời gian chạy.

Hạn chế hiện tại cần nói trung thực: đây là workflow demo-ready với OSM/Wikipedia, chưa phải hệ thống xác minh đa nguồn ở quy mô production. Nếu backlog lớn hơn 50 row/ngày trong thời gian dài thì cần tăng throughput.

## 4. Câu hỏi hội đồng có thể đặt ra

### Kiến trúc

**1. Tại sao không làm tất cả trong Supabase Edge Functions?**  
Edge Functions phù hợp với gateway và nghiệp vụ ngắn, nhưng thuật toán gợi ý, K-Means và Simulated Annealing thuận tiện hơn trong Python. Tách FastAPI giúp dùng hệ sinh thái Python và mở rộng tài nguyên tính toán độc lập.

**2. Tại sao app không gọi thẳng FastAPI?**  
Đi qua Edge Function giúp xác thực JWT, không tin `id_user` do client tự khai, tập trung CORS/error handling và không để lộ URL/secret backend nhạy cảm.

**3. Nếu một dịch vụ ngoài bị lỗi thì sao?**  
Mỗi luồng có xử lý lỗi riêng. Dịch offline vẫn dùng được khi model đã tải; Trip Planner có Haversine fallback nếu Goong không đủ dữ liệu; Data Freshness ghi lỗi thay vì kết luận nguồn biến mất; AI Recognition trả lỗi hoặc trạng thái an toàn thay vì ghi dữ liệu tùy tiện.

**4. Bảo mật database như thế nào?**  
Client dùng Supabase Auth JWT và quyền/RLS. Secret nhà cung cấp và service-role key chỉ nằm phía server. Admin API kiểm tra role admin. Checker cron dùng secret riêng trong header và secret này phải khớp giữa Vault với Edge Function.

### AI

**5. Tại sao cần cả ML Kit và DeepSeek cho dịch?**  
ML Kit tối ưu cho tốc độ, riêng tư và offline; DeepSeek tối ưu chất lượng ngữ cảnh. Hệ thống đánh đổi giữa khả dụng/chi phí và chất lượng.

**6. Dịch offline có hoạt động ngay khi không có mạng không?**  
Chỉ khi model ngôn ngữ cần thiết đã được tải xuống thiết bị trước đó.

**7. Làm sao giảm hallucination khi nhận diện ảnh?**  
Yêu cầu Gemini trả JSON theo schema, chuẩn hóa contract ở backend, dùng ngưỡng confidence, đưa ảnh thiếu bằng chứng về `unclear`, chặn nội dung ngoài phạm vi bằng `unsupported`, và chỉ đối chiếu database với món ăn khi đủ điều kiện.

**8. Gemini nhận diện được bao nhiêu nhóm?**  
Bốn nhóm nội dung chính: food, landmark, cultural object và sign text; thêm hai trạng thái an toàn là unclear và unsupported.

**9. Khi Gemini hết quota có tự chuyển sang DeepSeek không?**  
Không. Hiện tại hệ thống thử các Gemini key/model fallback đã cấu hình. DeepSeek đang dùng cho dịch và chat, không phải fallback đa phương thức cho nhận diện ảnh.

**10. Recommend có dùng LLM không?**  
Luồng Recommend/Trip Planner chính không phụ thuộc LLM. Nó dùng Content-Based, Collaborative Filtering, K-Means, Greedy Repair và Simulated Annealing. Đây là hệ gợi ý/tối ưu, không phải AI tạo sinh.

**11. Người dùng mới chưa có lịch sử thì gợi ý bằng gì?**  
Dùng sở thích onboarding, lựa chọn chuyến đi và tag địa điểm. Khi có thêm hành vi/CF factors, điểm Collaborative Filtering mới đóng góp nhiều hơn.

**12. Vì sao dùng Simulated Annealing?**  
Bài toán sắp thứ tự nhiều địa điểm với thời gian mở cửa và khung giờ có không gian nghiệm lớn. Simulated Annealing tìm nghiệm tốt trong thời gian hợp lý thay vì vét cạn mọi hoán vị.

### Data Freshness

**13. Crawler và checker có phải cùng một thứ không?**  
Không. Crawler nhập dữ liệu ban đầu và đang chạy thủ công. Checker là Edge Function chạy theo cron hoặc trigger, trực tiếp kiểm tra lại nguồn của row đã có.

**14. Checker so sánh với cache crawler đúng không?**  
Không. Checker gọi adapter OSM/Wikipedia tại thời điểm chạy, chuẩn hóa payload và so sánh với bản ghi hiện tại/source hash.

**15. Khi nào một row bị stale?**  
Trong domain hiện tại, `due` chỉ là đến hạn. `stale` là lần đầu nguồn hợp lệ báo không tìm thấy. Lần thứ hai liên tiếp mới tạo proposal `possibly_closed`.

**16. Phát hiện thay đổi có tự sửa database ngay không?**  
Không đối với thay đổi thông thường. Hệ thống tạo proposal để admin duyệt. Chỉ sự kiện đã qua `valid_until` được auto-expire.

**17. Checker kiểm tra những bảng và trường nào?**  
Mô hình hỗ trợ place, activity, culture, food và local products. Checker chỉ claim row có nguồn OSM/Wikipedia hợp lệ. Các trường nguồn allowlist gồm tên, địa chỉ, tọa độ, giờ mở/đóng, điện thoại, website và trạng thái.

**18. Cron chạy lúc nào?**  
Migration cấu hình `19:15 UTC`, tương đương `02:15` giờ Việt Nam. Cron chạy mỗi ngày nhưng chỉ lấy row đã đến `next_check_at`.

**19. Tại sao mỗi lượt chỉ kiểm tra tối đa 50 row?**  
Để giới hạn thời gian Edge Function và tránh rate limit OSM/Wikipedia. Đây là batch; có thể tăng throughput bằng nhiều lịch chạy hoặc queue sau khi đo tải.

**20. `partial_failure` có nghĩa toàn bộ lượt chạy thất bại không?**  
Không. Nó nghĩa có một số row kiểm tra thành công và một số row lỗi. Mỗi row vẫn được ghi audit riêng; lỗi nguồn không bị coi là missing.

**21. Nút “Check again” có chạy ngay không?**  
Không. Nó chỉ đặt row về `due` với `next_check_at = now()`. Cần lượt checker tiếp theo để thực sự gọi nguồn.

**22. Admin Data Freshness có chạy được Python crawler không?**  
Chưa. Admin hiện quản lý proposal, báo sai, stale và lịch sử checker; crawler vẫn là công cụ CLI riêng.

**23. Làm sao tránh nguồn ngoài sai làm hỏng dữ liệu?**  
Allowlist host và trường, retry/timeout hữu hạn, hai lần missing trước khi đề xuất đóng, không tăng missing khi lỗi kỹ thuật, proposal cần admin duyệt và kiểm tra optimistic concurrency trước khi apply.

**24. Hệ thống đã production-grade chưa?**  
Phần workflow, audit và quyền đã có; nhưng adapter mới tập trung OSM/Wikipedia và batch còn nhỏ. Cách nói đúng là “demo-ready và có nền tảng mở rộng”, chưa nên tuyên bố xác minh đa nguồn production-grade.

## 5. Những câu nên tránh khi thuyết trình

| Không nên nói | Nên nói |
|---|---|
| “Mọi bản ghi được checker kiểm tra mỗi ngày.” | “Cron chạy mỗi ngày và lấy tối đa 50 row đến hạn có nguồn được hỗ trợ.” |
| “Stale nghĩa là dữ liệu chắc chắn sai.” | “Stale nghĩa nguồn bị thiếu lần đầu; cần kiểm tra tiếp.” |
| “Checker lấy dữ liệu crawler đã cache để so sánh.” | “Checker trực tiếp truy vấn adapter OSM/Wikipedia khi chạy.” |
| “Hệ thống tự động cập nhật mọi thay đổi.” | “Thay đổi thường tạo proposal; chỉ sự kiện hết hạn được auto-expire.” |
| “DeepSeek fallback cho Gemini Recognition.” | “Recognition fallback giữa các Gemini key/model; DeepSeek phục vụ dịch/chat.” |
| “Trip Planner là Generative AI.” | “Trip Planner là pipeline gợi ý và tối ưu thuật toán.” |
| “Dịch offline luôn dùng được ngay.” | “Dịch offline dùng được sau khi model đã tải.” |

## 6. Bản ghi nhớ 60 giây

> Client của HelloVietnam gồm Flutter Android và Flutter Web Admin. Request cần backend đi qua Supabase Auth và Edge Functions để xác thực và điều phối. Supabase quản lý dữ liệu; FastAPI chạy Recommend và Trip Planner bằng Content-Based, Collaborative Filtering, K-Means và Simulated Annealing. DeepSeek phục vụ dịch/chat, Gemini phục vụ nhận diện ảnh, VBee và Android TTS phục vụ đọc văn bản.
>
> Data Freshness gồm crawler thủ công để nhập dữ liệu và checker tự động để kiểm tra lại nguồn. Cron được cấu hình lúc 02:15 giờ Việt Nam, mỗi lượt tối đa 50 row đến hạn từ OSM/Wikipedia. Nguồn thiếu lần đầu thành stale, lần hai mới tạo proposal possibly closed; thay đổi thông thường phải chờ admin duyệt. Chỉ scheduled event đã hết hạn được tự động expire. Admin xem proposal, report, stale data và run history nhưng chưa điều khiển Python crawler.

## 7. Nguồn đối chiếu trong repository

- Kiến trúc: [`README.md`](../README.md), [`cf_service/README.md`](../cf_service/README.md)
- Trip Planner: [`cf_service/services/trip_planner.py`](../cf_service/services/trip_planner.py)
- Recommend: [`cf_service/routes/recommend.py`](../cf_service/routes/recommend.py)
- Dịch/TTS: [`frontend/lib/features/translate/data`](../frontend/lib/features/translate/data), [`backend/supabase/functions/translate`](../backend/supabase/functions/translate)
- AI Recognition: [`ai-recognition-behavior.md`](ai-recognition-behavior.md), [`backend/supabase/functions/ai-search`](../backend/supabase/functions/ai-search)
- Data Freshness: [`data-freshness-operations.md`](data-freshness-operations.md), [`freshness_domain.ts`](../backend/supabase/functions/data-freshness-check/freshness_domain.ts)
- Migration/cron: [`20260803000100_data_freshness.sql`](../backend/supabase/migrations/20260803000100_data_freshness.sql)
