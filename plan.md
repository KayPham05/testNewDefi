# Plan triển khai DefiVault end-to-end (Frontend + Backend + Indexer + Vận hành)

> Mục tiêu: triển khai đầy đủ luồng nghiệp vụ DefiVault cho môi trường thử nghiệm Sepolia, tập trung vào sản phẩm và vận hành, không đi quá sâu vào thay đổi smart contract lõi.

## 0) Bối cảnh & mục tiêu

### 0.1 Mục tiêu chính
- Hoàn thiện luồng người dùng: **Connect Wallet -> Approve -> Preview -> Deposit/Withdraw -> Theo dõi lịch sử/ROI**.
- Xây backend/indexer để biến dữ liệu on-chain thành dữ liệu nghiệp vụ có thể dùng trên UI.
- Bổ sung quan sát hệ thống (monitoring/alert) để demo ổn định.
- Chuẩn hóa quy trình QA/UAT trước demo.

### 0.2 Phạm vi
- In scope:
  - Frontend module Vault.
  - Backend API cho Vault read model.
  - Indexer event `Deposited`, `Withdrawn`.
  - Dashboard chỉ số cơ bản TVL / user history / ROI.
  - Security ứng dụng, logging, monitoring.
- Out of scope (giai đoạn này):
  - Nâng cấp contract thành full ERC-4626.
  - Tích hợp strategy sinh yield thật (lending/AMM).

### 0.3 Đầu ra mong muốn
- 1 trang Vault chạy được end-to-end trên Sepolia.
- 1 backend có API lịch sử + thống kê.
- 1 indexer ổn định, có xử lý reorg cơ bản.
- 1 bộ test + checklist release.

---

## 1) Kiến trúc mục tiêu

### 1.1 Thành phần
- **Frontend (React/Vite)**
  - Vault Page
  - Vault hooks (read/write contract)
  - State + cache (query layer)
- **Backend (Spring Boot)**
  - Vault Controller / Service / Repository
  - API trả dữ liệu history + stats
- **Indexer/Worker**
  - Lắng nghe event DefiVault
  - Ghi DB theo block, tx, logIndex
  - Reorg handling (confirmations + rollback window)
- **Database**
  - Bảng sự kiện vault
  - Bảng snapshot thống kê (tuỳ chọn)
- **Observability**
  - Logging chuẩn
  - Metrics + health checks
  - Alert cơ bản

### 1.2 Luồng dữ liệu
1. User thao tác trên FE (approve/deposit/withdraw).
2. Tx lên chain Sepolia.
3. Indexer đọc logs DefiVault -> ghi DB.
4. Backend API tổng hợp DB + on-chain read.
5. FE lấy API/backend data để hiển thị lịch sử, ROI, TVL.

---

## 2) Kế hoạch chi tiết theo giai đoạn

## Giai đoạn A — Discovery & Thiết kế (2-3 ngày)

### A1. Chốt business rules
- Chuẩn hóa định nghĩa:
  - TVL = `totalAssets`
  - Price per share ~ `(totalAssets + offset)/(totalSupply + offset)`
  - ROI user = (assets withdraw - assets deposit ròng) / deposit ròng
- Quy tắc hiển thị rounding (Floor) để tránh hiểu nhầm số lẻ.
- Quy tắc timezone hiển thị thống nhất UTC hoặc local.

### A2. Thiết kế API contract FE-BE
- Tài liệu endpoint + response JSON + mã lỗi.
- Tài liệu pagination/sort/filter cho history.

### A3. Thiết kế DB schema
- `vault_events`
  - id, chain_id, contract_address, block_number, block_time, tx_hash, log_index,
  - user_address, action(deposit/withdraw), assets_raw, shares_raw,
  - assets_dec, shares_dec, created_at
  - unique key: (chain_id, tx_hash, log_index)
- `vault_user_daily_snapshot` (tuỳ chọn)
  - date, user, net_assets, net_shares, realized_pnl, roi

### A4. Chốt non-functional requirements
- Độ trễ indexer mục tiêu: < 30s sau block xác nhận.
- API p95 < 500ms với data demo.
- Uptime demo: >= 99% trong thời gian trình bày.

**Deliverables A:**
- `docs/vault-business-rules.md`
- `docs/vault-api-spec.md`
- `docs/vault-db-schema.md`

---

## Giai đoạn B — Frontend Vault Module (4-6 ngày)

### B1. Thiết kế UI/UX trang Vault
- Section Wallet:
  - Address, network, SKT balance, dvSKT balance
- Section Deposit:
  - Input assets, preview shares, approve status, submit
- Section Withdraw:
  - Input shares, preview assets, submit
- Section Analytics:
  - TVL, totalSupply, PPS, ROI user
- Section History:
  - Bảng giao dịch user

### B2. Implement hooks/contracts
- Hook `useVaultRead`
  - `asset()`, `totalAssets()`, `totalSupply()`, `previewDeposit()`, `previewWithdraw()`
- Hook `useVaultWrite`
  - `approveIfNeeded(amount)`
  - `deposit(amount)`
  - `withdraw(shares)`
- Chuẩn hóa số liệu bằng `bigint` + `parseUnits/formatUnits`.

### B3. UX safeguards
- Chặn double submit.
- Hiển thị trạng thái: pending/confirmed/failed.
- Cảnh báo khi chênh giữa preview và executed.
- Timeout mềm cho tx pending quá lâu.

### B4. FE Integration với backend
- Gọi history/stats API.
- Caching query + auto-refresh theo block interval.

### B5. FE tests
- Unit test formatter, conversion helpers.
- Component test cho form validation.
- Smoke e2e (deposit -> withdraw path với mock).

**Deliverables B:**
- `client/src/pages/VaultPage.tsx`
- `client/src/hooks/useVaultRead.ts`
- `client/src/hooks/useVaultWrite.ts`
- `client/src/lib/vaultApi.ts`

---

## Giai đoạn C — Backend Vault APIs (4-6 ngày)

### C1. API endpoints
- `GET /api/vault/health`
- `GET /api/vault/tvl`
- `GET /api/vault/pps`
- `GET /api/vault/history/{wallet}?page=&size=`
- `GET /api/vault/stats/{wallet}`

### C2. Service logic
- Chuẩn hóa địa chỉ wallet lowercase.
- Tính net deposit/withdraw.
- Tính ROI cơ bản từ lịch sử event.
- Hỗ trợ pagination + sort desc theo thời gian.

### C3. Security & validation
- Validate wallet format.
- Rate limit endpoint public.
- JWT guard cho endpoint nội bộ (nếu cần).

### C4. Backend tests
- Unit test service (ROI, aggregation).
- Integration test controller + repository.

**Deliverables C:**
- Controller/Service/Repository cho Vault.
- Migration DB + index cần thiết.

---

## Giai đoạn D — Indexer / Event Worker (4-7 ngày)

### D1. Event ingestion
- Theo dõi contract DefiVault tại Sepolia.
- Parse events:
  - `Deposited(address,uint256,uint256)`
  - `Withdrawn(address,uint256,uint256)`

### D2. Idempotency
- Upsert theo `(chain_id, tx_hash, log_index)`.
- Retry an toàn khi worker restart.

### D3. Reorg handling
- Chỉ finalize event sau N confirmations (ví dụ N=5).
- Lưu checkpoint block.
- Reconcile rollback window khi phát hiện fork ngắn.

### D4. Observability cho indexer
- Metrics:
  - block_lag
  - events_processed_total
  - events_failed_total
- Log structured theo txHash + block.

### D5. Job backfill
- Chạy backfill từ block deploy contract -> latest.
- So sánh count event on-chain vs DB.

**Deliverables D:**
- Worker chạy nền + cron/restart strategy.
- Tài liệu runbook indexer.

---

## Giai đoạn E — Analytics & Dashboard (2-4 ngày)

### E1. KPIs
- TVL hiện tại.
- Inflow/Outflow 24h.
- User active 24h/7d.
- Avg deposit size.

### E2. User metrics
- Tổng nạp, tổng rút, net position.
- ROI estimate.
- Lịch sử tx có filter theo action/date.

### E3. Data quality rules
- Không double-count event.
- Alert nếu event missing > ngưỡng.

**Deliverables E:**
- Dashboard cards + chart cơ bản.

---

## Giai đoạn F — QA, UAT, Release (3-5 ngày)

### F1. Test plan
- Happy path:
  - Approve -> Deposit -> Withdraw
- Edge cases:
  - amount = 0
  - insufficient balance
  - insufficient shares
  - tx dropped/replaced
- Consistency:
  - chain data vs backend aggregates

### F2. UAT scripts
- Script demo cho stakeholder:
  - User A deposit
  - Simulate yield/loss
  - User A withdraw
  - Check ROI/history

### F3. Release checklist
- Env vars đầy đủ (RPC, contract address, DB).
- Health checks pass.
- Indexer lag trong ngưỡng.
- Rollback plan có sẵn.

**Deliverables F:**
- `docs/vault-uat-checklist.md`
- `docs/vault-release-checklist.md`

---

## 3) Backlog chức năng chi tiết (ưu tiên)

### P0 (bắt buộc)
- FE Vault page + approve/deposit/withdraw + preview.
- Backend history/stats endpoints.
- Indexer ingest Deposited/Withdrawn.
- Basic monitoring + healthcheck.

### P1 (nên có)
- ROI/TVL dashboard.
- Reorg-safe confirmations + backfill job.
- Error handling UX tốt hơn (pending/replace tx).

### P2 (mở rộng)
- Slippage guard ở contract phiên bản mới.
- Full ERC-4626 compatibility.
- Strategy adapter sinh yield thật.

---

## 4) Phân công vai trò

### BA/PM
- Chốt business rules, KPI, acceptance criteria.
- Quản lý scope theo P0/P1/P2.

### FE Dev
- Xây trang Vault + hooks + UX safeguards.
- Tích hợp backend API lịch sử/stats.

### BE Dev
- Thiết kế schema + API vault.
- Logic aggregate/ROI + validation/rate-limit.

### Blockchain/Indexer Dev
- Worker ingestion + reorg handling + backfill.
- Kiểm tra data consistency với on-chain.

### QA
- Test matrix + regression.
- UAT demo script.

### DevOps
- Deploy FE/BE/DB/worker.
- Monitoring + alert + runbook.

---

## 5) Timeline đề xuất (3 sprint)

### Sprint 1 (tuần 1)
- Discovery + API/DB design.
- FE Vault basic read/write.
- Indexer PoC ingest events.

### Sprint 2 (tuần 2)
- Backend history/stats production-ready.
- FE hoàn thiện history + UX trạng thái tx.
- Indexer confirmations + idempotent upsert.

### Sprint 3 (tuần 3)
- Dashboard KPI + ROI.
- QA/UAT + hardening + release.

---

## 6) Acceptance Criteria (điều kiện nghiệm thu)

1. User có thể deposit/withdraw trên Sepolia từ UI không cần thao tác thủ công ngoài ví.
2. History của user hiển thị đúng tx đã phát sinh on-chain.
3. TVL và PPS hiển thị nhất quán với dữ liệu chain (sai số chỉ do timing block).
4. Hệ thống chịu được restart indexer mà không mất/nhân bản dữ liệu.
5. Demo end-to-end chạy ổn định trong ít nhất 60 phút liên tục.

---

## 7) Risk register & giảm thiểu

- **R1: RPC không ổn định**
  - Mitigation: fallback RPC + retry exponential backoff.
- **R2: Reorg gây sai dữ liệu event**
  - Mitigation: confirmation threshold + rollback window.
- **R3: UX mismatch preview/executed**
  - Mitigation: cảnh báo + refresh state ngay sau mined.
- **R4: Sai số số học FE**
  - Mitigation: chỉ dùng bigint, không dùng float cho on-chain amounts.
- **R5: Demo downtime**
  - Mitigation: health checks + restart policy + runbook.

---

## 8) Checklist triển khai nhanh (copy dùng ngay)

### FE
- [ ] Tạo VaultPage
- [ ] Hook useVaultRead
- [ ] Hook useVaultWrite
- [ ] Approve flow
- [ ] Deposit flow
- [ ] Withdraw flow
- [ ] Tx status UI
- [ ] History table + pagination

### BE
- [ ] Migration bảng vault_events
- [ ] API history/stats/tvl/pps
- [ ] Validation wallet
- [ ] Rate limit
- [ ] Unit/integration tests

### Indexer
- [ ] Parse 2 events
- [ ] Upsert idempotent
- [ ] Checkpoint block
- [ ] Confirmations logic
- [ ] Backfill script

### Ops
- [ ] Env config
- [ ] Health endpoints
- [ ] Metrics dashboard
- [ ] Alert rules
- [ ] Runbook on-call

---

## 9) Gợi ý phase sau khi hoàn thành plan này
- Upgrade DefiVault lên full ERC-4626.
- Thêm slippage/deadline guard on-chain.
- Tích hợp strategy thật (Aave/Compound/AMM sandbox).
- Bổ sung invariant/fuzz testing tài chính.

