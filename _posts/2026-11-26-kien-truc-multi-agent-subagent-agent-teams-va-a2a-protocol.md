---
title: "Kiến Trúc Multi-Agent Phổ Biến Nhất Hiện Nay: Subagent, Agent Teams và A2A Protocol"
description: "So sánh chuyên sâu 3 kiến trúc Multi-Agent phổ biến nhất: Subagent, Agent Teams và A2A Protocol — kèm chiến lược chọn mô hình và tối ưu chi phí token."
date: 2026-11-26 17:40:00 +0700
categories: [AI & Agent, Multi-Agent]
tags: [multi-agent, subagent, agent-teams, a2a, claude-code, architecture]
pin: false
toc: true
media_subpath: /assets/img/posts/
---

Chào anh em yêu công nghệ! Nếu như giai đoạn 2023–2024 là thời kỳ bùng nổ của các Single AI Agent xoay quanh việc gọi công cụ (Tool Calling / MCP), thì từ năm 2025–2026 trở đi, tâm điểm của thế giới AI Engineering đã dịch chuyển mạnh mẽ sang **Multi-Agent Systems** — với những tiêu chuẩn và kiến trúc thực chiến được định hình bởi **Anthropic** (qua hệ sinh thái Claude Code Subagents & Agent Teams) cùng **Google và Linux Foundation** (với giao thức mở Agent2Agent - A2A). Việc hiểu rõ cách các agent phối hợp, chia việc và nói chuyện với nhau sẽ giúp anh em thiết kế được những hệ thống AI vừa mạnh mẽ, vừa thoát khỏi cảnh "ngộp thở" vì giới hạn bộ nhớ ngữ cảnh.

---

## 1. Tổng quan & Sự khác biệt cốt lõi giữa 3 cơ chế Multi-Agent

Khi một bài toán trở nên quá lớn đối với một AI Agent đơn lẻ, phản xạ tự nhiên của chúng ta là chia nhỏ công việc cho nhiều agent cùng làm. Thế nhưng, chia việc theo mô hình nào để các agent không giẫm chân lên nhau?

Hiện nay, thế giới điều phối đa tác tử (**Multi-Agent Orchestration**) xoay quanh 3 cơ chế phổ biến nhất: **Subagent**, **Agent Teams** và **A2A (Agent2Agent Protocol)**. Điểm khác biệt cốt lõi tạo nên bản sắc của từng mô hình nằm ở **cấu trúc kết nối (communication topology)** và cách chúng phân chia công việc:

- **Subagent (Mô hình Hub-and-Spoke / Cha - Con)**: Đây là cấu trúc phân cấp nghiêm ngặt. Các subagent chỉ có quan hệ duy nhất với agent cha (**Parent / Lead Agent**). Mọi việc giao nhiệm vụ và báo cáo kết quả đều phải đi qua agent cha; các subagent hoàn toàn không biết đến sự tồn tại của nhau và không giao tiếp trực tiếp với nhau.
- **Agent Teams (Mô hình Team Lead - Teammates có bộ nhớ phối hợp chung)**: Vẫn giữ mối quan hệ giữa người dẫn dắt (**Team Lead**) và các thành viên (**Teammates / Workers**), nhưng rào cản giao tiếp đã được gỡ bỏ. Tất cả thành viên trong nhóm — cả Lead lẫn Worker — đều có thể chạm trực tiếp vào **Shared Task List** (danh sách công việc chung) để nhận hay cập nhật việc, đồng thời sử dụng **Mailbox** (hộp thư) để nhắn tin trao đổi trực tiếp với nhau.
- **A2A (Agent2Agent Protocol — Mô hình Ngang hàng / Peer-to-Peer)**: Phá vỡ cấu trúc cha - con cố định trong một phiên làm việc để hướng tới mối quan hệ ngang hàng giữa **Client Agent** và **Remote Agent**. Một agent có thể gửi trực tiếp từng `Task` sang một agent độc lập khác qua mạng và nhận lại kết quả (`Artifact` / `Message`).

Hiểu được bức tranh tổng thể về ba nhánh kiến trúc này rồi, hãy cùng mình mở nắp ca-pô từng cơ chế để xem bên trong chúng vận hành ra sao và đánh đổi những gì.

---

## 2. Cơ chế 1: Subagent — Nhanh, gọn và cô lập ngữ cảnh như một chiếc Tool

Khi bắt đầu bước từ Single Agent sang Multi-Agent, **Subagent** gần như luôn là điểm dừng chân đầu tiên vì sự đơn giản nhưng cực kỳ hiệu quả của nó. Sơ đồ dưới đây minh họa rõ cấu trúc phân cấp một chiều giữa Agent cha và các Subagent con:

![Mô hình Subagent (Hub-and-Spoke)](pasted-image-20261126111146.png)
*Mô hình Subagent (Hub-and-Spoke): Agent cha nắm toàn quyền khởi tạo, giao việc và nhận kết quả từ các agent con.*

### Cách hoạt động
Hãy hình dung Agent cha đang giải quyết một bài toán lớn và cần ai đó đi đọc 40 file log để tìm nguyên nhân lỗi. Nếu tự đọc hết, bộ nhớ của nó sẽ lập tức bị lấp đầy bởi dữ liệu thừa. Thay vào đó:

1. Agent cha khởi tạo các agent con với cửa sổ ngữ cảnh cô lập (**isolated context window**) để thực hiện một tác vụ cụ thể, sau đó chỉ nhận lại phần kết quả tóm tắt. Nếu anh em tự tay triển khai từ đầu, cách đơn giản và thực dụng nhất chính là dùng `child_process` mở một tiến trình CLI kèm theo prompt, hoặc bọc nó lại thành một MCP tool.
2. Dưới góc nhìn của agent cha, việc gọi một subagent không khác gì việc gọi một công cụ thực thi lệnh thông thường (như `Bash tool` hay `Agent tool`): truyền đầu vào, chờ xử lý xong và lấy đầu ra.

### Lợi ích mang lại
- **Chống tràn ngữ cảnh (Context Isolation / Context Preservation)**: Mọi dữ liệu thô, quá trình thử sai của agent con đều nằm gọn trong context riêng của nó, giữ cho context của agent cha luôn sạch và nhẹ.
- **Chạy song song (Parallel Execution)**: Agent cha có thể tung ra cùng lúc nhiều subagent để xử lý các nhiệm vụ độc lập.
- **Triển khai cực dễ**: Chỉ cần đóng gói nhanh thành một tool hoặc viết một MCP tool là hệ thống có thể chạy ngay.

### Nhược điểm cần lưu ý
Tuy gọn nhẹ, nhưng khi đưa vào các bài toán dài hơi, cấu trúc cha - con một chiều của Subagent bắt đầu bộc lộ giới hạn:
- **Tốn token và tăng độ trễ khi cần phối hợp chéo**: Với các tác vụ phức tạp (chẳng hạn Agent làm UI cần hỏi thông số API từ Agent làm Backend), chúng không thể hỏi thẳng nhau. Mọi thông điệp đều phải vòng qua Agent cha làm trung gian, gây đội chi phí token lên rất nhanh.
- **Nút cổ chai kiến thức ở Agent cha**: Vì danh sách công việc và kế hoạch chỉ do duy nhất Agent cha nắm giữ, một mình Agent cha phải ôm đồm toàn bộ kiến thức lẫn ngữ cảnh điều phối của dự án.
- **Gánh nặng quản lý**: Agent cha phải trực tiếp quản lý toàn bộ vòng đời (**lifecycle**) của các subagent từ lúc sinh ra đến lúc kết thúc.

---

## 3. Cơ chế 2: Agent Teams — Khi các Agent biết tự nhìn bảng việc và nhắn tin cho nhau

Nếu như ở mô hình Subagent, việc mọi thứ phải thông qua "sếp tổng" (Agent cha) gây ra điểm nghẽn giao tiếp, thì câu hỏi đặt ra là: *Tại sao không cho các agent một cái bảng công việc chung và một kênh chat riêng để chúng tự bàn bạc với nhau?* Đó chính là lý do **Agent Teams** ra đời. Nhìn vào sơ đồ kiến trúc dưới đây, anh em sẽ thấy luồng phối hợp đã thay đổi hoàn toàn nhờ sự xuất hiện của Task List và Mailbox:

![Mô hình Agent Teams với Task List và Mailbox](pasted-image-20261126111019.png)
*Mô hình Agent Teams: Cả Team Lead lẫn các Worker đều có thể tương tác trực tiếp với bảng Task List chung và trao đổi ngang hàng qua hệ thống Mailbox.*

### Cách hoạt động
Trong mô hình Agent Teams, không gian làm việc được tổ chức giống hệt một đội phát triển phần mềm ngoài đời thực:

1. Agent cha (**Team Lead**) khởi tạo các agent con (**Teammates**) chạy dưới dạng các phiên độc lập (ví dụ spawn các tiến trình CLI riêng biệt), kèm theo một thư mục **Task List** dùng chung. Bên trong thư mục này chứa nhiều file `task.json` lưu rõ thông tin công việc và subagent nào đang nhận việc (ví dụ trong Claude Code được lưu tại `~/.claude/tasks/{team-name}/`).
2. Để giao tiếp, mỗi agent trong đội được cấp một file hộp thư riêng (`mailbox.json`, ví dụ `~/.claude/teams/{team-name}/inboxes/{agent-name}.json`). Khi một agent muốn trao đổi với đồng đội, nó chỉ cần gửi message thẳng vào file mailbox của người nhận mà không cần làm phiền Team Lead.

### Lợi ích mang lại
- **Chống tràn ngữ cảnh tối đa**: Mỗi teammate vận hành trong một phiên độc lập với context window hoàn toàn tách biệt.
- **Chạy song song nhiều nhiệm vụ hiệu quả**: Các nhánh công việc lớn có thể tiến hành đồng thời mà không phải chờ đợi (chặn) nhau.
- **Linh hoạt thảo luận và cập nhật kế hoạch**: Điểm ăn tiền nhất của Agent Teams là các agent có thể trực tiếp thảo luận ngang hàng để làm rõ yêu cầu, tự nhận việc (**self-claim**) hoặc đề xuất thêm/sửa task trong Task List cho phù hợp với tình hình thực tế.

### Nhược điểm cần lưu ý
Trao quyền tự chủ và giao tiếp tự do cho các agent cũng đồng nghĩa với việc độ phức tạp kỹ thuật tăng lên đáng kể:
- **Nguy cơ "họp hành quá nhiều" nếu prompt không kỹ**: Đòi hỏi người thiết kế phải viết prompt thật chặt chẽ để quy định rõ khi nào cần giao tiếp, giao tiếp những gì. Nếu không kiểm soát tốt, các agent sẽ nhắn tin qua lại liên tục ngay cả ở những task không cần thiết, làm đốt token và giảm hiệu quả rõ rệt.
- **Bài toán xung đột dữ liệu (Race Conditions / Concurrency Control)**: Khi tự triển khai, anh em buộc phải xây dựng các cơ chế chống xung đột phức tạp — ví dụ xử lý tình huống hai hay nhiều agent cùng lao vào sửa file Task List tại cùng một thời điểm.
- **Quản lý vòng đời**: Dù các worker đã tự chủ hơn trong công việc, Team Lead vẫn phải chịu trách nhiệm giám sát và quản lý vòng đời (khởi tạo, theo dõi, đóng phiên) của toàn bộ các teammate.

---

## 4. Cơ chế 3: A2A (Agent2Agent Protocol) — Giao thức kết nối thế giới Agent ngang hàng

Subagent và Agent Teams giải quyết rất tốt bài toán phối hợp bên trong cùng một máy hoặc cùng một công cụ. Nhưng điều gì sẽ xảy ra khi Agent của bạn cần nhờ đến một Agent chuyên gia khác nằm ở một server khác, viết bằng một ngôn ngữ hay framework hoàn toàn khác? Lúc này, chúng ta bước sang địa hạt của **A2A (Agent2Agent Protocol)**. Hình minh họa dưới đây cho thấy cách các agent độc lập tìm thấy nhau thông qua Agent Card và gửi task trực tiếp:

![Mô hình giao tiếp ngang hàng A2A Protocol](pasted-image-20261126111112.png)
*Mô hình A2A Protocol: Các agent khám phá năng lực của nhau qua Agent Card và giao tiếp ngang hàng (Peer-to-Peer) để ủy quyền từng Task.*

### Cách hoạt động
Thay vì khởi tạo tiến trình con từ một mối, A2A coi mỗi agent là một thực thể độc lập trên mạng:

1. Mỗi agent tạo một hồ sơ năng lực gọi là **Agent Card** (`agent-card.json`) mô tả rõ khả năng, kỹ năng (`skills`), định dạng đầu vào/đầu ra và địa chỉ kết nối của mình. Anh em có thể đẩy Agent Card lên một "server A2A trung tâm" (Registry/Catalog) hoặc để mỗi agent tự đóng vai trò một A2A Server độc lập và công bố card của mình tại đường dẫn chuẩn `/.well-known/agent-card.json`.
2. Khi một agent cần nhờ agent khác hỗ trợ, nó sẽ xem các **Agent Card** để chọn đúng agent có chuyên môn phù hợp, sau đó gửi message kèm `Task` trực tiếp đến endpoint của agent đó.

### Lợi ích mang lại
- **Ủy quyền chuyên môn sâu (Delegation)**: Dễ dàng giao việc cho các agent khác có chuyên môn sâu hơn mình, bất kể chúng được xây dựng bằng công nghệ hay chạy trên hạ tầng nào.
- **Phối hợp song song diện rộng**: Một Client Agent có thể bắn yêu cầu giao nhiều việc song song cho hàng loạt Remote Agent khác nhau.
- **Mở rộng quy mô (Scalability) và tái sử dụng (Reusability) tuyệt vời**: Một agent chuyên biệt sau khi dựng xong và công bố Agent Card có thể được tái sử dụng bởi vô số agent và hệ thống khác nhau.

### Nhược điểm cần lưu ý
Vì được thiết kế như một giao thức giao tiếp liên mạng giữa các thực thể độc lập, A2A cũng có những giới hạn tự nhiên khi đặt cạnh Agent Teams:
- **Không có sẵn bảng phân việc chung (Shared Taskboard / Orchestration Backlog)**: Trong A2A, mỗi Remote Agent là một hộp đen (**opaque agent**) không biết gì về bức tranh toàn cảnh dự án của bạn; nó chỉ nhận từng `Task` được giao, xử lý rồi trả lại kết quả. Dù bản thân giao thức A2A vẫn quản lý chặt chẽ vòng đời trạng thái của từng `Task` đơn lẻ (như `SUBMITTED`, `WORKING`, `INPUT_REQUIRED`, `COMPLETED` kèm các API `GetTask`, `ListTasks`, `CancelTask`), nhưng nó không có sẵn một `tasklist` chung để các agent tự vào chia việc với nhau.
- **Phải tự custom nhiều cho bài toán lớn**: Nếu muốn dùng A2A cho một dự án lớn nhiều bước phụ thuộc lẫn nhau, phía gọi (Client/Orchestrator) sẽ phải tự xây dựng thêm tầng lập kế hoạch và điều phối luồng công việc (**custom orchestration layer**) khá vất vả.

---

## 5. Kinh nghiệm thực chiến: Chọn cơ chế nào, tối ưu chi phí và cấu hình ra sao?

Đi qua cả 3 cơ chế, có thể thấy không có kiến trúc nào là "viên đạn bạc" cho mọi bài toán. Để giúp anh em áp dụng ngay vào dự án thực tế mà không bị cháy túi vì token, dưới đây là 3 đúc kết quan trọng nhất về cách lựa chọn, tối ưu chi phí và cấu hình hệ thống.

### Bảng tiêu chí chọn nhanh cơ chế cho từng bài toán (Decision Matrix)
- **Dùng Subagent khi**: Tác vụ con mang tính độc lập, anh em chỉ quan tâm đến kết quả cuối cùng (ví dụ: đọc 30 file mã nguồn để tìm bug, chạy unit test, tóm tắt tài liệu) và muốn giữ độ phức tạp cũng như chi phí ở mức thấp nhất.
- **Dùng Agent Teams khi**: Nhiều agent cùng làm việc trên một repo hoặc dự án lớn nhưng chia theo tầng (Frontend, Backend, Test) hoặc cùng tranh luận nhiều giả thuyết debug, nơi kết quả tìm được của worker này sẽ trực tiếp làm thay đổi kế hoạch hành động của worker kia.
- **Dùng A2A khi**: Cần kết nối các agent **khác hệ sinh thái, khác team hoặc khác server** (ví dụ: Agent chăm sóc khách hàng trên Cloud A gọi sang Agent tra cứu kho vận viết bằng LangGraph trên Server B).

### Bài toán chi phí Token & Chiến lược phân tầng Model (`Model Tiering`)
Một sự thật thực chiến mà ít người để ý khi mới làm Multi-Agent là tốc độ "đốt" token của nó:
- **Bội số chi phí**: Theo thống kê thực tế từ hệ thống Multi-Agent của Anthropic, kiến trúc đa agent có thể tiêu tốn lượng token gấp **~15 lần** so với một cuộc hội thoại chat đơn lẻ (và gấp khoảng 3–4 lần so với Single Agent).
- **Cách tối ưu bằng `Model Tiering`**: Tuyệt đối không dùng một model đắt nhất cho tất cả các agent trong hệ thống. Chiến lược thông minh là dùng model có khả năng suy luận mạnh nhất (như `Opus`) làm **Team Lead / Orchestrator** để lập kế hoạch và tổng hợp kết quả, trong khi các **Subagents / Workers** chạy song song phía dưới thì gán các model nhanh và tiết kiệm chi phí hơn (như `Sonnet` hoặc `Haiku`).

### Cấu hình chuẩn để bật và kiểm tra Agent Teams / Subagents trong Claude Code
Nếu anh em muốn trải nghiệm trực tiếp sự khác biệt giữa Subagent và Agent Teams ngay trên Claude Code, đây là hai cấu hình cốt lõi cần nắm:
- `.claude/agents/<tên-agent>.md`: File định nghĩa Subagent dùng chung được cho cả chế độ Subagent thông thường lẫn khi gọi làm Teammate trong đội (khai báo rõ `name`, `description`, `tools`, `model` bên trong phần YAML frontmatter).
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`: Biến môi trường (đặt trực tiếp trong shell hoặc cấu hình vào `settings.json`) để kích hoạt tính năng Agent Teams trong phiên terminal tương tác.

---

## Lời kết

Nhìn lại toàn bộ bức tranh, sự tiến hóa từ **Subagent** → **Agent Teams** → **A2A Protocol** phản ánh đúng cách con người chúng ta tổ chức công việc: từ một cá nhân sai vặt các trợ lý riêng lẻ (Subagent), tiến lên một nhóm dự án cùng nhìn vào một bảng Kanban và nhắn tin phối hợp với nhau (Agent Teams), cho tới việc kết nối liên thông giữa các phòng ban và công ty độc lập qua một giao thức chuẩn hóa (A2A).

Mỗi cơ chế đều có một thế mạnh và vùng tối ưu riêng:
- Cần gọn nhẹ, dễ làm, chống tràn context nhanh: chọn **Subagent**.
- Cần giải quyết bài toán lớn có sự bàn bạc, tự chia lại task giữa các thành viên: chọn **Agent Teams**.
- Cần mở rộng quy mô, kết nối các agent chuyên biệt xuyên biên giới server và nền tảng: chọn **A2A**.

Hiện tại dự án của anh em đang dừng ở Single Agent, đang tự bọc Subagent bằng MCP/CLI hay đã bắt đầu thử nghiệm với Agent Teams và A2A rồi? Hãy chia sẻ góc nhìn và bài toán thực tế của anh em để chúng ta cùng bàn luận nhé. Chúc anh em xây dựng được những đội quân AI Agent thật mượt mà và hiệu quả!
