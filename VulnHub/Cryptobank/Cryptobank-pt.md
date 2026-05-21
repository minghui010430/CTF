# CryptoBank 模擬滲透測試報告


## 專案摘要

本次測試目標為 VulnHub 靶機 `CryptoBank: 1`。依目前已驗證的結果，攻擊者可先透過登入介面的 SQL injection 繞過驗證並枚舉資料庫內容，再結合站內可觀察到的帳號格式與密碼爆破取得有效憑證，進入 `/development` portal 後找到開發工具功能，最終透過命令執行取得 `www-data` shell，並完成本機權限提升至 `root`。

目前可確認的攻擊鏈如下：

1. Web 服務枚舉
2. 登入介面 SQL injection
3. 資料庫內容導出
4. 帳號格式推測與憑證爆破
5. 登入 `/development`
6. 已驗證命令執行
7. 取得 reverse shell
8. 本機權限提升至 `root`

## 測試範圍

- 測試目標：`CryptoBank: 1`
- 測試環境：本地 lab / 私有網段
- 測試目標 IP：`172.20.10.3`
- 說明：`172.20.10.3` 為測試時使用的內網 IP，屬於私有網段

## 目標資訊

- 平台：VulnHub
- 靶機名稱：`CryptoBank: 1`
- 作者：`emaragkos`
- 發布日期：`2020-04-18`
- 難度：`Intermediate`
- 虛擬機格式：`OVA`
- 作業系統：`Linux`
- 建議環境：以 `VirtualBox` 為主，`VMware` 應可運行
- 網路設定：`DHCP enabled`
- 測試目標：取得 `root flag`

## 攻擊路徑概述

本次測試顯示，目標系統可由外部 Web 入口一路擴大利用至主機層最高權限。整體攻擊路徑如下：

1. 透過資訊收集確認目標主機提供 Web 服務，並發現 `/development` 路徑。
2. 登入介面存在 SQL Injection，可繞過驗證並進一步枚舉資料庫內容。
3. 結合資料庫導出的憑證資料與公開頁面可觀察到的帳號命名規則後，成功取得 `/development` portal 的有效登入憑證。
4. `/development` 內的開發工具功能存在已驗證命令注入，可於伺服器端執行系統命令並取得 `www-data` shell。
5. 在低權限 shell 的基礎上，進一步利用 `CVE-2026-31431`（Copy Fail）完成本機權限提升至 `root`。

## 弱點整理

以下 CVSS 評分以各弱點單獨評估，不將後續利用鏈中的其他弱點效果併入同一分數。為了與弱點 4 所引用的廠商公開評分保持一致，本報告統一採用 `CVSS v3.1 Base Score`。

分級對照如下：

- `0.1 - 3.9`：低風險（Low）
- `4.0 - 6.9`：中風險（Medium）
- `7.0 - 8.9`：高風險（High）
- `9.0 - 10.0`：重大風險（Critical）

### 弱點 1：登入介面 SQL Injection

- 位置：`http://cryptobank.local/trade/`
- 受影響功能：`Secure Login`
- 對應分類：`OWASP Top 10 2021 A03: Injection`
- CWE：`CWE-89` Improper Neutralization of Special Elements used in an SQL Command (`SQL Injection`)
- 風險等級：高風險
- CVSS v3.1 Base Score：`8.2`（High）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N`

#### 弱點描述

目標系統的登入功能未妥善處理使用者輸入，導致攻擊者可於 `Username` 欄位插入 SQL payload，進而影響後端查詢邏輯並繞過原有登入驗證機制。

#### 驗證方式

本次以手動測試方式進行驗證。測試時於登入頁面的 `Username` 欄位輸入 SQL payload `"' or 1 -- -"`，即使未提供有效帳號密碼，系統仍成功登入並進入登入後頁面，畫面顯示 `Welcome, williamdelisle`，可確認此處存在 SQL Injection。

後續再透過攔截登入請求並交由 `sqlmap` 測試，確認該注入點可進一步枚舉資料庫內容，包括資料庫名稱、資料表與帳號密碼資料。

#### 影響說明

此弱點可使未經授權的使用者直接繞過登入驗證機制，存取受保護功能。進一步結合資料庫枚舉後，攻擊者可取得系統內部帳號與密碼資訊，作為後續憑證爆破、功能存取與進一步入侵的基礎，對系統機密性與整體安全性造成明顯風險。

#### 驗證結果

已成功：

- 繞過登入驗證
- 存取登入後頁面
- 枚舉資料庫名稱
- 枚舉 `cryptobank` 資料庫中的資料表
- 導出 `accounts` 資料表內容

#### 修補建議

- 將所有資料庫查詢改為使用參數化查詢或 prepared statements，避免將使用者輸入直接拼接進 SQL 語句。
- 重新檢查登入驗證流程，確認 `Username`、`Password` 等欄位在進入查詢前已經過妥善處理，且登入邏輯不會因單一欄位異常而被繞過。
- 限制資料庫帳號權限，避免 Web 應用程式帳號具備不必要的讀取或枚舉能力，以降低弱點被利用後的資料外洩範圍。
- 建立異常登入與查詢行為監控，針對明顯異常的登入請求、資料庫枚舉或大量錯誤回應進行告警與調查。

### 弱點 2：帳號格式可推測與暴力破解防護不足

- 位置：首頁 Team 區塊、`/development`
- 受影響功能：公開成員資訊、`/development` 登入入口
- 對應分類：`OWASP Top 10 2025 A07: Authentication Failures`
- CWE：`CWE-307` Improper Restriction of Excessive Authentication Attempts
- 風險等級：中風險
- CVSS v3.1 Base Score：`6.5`（Medium）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N`

#### 弱點描述

首頁 Team 區塊中的郵件圖示會暴露類似 `bill.w` 的個人連結格式，使攻擊者可據此推測站內帳號命名規則。測試過程亦確認，整理自資料庫內容的帳號與密碼資料可進一步用於 `/development` portal 的登入驗證，顯示該入口缺乏足夠的暴力破解防護機制，且不同入口之間存在帳號資訊可被串聯利用的風險。

#### 驗證方式

本次先以手動觀察方式檢查首頁 Team 區塊。將滑鼠移至人物卡片下方的郵件圖示時，瀏覽器左下角會顯示對應個人連結；截圖示例為 `http://172.20.10.3/bill.w`，其餘成員亦有類似情況。依此可推測系統可能使用短帳號格式作為登入識別。

後續將 `sqlmap` 從資料庫導出的帳號與密碼整理為字典，並額外加入首頁觀察到的短帳號格式，再以 `hydra` 對 `/development` 入口進行驗證，最終成功取得有效帳號與密碼 `julius.b / wJWm4CgV26`，並可正常登入 `/development` portal。

#### 影響說明

此問題會降低攻擊者推測有效帳號的成本，並使公開頁面資訊可直接作為登入攻擊的輔助線索。若敏感入口未實作足夠的登入嘗試限制，攻擊者即可進一步利用字典攻擊或憑證測試取得有效帳號。由於 `/development` portal 內含開發工具功能，此弱點亦可能成為後續高風險行為的入口。

#### 驗證結果

已成功：

- 從首頁觀察並推測短帳號命名規則
- 建立帳號與密碼字典
- 使用字典測試取得有效憑證
- 成功登入 `/development` portal

#### 修補建議

- 避免在公開頁面直接暴露可對應內部登入帳號的識別資訊；若需保留成員頁面連結，建議改用與登入帳號無直接關聯的公開識別碼。
- 針對 `/development` 等敏感入口實作登入速率限制、帳號鎖定、CAPTCHA 或多因素驗證，以降低暴力破解與字典攻擊成功率。
- 建立並落實密碼政策，避免使用可被推測或重複利用的憑證，並要求定期更換高風險帳號密碼。
- 監控並記錄異常登入行為，例如短時間內大量失敗嘗試、同一來源對多組帳號測試等情況。

### 弱點 3：已驗證命令注入導致系統命令執行

- 位置：`/development` 認證後功能中的 `Execute a command`
- 受影響功能：`Auth to execute system command`
- 對應分類：`OWASP Top 10 2021 A03: Injection`
- CWE：`CWE-78` Improper Neutralization of Special Elements used in an OS Command (`OS Command Injection`)
- 風險等級：重大風險
- CVSS v3.1 Base Score：`9.9`（Critical）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`

#### 弱點描述

目標系統於 `/development` 認證後提供一個 `Execute a command` 功能，進入後可見標題為 `Auth to execute system command` 的表單頁面。測試結果顯示，使用者輸入可經由 `Username` 欄位被帶入系統層執行，導致已驗證使用者可在伺服器端執行任意系統命令。

#### 驗證方式

本次先以前述取得的有效帳號與密碼登入 `/development` portal，之後在已登入狀態下枚舉出 `/tool` 頁面，並進入 `Execute a command` 功能。於表單中，先將 `whoami` 填入 `Username` 欄位作為測試指令，送出後頁面下方直接回顯 `www-data`，可確認輸入內容已在伺服器端執行。

後續進一步嘗試送出 `reverse shell` payload。直接送出原始 payload 時內容會被阻擋，因此改為先將 payload 進行 base64 編碼，再以解碼執行的形式填回 `Username` 欄位。最終於攻擊端成功接收到來自目標主機的 `www-data` reverse shell。

#### 影響說明

此弱點使已通過認證的攻擊者可直接在目標主機上執行系統命令，取得 Web 服務所屬權限並進一步操作主機環境。攻擊者可藉此讀取系統資訊、檢查檔案內容、建立反向連線、下載或上傳工具，並作為後續本機權限提升的起點。由於該弱點直接影響伺服器端執行環境，對系統機密性、完整性與可控性均造成高度風險。

#### 驗證結果

已成功：

- 於 `Username` 欄位執行 `whoami`
- 確認回顯結果為 `www-data`
- 於伺服器端執行經過編碼處理的 payload
- 成功取得 `www-data` reverse shell

#### 修補建議

- 若該功能並非正式業務需求，建議直接自正式環境移除 `Execute a command` 類型功能，避免將開發測試工具暴露於可登入介面中。
- 若確有保留需求，應改以固定功能選單或 allowlist 方式實作，禁止將使用者輸入直接傳入 shell、系統命令或外部程序。
- 將此類高風險功能限制於受信任管理網段、特定管理角色或額外驗證流程，不應僅依一般帳號登入狀態開放使用。
- 以最小權限原則執行 Web 服務，並搭配容器隔離、命令執行沙箱或系統層限制，降低命令執行成功後的影響範圍。

### 弱點 4：本機權限提升至 root

- 位置：作業系統本機環境
- 受影響元件：目標主機核心與本機權限邊界
- 相關 CVE：`CVE-2026-31431`（Copy Fail）
- 風險等級：高風險
- CVSS v3.1 Base Score：`7.8`（High）
- Vector：`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`
- 評分來源：Ubuntu Security

#### 弱點描述

在已取得 `www-data` 低權限 shell 的情況下，目標主機仍可進一步被利用以取得 `root` 權限。依本次測試結果與公開資料比對，此利用鏈對應公開揭露的 Linux kernel 本機權限提升漏洞 `CVE-2026-31431`（Copy Fail）。測試過程中確認，攻擊者可根據系統核心版本與本機環境條件調整 exploit 使用方式，最終成功突破原有權限邊界，取得完整系統控制權。

#### 驗證方式

本次先以 `uname -a` 確認目標主機核心版本為 `Linux 4.15`，並使用 `searchsploit` 尋找可能的本機權限提升方式。測試過程曾嘗試多個 exploit，其中部分方法因目標主機缺少 `gcc` 無法直接利用；另有腳本預設操作 `/usr/bin/su`，但實測該路徑不存在，因此無法直接成功。

後續以 `find / -perm -u=s -type f 2>/dev/null` 檢查系統中的 SUID 檔案，確認存在 `/bin/su`。由於其用途與 exploit 預設目標接近，因此改以 `/bin/su` 作為實際攻擊目標。另在利用過程中發現原始程式碼中依賴 `os.splice` 的部分無法於目標環境正常執行，因此將其改為透過 `ctypes` 呼叫 `libc.splice`。完成調整後，以 `python3 fail.py /bin/su` 成功取得 `root` 權限。根據公開揭露資料，`CVE-2026-31431` 屬於 Linux kernel `algif_aead` / `AF_ALG` 相關的本機權限提升漏洞，公開 PoC 預設目標即為 `/usr/bin/su`，也允許改指定其他 setuid binary。

#### 影響說明

此弱點使攻擊者在已取得低權限 shell 的前提下，可進一步突破本機權限限制並取得 `root` 權限。一旦提升成功，攻擊者即可完整控制系統，包括讀取與修改敏感檔案、建立持久化機制、竄改系統設定、停用安全控制，以及完全存取其他使用者與系統資源。此情況代表主機整體信任邊界已失效，影響範圍涵蓋系統機密性、完整性與可用性。

#### 驗證結果

已成功：

- 確認核心版本與可用利用方向
- 驗證 `/usr/bin/su` 路徑不存在
- 從 SUID 列表中找出 `/bin/su`
- 針對目標環境調整 exploit 程式碼
- 以 `python3 fail.py /bin/su` 取得 `root` 權限
- 成功讀取 `/root/flag.txt`

#### 修補建議

- 儘速將目標主機核心更新至供應商已修補 `CVE-2026-31431` 的版本，並確認相關套件與核心模組同步更新完成。
- 盤點並縮減系統中不必要的 SUID/SGID 二進位檔，降低本機權限提升漏洞被利用時可操作的高權限目標。
- 強化主機硬化措施，例如限制低權限服務帳號的本機執行能力、減少可用編譯與利用工具、加強檔案完整性監控。
- 針對 Web 服務主機與開發工具主機進行角色分離，避免一旦 Web 層被突破後即可直接進入具高權限利用條件的系統環境。

## 參考資料與附錄

### 靶機與背景資訊

- VulnHub 靶機條目：<https://www.vulnhub.com/entry/cryptobank-1,467/>

### 風險分類與弱點對應

- OWASP Top 10 2021 A03: Injection：<https://owasp.org/Top10/2021/A03_2021-Injection/>
- OWASP Top 10 2025 A07: Authentication Failures：<https://owasp.org/Top10/2025/A07_2025-Authentication_Failures/>
- CWE-89: SQL Injection：<https://cwe.mitre.org/data/definitions/89.html>
- CWE-78: OS Command Injection：<https://cwe.mitre.org/data/definitions/78.html>
- CWE-307: Improper Restriction of Excessive Authentication Attempts：<https://cwe.mitre.org/data/definitions/307.html>

### CVSS 評分參考

- FIRST CVSS v3.1 Specification：<https://www.first.org/cvss/v3.1/specification.document>
- FIRST CVSS v3.1 Calculator：<https://www.first.org/cvss/calculator/3.1>
- NVD CVSS v3 Calculator：<https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator>

### 弱點 4 與公開資訊

- Copy Fail 官方頁面：<https://copy.fail/>
- Ubuntu CVE-2026-31431 頁面：<https://ubuntu.com/security/CVE-2026-31431?pubDate=20260501>
- Ubuntu Copy Fail 公告：<https://ubuntu.com/blog/copy-fail-vulnerability-fixes-available>
- CERT-EU Security Advisory 2026-005：<https://cert.europa.eu/publications/security-advisories/2026-005/>
