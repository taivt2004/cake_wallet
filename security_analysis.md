# Phân tích Bảo mật Cake Wallet: Deep Dive & Implementation Details

Tài liệu này ghi lại cách Cake Wallet tổ chức lưu trữ dữ liệu trên thiết bị người dùng để đảm bảo an toàn, kèm “điểm bám” (file path + function) để học theo và mở rộng cho dự án.

## Mục tiêu & Phạm vi

*   **Research:** How is storage organized for security on user device
*   **Mức độ 1 (Highest):** Seed phrase đã mã hóa, các key dùng cho mã hóa (liên quan trực tiếp tài sản)
*   **Mức độ 2 (Medium):** Lịch sử giao dịch, app configuration, và dữ liệu sản phẩm tương lai (chat/exchange history)
*   **Platform:** Mobile + Desktop

## Phần 1: Mức độ 1 (Highest) - Seed Phrase & Encryption Keys

Đây là phần quan trọng nhất, chứa "chìa khóa" để truy cập tài sản của người dùng. Cake Wallet áp dụng mô hình **Non-custodial** (Người dùng tự giữ key), do đó việc bảo vệ dữ liệu này trên thiết bị là tối quan trọng.

### 1.1. Cách Cake Wallet bảo vệ Seed Phrase & Encryption Keys

*   **Seed phrase / private key không lưu plaintext trên ổ cứng.**
*   App sinh 1 **Master Key** (tên trong code: `walletPassword`) và lưu vào **Secure Storage** của OS.
*   File ví nằm trên disk (trong sandbox/app dir) dưới dạng **đã mã hóa** (thường là file `.keys`).
*   Khi cần dùng (mở ví, ký giao dịch), app **giải mã và load key lên RAM** trong phiên chạy; tắt app thì dữ liệu trên RAM mất.

---

### 1.2. Storage Map (Mức độ 1)

#### A. Quy trình 1: Sinh & Bảo vệ Master Key (`walletPassword`)

Đây là bước quan trọng nhất. `walletPassword` là chìa khóa vạn năng để mở mọi dữ liệu của ví.

1.  **Sinh Key (The Birth):**
    *   Khi tạo ví mới, App sẽ sinh ra một chuỗi ngẫu nhiên 512-bit (kèm 8-byte IV). Đây chính là `walletPassword`.
    *   **Code:** [lib/core/generate_wallet_password.dart](lib/core/generate_wallet_password.dart) sử dụng `generateKey()` trong [cw_core/lib/key.dart](cw_core/lib/key.dart).
    
2.  **Lưu Key (The Vault):**
    *   Key này **KHÔNG** được lưu vào file thường. Nó được đưa thẳng vào **Secure Storage** (Kho lưu trữ bảo mật của hệ điều hành).
    *   **iOS:** Lưu vào `Keychain` (chip bảo mật Apple).
    *   **Android:** Lưu vào `EncryptedSharedPreferences` (Keystore).
    *   **Code:** [lib/core/secure_storage.dart](lib/core/secure_storage.dart) là lớp bao bọc (Wrapper) để gọi xuống OS.

3.  **Quản lý Key (The Gatekeeper):**
    *   Lớp `KeyService` đứng giữa App và Secure Storage. App muốn lấy key phải hỏi qua ông này.
    *   **Code:** [lib/core/key_service.dart](lib/core/key_service.dart) có hàm `saveWalletPassword()` và `getWalletPassword()`.

#### B. Quy trình 2: Mã hóa & Lưu trữ Dữ liệu Ví (`.keys`)

Sau khi có Master Key, App dùng nó để khóa dữ liệu tài sản (Seed Phrase, Private Key).

1.  **Chuẩn bị Dữ liệu (The Content - Tài sản cần bảo vệ):**
    *   Dữ liệu nhạy cảm được gom lại thành một cấu trúc gọi là `WalletKeysData`. Đây chính là **"Seed Phrase đã mã hóa"** (khi ghi xuống đĩa).
    *   **Thành phần bên trong:**
        *   `mnemonic`: Seed phrase (12/24 từ).
        *   `privateKey`: Private Key (cho EVM/Bitcoin).
        *   `xPub`: Extended Public Key (cho Bitcoin/Electrum để watch-only).
        *   `passphrase`: Mật khẩu phụ (nếu có).
    *   **Code Reference:** [cw_core/lib/wallet_keys_file.dart](cw_core/lib/wallet_keys_file.dart) (Class `WalletKeysData`).

2.  **Các Key dùng cho Mã hóa (Encryption Keys):**
    *   **Master Key (`walletPassword`):** Key chính 512-bit sinh ngẫu nhiên.
        *   **Code:** `generateKey()` trong [cw_core/lib/key.dart](cw_core/lib/key.dart).
    *   **IV (Initialization Vector):** Chuỗi ngẫu nhiên 8-byte hoặc 12-byte đi kèm Master Key để đảm bảo mỗi lần mã hóa là duy nhất.
    *   **Salt:** Thêm vào để chống Rainbow Table.

3.  **Thuật toán Mã hóa (The Lock):**
    *   Sử dụng **XChaCha20Poly1305** (hiện đại, nhanh hơn AES trên mobile).
    *   Dữ liệu `WalletKeysData` -> JSON -> Encode UTF8 -> **Encrypt (dùng Master Key + IV)** -> File `.keys`.
    *   **Code Reference:** `XChaCha20EncryptionFileUtils` trong [cw_core/lib/encryption_file_utils.dart](cw_core/lib/encryption_file_utils.dart).

4.  **Lưu File (The Safe House):**
    *   Mớ dữ liệu đã mã hóa được ghi xuống đĩa cứng, vào file có đuôi `.keys`.
    *   Đường dẫn file được tính toán dựa trên Hệ điều hành (Sandbox).
    *   **Code:** [cw_core/lib/pathForWallet.dart](cw_core/lib/pathForWallet.dart) xác định vị trí đặt file.

#### C. Quy trình 3: Chi tiết từng loại Coin (Implementation)

Mỗi loại coin sẽ lưu những gì vào trong file `.keys` đó?

*   **Bitcoin/Electrum:** Lưu 3 thứ chính: `mnemonic` (12 từ), `xPub` (để soi số dư), và `passphrase` (nếu có).
    *   **Xem tại:** [cw_bitcoin/lib/electrum_wallet.dart](cw_bitcoin/lib/electrum_wallet.dart).
*   **Ethereum (EVM):** Lưu `privateKey` trực tiếp.
    *   **Xem tại:** [cw_evm/lib/evm_chain_wallet.dart](cw_evm/lib/evm_chain_wallet.dart).
*   **Monero:** Lưu phức tạp hơn (Binary Struct), gồm `spend_key` và `view_key`.

### 1.3. Luồng chạy thực tế (Mức độ 1)

#### A. Tạo ví mới (Create)

*   UI nhập tên ví và bấm tạo:
    *   [lib/src/screens/new_wallet/new_wallet_page.dart](lib/src/screens/new_wallet/new_wallet_page.dart) → `_confirmForm()` gọi `_walletNewVM.create(...)`
    *   [lib/view_model/wallet_new_vm.dart](lib/view_model/wallet_new_vm.dart) → `process(...)` gọi `walletCreationService.create(...)`
*   Sinh `walletPassword` và lưu Secure Storage:
    *   [lib/core/wallet_creation_service.dart](lib/core/wallet_creation_service.dart) → `WalletCreationService.create()` gọi `keyService.saveWalletPassword(...)`
*   Sinh seed trong RAM, dựng wallet object, rồi persist xuống disk:
    *   [cw_bitcoin/lib/bitcoin_wallet_service.dart](cw_bitcoin/lib/bitcoin_wallet_service.dart) → `BitcoinWalletService.create()`:
        *   `MnemonicBip39.generate(...)`
        *   `BitcoinWalletBase.create(...)`
        *   `wallet.save()` (điểm ghi `.keys` và các file liên quan)
        *   `wallet.init()`
*   Sau khi lưu xong mới chuyển sang màn hình seed:
    *   [lib/src/screens/seed/pre_seed_page.dart](lib/src/screens/seed/pre_seed_page.dart) → `Routes.seed`
    *   [lib/src/screens/seed/wallet_seed_page.dart](lib/src/screens/seed/wallet_seed_page.dart) (hiển thị seed)

#### B. Mở ví (Open)     

*   Lấy `walletPassword` từ Secure Storage:
    *   [lib/core/key_service.dart](lib/core/key_service.dart) → `KeyService.getWalletPassword()`
*   Đọc file `.keys` và đưa key lên RAM: 
    *   [cw_core/lib/wallet_keys_file.dart](cw_core/lib/wallet_keys_file.dart) → `WalletKeysFile.readKeysFile()`
    *   [cw_core/lib/encryption_file_utils.dart](cw_core/lib/encryption_file_utils.dart) → `EncryptionFileUtils.read(...)`
*   Init runtime data (địa chỉ, balance, history):
    *   Tùy chain, thường gọi `wallet.init()` (ví dụ Electrum wallet init sẽ load `transactionHistory.init()` và addresses)

### 1.4. Cấu trúc File Ví & Mô hình Bảo mật 3 Lớp (Deep Dive)

Để trả lời câu hỏi: *"File ví chứa những gì và được bảo vệ như thế nào?"*, hãy hình dung mô hình **Két sắt trong Két sắt**.

#### A. Mô hình Bảo vệ 3 Lớp (The 3-Layer Security Model)

1.  **Lớp 1 (User Layer):**
    *   **Bảo vệ:** Mã PIN (hoặc Biometric).
    *   **Nhiệm vụ:** Ngăn người lạ mở App trên điện thoại.
    *   **Hành động:** Khi nhập đúng PIN -> App được quyền truy cập vào *Lớp 2*.
 
2.  **Lớp 2 (System Layer - Secure Storage):**
    *   **Bảo vệ:** Keychain (iOS) / Keystore (Android).
    *   **Chứa:** **Master Key** (Wallet Password - 512 bit).
    *   **Nhiệm vụ:** Lưu giữ chìa khóa giải mã file ví. Master Key này **KHÔNG** bao giờ nằm trong file ví, mà nằm tách biệt hoàn toàn trong chip bảo mật của điện thoại.
    *   **Hành động:** App lấy Master Key -> Dùng nó để mở *Lớp 3*.

3.  **Lớp 3 (Application Layer - Encrypted Wallet File):**
    *   **Bảo vệ:** Mã hóa XChaCha20Poly1305 (dùng Master Key từ Lớp 2).
    *   **Chứa:** Toàn bộ tài sản số và bí mật của user (xem mục B dưới đây).
    *   **Vị trí:** File `.keys` nằm trên ổ cứng (Disk).

#### B. Nội dung bên trong File Ví (.keys)
Khi App dùng Master Key để giải mã file `.keys`, nó sẽ nhận được một JSON Object (đối với Bitcoin/Electrum) hoặc Binary Struct (đối với Monero) chứa các thông tin sau:

*   **Code Reference:** `WalletKeysData` trong [`wallet_keys_file.dart`](../cw_core/lib/wallet_keys_file.dart).
*   **Code Reference:** `WalletKeysData` trong [`wallet_keys_file.dart`](cw_core/lib/wallet_keys_file.dart).

*   **Mnemonic (Seed Phrase):** 12/24 từ khôi phục ví. (🔴 **CRITICAL** - Mất là mất tiền)
*   **Private Key (Spend Key):** Dùng để ký giao dịch chuyển tiền đi. (🔴 **CRITICAL**)
*   **Private View Key:** (Monero) Dùng để soi blockchain xem có tiền vào không. (🟠 HIGH)
*   **Public Address:** Địa chỉ ví để nhận tiền. (🟢 PUBLIC)
*   **Derivation Path:** Đường dẫn phái sinh key (VD: m/44'/0'/0'). (🟡 MEDIUM)
*   **Salt/IV:** Các tham số kỹ thuật dùng cho mã hóa. (🟡 MEDIUM)

**Kết luận:** File ví chính là "Két sắt" chứa Seed Phrase. Nhưng chìa khóa mở két (Master Key) lại được giấu ở một nơi khác an toàn hơn (Secure Storage), và người dùng giữ chìa khóa vào nơi đó (PIN).

---

## Phần 2: Mức độ 2 (Medium) - Dữ liệu User App

### 2.1. Lịch sử giao dịch (Transaction History)
Mặc dù là dữ liệu mức 2, Cake Wallet vẫn mã hóa nó tương đương mức 1 để đảm bảo tính riêng tư tuyệt đối (Privacy).

#### Code : `ElectrumTransactionHistory.save()`
*   **File:** [electrum_transaction_history.dart](cw_bitcoin/lib/electrum_transaction_history.dart)
*   **Flow:** Map Transaction -> JSON -> Encrypt (dùng chung Wallet Password) -> Save to Disk.
*   Entry liên quan:
    *   `cw_bitcoin/lib/electrum_transaction_history.dart` → `ElectrumTransactionHistory.save()`
    *   `cw_bitcoin/lib/electrum_transaction_history.dart` → `ElectrumTransactionHistory._read() / _load()`
    *   `cw_core/lib/encryption_file_utils.dart` → `EncryptionFileUtils.write()/read()`

### 2.2. App Configuration
Lưu trữ thông thường, không mã hóa để truy xuất nhanh.
*   **File:** [settings_store.dart](lib/store/settings_store.dart)
*   **Implementation:** Sử dụng `shared_preferences`.

### 2.3. Đề xuất & Mở rộng cho Dự án (Chat/Exchange)
Học hỏi từ Cake, chúng ta có thể thiết kế module Chat bảo mật hơn bằng cách **Derive Key** thay vì dùng chung Master Key.

#### Implementation Guide (Mã nguồn đề xuất)
*   Đề xuất mức thiết kế (không đưa code vào file):
    *   Derive “module key” từ `walletPassword` theo `context` (ví dụ: `ChatModule`, `ExchangeHistory`) để tách blast radius.
    *   Lưu trữ dữ liệu module bằng DB có encryption-at-rest (Hive/SQLite/SQLCipher), với encryption key là derived key.
    *   Không dùng trực tiếp `walletPassword` cho mọi module nếu module có vòng đời/permission khác nhau.

---

## Phần 3: Cross-Platform Strategy (Mobile & Desktop)

Để chạy được trên cả Mobile và Desktop mà vẫn bảo mật, Cake Wallet sử dụng các lớp Abstraction.

### 3.1. Secure Storage Configuration
Cấu hình này cực kỳ quan trọng để đảm bảo key không bị lộ trên Android (vấn đề fragmentation) và iOS.

#### Code Deep Dive: `DefaultSecureStorage`
*   **File:** [secure_storage.dart](lib/core/secure_storage.dart)
*   Entry liên quan:
    *   `lib/core/secure_storage.dart` → `DefaultSecureStorage` (FlutterSecureStorage iOptions/aOptions)
    *   iOS: `KeychainAccessibility.first_unlock`
    *   Android: `AndroidOptions(encryptedSharedPreferences: true)`

### 3.2. File Path Handling
Sử dụng `path_provider` để đảm bảo file luôn được lưu vào vùng an toàn (Sandbox) của từng OS.

*   **iOS/macOS:** `NSDocumentDirectory` (Sandboxed).
*   **Android:** `Context.getFilesDir()` (Private internal storage).
*   **Linux/Windows:** `XDG_DATA_HOME` hoặc `AppData`.

### 3.3. Ghi chú thực tế về path (theo code)

#### A. App dir gốc (nơi app bắt đầu lưu mọi thứ)

*   [root_dir.dart](cw_core/lib/root_dir.dart) → `getAppDir()`:
    *   Windows: `getApplicationSupportDirectory()`
    *   Linux: chọn path theo danh sách fallback + tương thích ngược, có case riêng cho Tails
    *   macOS/iOS/Android: `getApplicationDocumentsDirectory()`
*   [root_dir.dart](cw_core/lib/root_dir.dart) → `setRootDirFromEnv()`:
    *   Cho phép override bằng env `CAKE_WALLET_DIR` (dùng cho debug/portable)
*   [root_dir.dart](cw_core/lib/root_dir.dart) → `linuxSymlinkSharedPreferences()`:
    *   Trên Linux: migrate/symlink data cũ (XDG_DATA_HOME) để không “mất ví” khi đổi path
*   [root_dir.dart](cw_core/lib/root_dir.dart) → `isNonAmnesticTails`:
    *   Detect Tails + persistence để chọn đúng thư mục “Persistent”

#### B. Path cụ thể cho từng ví (wallet folder + wallet files)

*   [pathForWallet.dart](cw_core/lib/pathForWallet.dart):
    *   `pathForWalletDir(name, type)` → thư mục: `.../wallets/<type>/<name>/`
    *   `pathForWallet(name, type)` → file base path: `.../wallets/<type>/<name>/<name>`
*   [wallet_keys_file.dart](cw_core/lib/wallet_keys_file.dart):
    *   `makeKeysFilePath()` → `<basePath>.keys` (két sắt chứa seed/private keys đã mã hóa)
    *   `WalletKeysFile.readKeysFile(...)` / `saveKeysFile(...)` → đọc/ghi `.keys` (encrypt-at-rest)
*   Các file liên quan khác trong thư mục ví (tùy chain):
    *   [electrum_wallet.dart](cw_bitcoin/lib/electrum_wallet.dart) → `save()` ghi `<basePath>` (wallet snapshot/cache) + gọi `transactionHistory.save()`

---

## Tổng kết bài học (Takeaways)

1.  **Encryption Everywhere:** Không chỉ Seed Phrase, hãy mã hóa cả dữ liệu Transaction/History nếu nó chứa thông tin nhạy cảm.
2.  **Random Master Key:** Đừng dùng password người dùng làm key mã hóa. Hãy sinh Random Key (512-bit), lưu nó vào Secure Storage, và dùng Password/Biometric để bảo vệ quyền truy cập vào Secure Storage đó.
3.  **Key Derivation:** Với các module mở rộng (Chat, Social), hãy phái sinh key con từ Master Key.
4.  **Cross-Platform:** Cấu hình kỹ `AndroidOptions` cho Secure Storage, đừng dùng default settings.
