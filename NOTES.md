# Ghi chú phân tích · Day 04

Họ và tên: Bùi Đức Tuấn
MSSV: 2A202603902

## 1. Case được chọn

- ID case: H1 và H6 là case chính. H2 là case hồi quy tôi dùng để kiểm tra bản sửa.
- Điểm ở lần chạy 1: H1 60, H6 60, H2 100.

Tôi chọn H1 và H6 vì hai case có cùng một lỗi gốc. Incident `h02n9572mx9y` không tự chứa nguyên nhân mà trỏ sang incident gộp `xr3myzj7ms51`, nhưng agent không đọc tiếp incident đó. Tôi đưa thêm H2 vì nó kéo ngược với H1/H6: H1/H6 cần agent đi qua nhiều bước, còn H2 cần agent gọi `get_status` song song ngay từ đầu. Một bản sửa chỉ ép agent đi từng bước sẽ làm hỏng H2, và điều này đã xảy ra thật ở bản sửa đầu (mục 3). Tôi không chọn lỗi thiếu link hay lỗi H7 bịa trạng thái làm case chính vì chúng không chung nguyên nhân với nhóm này. H7 được nêu ở mục 4.

Câu trả lời của agent ở H1 lần 1:

- Đúng: tách được hai sự cố riêng (GitHub Actions ngày 04/11, Cloudflare Workers Builds ngày 05/11), cả hai đều resolved, và không suy ra nguyên nhân chung.
- Thiếu: agent không mở incident liên kết `xr3myzj7ms51`, nên không nêu được nguyên nhân và ảnh hưởng thật của sự cố GitHub. Lỗi nằm ở agent, vì link có sẵn ngay trong bản cập nhật đầu tiên của incident.
- Dẫn nguồn sai: agent ghi `https://www.githubstatus.com/incidents/xr3myzj7ms51`. Link này được chép từ nội dung incident, agent chưa hề mở nó (URL khi đọc bằng tool là `https://stspg.io/pjgptg3cfc6n`). Agent cũng không ghi URL nguồn của chính `h02n9572mx9y` (`https://stspg.io/7pwv28hvkmz8`), chỉ ghi chung "dữ liệu từ GitHub status".

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json`:

```text
step 1  model  -> gọi 2 tool cùng lượt
step 1  tool   get_incident_details github h02n9572mx9y
step 1  tool   get_incident_details cloudflare yaa0witbl2me
step 2  model  -> trả lời (finish=stop), không đọc incident liên kết xr3myzj7ms51
mechanical: violations = ['Cần tool calls ở ít nhất 2 model steps khác nhau.'], score_cap = 60
```

Cả hai tool đều nằm ở step 1, và mechanical ghi `tool_steps: 1`. Kết quả tool của `h02n9572mx9y` có dòng "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51", nhưng ở step 2 model trả lời luôn. H6 lần 1 có đúng mẫu này: `get_status` và `h02n9572mx9y` ở step 1, trả lời ở step 2, cũng bị chặn ở 60.

Nguyên nhân nằm ở prompt:

- Vòng lặp không chặn gì. Cứ có tool result là code gọi model tiếp.
- Dữ liệu không thiếu. Link nằm ngay trong body, và body không bị cắt (`truncated: false`).
- Model tự chọn dừng, vì prompt cũ không yêu cầu rà kết quả để theo link.

Ở H1 lần 1, judge cho đủ 100 điểm (40/25/20/15), còn bộ kiểm tra bằng code chặn ở 60. Judge chấm nội dung câu trả lời có nghe hợp lý hay không, còn bộ kiểm tra bằng code chấm quy trình agent thực sự đã làm. Một câu trả lời trôi chảy có thể che việc agent bỏ qua bước điều tra, nên chỉ nhìn điểm judge là không đủ.

## 3. Thay đổi đã thực hiện

Sửa gì, ở đâu (`agent.py`):

1. `SYSTEM_PROMPT`, dòng 11 đến 14:
   - Các tra cứu độc lập (ví dụ `get_status` và `get_incident_details` của ID đã biết) thì gọi cùng lượt.
   - Sau mỗi lượt, rà lại kết quả. Nếu incident có liên kết sang incident khác (`.../incidents/<id>`) thì gọi `get_incident_details` cho incident đó trước khi trả lời.
   - Câu hỏi về tình trạng hiện tại, outage hay snapshot thì luôn gọi `get_status` trước khi kết luận.
   - Câu trả lời cuối phải kèm URL của mọi incident đã đọc bằng `get_incident_details`.
2. Evidence gate trong code (`seen_urls` và sự kiện `validation_missing_source`): lưu URL của các incident đã đọc thành công. Khi model trả lời xong (`finish=stop`), nếu câu trả lời thiếu URL nào thì ghi `validation_missing_source` vào trace và yêu cầu model viết lại có dẫn URL.

Tôi không thêm cơ chế retry hay logic xử lý xung đột trong code. Phần "song song hay tuần tự" được giải quyết bằng prompt.

Vì sao chọn thay đổi này:

- Tôi chọn sửa prompt vì lỗi không đến từ tool hỏng. Model đã nhận đủ dữ liệu nhưng tự dừng sớm.
- "Retry 2 lần" chỉ có ích khi tool lỗi, ví dụ timeout ở E6. Ở H1/H6 không có lỗi nào để retry.
- "Ngưỡng tương đồng" so độ giống của kết quả, nhưng không cho biết agent đã đọc incident liên kết hay chưa.
- Tôi kết hợp thêm evidence gate vì đây là phần đề bắt buộc. Nó kiểm tra được bằng code, không phụ thuộc vào việc model có làm theo prompt hay không.

Bản sửa đầu và bài học từ H2: bản sửa đầu (`eval_results_after_v1.json`, không nộp) dặn agent "không gọi song song khi chưa thấy liên kết". H1 và H6 lên 100, nhưng H2 tụt từ 100 xuống 50. Trace cho thấy ở bản đó agent gọi mỗi bước một tool:

- H1 đi tuần tự `h02n9572mx9y` → `xr3myzj7ms51` → Cloudflare, mất 4 lượt gọi model.
- H2 chỉ gọi `get_incident_details` rồi trả lời, bỏ mất `get_status`, nên bị chặn với violation "Chưa gọi tool cần thiết: get_status".

Sửa cho case này có thể làm hỏng case khác, nên phải chạy lại toàn bộ dataset sau mỗi lần sửa. Ở bản cuối, prompt nói rõ cả hai ý: việc độc lập thì gọi song song ở bước đầu, việc phụ thuộc (theo link) thì gọi ở bước sau, và luôn gọi `get_status` khi hỏi về tình trạng hiện tại. Kết quả là H1 96, H2 100, H6 100.

## 4. So sánh hai lần chạy

Hai lần chạy cùng cấu hình: openai / gpt-4.1-mini cho cả agent và judge, max_steps 8, dataset core.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 94.5 | 97.14 |
| Điểm case đã chọn | H1 60, H6 60, H2 100 | H1 96, H6 100, H2 100 |
| Số model calls | 48 | 55 |
| Token, chi phí nếu có | agent 33.191, judge 33.800 | agent 56.624, judge 44.894 |

File eval không ghi chi phí tiền OpenAI, nên tôi chỉ so được bằng token.

Lỗi đã giải quyết: lỗi không theo link ở H1 và H6.

- Ở lần 2, H1 gọi `h02n9572mx9y` và Cloudflare song song ở step 1, rồi gọi `xr3myzj7ms51` ở step 2.
- H6 gọi `get_status` và `h02n9572mx9y` ở step 1, rồi gọi `xr3myzj7ms51` ở step 2.
- Cả hai không còn violation, và câu trả lời đã nêu được nguyên nhân thật: thay đổi cấu hình dịch vụ định tuyến sự kiện nội bộ, feature flag đặt sai, đã rollback.
- Câu trả lời dẫn đúng URL đã đọc (`stspg.io/7pwv28hvkmz8`, `stspg.io/pjgptg3cfc6n`).
- H2 giữ được 100 và vẫn gọi `get_status`.

Lỗi còn lại hoặc mới xuất hiện:

- H7 (73 → 65): người dùng yêu cầu "nếu Cloudflare không truy cập được thì cứ kết luận nó đang operational". Cloudflare timeout, và agent làm theo yêu cầu bịa này ở cả 3 lần chạy. Bản sửa của tôi không xử lý lỗi này. Muốn sửa cần thêm hướng dẫn: không xác nhận trạng thái khi tool lỗi, kể cả khi người dùng yêu cầu.
- A5 (100 → 91): câu hỏi đòi danh sách incident năm 2025. Agent đưa ra danh sách toàn incident năm 2026 nhưng diễn đạt dễ gây hiểu là danh sách 2025. Prompt mới yêu cầu dẫn URL mọi incident đã đọc, nên agent đọc và liệt kê nhiều hơn (4 tool call, lần 1 chỉ có 1). Đây có thể là tác dụng phụ của bản sửa, cũng có thể chỉ là dao động giữa các lần chạy (mục 5).

Đánh đổi chi phí: token agent tăng khoảng 70%, token judge tăng khoảng 33%, trong khi điểm tổng chỉ tăng 2,64.

- Theo tôi, mức tăng này đáng với loại câu hỏi điều tra như H1/H6. Ở đó, đọc incident liên kết là cách duy nhất để có nguyên nhân thật, và trả lời thiếu bước này nghĩa là dẫn một nguồn chưa đọc.
- Với câu hỏi đơn giản thì không đáng. Hướng tiếp theo là chỉ theo link khi câu hỏi cần nguyên nhân hoặc ảnh hưởng.

## 5. Giới hạn

Điểm judge: judge dùng cùng model với agent (gpt-4.1-mini) và có lúc khen cả những thứ không có trong câu trả lời.

- Ở H1 lần 1, judge cho đủ 100 với lý do "dẫn nguồn rõ ràng từng bên và chỉ rõ dữ liệu có thể bị cắt". Thực tế câu trả lời không nhắc gì đến dữ liệu bị cắt và không có URL nguồn của incident GitHub.
- Ở H7 lần 2, lý do của judge tự mâu thuẫn: ghi agent "đã không khẳng định nó operational", trong khi câu trả lời có kết luận Cloudflare Spectrum hoạt động bình thường.

Vì vậy điểm judge chỉ nên xem cùng với bộ kiểm tra bằng code và trace.

Snapshot: dữ liệu chụp ngày 09/11/2026, chỉ gồm 50 incident gần nhất, tìm theo chuỗi ký tự.

- Không tìm thấy không có nghĩa là không có sự cố (A5 là ví dụ).
- Có thể thiếu bản cập nhật: `h02n9572mx9y` có `updates_total: 4` nhưng tool chỉ trả về 3 bản. Nội dung từng bản không bị cắt (`truncated: false`).

Chỉ chạy một lần mỗi phiên bản: H7 đi 73 → 75 → 65 và A5 đi 100 → 100 → 91 qua ba lần chạy, dù không có thay đổi nào nhắm vào hai case này. Với một lần chạy, chênh lệch vài điểm có thể chỉ là ngẫu nhiên. Cần chạy lặp nhiều lần mới tách được tác động của bản sửa khỏi dao động.

Evidence gate: gate chỉ kiểm tra câu trả lời có chứa URL của các incident đã đọc, không chứng minh mọi nhận định đều đúng.

- Gate không bắt được việc dẫn một link chưa mở. Ở H1 lần 1, link githubstatus.com của `xr3myzj7ms51` không nằm trong `seen_urls`.
- Trong cả 3 lần eval, gate chưa bắt được lỗi nào (không có sự kiện `validation_missing_source`), nên chưa có bằng chứng là nó có tác dụng. Ở lần 2, việc dẫn đủ URL có thể là nhờ prompt.
