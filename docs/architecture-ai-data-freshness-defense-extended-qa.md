# Bộ câu hỏi phản biện mở rộng: Kiến trúc, AI và Data Freshness

> Tài liệu này bổ sung cho [`architecture-ai-data-freshness-defense-guide.md`](architecture-ai-data-freshness-defense-guide.md), không lặp lại các câu hỏi cơ bản đã có trong file đó. Nội dung được đối chiếu với code trên nhánh `final` ngày 13/08/2026.

## 1. Cách dùng tài liệu

Mỗi câu có hai mức trả lời:

- **Trả lời nhanh:** dùng trong khoảng 15–30 giây.
- **Nếu bị hỏi sâu:** bổ sung bằng chi tiết kỹ thuật hoặc giới hạn hiện tại.

Khi gặp một câu ngoài dự kiến, nên trả lời theo thứ tự:

1. Hệ thống **đang làm gì**.
2. Cơ chế nào trong code bảo đảm điều đó.
3. **Giới hạn hiện tại** là gì.
4. Nếu đưa vào production thì sẽ nâng cấp thế nào.

Không nên biến một hướng nâng cấp thành tính năng đã triển khai. Một câu trả lời thẳng thắn như “phiên bản hiện tại mới ở mức đồ án và đây là phần nhóm sẽ harden khi production” tốt hơn một khẳng định không đúng với code.

---

## 2. Kiến trúc, triển khai và bảo mật

### 1. Kiến trúc này là monolith, microservices hay serverless?

**Trả lời nhanh:** Đây là kiến trúc lai. Flutter là client; Supabase cung cấp Auth, PostgreSQL và Edge Functions theo hướng backend-as-a-service/serverless; còn Trip Planner và Recommendation chạy trong một FastAPI service riêng. Vì vậy không nên gọi toàn bộ hệ thống là microservices thuần túy.

**Nếu bị hỏi sâu:** Nhóm tách FastAPI vì phần recommendation, WALS và tối ưu lịch trình phù hợp với Python hơn Edge Function. Tuy nhiên FastAPI hiện vẫn chứa nhiều module nghiệp vụ trong cùng một service, nên cách gọi chính xác là **modular service kết hợp serverless backend**.

### 2. Vercel đang chạy toàn bộ hệ thống phải không?

**Trả lời nhanh:** Không. Vercel chỉ phân phối bản build tĩnh của Flutter Web, chủ yếu là trang admin. Auth, database và Edge Functions vẫn ở Supabase; Trip Planner/Recommendation ở FastAPI; app Android không chạy trên Vercel.

**Nếu bị hỏi sâu:** Khi Vercel lỗi, admin Web có thể không truy cập được, nhưng điều đó không đồng nghĩa PostgreSQL, Edge Functions hoặc app Android đều ngừng hoạt động. Đây là các deployment độc lập.

### 3. Vì sao user app và admin đều dùng Flutter? Có làm lộ chức năng admin không?

**Trả lời nhanh:** Flutter giúp dùng chung theme, model và hạ tầng client, nhưng user app và admin có entry point/build target riêng. Việc ẩn route ở giao diện chỉ phục vụ UX; quyền admin thật phải được kiểm tra ở backend.

**Nếu bị hỏi sâu:** `adminSignIn` kiểm tra role và các API nhạy cảm tiếp tục kiểm tra JWT/role. Kể cả người dùng tự sửa URL hoặc JavaScript của Flutter Web, họ vẫn không được phép thao tác nếu Edge Function, RPC và RLS thực thi đúng quyền.

### 4. Nếu người dùng biết URL trang admin rồi truy cập trực tiếp thì sao?

**Trả lời nhanh:** Router không phải lớp bảo mật. Backend mới là lớp quyết định: xác thực Supabase JWT, lấy user thật từ token và kiểm tra role `admin` trước thao tác nhạy cảm.

**Nếu bị hỏi sâu:** Route guard chỉ ngăn truy cập nhầm từ giao diện. Nguyên tắc của hệ thống là không tin `userId`, role hoặc giá trị đặc quyền do client tự gửi. API admin phải trả `401/403` dù giao diện đã bị bypass.

### 5. Supabase anon key nằm trong app có phải là lộ secret không?

**Trả lời nhanh:** Không. Anon key được thiết kế để xuất hiện ở client. Nó chỉ xác định project/role công khai; quyền dữ liệu vẫn do JWT, RLS và policy quyết định. `service_role` mới là secret và chỉ được đặt ở Edge Function/FastAPI.

**Nếu bị hỏi sâu:** Nếu RLS hoặc quyền RPC cấu hình sai thì anon key vẫn có thể bị lạm dụng. Vì vậy bảo mật không dựa vào việc giấu anon key mà dựa vào policy, revoke/grant và kiểm tra quyền phía server. Tuyệt đối không đưa `SUPABASE_SERVICE_ROLE_KEY`, DeepSeek, Gemini hay Stripe secret vào Flutter asset.

### 6. Tại sao app phải đi qua Edge Function trước khi tới FastAPI?

**Trả lời nhanh:** Edge Function là cổng xác thực và chuẩn hóa request. Nó lấy user từ JWT thay vì tin `id_user` do client gửi, sau đó mới chuyển nghiệp vụ tính toán sang FastAPI.

**Nếu bị hỏi sâu:** Đây là **luồng thiết kế** hiện tại. Tuy nhiên các endpoint Trip/Recommend thông thường của FastAPI chưa tự xác minh một chữ ký service-to-service. Nếu URL FastAPI được public khi production, cần đặt nó trong mạng riêng hoặc bổ sung HMAC/API gateway/mTLS và chỉ cho Edge Function gọi. Không nên nói rằng Edge Function tự động bảo vệ được một FastAPI URL đang mở công khai.

### 7. CORS đặt `*` có phải là một lỗ hổng xác thực không?

**Trả lời nhanh:** CORS và authentication là hai vấn đề khác nhau. CORS chỉ kiểm soát trình duyệt nào được phép gọi; nó không chứng minh người gọi là ai. JWT, role và RLS vẫn phải bảo vệ tài nguyên.

**Nếu bị hỏi sâu:** Shared Edge headers và FastAPI hiện cho phép origin rộng để tiện Android, local và demo. Với production Web nên thu hẹp danh sách origin, nhưng vẫn phải giữ xác thực server-side. Đối với Stripe callback origin, hệ thống đã dùng allowlist chính xác và không dùng `*`.

### 8. FastAPI có mở rộng được khi nhiều người lập lịch trình cùng lúc không?

**Trả lời nhanh:** Các request API không giữ session trong RAM nên có thể scale nhiều worker/instance, nhưng bước Simulated Annealing dùng CPU cần được giới hạn. Code hiện có semaphore theo từng worker và timeout hàng đợi để tránh làm nghẽn toàn service.

**Nếu bị hỏi sâu:** Mặc định chỉ cho một tác vụ SA chạy đồng thời trên mỗi worker qua `SA_MAX_CONCURRENCY_PER_WORKER`. Khi quá tải, hệ thống ưu tiên trả lỗi “planner busy, please retry” thay vì nhận vô hạn. Scale production cần benchmark rồi điều chỉnh số worker, CPU, queue và timeout; số lượng test không tự chứng minh năng lực tải production.

### 9. Cache của FastAPI có làm người dùng nhìn thấy dữ liệu của nhau không?

**Trả lời nhanh:** Cache dùng cho dữ liệu đọc dùng chung hoặc factor được khóa theo user/place; implementation trả deep copy để tránh request sửa chung một object trong RAM. Cache có TTL và được xóa chủ động sau CF retrain.

**Nếu bị hỏi sâu:** Đây là process-local cache, không phải Redis dùng chung toàn cluster. Mỗi instance có cache riêng, nên nó giúp giảm truy vấn nhưng không phải nguồn dữ liệu chuẩn. PostgreSQL vẫn là source of truth; nếu scale nhiều instance thì cần cơ chế invalidation phân tán nếu yêu cầu nhất quán tức thời.

### 10. Hệ thống đang giám sát lỗi bằng cách nào?

**Trả lời nhanh:** Các luồng quan trọng có log và lịch sử riêng: AI Chat ghi request ID, latency theo stage và token; CF retrain ghi `cf_retrain_log`; Data Freshness ghi từng run, số selected/checked/failed và proposal; payment lưu trạng thái trong `payment_attempt` và có log xử lý webhook.

**Nếu bị hỏi sâu:** Đây là observability theo log/database, chưa phải một hệ thống APM và cảnh báo tập trung hoàn chỉnh. Khi production nên đẩy log có cấu trúc vào Sentry/Log Drain, đặt alert cho error rate, P95, cron miss, backlog và webhook failure.

### 11. Nếu Supabase hoặc một nhà cung cấp ngoài bị lỗi thì hệ thống có single point of failure không?

**Trả lời nhanh:** Supabase là hạ tầng trung tâm nên sự cố Supabase ảnh hưởng nhiều tính năng online. Các provider khác được cô lập theo chức năng: Gemini lỗi chủ yếu ảnh hưởng Recognition, DeepSeek ảnh hưởng dịch online/chat, Goong có thể fallback Haversine trong một số luồng.

**Nếu bị hỏi sâu:** Local cache và ML Kit giúp một số trải nghiệm vẫn dùng được, nhưng app không phải offline-first hoàn toàn. Production cần health check, timeout, circuit breaker, alert và kế hoạch khôi phục dữ liệu. Không nên khẳng định repo hiện tại đã có multi-region failover hoặc disaster recovery nếu chưa kiểm chứng cấu hình hạ tầng Supabase.

---

## 3. AI, thuật toán và quyền riêng tư

### 12. Tất cả Recommend và Trip Planner có được xem là Generative AI không?

**Trả lời nhanh:** Không. Gemini Recognition, DeepSeek Chat và dịch online là Generative AI. Recommendation kết hợp content-based và collaborative filtering; Trip Planner dùng heuristic, ràng buộc lịch và Simulated Annealing. Đó là ML/optimization, không phải LLM sinh lịch trình tự do.

**Nếu bị hỏi sâu:** Cách phân loại này quan trọng vì mỗi phần được đánh giá khác nhau: LLM cần kiểm tra factuality/hallucination; recommender cần relevance/cold-start; optimizer cần chi phí tuyến đường, vi phạm ràng buộc và thời gian chạy.

### 13. Nhóm đánh giá chất lượng AI bằng gì, ngoài việc tự dùng thử?

**Trả lời nhanh:** Repo có bộ evaluation gọi đúng các Edge Function thật. AI Search đo classification/name/database-match/end-to-end accuracy và calibration; Chat đo action correctness và fact coverage; Translation dùng token F1, chrF và bảo toàn số; tất cả đều ghi success rate và latency P50/P95/P99.

**Nếu bị hỏi sâu:** TTS tự động chỉ xác minh audio có tạo và truy cập được; naturalness, intelligibility và factuality vẫn cần người chấm. Cách nói an toàn là framework và metric **đã có**, còn một kết luận chất lượng có ý nghĩa phải đi kèm dataset đại diện, kết quả run cụ thể và tối thiểu vài người chấm độc lập.

### 14. Confidence của Gemini có phải xác suất chính xác thật không?

**Trả lời nhanh:** Không nên coi đó là xác suất tuyệt đối. Nó là confidence do model trả về, được server clamp về 0–1 và dùng ngưỡng để chuyển kết quả quá yếu thành `unclear`.

**Nếu bị hỏi sâu:** Code dùng ngưỡng dưới `0.55` cho kết quả supported và chia band từ `0.8`. Muốn biết confidence đáng tin tới đâu phải đo calibration, ví dụ ECE, trên bộ ảnh có ground truth. Vì vậy UI nên diễn đạt là mức tin cậy, không phải bảo đảm đúng.

### 15. Làm sao chống prompt injection hoặc model trả một route độc hại?

**Trả lời nhanh:** AI Chat yêu cầu JSON, chỉ chấp nhận action nằm trong allowlist và loại bỏ URL/route/deep-link do model tự tạo. App ánh xạ action key đã biết sang route nội bộ; model không được tự chọn URL tùy ý.

**Nếu bị hỏi sâu:** Payload còn được giới hạn độ sâu, độ dài, số phần tử và loại các key nguy hiểm. Tuy vậy prompt injection không thể giải quyết chỉ bằng prompt; nguyên tắc chính là **không cấp quyền trực tiếp cho model** và luôn validate output trước khi thực thi. Hiện các action chủ yếu là gợi ý điều hướng, không tự mua hàng hay tự sửa database.

### 16. Ảnh và nội dung người dùng được gửi tới đâu?

**Trả lời nhanh:** Recognition đọc ảnh trên thiết bị, mã hóa Base64 rồi gửi qua Supabase Edge Function tới Gemini. Dịch/Chat gửi text cần xử lý tới DeepSeek; TTS gửi text tới VBee; dữ liệu tuyến đường cần thiết được gửi tới dịch vụ bản đồ.

**Nếu bị hỏi sâu:** Edge Function Recognition không ghi ảnh gốc vào database. App chỉ lưu thumbnail và kết quả Recognition trong SharedPreferences trên thiết bị, tối đa 30 entry và tách key theo user. Tuy nhiên dữ liệu vẫn đi qua provider ngoài, nên production cần privacy notice/consent, giới hạn kích thước, retention policy và không cho người dùng gửi giấy tờ hoặc dữ liệu nhạy cảm.

### 17. AI Recognition có nhận diện người, CCCD hoặc thẻ ngân hàng không?

**Trả lời nhanh:** Không. Prompt yêu cầu không định danh người, không chép giấy tờ nhạy cảm và trả `unsupported` cho selfie, giấy tờ định danh hoặc thẻ thanh toán. Safety block cũng được chuyển thành kết quả an toàn thay vì cố phân tích.

**Nếu bị hỏi sâu:** Đây là guardrail ở prompt, schema và normalization, không phải một hệ thống DLP độc lập. Với production nên bổ sung kiểm tra file/MIME/kích thước, moderation trước provider và quota theo user.

### 18. AI Recognition có thật sự xác thực từng người dùng và giới hạn quota chưa?

**Trả lời nhanh:** Chưa đầy đủ theo từng user. Client có gửi authorization header, nhưng `ai-search` hiện không gọi `getUser()` để lấy user thật và chưa có quota theo user trong handler.

**Nếu bị hỏi sâu:** Đây là điểm cần harden: dùng session JWT thay vì chỉ anon credential, xác minh user trong Edge Function, đặt rate limit/ngày, giới hạn dung lượng ảnh và log usage. Không nên trả lời rằng Recognition đã có cơ chế quota giống Translation hoặc AI Chat.

### 19. API key AI được bảo vệ như thế nào?

**Trả lời nhanh:** Gemini, DeepSeek và VBee key nằm trong Supabase secrets; Flutter chỉ gọi Edge Function. FastAPI dùng server environment cho secret riêng. Key không được ghi vào source, asset, log hay ảnh demo.

**Nếu bị hỏi sâu:** Tách secret khỏi client ngăn việc decompile APK lấy provider key. Khi nghi lộ key phải rotate và kiểm tra log usage. Anon key của Supabase là ngoại lệ vì nó được thiết kế công khai và không thay vai trò của RLS.

### 20. Nhóm kiểm soát chi phí AI như thế nào?

**Trả lời nhanh:** Dịch có rate limit ngày/phút; AI Chat yêu cầu Premium, giới hạn request và ghi input/output token cùng estimated cost; request provider có timeout và `max_tokens`. Recognition có fallback key/model nhưng còn thiếu quota theo user.

**Nếu bị hỏi sâu:** Fallback tăng độ sẵn sàng nhưng cũng có thể làm tăng chi phí, nên production cần budget theo feature/user, dashboard usage, alert và hard cap ở server. Không nên dựa vào việc giấu nút ở UI để kiểm soát chi phí.

### 21. Nếu đổi Gemini/DeepSeek sang nhà cung cấp khác có phải viết lại toàn bộ app không?

**Trả lời nhanh:** Không cần viết lại UI nếu giữ nguyên contract Edge Function. Phần phải thay chủ yếu là adapter gọi provider, tên model, cách gửi ảnh/text và prompt/schema tương thích.

**Nếu bị hỏi sâu:** Không thể bảo đảm “chỉ đổi URL”. Mỗi provider khác nhau về multimodal payload, structured output, safety, token, error và rate limit. Cần chạy lại evaluation vì cùng prompt chưa chắc cho chất lượng tương đương. Contract chuẩn hóa ở server giúp giới hạn phạm vi thay đổi.

### 22. Khi model được nhà cung cấp cập nhật, kết quả có tái lập được không?

**Trả lời nhanh:** Kết quả LLM không hoàn toàn deterministic. Hệ thống có lưu model trong AI Chat metrics và Recognition trả provider/model để truy vết, nhưng vẫn phải quản lý version và evaluation khi đổi model.

**Nếu bị hỏi sâu:** Production nên pin model/version nếu provider hỗ trợ, lưu prompt version, schema version, thời điểm và metric. Không lưu key hoặc dữ liệu nhạy cảm trong log. Với thuật toán recommender/optimizer, seed, dữ liệu đầu vào và model snapshot cũng cần được lưu nếu muốn tái lập hoàn toàn.

### 23. Recommendation có thể tạo thiên lệch hoặc “filter bubble” không?

**Trả lời nhanh:** Có khả năng. CF học từ hành vi nên địa điểm phổ biến hoặc nhóm người dùng hoạt động nhiều có thể được ưu tiên. Hybrid recommendation giảm vấn đề bằng content preference và fallback cho cold-start, nhưng không loại bỏ bias hoàn toàn.

**Nếu bị hỏi sâu:** Hướng nâng cấp là thêm diversity/novelty, giới hạn lặp tỉnh/category, theo dõi coverage của địa điểm ít phổ biến và cho người dùng điều chỉnh sở thích. Không nên chỉ tối ưu click vì có thể làm gợi ý ngày càng hẹp.

### 24. Trip Planner có bảo đảm tìm tuyến tối ưu toàn cục không?

**Trả lời nhanh:** Không. Simulated Annealing là heuristic, tìm nghiệm tốt trong thời gian hữu hạn chứ không chứng minh global optimum. Mục tiêu thực tế là lịch hợp lệ, khoảng cách hợp lý và phản hồi đủ nhanh cho app.

**Nếu bị hỏi sâu:** Chất lượng phụ thuộc dữ liệu tọa độ, thời gian mở cửa, duration, ma trận di chuyển và tham số SA. Hệ thống dùng greedy/repair để tạo lịch khả thi rồi SA cải thiện thứ tự. Câu đúng là “best found under the configured budget”, không phải “tuyến ngắn nhất tuyệt đối”.

### 25. Collaborative Filtering được cập nhật tự động như thế nào?

**Trả lời nhanh:** `pg_cron` gọi endpoint retrain hằng ngày bằng shared secret. Job đọc favorite, review và các event như view/share; huấn luyện WALS; ghi top score và user/place factors; cuối cùng xóa factor cache để request sau đọc model mới.

**Nếu bị hỏi sâu:** Lịch mặc định trong migration là `19:00 UTC`, tương ứng `02:00` ngày kế tiếp ở Việt Nam, và admin có cấu hình lịch. Cron trigger không bảo đảm job thành công; phải kiểm tra `cf_retrain_log`, Vault secret, endpoint FastAPI và số row được ghi.

---

## 4. Thanh toán, voucher và tiền tệ

### 26. App có hỗ trợ Google Pay hoặc thanh toán VISA trực tiếp không?

**Trả lời nhanh:** Không. Giao diện hiện chỉ mô tả đúng luồng **Stripe Sandbox Checkout**. Các lựa chọn Google Pay/VISA giả trước đây đã được bỏ vì app không có tích hợp tương ứng.

**Nếu bị hỏi sâu:** Stripe Checkout có thể hiển thị phương thức nào còn phụ thuộc cấu hình Stripe, quốc gia và session; nhóm không nên tự tuyên bố Google Pay nếu chưa cấu hình và kiểm thử end-to-end.

### 27. Thanh toán hiện tại có dùng được với tiền thật chưa?

**Trả lời nhanh:** Chưa. Hệ thống cố ý chạy Stripe **test mode** để demo, không thu tiền thật. UI cũng hiển thị “TEST MODE · No real charge”.

**Nếu bị hỏi sâu:** Đưa lên production cần live Stripe account/key, chính sách giá/thuế/hoàn tiền, webhook live, domain/deep link chính thức, kiểm thử pháp lý và vận hành. Không được chỉ đổi `sk_test` thành `sk_live` rồi gọi là hoàn tất.

### 28. Nếu client sửa giá hoặc voucher trong request thì sao?

**Trả lời nhanh:** Server không tin giá từ client. Client chỉ gửi plan code và voucher tùy chọn; server đọc giá plan, quyền sở hữu voucher và điều kiện giảm từ database rồi tạo `payment_attempt` snapshot.

**Nếu bị hỏi sâu:** Khi Stripe báo paid, server còn đối chiếu session ID, user, amount và currency với attempt. Premium chỉ được cấp sau bước verify/finalize ở server, không dựa vào màn hình “thanh toán thành công” phía Flutter.

### 29. Callback và webhook tới hai lần có cấp Premium hoặc điểm loyalty hai lần không?

**Trả lời nhanh:** Không theo thiết kế hiện tại. Checkout dùng idempotency key; `finalize_verified_payment` chạy atomically; payment/session có unique identity; loyalty được award theo payment. Replay trả kết quả đã có.

**Nếu bị hỏi sâu:** Webhook Stripe có chữ ký là nguồn xác nhận authoritative; callback app chỉ hỗ trợ UX/recovery. Việc gửi lại webhook là bình thường và hệ thống phải idempotent thay vì giả định mỗi event chỉ tới một lần.

### 30. Vì sao lỗi voucher loyalty trước đây có thể làm `INTERNAL_ERROR`?

**Trả lời nhanh:** Luồng voucher chạm nhiều dữ liệu cùng lúc: wallet ownership, trạng thái voucher, plan áp dụng, snapshot discount và finalization. Nếu client và server tính khác nhau hoặc RPC không xử lý transaction/idempotency đúng thì dễ rơi vào lỗi chung.

**Nếu bị hỏi sâu:** Luồng hiện tại chuyển việc định giá về server, khóa voucher theo đúng user, snapshot số tiền vào `payment_attempt` và chỉ consume/finalize atomically khi Stripe đã verify. Khi trình bày nên nói cơ chế sau khi sửa, không chỉ nói “đổi UI”.

### 31. Chọn VND/EUR trong Currency có làm Stripe thu bằng VND/EUR không?

**Trả lời nhanh:** Không. Currency preference dùng để hiển thị/quy đổi các giá trị trong app. Subscription Checkout hiện cố định USD từ bảng plan, `payment_attempt` tới Stripe và UI thanh toán.

**Nếu bị hỏi sâu:** Đây là phân biệt giữa **display currency** và **settlement currency**. Muốn thanh toán đa tiền tệ phải thiết kế price/rounding/discount theo từng currency, tạo Stripe session đúng currency và đối chiếu webhook tương ứng. Không được chỉ đổi ký hiệu `$` ở client.

### 32. Nếu người dùng trả tiền xong nhưng đóng app trước callback thì sao?

**Trả lời nhanh:** Signed webhook vẫn có thể hoàn tất subscription độc lập với app. Khi người dùng mở lại, app đọc trạng thái Premium từ server; callback có thể retry với cùng Session ID mà không tạo bản ghi trùng.

**Nếu bị hỏi sâu:** Nếu webhook tạm lỗi, Stripe test dashboard có thể resend. Không nên cấp Premium tạm thời chỉ dựa vào redirect vì redirect có thể bị giả mạo hoặc bị mất.

---

## 5. Data Freshness, tính nhất quán và vận hành

### 33. “Stale” có nghĩa dữ liệu đã sai không?

**Trả lời nhanh:** Không. `stale/due` nghĩa bản ghi đã tới hạn cần kiểm tra theo cadence. Chỉ sau khi checker đọc nguồn ngoài và phát hiện khác biệt hợp lệ thì mới có proposal chờ admin duyệt.

**Nếu bị hỏi sâu:** Vì vậy số Due/Stale lớn không đồng nghĩa database có từng đó lỗi. Nó phản ánh backlog kiểm tra, không phải số row chắc chắn sai.

### 34. Tại sao không tự động ghi mọi thay đổi từ nguồn ngoài vào database?

**Trả lời nhanh:** Nguồn ngoài có thể sai, tạm thiếu hoặc thay đổi format. Hệ thống ưu tiên human-in-the-loop: checker tạo proposal, admin xem diff rồi xác nhận; chỉ một số trạng thái theo rule an toàn mới có thể xử lý tự động.

**Nếu bị hỏi sâu:** Việc tách detect và apply làm giảm blast radius. Production có thể auto-apply các trường rủi ro thấp khi confidence cao, nhưng name/address/status quan trọng vẫn cần rule, nhiều nguồn hoặc phê duyệt.

### 35. Nếu hai lượt cron hoặc cron và admin chạy cùng lúc thì có kiểm tra trùng một row không?

**Trả lời nhanh:** Batch claim dùng `FOR UPDATE SKIP LOCKED`, nên worker bỏ qua row đang bị worker khác khóa thay vì cùng claim. Điều này hỗ trợ chạy đồng thời an toàn hơn.

**Nếu bị hỏi sâu:** Khóa chỉ giải quyết tranh chấp lúc chọn row. Apply proposal còn cần optimistic concurrency để bảo đảm dữ liệu gốc chưa bị người khác sửa kể từ lúc tạo diff.

### 36. Với tối đa 50 row/ngày, backlog lớn bao giờ mới hết?

**Trả lời nhanh:** Thời gian tối thiểu xấp xỉ `ceil(số row đang due / 50)` ngày nếu không có row mới tới hạn. Vì vậy 50 là giới hạn demo/an toàn cho quota nguồn, không phải cấu hình đủ cho mọi quy mô production.

**Nếu bị hỏi sâu:** Muốn đạt SLA phải tính throughput theo tổng row và cadence, rồi tăng batch, chạy nhiều khung giờ hoặc chia nguồn/queue. Ví dụ 7.500 row due với đúng một batch 50/ngày cần khoảng 150 ngày trong điều kiện lý tưởng. Khi bảo vệ nên thừa nhận đây là tuning vận hành, không nói 50 luôn tối ưu.

### 37. Có cron rồi thì làm sao biết nó thực sự chạy thành công mỗi ngày?

**Trả lời nhanh:** Không chỉ nhìn cấu hình cron. Phải kiểm tra run history: trigger, thời gian bắt đầu/kết thúc, selected, checked, failed, proposal và status `completed/partial_failure/failed`.

**Nếu bị hỏi sâu:** Cron chỉ bảo đảm lịch kích hoạt khi database, Vault secret và endpoint hoạt động. Cần alert khi không có run đúng hạn, failed tăng hoặc backlog không giảm. Admin “Run history” là evidence trực quan, còn `cron.job`/run detail là evidence vận hành.

### 38. Nếu OSM/Wikipedia trả 429 hoặc timeout thì hệ thống có đánh dấu địa điểm là đóng không?

**Trả lời nhanh:** Không nên. Lỗi kỹ thuật được tính là failed/partial failure và retry sau; không tăng missing count như một bằng chứng địa điểm biến mất.

**Nếu bị hỏi sâu:** Luồng đóng địa điểm cần nhiều lần missing hợp lệ; timeout, 429, 5xx hoặc parse error không được hiểu là dữ liệu nguồn xác nhận “đã đóng”. Đây là lý do checker tách `failed` khỏi proposal.

### 39. Nếu nguồn ngoài sai nhưng admin bấm xác nhận thì sao?

**Trả lời nhanh:** Human review giảm rủi ro nhưng không bảo đảm tuyệt đối. Admin cần xem source, field diff và ngữ cảnh trước khi apply; thay đổi vẫn có audit trail để truy vết.

**Nếu bị hỏi sâu:** Hướng nâng cấp là đối chiếu nhiều nguồn, confidence/provenance, quyền phê duyệt hai bước cho field nhạy cảm và chức năng rollback. Hiện hệ thống chưa nên được mô tả là xác minh sự thật từ nhiều nguồn ở mức production-grade.

### 40. Nếu row đã bị sửa sau khi proposal được tạo thì apply có ghi đè không?

**Trả lời nhanh:** Apply phải kiểm tra optimistic concurrency: dữ liệu hiện tại/hash phải còn khớp snapshot khi proposal được tạo. Nếu không khớp thì proposal cần được kiểm tra lại thay vì ghi đè mù.

**Nếu bị hỏi sâu:** Đây là cách tránh lost update giữa admin CRUD và Data Freshness. Nút “Check again” đưa row vào lượt kiểm tra mới; nó không tự ép proposal cũ lên dữ liệu mới.

### 41. Có rollback được một thay đổi Data Freshness không?

**Trả lời nhanh:** Hệ thống lưu proposal, diff, quyết định admin và run history để audit. Tuy nhiên không nên khẳng định có nút rollback hoàn chỉnh nếu giao diện hiện tại chưa cung cấp.

**Nếu bị hỏi sâu:** Khi production nên lưu before/after snapshot bất biến và cung cấp compensating action có phân quyền. Trước mắt có thể phục hồi bằng một thay đổi admin được kiểm soát dựa trên audit evidence, không xóa lịch sử.

### 42. Muốn thêm một bảng hoặc một trường mới vào checker có tự động được không?

**Trả lời nhanh:** Không tự động. Phải thêm metadata/cadence, adapter nguồn, allowlist field, normalize/hash/diff rule, cách apply và test. Việc cố ý dùng allowlist ngăn nguồn ngoài sửa một field chưa được review.

**Nếu bị hỏi sâu:** Mỗi field có semantics khác nhau. Ví dụ `address` có thể chuẩn hóa text; `status` cần chính sách missing; ảnh cần kiểm tra URL/license; nội dung dịch cần nguồn hoặc quy trình riêng. Không nên dùng một comparator chung cho mọi dữ liệu.

### 43. Python crawler thủ công còn có ý nghĩa gì nếu checker đã trực tiếp lấy nguồn?

**Trả lời nhanh:** Crawler dùng cho import/backfill/demo và tạo artifact để review; checker là luồng vận hành định kỳ cho các row đã đăng ký freshness. Hai luồng không dùng chung cache và không thay thế nhau.

**Nếu bị hỏi sâu:** Crawler phù hợp khi cần thu thập tập dữ liệu mới với quy trình làm sạch/duyệt hàng loạt. Checker phù hợp phát hiện thay đổi nhỏ theo thời gian. Việc nhập crawler trực tiếp vào production vẫn phải qua validation và review.

### 44. Nếu một bản ghi không có `external_id` hoặc nguồn hợp lệ thì checker làm gì?

**Trả lời nhanh:** Checker không nên đoán nguồn. Row thiếu định danh/URL cần được báo failed hoặc cần bổ sung metadata trước khi đối chiếu.

**Nếu bị hỏi sâu:** Khả năng kiểm tra được phụ thuộc provenance. Một row chỉ có tên địa điểm không đủ để khẳng định OSM/Wikipedia record nào là đúng, vì có thể trùng tên. Backfill `source_system`, `external_id`, source URL và cadence là bước onboarding bắt buộc.

---

## 6. Kiểm thử, giới hạn và cách trả lời khi bị truy vấn

### 45. Nhóm có bằng chứng nào ngoài việc demo chạy được?

**Trả lời nhanh:** Nhánh `final` đã có test ở nhiều lớp: Flutter, FastAPI, crawler, AI evaluation và Edge Function; có build Web user/admin và Android debug. Các số baseline gần nhất được ghi trong `final-cleanup-report.md`.

**Nếu bị hỏi sâu:** Test chứng minh các case đã định nghĩa không regression, không chứng minh hệ thống không còn bug hoặc chịu được tải production. Cần phân biệt unit/integration/build check với load test, security audit và user acceptance test.

### 46. Hệ thống hiện đã production-grade chưa?

**Trả lời nhanh:** Chưa hoàn toàn; đây là phiên bản đồ án/demo có luồng end-to-end và nhiều guard quan trọng. Production vẫn cần harden auth service-to-service, quota Recognition, CORS, centralized monitoring, backup/DR, multi-source freshness và Stripe live compliance.

**Nếu bị hỏi sâu:** Trả lời này không làm yếu đồ án. Nó cho thấy nhóm phân biệt được “feature hoạt động” với “vận hành production”, đồng thời chỉ ra đúng backlog kỹ thuật dựa trên code thay vì nói chung chung.

### 47. Repo còn technical debt nào cần nói trung thực?

**Trả lời nhanh:** Báo cáo cleanup ghi nhận hai Flutter analyzer `info`, dependency audit còn advisory, một số Deno type/runtime baseline chưa được sửa. Các lỗi này không tăng sau cleanup nhưng vẫn là debt cần xử lý.

**Nếu bị hỏi sâu:** Không cần tự nêu toàn bộ debt nếu không được hỏi. Nếu hội đồng hỏi về chất lượng, hãy nói rõ phạm vi verification đã pass và phần baseline nào chưa pass; tránh câu “toàn bộ test đều xanh” khi báo cáo không nói vậy.

### 48. Nếu lúc demo một provider AI hoặc nguồn crawl bị lỗi thì nên giải thích thế nào?

**Trả lời nhanh:** Trình bày đúng failure mode: provider ngoài đang timeout/quota/rate-limit, còn hệ thống đã trả lỗi có kiểm soát và không ghi dữ liệu sai. Sau đó dùng artifact hoặc run history đã chuẩn bị để giải thích luồng.

**Nếu bị hỏi sâu:** Không nên giả vờ response cache là kết quả live. Với Data Freshness, `partial_failure` vẫn chứng minh batch đã tiếp tục xử lý các row khác. Với AI, nên có ảnh/text mẫu nhỏ, mạng ổn định và kết quả evaluation/run gần nhất làm evidence dự phòng.

---

## 7. Mười câu cần nhớ nhất

1. **Vercel chỉ host Flutter Web**, không host Supabase hoặc FastAPI.
2. **Route admin không phải bảo mật**; JWT, role và RLS mới là lớp quyết định.
3. **Anon key được phép public**, service-role/provider keys thì không.
4. Luồng intended đi Edge → FastAPI; **FastAPI public cần thêm service-to-service auth**.
5. Confidence của LLM là tín hiệu, **không phải xác suất đúng tuyệt đối**.
6. Recognition gửi ảnh tới Gemini và hiện **chưa có quota/xác thực theo từng user đầy đủ**.
7. Recommend/Trip Planner **không phải Generative AI**; SA không bảo đảm global optimum.
8. Stripe hiện là **Sandbox USD**, Currency preference không biến nó thành thanh toán đa tiền tệ.
9. **Stale không đồng nghĩa sai**; proposal không mặc định tự ghi vào dữ liệu chính.
10. Batch 50 là giới hạn an toàn/demo; throughput production phải tính theo backlog và SLA.

---

## 8. Các câu tuyệt đối không nên nói

| Không nên nói | Nên nói |
|---|---|
| “Vercel đang chạy backend của hệ thống.” | “Vercel phân phối Flutter Web; backend nằm ở Supabase và FastAPI.” |
| “Có route guard nên user không thể hack admin.” | “Route guard là UX; backend role/RLS mới bảo vệ dữ liệu.” |
| “CORS `*` là an toàn vì đã có anon key.” | “CORS không phải authentication; production Web nên giới hạn origin và vẫn kiểm tra JWT/role.” |
| “AI confidence 90% nghĩa chắc chắn đúng 90%.” | “Đó là confidence của model; cần calibration trên ground truth.” |
| “Ảnh chỉ xử lý trong máy.” | “Ảnh Recognition được gửi qua Edge Function tới Gemini; thumbnail lịch sử lưu cục bộ.” |
| “AI Recognition đã giới hạn quota theo từng user.” | “Recognition hiện còn thiếu quota/user verification trong handler và cần harden.” |
| “Trip Planner luôn tìm tuyến tối ưu nhất.” | “SA tìm nghiệm tốt trong budget, không chứng minh global optimum.” |
| “App hỗ trợ Google Pay và VISA.” | “Luồng hiện tại là Stripe Sandbox Checkout; không quảng bá phương thức chưa tích hợp.” |
| “Chọn VND thì Stripe sẽ thu VND.” | “Currency là display preference; Checkout hiện settlement bằng USD.” |
| “7.500 stale nghĩa có 7.500 địa điểm sai.” | “Đó là số row đến hạn kiểm tra, không phải số lỗi đã xác nhận.” |
| “Có cron là chắc chắn ngày nào cũng thành công.” | “Cron tạo trigger; run history mới chứng minh kết quả.” |
| “Toàn bộ hệ thống đã production-ready.” | “Luồng demo đã hoàn chỉnh; các điểm hardening production đã được xác định.” |

---

## 9. Nguồn đối chiếu trong repository

- Kiến trúc và entry point: [`README.md`](../README.md), [`frontend/lib/main_admin.dart`](../frontend/lib/main_admin.dart), [`cf_service/main.py`](../cf_service/main.py)
- Auth và role: [`frontend/lib/core/auth/auth_repository.dart`](../frontend/lib/core/auth/auth_repository.dart), [`backend/supabase/functions/auth/auth_guard.ts`](../backend/supabase/functions/auth/auth_guard.ts)
- Edge → FastAPI: [`backend/supabase/functions/trip-planner/trip_handler.ts`](../backend/supabase/functions/trip-planner/trip_handler.ts), [`backend/supabase/functions/recommend/recommend_handler.ts`](../backend/supabase/functions/recommend/recommend_handler.ts)
- AI Recognition: [`backend/supabase/functions/ai-search/index.ts`](../backend/supabase/functions/ai-search/index.ts), [`recognition_contract.ts`](../backend/supabase/functions/ai-search/recognition_contract.ts), [`recognition_prompt.ts`](../backend/supabase/functions/ai-search/recognition_prompt.ts)
- Recognition history: [`frontend/lib/features/ai_search/data/ai_recognition_history_repository.dart`](../frontend/lib/features/ai_search/data/ai_recognition_history_repository.dart)
- AI Chat validation, action allowlist và metrics: [`backend/supabase/functions/ai-chat/ai_chat_domain.ts`](../backend/supabase/functions/ai-chat/ai_chat_domain.ts), [`ai_chat_handler.ts`](../backend/supabase/functions/ai-chat/ai_chat_handler.ts), [`index.ts`](../backend/supabase/functions/ai-chat/index.ts)
- AI evaluation: [`evaluation/ai_quality/README.md`](../evaluation/ai_quality/README.md)
- Trip/Recommend và concurrency: [`cf_service/services/trip_planner.py`](../cf_service/services/trip_planner.py), [`cf_service/services/sa_concurrency.py`](../cf_service/services/sa_concurrency.py), [`cf_service/services/ttl_cache.py`](../cf_service/services/ttl_cache.py)
- CF retrain: [`cf_service/jobs/cf_retrain.py`](../cf_service/jobs/cf_retrain.py), [`20260811000300_move_cf_retrain_to_pg_cron.sql`](../backend/supabase/migrations/20260811000300_move_cf_retrain_to_pg_cron.sql)
- Stripe Sandbox: [`backend/supabase/functions/subscription-payment/README.md`](../backend/supabase/functions/subscription-payment/README.md), [`backend/supabase/functions/_shared/stripe_payment_store.ts`](../backend/supabase/functions/_shared/stripe_payment_store.ts)
- Currency: [`frontend/lib/features/profile/data/currency_service.dart`](../frontend/lib/features/profile/data/currency_service.dart), [`backend/supabase/functions/currency-rates/index.ts`](../backend/supabase/functions/currency-rates/index.ts)
- Data Freshness: [`docs/data-freshness-operations.md`](data-freshness-operations.md), [`backend/supabase/functions/data-freshness-check`](../backend/supabase/functions/data-freshness-check), [`20260803000100_data_freshness.sql`](../backend/supabase/migrations/20260803000100_data_freshness.sql)
- Baseline verification và technical debt: [`docs/final-cleanup-report.md`](final-cleanup-report.md)
