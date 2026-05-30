# Breakout 模擬滲透測試報告


## 專案摘要

本次測試目標為 VulnHub 靶機 `Empire: Breakout`。依目前已驗證的結果，攻擊者可先透過 Web 與 SMB 服務枚舉取得有效使用者資訊，再從首頁原始碼中取得以 Brainfuck 形式隱藏、但可逆向還原的密碼內容，進而成功登入 `Usermin`。登入後，因 `Usermin` 對一般使用者暴露 `Command Shell` 高風險功能，攻擊者可直接在目標主機上執行系統命令並建立 shell；後續再結合 Linux kernel `Dirty Pipe` 弱點，最終完成本機權限提升至 `root`。

目前可確認的攻擊鏈如下：

1. Web / SMB 服務枚舉
2. 未經驗證的使用者資訊枚舉
3. 首頁原始碼中的可逆密碼資訊還原
4. 使用有效憑證登入 `Usermin`
5. 透過 `Command Shell` 取得系統命令執行能力
6. 建立低權限 shell
7. 利用 `CVE-2022-0847` 完成本機權限提升至 `root`

## 測試範圍

- 測試目標：`Empire: Breakout`
- 測試環境：本地 lab / 私有網段
- 測試目標 IP：`172.20.10.3`
- 說明：`172.20.10.3` 為測試時使用的內網 IP，屬於私有網段

## 目標資訊

- 平台：VulnHub
- 靶機名稱：`Empire: Breakout`
- 作者：`icex64 & Empire Cybersecurity`
- 發布日期：`2021-10-21`
- 系列：`Empire`
- 難度：`Easy`
- 虛擬機格式：`VirtualBox - OVA`
- 作業系統：`Linux`
- 網路設定：`DHCP enabled`
- 測試目標：取得 `user flag` 與 `root flag`

## 攻擊路徑概述

本次測試顯示，目標系統可由外部可見服務一路擴大利用至主機層最高權限。整體攻擊路徑如下：

1. 透過 `nmap` 與 `enum4linux` 確認目標主機提供 Web、SMB、`Webmin` 與 `Usermin` 服務，並從 SMB 枚舉中取得有效使用者 `cyber`。
2. 檢視 `80` port 首頁原始碼後，發現一段以 Brainfuck 表示的密碼資訊；將其解碼後得到可直接使用的密碼 `.2uqPEfj3D<P'a-3`。
3. 使用 `cyber` 與該密碼成功登入 `20000` port 的 `Usermin`，確認可進入已驗證功能頁面。
4. 登入後可直接存取 `Login > Command Shell`，代表一般使用者被授予可於主機端執行系統命令的高風險功能。
5. 利用該功能建立 shell，並確認目標核心版本為 `5.10.0-9-amd64`。
6. 後續利用 `Dirty Pipe`（`CVE-2022-0847`）成功取得 `root` 權限，完成最終目標。

## 弱點整理

以下 CVSS 評分以各弱點單獨評估，不將後續利用鏈中的其他弱點效果併入同一分數。為與公開漏洞資訊一致，本報告統一採用 `CVSS v3.1 Base Score`。

分級對照如下：

- `0.1 - 3.9`：低風險（Low）
- `4.0 - 6.9`：中風險（Medium）
- `7.0 - 8.9`：高風險（High）
- `9.0 - 10.0`：重大風險（Critical）

### 弱點 1：未經驗證的 SMB 使用者資訊枚舉

- 位置：`SMB` / `NetBIOS` 服務（`139` / `445`）
- 受影響功能：帳號資訊與主機資訊暴露
- 對應分類：`OWASP Top 10 2021 A01: Broken Access Control`
- CWE：`CWE-200` Exposure of Sensitive Information to an Unauthorized Actor
- 風險等級：中風險
- CVSS v3.1 Base Score：`5.3`（Medium）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N`

#### 弱點描述

目標主機的 `SMB` 服務可在未提供有效認證的情況下回應帳號相關資訊，導致外部攻擊者能夠枚舉出可用使用者名稱。此類資訊本身雖不直接構成完整入侵，但會顯著降低後續憑證測試與登入攻擊的成本。

#### 驗證方式

本次使用 `enum4linux 172.20.10.3 -a` 對 `SMB` 服務進行枚舉。測試結果中可辨識出有效使用者 `cyber`，且過程中未先取得任何已知帳號密碼或其他高權限存取條件，顯示相關資訊對未經授權對象可見。

#### 影響說明

帳號資訊暴露會降低攻擊者進行密碼猜測、憑證重用與服務登入測試的難度。當系統同時存在其他弱點，例如密碼外洩、弱密碼或過度授權功能時，這類枚舉結果可直接成為攻擊鏈的起點。

#### 驗證結果

已成功：

- 枚舉出目標主機開啟的 `SMB` 服務
- 從枚舉結果中辨識有效使用者 `cyber`
- 將該帳號用於後續登入測試

#### 修補建議

- 限制 `SMB` / `NetBIOS` 服務對未授權來源回應過多主機與帳號資訊。
- 關閉不必要的匿名枚舉能力，並檢查目前 `SMB` 版本與分享設定是否暴露額外識別資訊。
- 若該服務並非必要對外提供，建議限制於管理網段或內部可信任來源。
- 建立服務存取監控，針對異常列舉行為、短時間大量查詢與未授權連線進行告警。

### 弱點 2：首頁原始碼暴露可逆向還原的密碼資訊

- 位置：`http://172.20.10.3/` 首頁原始碼
- 受影響功能：憑證保護機制
- 對應分類：`OWASP Top 10 2021 A02: Cryptographic Failures`
- CWE：`CWE-257` Storing Passwords in a Recoverable Format
- 風險等級：中風險
- CVSS v3.1 Base Score：`6.5`（Medium）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N`

#### 弱點描述

目標首頁原始碼中直接包含一段經過 Brainfuck 表示的敏感資訊，內容實際上可被還原為可用密碼。這種做法僅屬於可逆的編碼或混淆，而非真正的安全保護機制；任何能檢視頁面原始碼的攻擊者都可還原出該憑證內容。

#### 驗證方式

本次先檢視首頁原始碼，在頁面底部發現一段由 `+`、`-`、`<`、`>`、`[`、`]`、`.`、`,` 組成的字串。將其交由 Brainfuck 解碼後，得到密碼 `.2uqPEfj3D<P'a-3`。再結合前一步經由 `SMB` 枚舉取得的使用者 `cyber`，成功登入 `Usermin`。

#### 影響說明

此弱點使未授權攻擊者可在不突破任何登入保護的情況下，直接取得有效憑證並冒用既有使用者身份。若該帳號同時具備敏感功能存取權限，則帳號洩漏可迅速擴大為主機層風險。

#### 驗證結果

已成功：

- 從首頁原始碼取得可疑密文
- 將密文還原為有效密碼
- 結合有效帳號 `cyber` 成功登入 `Usermin`

#### 修補建議

- 立即移除所有出現在前端原始碼、靜態檔案與註解中的帳號密碼或其他敏感資訊。
- 不得以可逆編碼、字串混淆或簡單轉換方式保存密碼；密碼應以不可逆雜湊形式於後端安全儲存。
- 針對已暴露的帳號進行密碼輪替，並檢查是否存在相同或相似憑證被重用於其他服務。
- 對具管理或系統操作能力的帳號導入額外保護措施，例如多因素驗證、來源限制與行為監控。

### 弱點 3：Usermin 權限配置不當導致已驗證系統命令執行

- 位置：`Usermin` -> `Login` -> `Command Shell`
- 受影響功能：`Command Shell`
- 對應分類：`OWASP Top 10 2021 A01: Broken Access Control`
- CWE：`CWE-284` Improper Access Control
- 風險等級：高風險
- CVSS v3.1 Base Score：`8.8`（High）
- Vector：`CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`

#### 弱點描述

目標系統允許一般已驗證使用者直接存取 `Usermin` 的 `Command Shell` 功能，導致低權限帳號可在主機端執行系統命令。該功能本質上屬於高風險管理能力，若未正確限制於受信任角色或受控網段，即等同將遠端命令執行能力授予一般使用者。

#### 驗證方式

本次以前述取得的 `cyber` 帳號登入 `Usermin` 後，直接在介面中找到 `Login > Command Shell`。於該功能內執行 `uname -a` 可成功取得系統版本資訊，確認命令會在目標主機上實際執行。後續再透過同一功能送出反向 shell 指令，成功建立低權限 shell 並在主機端持續互動。

#### 影響說明

此弱點使任一持有有效帳號的攻擊者可直接在目標主機上執行命令、探索檔案系統、建立反向連線、下載與執行工具，並以該帳號所屬權限為基礎進一步展開本機權限提升。其影響已超出單純 Web 功能濫用，直接延伸至作業系統層級的控制風險。

#### 驗證結果

已成功：

- 以一般使用者身份存取 `Command Shell`
- 執行 `uname -a` 並取得系統版本資訊
- 建立低權限 shell
- 以該 shell 作為後續本機提權起點

#### 修補建議

- 重新檢查 `Usermin` 權限配置，禁止一般使用者存取 `Command Shell`、檔案管理與其他高風險管理功能。
- 若確有管理需求，應改由獨立管理帳號、限定來源網段與額外驗證機制保護，不應與一般使用者入口共用。
- 落實最小權限原則，確保應用層帳號僅具備完成業務所需的最低權限。
- 針對管理介面啟用完整審計紀錄與異常命令告警，以利發現未授權操作。

### 弱點 4：Linux Kernel `Dirty Pipe` 本機權限提升至 root

- 位置：作業系統本機環境
- 受影響元件：Linux kernel `5.10.0-9-amd64`
- 相關 CVE：`CVE-2022-0847`（Dirty Pipe）
- 風險等級：高風險
- CVSS v3.1 Base Score：`7.8`（High）
- Vector：`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`
- 評分來源：NVD

#### 弱點描述

目標主機核心版本落在 `Dirty Pipe`（`CVE-2022-0847`）的受影響範圍內。此漏洞允許本機低權限使用者覆寫只讀檔案對應的 page cache 內容，進而達成權限提升。當攻擊者已先取得一般使用者或低權限 shell 時，即可利用該弱點突破原有權限邊界並取得 `root` 權限。

#### 驗證方式

本次先於目標主機執行 `uname -a`，確認核心版本為 `5.10.0-9-amd64`。後續利用 `Dirty Pipe` 利用鏈進行驗證，成功建立 `root` session 並切換至 root shell；另以 `searchsploit` 搜尋本地提權程式時，也能找到對應的 `50808.c`，進一步驗證該核心版本存在可利用的公開提權方式。根據 NVD 公開資訊，`CVE-2022-0847` 影響 Linux kernel `5.8` 起至修補前的多個版本分支，其中包含 `5.10` 分支在 `5.10.102` 之前的版本。

#### 影響說明

此弱點使攻擊者在已取得低權限主機存取的前提下，可進一步取得 `root` 權限。一旦提升成功，攻擊者即可完整控制系統，包括讀取與修改敏感檔案、建立持久化機制、竄改系統設定、存取所有使用者資料，以及停用原有安全控制。

#### 驗證結果

已成功：

- 確認核心版本位於 `Dirty Pipe` 受影響範圍
- 以低權限 shell 作為起點進行本機提權
- 成功取得 `root` session
- 成功讀取 `root flag`

#### 修補建議

- 儘速將核心更新至已修補 `CVE-2022-0847` 的版本，並確認供應商套件已完整套用。
- 盤點並淘汰已知存在高風險本機提權漏洞的核心與映像版本，避免弱點持續存在於測試或正式環境。
- 限制一般使用者與應用服務帳號在主機上的互動能力，降低低權限 shell 被進一步利用的機會。
- 搭配主機層防護，例如 EDR、完整性監控與最小權限設計，以降低本機提權成功後的擴散範圍。

## 參考資料與附錄

### 靶機與背景資訊

- VulnHub 靶機條目：<https://www.vulnhub.com/entry/empire_breakout,751/>

### 風險分類與弱點對應

- OWASP Top 10 2021 A01: Broken Access Control：<https://owasp.org/Top10/en/A01_2021-Broken_Access_Control/>
- OWASP Top 10 2021 A02: Cryptographic Failures：<https://owasp.org/Top10/2021/A02_2021-Cryptographic_Failures/>
- CWE-200: Exposure of Sensitive Information to an Unauthorized Actor：<https://cwe.mitre.org/data/definitions/200.html>
- CWE-257: Storing Passwords in a Recoverable Format：<https://cwe.mitre.org/data/definitions/257.html>
- CWE-284: Improper Access Control：<https://cwe.mitre.org/data/definitions/284.html>

### CVSS 評分參考

- FIRST CVSS v3.1 Specification：<https://www.first.org/cvss/v3.1/specification-document>
- FIRST CVSS v3.1 Calculator：<https://www.first.org/cvss/calculator/3.1>
- NVD CVSS v3 Calculator：<https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator>

### 弱點 4 與公開資訊

- NVD CVE-2022-0847 頁面：<https://nvd.nist.gov/vuln/detail/CVE-2022-0847>
- Dirty Pipe 原始揭露頁面：<https://dirtypipe.cm4all.com/>
- CISA Known Exploited Vulnerabilities Catalog（CVE-2022-0847）：<https://www.cisa.gov/known-exploited-vulnerabilities-catalog?field_cve=CVE-2022-0847>
