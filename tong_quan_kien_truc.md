1) Sơ đồ kiến trúc tổng quan


flowchart LR
  %% Clients
  U[user-web-app-client] -->|HTTPS| K[Kong API Gateway]
  A[admin-web-app-client] -->|HTTPS| K

  %% Gateway core capabilities
  K -->|mTLS + route| CDSM[cds management service]
  K -->|mTLS + route| MW[mifox service (core banking)]
  K -->|mTLS + route| UWS[user wallet service (AWS KMS)]
  K -->|mTLS + route| R[relayer service]
  K -->|mTLS + route| IPFS[ipfs service]

  %% Core flows
  CDSM -->|create/update CD product, schedule| MW
  CDSM -->|store metadata| IPFS
  CDSM -->|call onchain ops| R
  UWS -->|sign (EIP-712/tx)| R
  R -->|submit tx + pay fee| L1[Blockchain Layer 1 (CD smart contracts)]
  IPFS -->|CID/hash reference| L1

  %% Optional read paths
  U -->|query CD status| K --> CDSM
  A -->|manage configs| K --> CDSM
  CDSM -->|read onchain state| L1

  Ý nghĩa luồng chính:
	•	Admin/User chỉ gọi vào Kong (entrypoint duy nhất).
	•	CDS Management Service là “bộ não nghiệp vụ CD”.
	•	Mifox Service xử lý logic core banking (đối soát, trạng thái, tính lãi…).
	•	IPFS Service lưu metadata off-chain, trả CID/hash để on-chain reference.
	•	User Wallet Service (AWS KMS) ký giao dịch cho user (không lộ private key).
	•	Relayer Service gom chữ ký + trả gas, gửi tx lên L1.


    2) Thành phần hệ thống

2.1 API Gateway / Kong

Vai trò: Entrypoint duy nhất cho toàn hệ thống.
	•	Route đến các service nội bộ
	•	mTLS nội bộ service-to-service
	•	Rate limit / API key / JWT validation (tuỳ thiết kế Auth)
	•	Logging, tracing, request transformation (nếu cần)

⸻

2.2 user-web-app-client

Vai trò: UI để người dùng:
	•	xem danh sách CD, lãi suất, kỳ hạn
	•	mua/đăng ký CD
	•	theo dõi trạng thái (ACTIVE/MATURED/REDEEMED…)
	•	xem lịch trả lãi / lịch đáo hạn

⸻

2.3 admin-web-app-client

Vai trò: UI quản trị:
	•	tạo/sửa sản phẩm CD (kỳ hạn, rate, rule)
	•	cấu hình lịch trả lãi (coupon schedule)
	•	giám sát trạng thái phát hành, freeze/unfreeze (theo chính sách)
	•	quản lý metadata template và rule compliance (nếu có)

⸻

2.4 user wallet service (AWS KMS)

Vai trò: Ký giao dịch on-chain cho user theo mô hình “custodial signing”.
	•	Private key nằm trong AWS KMS
	•	Cung cấp signing API (EIP-712 / raw tx signing)
	•	Có thể gắn policy theo user/app (quota, allowlist method)

⸻

2.5 relayer service

Vai trò: “Gas sponsor + transaction submitter”
	•	Nhận request giao dịch từ CDS Management Service
	•	Gọi User Wallet Service để lấy chữ ký (nếu cần user-sign)
	•	Tự trả phí gas (meta-tx / sponsored tx)
	•	Submit tx lên Blockchain L1
	•	Retry / bump gas / monitor tx status

⸻

2.6 mifox service (core banking)

Vai trò: Xử lý nghiệp vụ core banking liên quan CD:
	•	ghi nhận mua CD (fiat settlement/ledger)
	•	tính lãi, đối soát lịch trả lãi
	•	cập nhật trạng thái khi đáo hạn, chi trả
	•	quản lý mapping CD on-chain ↔ record banking

⸻

2.7 cds management service

Vai trò: Service trung tâm quản lý CD end-to-end
	•	quản lý sản phẩm CD (product)
	•	tạo CD instance theo user
	•	cấu hình lịch trả lãi (schedule)
	•	orchestration:
	•	gọi Mifox để đảm bảo backing/settlement
	•	lưu metadata lên IPFS
	•	gọi Relayer để thực thi on-chain (mint/issue, update state, redeem…)

⸻

2.8 ipfs service

Vai trò: Lưu metadata off-chain cho CD
	•	lưu JSON metadata (principal, rate, maturity, issuer, terms hash…)
	•	trả về CID / hash
	•	đảm bảo tính bất biến & truy xuất nhanh (gateway nội bộ)

⸻

2.9 blockchain layer 1

Vai trò: Nơi lưu state & logic bất biến của CD
	•	Smart contracts:
	•	issue/mint CD
	•	state machine (ACTIVE/MATURED/REDEEMED/FROZEN…)
	•	reference metadata (CID/hash)
	•	event log phục vụ audit