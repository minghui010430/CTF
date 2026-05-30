# Earth 模擬滲透測試報告


## 專案摘要

本次測試目標為 VulnHub 靶機 `The Planets: Earth`。依目前已驗證的結果，攻擊者可先透過 HTTPS 憑證與公開測試站內容枚舉虛擬主機與測試檔案，再從公開可存取的測試筆記與訊息資料中還原出管理者憑證，成功登入 `earth.local` 的管理後台。登入後，由於後台直接提供主機命令執行能力，攻擊者可立即取得低權限 shell；後續再結合一支缺乏授權檢查的 SUID 程式 `reset_root`，重設 `root` 密碼並完成本機權限提升。

目前可確認的攻擊鏈如下：

1. 主機與虛擬主機資訊枚舉
2. 公開測試站與測試檔案發現
3. XOR 訊息還原與管理者憑證取得
4. 登入 `earth.local/admin`
5. 已驗證系統命令執行
6. 取得低權限 shell
7. 利用 `reset_root` 完成本機權限提升至 `root`

## 測試範圍

- 測試目標：`The Planets: Earth`
- 測試環境：本地 lab / 私有網段
- 測試目標 IP：`192.168.233.136`
- 說明：`192.168.233.136` 為測試時使用的內網 IP，屬於私有網段

## 目標資訊

- 平台：VulnHub
- 靶機名稱：`The Planets: Earth`
- 作者：`SirFlash`
- 發布日期：`2021-11-02`
- 系列：`The Planets`
- 難度：`Easy`
- 虛擬機格式：`VirtualBox - OVA`
- 作業系統：`Linux`
- 建議環境：以 `VirtualBox` 為主，`VMware` 可能無法正確運行
- 網路設定：`DHCP enabled`
- 測試目標：取得 `user flag` 與 `root flag`

## 攻擊路徑概述

本次測試顯示，目標系統可由對外公開的 Web 與 TLS 資訊一路擴大利用至主機層最高權限。整體攻擊路徑如下：

1. 透過 `nmap` 對目標主機進行掃描，確認主機提供 `22`、`80`、`443` 三項主要服務。
2. 從 `443` 的 TLS 憑證資訊中可直接得知 `earth.local` 與 `terratest.earth.local` 兩個虛擬主機名稱；後續在 `terratest.earth.local` 的 `robots.txt` 中又可發現 `/testingnotes.*` 線索。
3. 公開的 `testingnotes.txt` 與 `testdata.txt` 揭露了管理者帳號 `terra`、訊息加密方式與 XOR key 來源；再結合訊息頁面上的歷史密文，即可還原管理者密碼 `earthclimatechangebad4humans`。
4. 使用前述憑證成功登入 `earth.local/admin` 後，可直接存取 `Admin Command Tool` 並於主機端執行系統命令。
5. 在低權限 shell 的基礎上，進一步發現並分析 SUID 程式 `reset_root`，確認其僅透過幾個固定路徑的觸發檔案判斷是否重設 `root` 密碼，缺乏對呼叫者身份的授權檢查。
6. 建立對應觸發檔案後執行 `reset_root`，即可將 `root` 密碼重設為 `Earth`，最終成功切換為 `root` 並讀取 `root_flag.txt`。

## 弱點整理

以下 CVSS 評分以各弱點單獨評估，不將後續利用鏈中的其他弱點效果併入同一分數。為了與既有報告格式一致，本報告統一採用 `CVSS v3.1 Base Score`。

分級對照如下：

- `0.1 - 3.9`：低風險（Low）
- `4.0 - 6.9`：中風險（Medium）
- `7.0 - 8.9`：高風險（High）
- `9.0 - 10.0`：重大風險（Critical）

### 弱點 1：公開服務洩露虛擬主機與測試端點資訊

- 位置：`443` TLS 憑證、`terratest.earth.local/robots.txt`
- 受影響功能：主機識別資訊、測試端點暴露
- 對應分類：`OWASP Top 10 2021 A05: Security Misconfiguration`
- CWE：`CWE-538` Insertion of Sensitive Information into Externally-Accessible File or Directory
- 風險等級：中風險
- CVSS v3.1 Base Score：`5.3`（Medium）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N`

#### 弱點描述

目標主機的 HTTPS 服務會在 TLS 憑證中直接暴露虛擬主機名稱 `earth.local` 與 `terratest.earth.local`，而公開可存取的 `robots.txt` 又進一步揭露 `/testingnotes.*` 這類測試用端點規則。這些資訊雖不直接構成登入繞過，但能顯著降低攻擊者發現隱藏站點、測試功能與後續敏感內容的成本。

#### 驗證方式

本次以 `nmap 192.168.233.136 -sV -A` 掃描目標時，在 `443` 服務的 TLS 資訊中直接看到 `earth.local` 與 `terratest.earth.local`。後續將兩個名稱加入本機 `hosts` 後，即可正常存取對應網站。再對 `https://terratest.earth.local` 進行目錄枚舉，並檢視其 `robots.txt`，可以發現 `Disallow: /testingnotes.*` 規則，成功導引出後續可利用的測試筆記檔案。

#### 影響說明

此弱點會讓未授權攻擊者更容易辨識內部或測試用虛擬主機、隱藏站點與非預期暴露的端點，縮短枚舉時間並提高後續攻擊效率。若站點中同時存在測試資訊、敏感檔案或弱保護功能，這類資訊洩露往往可直接成為完整攻擊鏈的起點。

#### 驗證結果

已成功：

- 從 TLS 憑證資訊辨識虛擬主機名稱
- 成功存取 `earth.local` 與 `terratest.earth.local`
- 從 `robots.txt` 發現 `/testingnotes.*` 線索
- 依此找到公開可讀的測試筆記檔案

#### 修補建議

- 重新檢查對外 TLS 憑證是否包含不必要的內部、測試或替代主機名稱，避免將未預期資訊暴露於外部服務中。
- 不要依賴 `robots.txt` 隱藏敏感或測試端點；對不應對外公開的路徑應以正確的存取控制保護，必要時完全自公開環境移除。
- 測試站與正式站應分離部署，避免測試端點與正式管理入口同時暴露於相同網段與服務面。
- 建立對外服務盤點與定期檢查機制，確認憑證、虛擬主機與公開檔案不會持續洩露環境資訊。

### 弱點 2：公開測試檔案與可逆加密流程導致管理者憑證外洩

- 位置：`terratest.earth.local/testingnotes.txt`、`terratest.earth.local/testdata.txt`、訊息頁面歷史密文
- 受影響功能：管理者憑證保護、訊息加密機制
- 對應分類：`OWASP Top 10 2021 A02: Cryptographic Failures`
- CWE：`CWE-257` Storing Passwords in a Recoverable Format
- 風險等級：高風險
- CVSS v3.1 Base Score：`8.2`（High）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N`

#### 弱點描述

目標系統的公開測試站直接暴露加密方式、測試資料與管理者帳號資訊，且歷史訊息內容僅以可逆的 XOR 形式處理。由於 key 來源與密文皆可自公開頁面取得，攻擊者可在未授權的情況下還原出有效管理者密碼，進而接管後台帳號。

#### 驗證方式

本次先檢視 `testingnotes.txt`，確認其中明確寫出管理者帳號為 `terra`，並指出 `testdata.txt` 可作為加密測試用資料。接著讀取 `testdata.txt`，取得 XOR key 來源，再將訊息頁面中的歷史十六進位密文交由 `CyberChef` 做 `From Hex` 與 XOR 還原。最終成功從密文中整理出管理者密碼 `earthclimatechangebad4humans`，並使用 `terra / earthclimatechangebad4humans` 成功登入 `earth.local/admin`。

#### 影響說明

此弱點使未授權攻擊者可直接還原出有效管理者憑證，而不需要暴力破解或其他額外前置條件。一旦取得管理者帳號，攻擊者即可存取後台功能、執行高風險管理操作，並進一步擴大對應用程式與主機環境的控制範圍。

#### 驗證結果

已成功：

- 從公開測試站取得管理者帳號名稱
- 取得可用於還原訊息的 XOR key 資料
- 從歷史密文中還原出有效管理者密碼
- 以 `terra` 身份成功登入管理後台

#### 修補建議

- 立即移除所有公開可存取的測試筆記、測試資料與任何與正式帳號密碼保護機制相關的開發資訊。
- 不得以可逆加密、簡單混淆或前端可還原方式保護密碼或等價敏感資訊；密碼應以不可逆雜湊形式儲存於後端。
- 對已暴露之管理者帳號進行密碼輪替，並檢查是否存在相同或相似密碼被重用於其他系統。
- 對管理帳號啟用更嚴格的保護措施，例如多因素驗證、來源限制與高風險登入監控。

### 弱點 3：管理後台直接提供主機命令執行能力

- 位置：`http://earth.local/admin`
- 受影響功能：`Admin Command Tool`
- 對應分類：`OWASP Top 10 2021 A05: Security Misconfiguration`
- CWE：`CWE-284` Improper Access Control
- 風險等級：重大風險
- CVSS v3.1 Base Score：`9.9`（Critical）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`

#### 弱點描述

目標系統的管理後台直接提供 `Admin Command Tool`，允許後台使用者在主機端執行任意 CLI 命令。這類功能本質上等同於 Web 入口的遠端命令執行介面；若管理帳號一旦外洩或管理入口缺乏額外保護，攻擊者即可立即將應用層存取提升為作業系統層級的主機操作能力。

#### 驗證方式

本次以前述取得的 `terra` 憑證登入 `earth.local/admin` 後，於 `Admin Command Tool` 內輸入 `whoami`，頁面成功回顯 `apache`，可確認命令確實在目標主機上執行。後續再透過同一功能送出經過 base64 包裝的反向 shell payload，成功在攻擊端取得來自目標主機的 shell，並以 Python 升級成互動式 shell。

#### 影響說明

此弱點使持有有效後台帳號的攻擊者能直接在目標主機上執行系統命令、探索檔案系統、讀取旗標檔案、建立反向連線、傳輸工具，以及作為進一步本機權限提升的起點。由於影響已從 Web 後台延伸至主機作業系統，對機密性、完整性與可用性均具有極高風險。

#### 驗證結果

已成功：

- 於後台執行 `whoami`
- 確認回顯結果為 `apache`
- 送出 payload 並取得低權限 shell
- 以互動式 shell 持續操作主機環境

#### 修補建議

- 若該功能並非絕對必要，建議直接自正式環境移除 `Admin Command Tool` 類型功能。
- 若確需保留，應將其限制於受信任管理網段、專屬管理帳號與額外驗證流程，不應單靠一般 Web 後台登入保護。
- 不應讓後台使用者直接輸入任意系統命令；若有維運需求，應改為固定選單、受控操作或 allowlist 型式的管理介面。
- 強化後台審計紀錄與高風險操作告警，針對命令執行、檔案讀取、外連線與反向 shell 特徵行為進行監控。

### 弱點 4：SUID 程式 `reset_root` 缺乏授權檢查，可直接重設 root 密碼

- 位置：`/usr/bin/reset_root`
- 受影響元件：本機 SUID 密碼重設工具
- 相關 CVE：未對應公開 CVE，屬於目標主機自訂程式邏輯缺陷
- 風險等級：高風險
- CVSS v3.1 Base Score：`7.8`（High）
- Vector：`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`

#### 弱點描述

目標主機上的 SUID 程式 `reset_root` 會以高權限身份執行，並在檢查幾個固定路徑的檔案存在後，直接重設 `root` 密碼。然而程式本身未對執行者身份做適當授權檢查，也未限制低權限使用者建立這些觸發檔案，導致任何已取得本機 shell 的使用者都能藉此將 `root` 密碼重設為已知值。

#### 驗證方式

本次先以 `find / -perm -u=s -type f 2>/dev/null` 發現 `/usr/bin/reset_root`。接著將該程式傳回攻擊端進行分析，並以 `ltrace` 觀察其執行邏輯。結果顯示它會檢查下列三個檔案是否存在：

- `/dev/shm/kHgTFI5G`
- `/dev/shm/Zw7bV9U5`
- `/tmp/kcM0Wewe`

在目標主機上建立這三個檔案後再執行 `reset_root`，程式便回應 `RESET TRIGGERS ARE PRESENT, RESETTING ROOT PASSWORD TO: Earth`。後續以 `su root` 並輸入密碼 `Earth`，即可成功切換為 `root`，並讀取 `root_flag.txt`。

#### 影響說明

此弱點使攻擊者在已取得低權限本機存取的前提下，可直接重設 `root` 密碼並取得完整系統控制權。一旦提升成功，攻擊者即可存取與修改所有系統資料、建立持久化機制、變更安全設定、讀取其他使用者資訊，並完全破壞主機原有的權限邊界。

#### 驗證結果

已成功：

- 發現可疑的 SUID 程式 `reset_root`
- 透過 `ltrace` 還原其觸發條件
- 建立觸發檔案並成功重設 `root` 密碼
- 以新密碼切換為 `root`
- 成功讀取 `/root/root_flag.txt`

#### 修補建議

- 立即移除或停用 `reset_root` 這類不必要的 SUID 工具，並重新檢查所有 SUID/SGID 程式是否具備明確且安全的授權邏輯。
- 若確有密碼重設需求，應改由受控管理流程、受信任管理者身份與嚴格授權檢查執行，不應透過可由一般使用者觸發的固定檔案條件判斷。
- 強化 `/tmp`、`/dev/shm` 等可寫目錄的使用策略，避免高權限程式將安全決策建立在一般使用者可控制的暫存檔存在與否上。
- 對主機進行定期的 SUID 稽核與二進位檔審查，及早發現類似的高權限邏輯缺陷。

## 參考資料與附錄

### 靶機與背景資訊

- VulnHub 靶機條目：<https://www.vulnhub.com/entry/the-planets-earth,755/>

### 風險分類與弱點對應

- OWASP Top 10 2021 A02: Cryptographic Failures：<https://owasp.org/Top10/2021/A02_2021-Cryptographic_Failures/>
- OWASP Top 10 2021 A05: Security Misconfiguration：<https://owasp.org/Top10/2021/A05_2021-Security_Misconfiguration/>
- OWASP Top 10 2021 A01: Broken Access Control：<https://owasp.org/Top10/en/A01_2021-Broken_Access_Control/>
- CWE-538: Insertion of Sensitive Information into Externally-Accessible File or Directory：<https://cwe.mitre.org/data/definitions/538.html>
- CWE-257: Storing Passwords in a Recoverable Format：<https://cwe.mitre.org/data/definitions/257.html>
- CWE-284: Improper Access Control：<https://cwe.mitre.org/data/definitions/284.html>
- CWE-862: Missing Authorization：<https://cwe.mitre.org/data/definitions/862.html>

### CVSS 評分參考

- FIRST CVSS v3.1 Specification：<https://www.first.org/cvss/v3.1/specification-document>
- FIRST CVSS v3.1 Calculator：<https://www.first.org/cvss/calculator/3.1>
- NVD CVSS v3 Calculator：<https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator>

### 相關公開資訊

- VulnHub 系列頁：<https://www.vulnhub.com/series/the-planets,362/>
