# Where do self-custody wallet providers “create/restore account” on blockchain? (Front-end vs Back-end)

## 1. “Create wallet/account” thực sự là gì?
Trong self-custody, “create” = **tạo key material ở client** (offline), không phải “đăng ký tài khoản” với blockchain.

*   **Bước 1 (Offline - trong App):** Sinh hoặc import **seed phrase / private key** trong RAM.
*   **Bước 2 (Offline - trong App):** Derive ra các dữ liệu public để sử dụng:
    *   address (địa chỉ nhận tiền)
    *   public key
    *   derivation path / xpub (tuỳ chain)
*   **Bước 3 (Local - trên thiết bị):** Lưu key material vào storage cục bộ theo dạng **đã mã hoá**.

Điểm quan trọng: các bước trên **không cần** gọi node/RPC. Vì vậy user có thể tạo ví khi offline.

*   **Code reference (Cake Wallet):**
    *   Tạo ví + sinh/lưu `walletPassword` (master key mã hoá file ví) vào Secure Storage: [wallet_creation_service.dart:L51-L71](lib/core/wallet_creation_service.dart#L51-L71)
    *   Lưu `walletPassword` qua `KeyService` (encode + write Secure Storage): [key_service.dart:L10-L23](lib/core/key_service.dart#L10-L23)
    *   Secure Storage wrapper (Keychain/EncryptedSharedPreferences): [secure_storage.dart:L18-L41](lib/core/secure_storage.dart#L18-L41)

## 2. “Restore wallet/account” thực sự là gì?
Restore không phải “tạo lại account trên blockchain”, mà là **khôi phục key ở client + sync lại trạng thái**.

*   **Bước 1 (Offline - trong App):** Nhập **seed/private key** (và passphrase nếu có) -> app **derive lại** đúng address/xpub.
*   **Bước 2 (Online - đọc dữ liệu):** App sync trạng thái bằng cách query qua:
    *   light client protocol (Electrum) cho BTC/LTC
    *   JSON-RPC node cho EVM (Ethereum, BSC, …)
    *   indexer/explorer (nếu app dùng để tăng tốc)

Restore thường **không ghi** gì lên chain. Nó chủ yếu là “đọc/scan/sync”.

*   **Sync (read-only):** dùng address/xpub/public info để lấy balance/nonce/UTXO/logs.
*   **Khi user gửi giao dịch:** app tạo raw tx, **ký bằng private key ở client**, rồi mới broadcast qua node/relayer.

*   **Code reference (Cake Wallet):**
    *   Tạo credentials từ seed/private key (restore modes): [wallet_restore_view_model.dart:L118-L315](lib/view_model/wallet_restore_view_model.dart#L118-L315)
    *   Gọi restore từ seed/keys về `WalletCreationService`: [wallet_restore_view_model.dart:L358-L364](lib/view_model/wallet_restore_view_model.dart#L358-L364)
    *   Restore từ seed + lưu `walletPassword` vào Secure Storage: [wallet_creation_service.dart:L118-L135](lib/core/wallet_creation_service.dart#L118-L135)

## 3. Front-end/Back-end làm gì?
### 3.1. Front-end (App) — phần bắt buộc trong self-custody
*   Sinh/nhập seed/private key
*   Derive key/address
*   Ký giao dịch (sign)
*   Lưu trữ key material ở thiết bị (encrypt-at-rest)

*   **Vì sao “seed/private key”?** Khi tạo ví mới, app thường **sinh seed phrase** (mnemonic) rồi derive ra các private key con theo path. Khi **import ví**, tuỳ chain, người dùng có thể nhập **seed phrase** hoặc **nhập trực tiếp private key** (phổ biến ở EVM). Cả hai đều là nguồn gốc để derive ra public key/address.
*   **Key material là gì?** Toàn bộ bí mật mật mã cần để kiểm soát ví:
    *   Seed phrase (mnemonic), passphrase (nếu có)
    *   Master/child private key (BTC/EVM), Monero spend/view keys
    *   Thông tin liên quan ví (xpub cho BTC/Electrum)
    *   Master key dùng để **mã hoá file ví** trên thiết bị (walletPassword)
*   **Encrypt-at-rest:** Master key (walletPassword) được lưu trong **Secure Storage** của hệ điều hành ([lib/core/secure_storage.dart](lib/core/secure_storage.dart)). Dữ liệu ví (seed/private keys, xpub, passphrase) được lưu trong file `.keys` **đã mã hoá** ([cw_core/lib/wallet_keys_file.dart](cw_core/lib/wallet_keys_file.dart), [cw_core/lib/encryption_file_utils.dart](cw_core/lib/encryption_file_utils.dart)).

### 3.2. Back-end — chỉ là dịch vụ phụ trợ (không được giữ key)
Tuỳ kiến trúc, backend có thể cung cấp:
*   RPC gateway (che API keys, rate-limit, caching)
*   Indexer (tăng tốc query lịch sử/balance)
*   Relayer (gasless/meta-tx) hoặc broadcaster
*   Notifications (push)

Nhưng backend **không** được:
*   nhận seed/private key của user
*   ký giao dịch thay user (trừ các mô hình “custodial” hoặc “smart contract wallet + guardian” được thiết kế riêng)

## 4. Vì sao nhiều người tưởng “phải tạo account trên blockchain”?
Tuỳ loại chain, có những “ảo giác” sau:

### 4.1. Account-based (EVM) - Ví dụ: Ethereum, BSC
Tưởng tượng giống như **Tài khoản Ngân hàng**.

*   **Trên App (Client):**
    *   App sinh/nhập **seed phrase** rồi derive ra **private key -> public key -> address `0x...`**.
    *   Địa chỉ này hợp lệ ngay lập tức về mặt toán học. User có thể gửi tiền vào đó ngay.
*   **Trên Blockchain (Server/Node):**
    *   Lúc mới tạo, Blockchain **CHƯA HỀ BIẾT** địa chỉ này là ai. Trong database (State Trie) chưa có dòng nào ghi nhận.
    *   **Khi nào nó xuất hiện?**
        1.  **Nhận tiền:** Ai đó gửi ETH vào. Blockchain tạo bản ghi: `{ Address: 0x..., Balance: X, Nonce: 0 }`.
        2.  **Gửi tiền (Deploy):** User nạp tiền vào rồi gửi đi. Giao dịch đầu tiên sẽ làm tăng `Nonce` từ 0 lên 1.
*   **Kết luận:** "Tạo account" thực chất chỉ là sinh key offline. Account chỉ thực sự được "khởi tạo" (initialized) trên state của blockchain khi có giao dịch đầu tiên đụng đến nó.

### 4.2. UTXO (BTC/LTC)
*   **Không có khái niệm "một tài khoản duy nhất" chứa số dư:** Số dư là tổng của các UTXO (Unspent Transaction Outputs) nằm rải rác trên blockchain.
*   **Cơ chế:** Khi "tạo" ví BTC, thực chất là ví bắt đầu lắng nghe trên một loạt các địa chỉ được sinh ra từ Seed Phrase.
*   **Privacy:** Để tăng tính riêng tư, mỗi khi nhận tiền, ví thường sinh ra một địa chỉ mới (nhưng vẫn thuộc về Seed Phrase đó).
*   **Restore:** Ví phải quét (scan) blockchain trên danh sách các địa chỉ con để tìm ra UTXO nào thuộc về mình. Đây là lý do restore ví Bitcoin thường lâu hơn ví Ethereum.

### 4.3. Ngoại lệ: Một số hệ cần "giao dịch kích hoạt"
Một số chain (Ripple, Stellar, Polkadot) yêu cầu ví phải có **số dư tối thiểu** (Minimum Balance) mới được coi là tồn tại trên Ledger.

*   Tuy nhiên, việc này **không thay đổi bản chất**: App vẫn sinh key offline.
*   Việc "kích hoạt" chỉ xảy ra khi **có người gửi tiền vào** (giao dịch on-chain), không phải do App gọi API "đăng ký tài khoản" với server.

## 5. Đối chiếu với Cake Wallet (điểm bám code)
Cake Wallet triển khai đúng mô hình self-custody: tạo/restore diễn ra local, sau đó mới sync với node/RPC.

### 5.1. Luồng create/restore (local)
*   Wallet creation/restore service: [lib/core/wallet_creation_service.dart](lib/core/wallet_creation_service.dart)
    *   `create(...)`: tạo walletPassword, lưu Secure Storage, gọi `WalletService.create(...)`.
    *   `restoreFromSeed(...)`: tương tự, gọi `WalletService.restoreFromSeed(...)`.
*   ViewModel tạo credentials và gọi service:
    *   Create: [lib/view_model/wallet_new_vm.dart](lib/view_model/wallet_new_vm.dart#L56-L161)
    *   Restore: [lib/view_model/wallet_restore_view_model.dart](lib/view_model/wallet_restore_view_model.dart#L118-L228)

### 5.2. Luồng “chạm blockchain” (sync/broadcast)
*   Nhóm BTC/LTC: query/broadcast qua Electrum protocol: [cw_bitcoin/lib/electrum.dart](cw_bitcoin/lib/electrum.dart)
*   Nhóm EVM: query/broadcast qua JSON-RPC: [cw_evm/lib/clients/evm_chain_client.dart](cw_evm/lib/clients/evm_chain_client.dart)

## 6. Kết luận
*   Self-custody tạo/restore account diễn ra ở **front-end (app)** vì liên quan seed/private key.
*   Backend (nếu có) chỉ hỗ trợ **RPC/indexer/relayer**, không giữ key.
*   “Trên blockchain” chỉ có state khi có **giao dịch** hoặc **nhận tài sản**; một số chain cần tx “khởi tạo/activate”, nhưng đó vẫn là tx do client ký.
