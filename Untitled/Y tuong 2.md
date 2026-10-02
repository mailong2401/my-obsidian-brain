

## Nhóm 1: Tập trung và học tập

**1. Focus Dungeon**  
Mỗi phiên Pomodoro 25 phút là một tầng hầm ngục. Bạn ở lại tập trung thì nhân vật đánh quái, thoát app giữa chừng thì quái đánh lại và mất máu. Càng xuống sâu càng có loot hiếm.

**2. Code Dungeon (Roguelike luyện code)**  
Mỗi bài tập code, SQL hoặc thuật toán là một con quái. AI sinh đề theo trình độ, giải đúng thì đánh bại quái, và sau mỗi lần chơi bạn chọn một "perk" như hint, thêm thời gian hay bỏ qua một bài.

**3. Language Tavern**  
Quán rượu thời trung cổ với các NPC do LLM điều khiển. Bạn luyện tiếng Anh bằng cách nói chuyện để hoàn thành nhiệm vụ như "gọi món", "mặc cả với thương nhân", "phỏng vấn xin việc". AI chấm phát âm và ngữ pháp, rồi thưởng XP.

**4. Flashcard Battler**  
Spaced repetition kết hợp card game. Mỗi flashcard là một lá bài tấn công, nhớ đúng thì đánh trúng, quên thì bị phản đòn. Thẻ bạn hay quên sẽ trở thành "boss" cần hạ gục.

## Nhóm 2: Sức khỏe và đời sống

**5. Sleep Garden**  
Ngủ đủ và đúng giờ thì khu vườn nở hoa, thức khuya thì cây héo. Có thể gắn thêm thời tiết trong vườn thay đổi theo chất lượng giấc ngủ của bạn.

**6. Walk the World**  
Số bước chân thật sẽ di chuyển nhân vật trên bản đồ phiêu lưu: 10.000 bước đi qua một khu rừng, 100.000 bước tới một thị trấn mới. Có thể làm bản đồ theo các vùng của Việt Nam.

**7. Mood Island**  
Bạn viết nhật ký cảm xúc hằng ngày và hòn đảo thay đổi theo đó: nắng, mưa, cầu vồng. AI phản hồi như một người bạn và gợi ý hoạt động giúp tinh thần tốt hơn.

**8. Healthy Farm**  
Ăn uống lành mạnh thì có hạt giống để trồng, ăn đồ ăn vặt thì sâu bệnh phá hoại. Chụp ảnh món ăn để AI nhận diện và tính điểm.

## Nhóm 3: Xã hội và cạnh tranh

**9. Study Raid (co-op)**  
Cả nhóm bạn cùng học để đánh một Raid Boss. Tổng số giờ học của cả nhóm trong tuần chính là sát thương, và nếu một người lười thì cả nhóm bị ảnh hưởng. Rất hợp lớp học hoặc CLB sinh viên.

**10. Habit Duel**  
Thách đấu 1v1 với bạn bè, ai giữ streak lâu hơn thì thắng. Có thể cược bằng coin trong game hoặc "hình phạt vui" như mời trà sữa.

**11. Chore Quest (gia đình)**  
Việc nhà thành nhiệm vụ: con rửa chén được +coin, bố mẹ duyệt nhiệm vụ. Đổi coin lấy phần thưởng thật như xem phim hay đi chơi.

## Nhóm 4: Tài chính và sáng tạo

**12. Savings City Builder**  
Mỗi mục tiêu tiết kiệm xây một công trình. Tiết kiệm đủ thì lên tòa nhà mới, chi tiêu quá tay thì thành phố bị "khủng hoảng". Có thể đặt mục tiêu chung cho cặp đôi hoặc nhóm bạn.

**13. Library Kingdom**  
Mỗi cuốn sách bạn đọc xong thêm một kệ sách vào thư viện của bạn. Ghi chú và highlight được "ma thuật hóa" thành vật phẩm, còn AI tóm tắt và hỏi lại để kiểm tra bạn đã hiểu chưa.

**14. Local Explorer**  
Game khám phá bằng GPS: đi quán cà phê, công viên, điểm du lịch thì mở khóa "huy hiệu" và xây bộ sưu tập địa điểm. Có thể bán cho quán và đơn vị du lịch địa phương như một kênh quảng bá.

## Nhóm 5: Dành cho dev

**15. GitHub Quest**  
Kết nối GitHub API: commit, PR, review code được quy đổi thành XP. Có skill tree theo tech stack, huy hiệu như "7 ngày liên tục commit", và bảng xếp hạng giữa các bạn dev.

**16. Idle Dev Tycoon**  
Game idle: bạn mở công ty phần mềm ảo. Thời gian học thật và số bài tập hoàn thành tạo ra "thu nhập thụ động", dùng để thuê dev ảo, nâng cấp server, mở rộng văn phòng.

## Đánh giá nhanh

|Ý tưởng|Độ khó MVP|Tiềm năng|Điểm nhấn kỹ thuật|
|---|---|---|---|
|Focus Dungeon|Dễ|⭐⭐⭐⭐|Timer, state machine|
|Code Dungeon|Trung bình|⭐⭐⭐⭐⭐|LLM sinh đề, sandbox chấm bài|
|Language Tavern|Khó|⭐⭐⭐⭐⭐|LLM, speech|
|Study Raid|Trung bình|⭐⭐⭐⭐|Realtime, Redis leaderboard|
|Sleep Garden|Dễ|⭐⭐⭐|Health API|
|GitHub Quest|Dễ|⭐⭐⭐⭐|OAuth, webhook, queue|
|Savings City Builder|Dễ|⭐⭐⭐⭐|CRUD, hệ thống phần thưởng|
|Local Explorer|Khó|⭐⭐⭐⭐|GPS, bản đồ|

## 3 ý mình thấy đáng làm nhất

1. **GitHub Quest**: MVP nhanh, bạn tự dùng được ngay, và nhà tuyển dụng nhìn vào thấy luôn kỹ năng backend của bạn (OAuth, webhook, queue, bảng xếp hạng).
2. **Code Dungeon**: kết hợp backend + AI/LLM, đúng hướng các JD đang tuyển, và có thể mở rộng thành sản phẩm thật cho sinh viên IT.
3. **Study Raid**: có yếu tố xã hội nên dễ lan truyền trong trường, đồng thời luyện được realtime (WebSocket) và scale với Redis.

Bạn muốn mình đi sâu vào ý nào không? Mình có thể phác thảo luôn tính năng MVP, kiến trúc backend và lộ trình làm trong 4-8 tuần.