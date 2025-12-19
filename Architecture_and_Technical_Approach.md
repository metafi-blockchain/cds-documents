L1 Architecture and Technical Approach

Kiến trúc Layer-1 cho Chứng chỉ Tiền gửi được token hóa – Thiết kế ưu tiên tuân thủ

🎯 Triết lý thiết kế

Chứng chỉ tiền gửi (Certificate of Deposit – CD) là sản phẩm tài chính có quản lý.
Vì vậy, kiến trúc của chúng tôi được xây dựng theo nguyên tắc tuân thủ trước – công nghệ sau.

Hệ thống không thay thế ngân hàng, mà kết nối core banking truyền thống với một Layer-1 blockchain chuyên biệt, trong đó:
	•	Ngân hàng vẫn giữ vai trò lưu ký và đối soát tiền,
	•	Blockchain cung cấp minh bạch, tự động hóa và khả năng kiểm toán.

⸻

🧱 Tổng quan kiến trúc

Kiến trúc hệ thống được chia tách rõ ràng giữa:
	•	Trải nghiệm người dùng,
	•	Nghiệp vụ ngân hàng,
	•	Thanh toán và lưu trữ on-chain.

Người dùng và quản trị viên thao tác qua các ứng dụng web, tất cả đi qua API Gateway (Kong).
Logic nghiệp vụ CD được xử lý tại CDS Management Service, tích hợp trực tiếp với Core Banking (Mifox).
Metadata của CD được lưu off-chain trên IPFS, trong khi trạng thái, vòng đời và logic bất biến được lưu on-chain trên Layer-1.

Toàn bộ giao dịch on-chain được:
	•	ký an toàn bằng AWS KMS,
	•	và Relayer Service chi trả phí giao dịch, giúp người dùng không cần nắm giữ token gas.

⸻

🧩 Các thành phần chính của hệ thống

1️⃣ API Gateway (Kong)

Kong đóng vai trò điểm truy cập duy nhất:
	•	Định tuyến request đến các service nội bộ,
	•	Áp dụng mTLS cho giao tiếp nội bộ,
	•	Rate-limit, xác thực và logging.

Đảm bảo tiêu chuẩn bảo mật và vận hành cấp doanh nghiệp.

⸻

2️⃣ Ứng dụng Web cho Người dùng & Quản trị
	•	User Web App: đăng ký mua CD, theo dõi lãi suất, lịch trả lãi và trạng thái đáo hạn.
	•	Admin Web App: cấu hình sản phẩm CD, kỳ hạn, lãi suất, quy tắc vận hành và giám sát hệ thống.

Người dùng không cần hiểu blockchain khi sử dụng hệ thống.

⸻

3️⃣ CDS Management Service (Lớp nghiệp vụ trung tâm)

Đây là bộ não của hệ thống, chịu trách nhiệm:
	•	Quản lý sản phẩm và từng chứng chỉ tiền gửi,
	•	Thiết lập lịch trả lãi và đáo hạn,
	•	Điều phối luồng nghiệp vụ giữa Core Banking, IPFS và Blockchain,
	•	Kích hoạt các hành động on-chain thông qua Relayer.

Mọi giao dịch on-chain đều phải phản ánh trạng thái tài chính hợp lệ off-chain.

⸻

4️⃣ Tích hợp Core Banking – Mifox Service

Mifox là nguồn dữ liệu tài chính gốc:
	•	Ghi nhận tiền gửi bảo chứng cho mỗi CD,
	•	Tính toán lãi suất, số tiền đáo hạn,
	•	Xác nhận đối soát trước khi phát hành hoặc tất toán on-chain.

Đảm bảo tỷ lệ bảo chứng 1:1 giữa token CD và tiền gửi thực.

⸻

5️⃣ Lưu trữ Metadata Off-chain – IPFS

Các thông tin như:
	•	mệnh giá,
	•	lãi suất,
	•	kỳ hạn,
	•	đơn vị phát hành,
	•	hash tài liệu pháp lý,

được lưu trữ off-chain trên IPFS.
Blockchain chỉ lưu CID/hash tham chiếu, giúp dữ liệu:
	•	bất biến,
	•	dễ kiểm toán,
	•	tối ưu chi phí on-chain.

⸻

6️⃣ User Wallet Service – Ký giao dịch bằng AWS KMS

Private key của người dùng được:
	•	quản lý tập trung và bảo mật bằng AWS KMS,
	•	không bao giờ lộ ra ngoài.

Service này:
	•	ký giao dịch theo chuẩn EIP-712 hoặc raw transaction,
	•	tuân thủ chính sách kiểm soát và phân quyền.

Phù hợp với tiêu chuẩn bảo mật của tổ chức tài chính.

⸻

7️⃣ Relayer Service – Giao dịch không cần gas cho người dùng

Relayer chịu trách nhiệm:
	•	chi trả phí giao dịch on-chain,
	•	thu thập chữ ký từ Wallet Service,
	•	gửi giao dịch lên Layer-1,
	•	theo dõi và retry khi cần.

Người dùng có trải nghiệm gasless, tương tự ứng dụng tài chính truyền thống.

⸻

8️⃣ Blockchain Layer-1

Layer-1 lưu trữ:
	•	smart contract quản lý CD,
	•	trạng thái vòng đời (ISSUED → ACTIVE → MATURED → REDEEMED),
	•	tham chiếu metadata IPFS,
	•	event log phục vụ audit và giám sát.

Blockchain đóng vai trò lớp settlement và kiểm toán minh bạch, không thay thế hệ thống ngân hàng.

⸻

🔐 Bảo mật & Tuân thủ
	•	Validator được kiểm soát (permissioned),
	•	Smart contract có thể audit,
	•	Event on-chain phục vụ giám sát và thanh tra,
	•	Phân tách rõ ràng giữa custody, nghiệp vụ và settlement.

⸻

🚀 Giá trị của kiến trúc

✔️ Thiết kế riêng cho tài sản tài chính có quản lý
✔️ Thân thiện với ngân hàng và cơ quan quản lý
✔️ Trải nghiệm người dùng đơn giản, không cần gas
✔️ Phân tách on-chain / off-chain rõ ràng
✔️ Sẵn sàng triển khai thực tế, không chỉ là ý tưởng