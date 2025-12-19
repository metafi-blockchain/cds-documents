# Architecture and Technical Approach

## Kiến trúc Layer-1 cho Chứng chỉ Tiền gửi được Token hóa – Thiết kế ưu tiên tuân thủ

---

## Vấn đề chúng tôi giải quyết

Chứng chỉ tiền gửi (Certificate of Deposit – CD) là sản phẩm tài chính an toàn và phổ biến trong hệ thống ngân hàng, tuy nhiên vẫn tồn tại nhiều hạn chế:

- Thanh khoản thấp, gần như không có thị trường thứ cấp,
- Quy trình phát hành, quản lý và tất toán còn thủ công,
- Khó tích hợp vào hệ sinh thái tài chính số (treasury, collateral, DeFi có kiểm soát).

Giải pháp của chúng tôi là xây dựng một **Layer-1 blockchain chuyên biệt** cho tài sản tài chính có quản lý, trong đó CD được:

- token hóa,
- quản lý vòng đời đầy đủ,
- và vận hành minh bạch, có thể kiểm toán,

trong khi ngân hàng vẫn giữ vai trò trung tâm về lưu ký và đối soát tiền.

---

## Triết lý thiết kế: Compliance-first

Chứng chỉ tiền gửi là financial instrument có ràng buộc pháp lý, vì vậy kiến trúc hệ thống được xây dựng theo nguyên tắc:

**Tuân thủ trước – công nghệ sau**

Hệ thống:

- không thay thế ngân hàng,
- mà kết nối core banking truyền thống với một Layer-1 blockchain chuyên biệt.

Trong mô hình này:

- Ngân hàng chịu trách nhiệm lưu ký tiền fiat và tính toán lãi,
- Blockchain đóng vai trò lớp settlement, tự động hóa và kiểm toán minh bạch.

---

## Vì sao cần Layer-1 riêng (Application-Specific L1)?

Việc triển khai CD chỉ bằng smart contract trên public chain không đáp ứng được các yêu cầu sau:

- Kiểm soát validator và participant,
- Tuân thủ KYC/AML và chính sách nội bộ,
- Phí giao dịch ổn định, hiệu năng cao,
- Tích hợp chặt chẽ với core banking.

Do đó, chúng tôi lựa chọn **Application-Specific Layer-1**, được thiết kế riêng cho tài sản tài chính có quản lý, thay vì chỉ triển khai smart contract trên public blockchain.

---

## Tổng quan kiến trúc hệ thống

### Sơ đồ luồng tổng quan (Flowchart)

```mermaid
flowchart TB
    subgraph Users["👥 NGƯỜI DÙNG"]
        U1[Khách hàng cá nhân]
        U2[Khách hàng doanh nghiệp]
        U3[Quản trị viên ngân hàng]
    end

    subgraph Presentation["🖥️ PRESENTATION LAYER"]
        WebApp[User Web App<br/>- Mua/Bán CD<br/>- Theo dõi lãi suất]
        AdminApp[Admin Web App<br/>- Cấu hình CD<br/>- Giám sát hệ thống]
    end

    subgraph Gateway["🚪 API GATEWAY"]
        Kong[Kong Gateway<br/>- mTLS Security<br/>- Rate Limiting<br/>- Auth & Routing]
    end

    subgraph Business["⚙️ BUSINESS LAYER"]
        CDS[CDS Management Service<br/>- Quản lý sản phẩm CD<br/>- Điều phối nghiệp vụ<br/>- Lịch trả lãi]
        Wallet[Wallet Service<br/>- Quản lý private keys<br/>- Ký giao dịch<br/>- AWS KMS]
        Relayer[Relayer Service<br/>- Gasless transactions<br/>- Submit to chain]
        Mifox[Core Banking - Mifox<br/>- Lưu ký tiền fiat<br/>- Tính toán lãi<br/>- Đối soát]
    end

    subgraph Settlement["🔗 SETTLEMENT LAYER"]
        IPFS[IPFS<br/>- Metadata storage<br/>- Document hashing]
        L1[Blockchain Layer-1<br/>- Smart contracts<br/>- State management<br/>- Event logs]
    end

    %% Luồng chính
    U1 & U2 --> WebApp
    U3 --> AdminApp
    WebApp & AdminApp --> Kong
    Kong --> CDS

    %% Business logic flow
    CDS --> Mifox
    CDS --> IPFS
    CDS --> Wallet

    Wallet --> Relayer
    Relayer --> L1
    IPFS -.Tham chiếu CID.-> L1

    %% Feedback loops
    L1 -.Events.-> CDS
    Mifox -.Xác nhận.-> CDS

    style Users fill:#e1f5ff
    style Presentation fill:#fff4e6
    style Gateway fill:#f3e5f5
    style Business fill:#e8f5e9
    style Settlement fill:#fce4ec
```

### Luồng giao dịch chi tiết - Use Case: Mua CD

```mermaid
sequenceDiagram
    actor User as 👤 Người dùng
    participant App as Web App
    participant Kong as API Gateway
    participant CDS as CDS Management
    participant Bank as Core Banking
    participant Wallet as Wallet Service
    participant KMS as AWS KMS
    participant Relayer as Relayer
    participant IPFS as IPFS
    participant L1 as Blockchain L1

    User->>App: 1. Chọn sản phẩm CD<br/>(mệnh giá, kỳ hạn)
    App->>Kong: 2. POST /cd/purchase
    Kong->>Kong: 3. Xác thực & Rate limit
    Kong->>CDS: 4. Forward request

    CDS->>Bank: 5. Verify tài khoản<br/>& số dư khả dụng
    Bank-->>CDS: 6. Xác nhận OK

    CDS->>Bank: 7. Khóa số tiền<br/>(reserve funds)
    Bank-->>CDS: 8. Transaction ID

    CDS->>IPFS: 9. Upload metadata<br/>(terms, rate, docs)
    IPFS-->>CDS: 10. Return CID

    CDS->>Wallet: 11. Request signature<br/>for mint transaction
    Wallet->>KMS: 12. Sign with user key
    KMS-->>Wallet: 13. Signature
    Wallet-->>CDS: 14. Signed payload

    CDS->>Relayer: 15. Submit transaction<br/>(with IPFS CID)
    Relayer->>L1: 16. Broadcast TX<br/>(Relayer pays gas)
    L1-->>Relayer: 17. TX receipt
    Relayer-->>CDS: 18. On-chain confirmation

    alt Blockchain TX Success
        CDS->>Bank: 19. Finalize deposit<br/>(chuyển tiền vào CD)
        Bank-->>CDS: 20. Settlement complete

        CDS-->>App: 21. Success response<br/>(CD ID, blockchain TX)
        App-->>User: 22. Hiển thị CD<br/>đã mua thành công

        Note over L1: Event: CDIssued<br/>(user, amount, CID, maturity)
    else Blockchain TX Failed
        CDS->>Bank: 19b. Rollback - Unfreeze funds
        Bank-->>CDS: 20b. Funds released

        CDS-->>App: 21b. Error response<br/>(TX failed, funds returned)
        App-->>User: 22b. Thông báo lỗi<br/>& retry option

        Note over CDS: Log error & retry queue
    end
```

### Luồng đáo hạn CD

```mermaid
sequenceDiagram
    participant Scheduler as Cron Scheduler
    participant CDS as CDS Management
    participant Bank as Core Banking
    participant Wallet as Wallet Service
    participant Relayer as Relayer
    participant L1 as Blockchain L1
    actor User as 👤 Người dùng

    Scheduler->>CDS: 1. Daily check<br/>matured CDs
    CDS->>L1: 2. Query on-chain<br/>CD state
    L1-->>CDS: 3. List of active CDs

    CDS->>CDS: 4. Filter CDs<br/>maturity_date <= today

    loop For each matured CD
        CDS->>Bank: 5. Calculate final<br/>amount (principal + interest)
        Bank-->>CDS: 6. Settlement amount

        CDS->>Wallet: 7. Sign maturity TX
        Wallet-->>CDS: 8. Signature

        CDS->>Relayer: 9. Submit maturity TX
        Relayer->>L1: 10. Update state<br/>ACTIVE → MATURED
        L1-->>Relayer: 11. Confirmation

        alt On-chain TX Success
            CDS->>Bank: 12. Release funds<br/>to user account

            alt Bank Release Success
                Bank-->>User: 13. Credit account
                Note over L1: Event: CDMatured<br/>(user, amount, interest)
                CDS->>User: 14. Notification<br/>(email/push: Success)
            else Bank Release Failed
                CDS->>CDS: 15. Add to retry queue
                CDS->>Admin: 16. Alert: Manual intervention needed
                Note over CDS: Reconciliation required
            end
        else On-chain TX Failed
            CDS->>CDS: 12b. Retry transaction<br/>(max 3 attempts)
            CDS->>Admin: 13b. Alert if retry exhausted
            Note over CDS: Add to failed queue<br/>for manual review
        end
    end
```

### Luồng chuyển nhượng CD (Secondary Market)

```mermaid
flowchart LR
    subgraph Seller["🔴 NGƯỜI BÁN"]
        S1[List CD for sale]
        S2[Approve transfer]
        S3[Receive payment]
    end

    subgraph Platform["📊 MARKETPLACE"]
        M1{Verify CD<br/>ownership &<br/>Seller KYC}
        M2{Verify<br/>Buyer KYC}
        M3[Create escrow]
        M4[Match buyer/seller]
        M5[Settlement]
    end

    subgraph Buyer["🟢 NGƯỜI MUA"]
        B1[Browse CDs]
        B2[Submit KYC]
        B3[Place order]
        B4[Transfer funds]
        B5[Receive CD]
    end

    subgraph Blockchain["⛓️ BLOCKCHAIN"]
        BC1[Lock CD in<br/>escrow contract]
        BC2[Verify payment]
        BC3[Transfer ownership]
        BC4[Update registry]
    end

    S1 --> M1
    M1 -->|Valid| M3

    B1 --> B2
    B2 --> M2
    M2 -->|KYC Approved| B3
    B3 --> M4
    M3 --> M4
    M4 --> BC1

    B4 --> BC2
    BC2 -->|Confirmed| BC3

    S2 -.Signature.-> BC3
    BC3 --> BC4
    BC4 --> S3
    BC4 --> B5

    M1 -->|Invalid| S1
    M2 -->|KYC Failed| B1

    style Seller fill:#ffebee
    style Buyer fill:#e8f5e9
    style Platform fill:#fff3e0
    style Blockchain fill:#e3f2fd
```

### Kiến trúc bảo mật đa lớp

```mermaid
graph TB
    subgraph Internet["🌐 INTERNET"]
        Client[Client Apps]
    end

    subgraph DMZ["🛡️ DMZ - FIREWALL"]
        WAF[Web Application<br/>Firewall]
        DDoS[DDoS Protection]
    end

    subgraph AppZone["🔒 APPLICATION ZONE"]
        LB[Load Balancer<br/>+ mTLS]
        Kong[API Gateway<br/>+ JWT Auth]
    end

    subgraph PrivateZone["🔐 PRIVATE ZONE"]
        Services[Microservices<br/>- CDS<br/>- Wallet<br/>- Relayer]
        DB[(Database<br/>Encrypted)]
    end

    subgraph SecureZone["🏦 SECURE ZONE - HSM"]
        KMS[AWS KMS<br/>Private Keys]
        Secrets[Secret Manager<br/>API Keys]
    end

    subgraph Monitoring["📊 MONITORING & AUDIT"]
        Logs[Centralized Logging<br/>ELK Stack / CloudWatch]
        Metrics[Metrics & Alerting<br/>Prometheus / Grafana]
        Audit[Audit Trail<br/>Immutable Logs]
    end

    subgraph External["🔗 EXTERNAL SYSTEMS"]
        Bank[Core Banking<br/>VPN/Dedicated Line]
        Chain[Blockchain<br/>Public Network]
    end

    Client --> WAF
    WAF --> DDoS
    DDoS --> LB
    LB --> Kong
    Kong --> Services
    Services --> DB
    Services -.Secure API.-> KMS
    Services -.Secure API.-> Secrets
    Services --> Bank
    Services --> Chain

    %% Monitoring connections
    WAF -.Logs.-> Logs
    Kong -.Logs & Metrics.-> Logs
    Kong -.Metrics.-> Metrics
    Services -.Logs & Metrics.-> Logs
    Services -.Metrics.-> Metrics
    Services -.Audit Events.-> Audit
    Chain -.Events.-> Audit

    style DMZ fill:#ffcdd2
    style AppZone fill:#fff9c4
    style PrivateZone fill:#c8e6c9
    style SecureZone fill:#b39ddb
    style Monitoring fill:#e1bee7
    style External fill:#b0bec5
```

### Cấu trúc 3 tầng

Kiến trúc được chia tách rõ ràng thành ba lớp:

```
┌─────────────────────────────────────────────────────────────┐
│              1️⃣ PRESENTATION LAYER                          │
│        (Trải nghiệm người dùng)                             │
│                                                             │
│  ┌──────────────────┐        ┌──────────────────┐          │
│  │  User Web App    │        │  Admin Web App   │          │
│  └──────────────────┘        └──────────────────┘          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              2️⃣ BUSINESS LAYER                              │
│        (Nghiệp vụ ngân hàng & điều phối)                    │
│                                                             │
│            ┌──────────────────────┐                         │
│            │   API Gateway (Kong) │                         │
│            └──────────────────────┘                         │
│                       │                                     │
│         ┌─────────────┼─────────────┐                       │
│         ▼             ▼             ▼                       │
│  ┌─────────────┐ ┌──────────┐ ┌─────────────┐             │
│  │    CDS      │ │  Wallet  │ │   Relayer   │             │
│  │ Management  │ │  Service │ │   Service   │             │
│  └─────────────┘ └──────────┘ └─────────────┘             │
│         │            │              │                       │
│         ▼            ▼              │                       │
│  ┌─────────────┐ ┌──────────┐     │                       │
│  │   Mifox     │ │ AWS KMS  │     │                       │
│  │ (Core Bank) │ │          │     │                       │
│  └─────────────┘ └──────────┘     │                       │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              3️⃣ SETTLEMENT LAYER                            │
│        (Settlement và lưu trữ on-chain)                     │
│                                                             │
│  ┌──────────────┐              ┌────────────────┐          │
│  │     IPFS     │              │  Blockchain    │          │
│  │  (Metadata)  │◄────────────►│   Layer-1      │          │
│  └──────────────┘              │ (State & Logic)│          │
│                                └────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### Luồng hoạt động chính

#### 1. Presentation Layer (Tầng trình bày)

Người dùng và quản trị viên thao tác qua **User Web App** và **Admin Web App**, tất cả request đều đi qua **API Gateway (Kong)** – điểm truy cập duy nhất.

**User Web App:**
- Đăng ký mua CD
- Theo dõi lãi suất
- Xem lịch trả lãi
- Kiểm tra trạng thái đáo hạn

**Admin Web App:**
- Cấu hình sản phẩm CD
- Thiết lập kỳ hạn và lãi suất
- Giám sát hệ thống
- Quản lý quy tắc vận hành

#### 2. Business Layer (Tầng nghiệp vụ)

**API Gateway (Kong):**
- Entry point duy nhất cho toàn hệ thống
- Routing, authentication, rate limiting
- mTLS security, logging & monitoring

**CDS Management Service:**

Logic nghiệp vụ CD được xử lý tại **CDS Management Service**, tích hợp trực tiếp với **Core Banking Service (Mifox)**.

Chức năng:
- Quản lý sản phẩm CD và CD instances
- Thiết lập lịch trả lãi và đáo hạn
- Điều phối giữa Core Banking, IPFS và Blockchain
- Kích hoạt các hành động on-chain

**Wallet Service + AWS KMS:**
- Quản lý private keys an toàn
- Ký giao dịch theo chuẩn EIP-712
- Không lộ key ra ngoài hệ thống

**Relayer Service:**
- Chi trả phí giao dịch (gasless UX)
- Thu thập chữ ký và submit lên chain
- Theo dõi trạng thái và retry

**Core Banking (Mifox):**
- Nguồn dữ liệu tài chính gốc
- Ghi nhận tiền gửi bảo chứng
- Tính toán lãi suất và số tiền đáo hạn
- Xác nhận đối soát trước khi on-chain

#### 3. Settlement Layer (Tầng thanh toán)

**IPFS (Off-chain Metadata):**

- Metadata CD được lưu off-chain trên **IPFS**:
  - Mệnh giá
  - Lãi suất
  - Kỳ hạn
  - Đơn vị phát hành
  - Hash tài liệu pháp lý

**Blockchain Layer-1 (On-chain State):**

- Trạng thái, vòng đời và logic bất biến được lưu on-chain trên **Layer-1**:
  - Smart contract quản lý CD
  - Vòng đời CD (ISSUED → ACTIVE → MATURED → REDEEMED)
  - Tham chiếu metadata IPFS (CID/hash)
  - Event log phục vụ audit

### Luồng giao dịch end-to-end

```
User Action → Web App → API Gateway → CDS Management
                                            ↓
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                         Core Banking    IPFS        Wallet Service
                         (Verify $)    (Store data)   (Sign TX)
                              │             │             │
                              └─────────────┼─────────────┘
                                            ▼
                                      Relayer Service
                                      (Pay gas & submit)
                                            │
                                            ▼
                                    Blockchain Layer-1
                                    (Finalize & emit events)
```

### Điểm nổi bật

Giao dịch on-chain được:

- ✅ Ký an toàn thông qua **User Wallet Service** sử dụng AWS KMS,
- ✅ **Relayer Service** chi trả phí giao dịch, giúp người dùng có trải nghiệm gasless,
- ✅ Đảm bảo bảo chứng 1:1 giữa token CD và tiền gửi thực tế trong Core Banking,
- ✅ Minh bạch, có thể audit thông qua event log on-chain.

---

## Các thành phần chính

### 1️⃣ API Gateway (Kong)

- Entrypoint duy nhất cho toàn hệ thống,
- Định tuyến request đến các service nội bộ,
- Áp dụng mTLS, rate-limit, xác thực và logging,
- Đáp ứng tiêu chuẩn bảo mật cấp doanh nghiệp.

---

### 2️⃣ User Web App & Admin Web App

- **User Web App**: đăng ký mua CD, theo dõi lãi suất, lịch trả lãi và trạng thái đáo hạn.
- **Admin Web App**: cấu hình sản phẩm CD, kỳ hạn, lãi suất, quy tắc vận hành và giám sát hệ thống.

Người dùng không cần hiểu blockchain để sử dụng hệ thống.

---

### 3️⃣ CDS Management Service (Lớp nghiệp vụ trung tâm)

Đây là bộ não điều phối của hệ thống:

- Quản lý sản phẩm CD và từng CD instance,
- Thiết lập lịch trả lãi và đáo hạn,
- Điều phối giữa Core Banking, IPFS và Blockchain,
- Kích hoạt các hành động on-chain thông qua Relayer.

Mọi giao dịch on-chain đều phải phản ánh trạng thái tài chính hợp lệ off-chain.

---

### 4️⃣ Core Banking Integration – Mifox Service

Mifox đóng vai trò nguồn dữ liệu tài chính gốc:

- Ghi nhận tiền gửi bảo chứng cho mỗi CD,
- Tính toán lãi suất và số tiền đáo hạn,
- Xác nhận đối soát trước khi phát hành hoặc tất toán on-chain.

Đảm bảo bảo chứng 1:1 giữa token CD và tiền gửi thực.

#### Quy trình Reconciliation (Đối soát)

**Mục tiêu:** Đảm bảo tính nhất quán giữa on-chain state và off-chain banking records.

**1. Real-time Reconciliation (Mỗi giao dịch):**

```mermaid
sequenceDiagram
    participant CDS
    participant Bank as Core Banking
    participant L1 as Blockchain
    participant Recon as Reconciliation Service

    Note over CDS,Recon: Real-time check after each transaction

    CDS->>Bank: Query balance & CD records
    CDS->>L1: Query on-chain CD state
    CDS->>Recon: Submit for reconciliation

    Recon->>Recon: Compare:<br/>- Total supply vs Total deposits<br/>- Individual CD amounts<br/>- Interest calculations

    alt Match
        Recon-->>CDS: ✅ Reconciled
        Note over Recon: Log success
    else Mismatch
        Recon-->>CDS: ❌ Discrepancy detected
        Recon->>Admin: Alert with details
        Note over Recon: Pause new transactions<br/>pending investigation
    end
```

**2. Scheduled Reconciliation:**

| Tần suất | Scope | Action |
|---------|-------|--------|
| **Hourly** | Active CDs | Verify state consistency |
| **Daily** | Full portfolio | Complete balance check |
| **Weekly** | Interest accrual | Verify interest calculations |
| **Monthly** | Audit report | Generate compliance report |

**3. Reconciliation Checks:**

```
┌─────────────────────────────────────────────────────┐
│         RECONCILIATION CHECKPOINTS                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ✓ Total Supply Check                              │
│    ON-CHAIN: Σ(CD tokens minted)                   │
│    OFF-CHAIN: Σ(Deposits in Mifox)                 │
│    MUST MATCH: 1:1 ratio                           │
│                                                     │
│  ✓ Individual CD Verification                      │
│    For each CD ID:                                 │
│    - Principal amount matches                      │
│    - Maturity date matches                         │
│    - Interest rate matches                         │
│    - Owner address matches account                 │
│                                                     │
│  ✓ State Consistency                               │
│    CD state on-chain = CD status in bank           │
│    (ACTIVE/MATURED/REDEEMED)                       │
│                                                     │
│  ✓ Interest Calculation                            │
│    Bank-calculated interest = Smart contract calc  │
│    Tolerance: ± 0.01% (rounding differences)       │
│                                                     │
│  ✓ Transaction History                             │
│    All on-chain events have corresponding          │
│    bank transactions (and vice versa)              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**4. Discrepancy Resolution Process:**

```mermaid
flowchart TD
    A[Discrepancy Detected] --> B{Type of Mismatch}

    B -->|Amount Mismatch| C[Check transaction logs]
    B -->|State Mismatch| D[Check state transitions]
    B -->|Missing Record| E[Check pending queue]

    C --> F{Root Cause}
    D --> F
    E --> F

    F -->|Failed TX| G[Retry transaction]
    F -->|Double Processing| H[Rollback duplicate]
    F -->|Data Entry Error| I[Manual correction]
    F -->|System Bug| J[Escalate to Dev team]

    G --> K{Resolved?}
    H --> K
    I --> K
    J --> K

    K -->|Yes| L[Update both systems<br/>Mark as reconciled]
    K -->|No| M[Freeze affected CDs<br/>Manual intervention]

    L --> N[Resume operations]
    M --> O[Admin review required]
```

**5. Automated Alerts:**

```yaml
Alert Levels:
  WARNING:
    - Minor discrepancy (< 0.01%)
    - Delayed reconciliation (> 1 hour)
    - Single CD mismatch
    Action: Log, auto-retry

  ERROR:
    - Significant mismatch (> 0.1%)
    - Multiple CD mismatches
    - Failed retry (3 attempts)
    Action: Notify DevOps, pause new CDs

  CRITICAL:
    - Total supply mismatch (> 1%)
    - System-wide discrepancy
    - Security breach suspected
    Action: Freeze all operations, notify C-level
```

**6. Audit Trail:**

Tất cả reconciliation events được lưu vào:
- **Database**: Structured logs với timestamp, before/after state
- **Blockchain**: Hash của reconciliation report (immutable proof)
- **S3**: Full reconciliation reports (regulatory compliance)

**7. Monthly Audit Report:**

```
Generated automatically on 1st of each month:

📊 Reconciliation Summary Report
├─ Total CDs issued: X
├─ Total value locked: $Y
├─ Successful reconciliations: Z%
├─ Discrepancies detected: N
│  ├─ Auto-resolved: M
│  └─ Manual intervention: (N-M)
├─ Average reconciliation time: T seconds
└─ Compliance status: ✅ PASS / ❌ FAIL

Submitted to:
- Internal Audit team
- Compliance officer
- External auditor (if required)
```

**Lợi ích:**

- ✅ **Real-time detection** của discrepancies
- ✅ **Automated resolution** cho 95% cases
- ✅ **Audit-ready** reports
- ✅ **Regulatory compliance** (Basel III, SOX)
- ✅ **Transparent trail** for investigations

---

### 5️⃣ Off-chain Metadata – IPFS

Các thông tin như:

- mệnh giá,
- lãi suất,
- kỳ hạn,
- đơn vị phát hành,
- hash tài liệu pháp lý,

được lưu trữ trên **IPFS**.

Blockchain chỉ lưu CID/hash tham chiếu, đảm bảo dữ liệu bất biến, dễ kiểm toán và tối ưu chi phí on-chain.

#### Chiến lược Pinning & High Availability

**1. Multi-tier Pinning Strategy:**

```
┌─────────────────────────────────────────────────────┐
│           IPFS STORAGE ARCHITECTURE                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Tier 1: Local IPFS Nodes (Hot Storage)            │
│  ├─ Primary Node (Data Center A)                   │
│  ├─ Replica Node (Data Center B)                   │
│  └─ Replica Node (Data Center C)                   │
│                                                     │
│  Tier 2: Pinning Services (Managed)                │
│  ├─ Pinata (Primary pinning service)               │
│  ├─ Web3.Storage (Backup service)                  │
│  └─ Filebase (S3-compatible backup)                │
│                                                     │
│  Tier 3: Archive Storage (Cold Storage)            │
│  ├─ AWS S3 Glacier (Long-term archive)             │
│  └─ Filecoin (Decentralized archive)               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**2. Replication & Redundancy:**

- **Minimum 5 copies** của mỗi CID:
  - 3 copies trên local IPFS nodes (different regions)
  - 2 copies trên pinning services
- **Geographic distribution**: Nodes ở 3 châu lục khác nhau
- **Auto-replication**: Nếu node offline, tự động pin lên node khác

**3. Backup & Recovery:**

```
Daily:  Snapshot metadata → S3
Weekly: Full backup → Glacier
Monthly: Archive → Filecoin

Recovery Time Objective (RTO): < 1 hour
Recovery Point Objective (RPO): < 24 hours
```

**4. Monitoring & Alerting:**

- **Health checks** (mỗi 5 phút): Verify CID accessibility
- **Alert triggers**:
  - CID không accessible từ > 2 nodes
  - Pin count < 3
  - Gateway response time > 2s
- **Auto-healing**: Tự động re-pin nếu detect missing

**5. Gateway Strategy:**

```
User Request → CDN (CloudFlare)
              ↓
    ┌─────────┴─────────┐
    ▼                   ▼
IPFS Gateway 1    IPFS Gateway 2
(Primary)         (Failover)
    │                   │
    └─────────┬─────────┘
              ▼
        IPFS Network
```

**Lợi ích:**

- ✅ **High Availability**: 99.99% uptime
- ✅ **Disaster Recovery**: Multi-region, multi-provider
- ✅ **Cost Optimization**: Hot/Cold tiering
- ✅ **Compliance**: Immutable audit trail
- ✅ **Performance**: CDN caching cho metadata access

---

### 6️⃣ User Wallet Service – AWS KMS

- Private key được quản lý tập trung và bảo mật bằng **AWS KMS**,
- Không bao giờ lộ ra ngoài hệ thống,
- Ký giao dịch theo chuẩn EIP-712 hoặc raw transaction,
- Phù hợp với tiêu chuẩn bảo mật của tổ chức tài chính.

---

### 7️⃣ Relayer Service – Gasless Transaction

Relayer:

- Chi trả phí giao dịch on-chain,
- Thu thập chữ ký từ Wallet Service,
- Gửi giao dịch lên Layer-1,
- Theo dõi trạng thái và retry khi cần.

Người dùng có trải nghiệm tương đương ứng dụng tài chính truyền thống.

---

### 8️⃣ Blockchain Layer-1

Layer-1 lưu trữ:

- Smart contract quản lý CD,
- Vòng đời CD (state machine - xem chi tiết bên dưới),
- Tham chiếu metadata IPFS,
- Event log phục vụ audit và giám sát.

Blockchain đóng vai trò lớp settlement và kiểm toán minh bạch, không thay thế hệ thống ngân hàng.

#### Vòng đời CD (State Machine)

```mermaid
stateDiagram-v2
    [*] --> PENDING: User submit purchase

    PENDING --> ISSUED: Bank confirms & locks funds
    PENDING --> CANCELLED: User cancels / Timeout

    ISSUED --> ACTIVE: On-chain mint successful
    ISSUED --> CANCELLED: On-chain mint failed

    ACTIVE --> TRANSFERRED: Secondary market sale
    ACTIVE --> LOCKED: Used as collateral
    ACTIVE --> MATURED: Maturity date reached

    TRANSFERRED --> ACTIVE: Transfer complete
    LOCKED --> ACTIVE: Collateral released

    MATURED --> REDEEMED: User redeems / Auto-redeem

    CANCELLED --> [*]: Funds returned
    REDEEMED --> [*]: Settlement complete

    note right of PENDING
        Waiting for bank confirmation
        Timeout: 15 minutes
    end note

    note right of ACTIVE
        CD earning interest
        Can be traded or locked
    end note

    note right of LOCKED
        Cannot trade while locked
        Unlocked when loan repaid
    end note

    note right of MATURED
        Ready for redemption
        Principal + Interest available
    end note
```

**Các trạng thái:**

- **PENDING**: Đơn hàng mới, chờ xác nhận từ Core Banking
- **ISSUED**: Bank đã khóa tiền, chờ mint on-chain
- **ACTIVE**: CD đang hoạt động, tích lũy lãi suất
- **TRANSFERRED**: Đang trong quá trình chuyển nhượng trên secondary market
- **LOCKED**: Đang được sử dụng làm collateral (DeFi)
- **MATURED**: Đã đáo hạn, sẵn sàng tất toán
- **REDEEMED**: Đã tất toán, hoàn tất vòng đời
- **CANCELLED**: Đã hủy (timeout hoặc lỗi)

**Chuyển trạng thái:**

| Từ | Đến | Trigger | Condition |
|---|---|---|---|
| PENDING | ISSUED | Bank confirmation | Funds available |
| PENDING | CANCELLED | Timeout / User cancel | 15 min timeout |
| ISSUED | ACTIVE | Blockchain TX success | Mint successful |
| ISSUED | CANCELLED | Blockchain TX failed | Rollback funds |
| ACTIVE | TRANSFERRED | Secondary market list | Owner signature |
| ACTIVE | LOCKED | Collateral deposit | Smart contract lock |
| ACTIVE | MATURED | Time-based | Maturity date reached |
| TRANSFERRED | ACTIVE | Transfer complete | New owner confirmed |
| LOCKED | ACTIVE | Collateral release | Loan repaid |
| MATURED | REDEEMED | User/Auto redeem | Settlement complete |

---

## 🔐 Bảo mật & Tuân thủ

- Validator được kiểm soát (permissioned),
- Smart contract có thể audit,
- Event on-chain phục vụ giám sát và thanh tra,
- Phân tách rõ ràng giữa custody – nghiệp vụ – settlement.

---

## 🚀 Giá trị cốt lõi của kiến trúc

✔️ Thiết kế riêng cho tài sản tài chính có quản lý
✔️ Thân thiện với ngân hàng và cơ quan quản lý
✔️ Trải nghiệm người dùng đơn giản, không cần gas
✔️ Phân tách on-chain / off-chain rõ ràng
✔️ Sẵn sàng triển khai thực tế, không chỉ là ý tưởng

---

## Tổng kết

Chúng tôi xây dựng một Layer-1 ưu tiên tuân thủ, cho phép token hóa chứng chỉ tiền gửi trong khi ngân hàng vẫn kiểm soát dòng tiền, người dùng không cần trả gas và toàn bộ vòng đời được kiểm toán minh bạch trên blockchain.
