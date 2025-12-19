1) L1 Architecture and Technical Approach

Tokenized Certificate of Deposit on a Purpose-Built L1 Blockchain

🎯 Problem We Solve

Chứng chỉ tiền gửi (CD) là một sản phẩm tài chính an toàn, phổ biến trong ngân hàng, nhưng:
	•	Thanh khoản thấp (không giao dịch thứ cấp hiệu quả),
	•	Quy trình phát hành & quản lý thủ công,
	•	Khó tích hợp vào hệ sinh thái tài chính số (DeFi, treasury, collateral).

Chúng tôi xây dựng một Layer-1 blockchain chuyên biệt cho tài sản tài chính được quản lý (regulated financial assets), trong đó CD được token hóa, quản lý vòng đời, và giao dịch một cách minh bạch, tuân thủ và hiệu quả.

🔹 Why L1 (not just smart contracts on public chain)?

CD là financial instrument có ràng buộc pháp lý, cần:
	•	Kiểm soát validator & participant,
	•	Tuân thủ KYC/AML & compliance,
	•	Hiệu năng cao, phí ổn định,
	•	Tích hợp ngân hàng & core banking.

➡️ Vì vậy, chúng tôi chọn Application-Specific L1 thay vì chỉ triển khai smart contract trên public chain.

🔹 L1 Technology Stack

Layer
Technology
Base L1
Avalanche Subnet (EVM-compatible)
Consensus
Avalanche Consensus (Snowman++)
Execution
EVM (Solidity)
Identity
DID-based (on-chain identity mapping)
Token Standard
ERC-20 / ERC-1155 (Tokenized CD)
Cross-chain
Avalanche Teleporter
Off-chain Services
KYC, Custodian, Banking Integration

Core L1 Components

1️⃣ CD Tokenization Layer
	•	Mỗi CD được phát hành dưới dạng on-chain token:
	•	ERC-20 → CD chuẩn hóa, mệnh giá cố định,
	•	ERC-1155 → CD phân mảnh (fractional CD).
	•	Metadata on-chain / IPFS:
	•	Principal,
	•	Interest rate,
	•	Maturity date,
	•	Issuing bank,
	•	Legal document hash.

➡️ Token đại diện quyền lợi pháp lý của người sở hữu CD.

2️⃣ Lifecycle Management Smart Contracts


Smart contracts kiểm soát:
	•	Phát hành (mint) CD,
	•	Chuyển nhượng thứ cấp (nếu được phép),
	•	Tự động đáo hạn (maturity),
	•	Thanh toán gốc + lãi,
	•	Đóng băng / thu hồi khi có yêu cầu pháp lý.

3️⃣ Permission & Compliance Layer (L1-Native)


Validator & participant được whitelist:
	•	Banks,
	•	Custodians,
	•	Regulated institutions.
	•	Mỗi địa chỉ ví được map với:
	•	DID,
	•	KYC status,
	•	Role (issuer, investor, regulator).

➡️ Smart contract enforce:
	•	Ai được mua CD,
	•	Ai được giao dịch,
	•	Ai được redeem.

    4️⃣ Banking & Custodian Integration
	•	L1 không thay thế ngân hàng – L1 kết nối ngân hàng:
	•	CD backing bằng tiền fiat thật tại ngân hàng phát hành,
	•	Custodian xác nhận tài sản bảo chứng,
	•	Oracle cập nhật trạng thái off-chain → on-chain.

➡️ Đảm bảo 1:1 backing giữa token CD và tiền gửi thực.

5️⃣ Cross-Chain Liquidity & Expansion
	•	CD token có thể:
	•	Bridge sang Avalanche C-Chain,
	•	Sử dụng làm collateral,
	•	Kết hợp DeFi có kiểm soát.
	•	Teleporter cho phép:
	•	Cross-L1 messaging,
	•	Mở rộng sang các subnet tài chính khác.
