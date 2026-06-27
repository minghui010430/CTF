# Information Gathering

- 這份主要在練 OSINT、DNS、vhost、fingerprinting 和 archives 的查找流程。
- 重點不是 exploitation，而是看到題目後能先判斷要用哪一類工具去收答案。

## Utilising WHOIS

### Perform a WHOIS lookup against the paypal.com domain. What is the registrar Internet Assigned Numbers Authority (IANA) ID number?

![[assets/information-gathering-main-01-whois-paypal-query.png]]

- 先對 `paypal.com` 做 WHOIS lookup。

![[assets/information-gathering-main-02-whois-paypal-iana-id.png]]

- 在 WHOIS 結果裡找 `Registrar IANA ID` 欄位，這題要的就是這個值。

```text
292
```

### What is the admin email contact for the tesla.com domain (also in-scope for the Tesla bug bounty program)?

![[assets/information-gathering-main-03-whois-tesla-query.png]]

- 同樣先對 `tesla.com` 做 WHOIS 查詢。

![[assets/information-gathering-main-04-whois-tesla-admin-email.png]]

- 這次改看管理聯絡資訊，在結果中找 `Admin Email` 對應的欄位。

```text
admin@dnstinations.com
```

## DNS

### Which IP address maps to inlanefreight.com?

![[assets/information-gathering-main-06-dns-a-record-inlanefreight.png]]

- 這題就是查 `inlanefreight.com` 的 `A` record，看 domain 會解析到哪個 IPv4 位址。

```text
134.209.24.248
```

### Which domain is returned when querying the PTR record for 134.209.24.248?

![[assets/information-gathering-main-07-dns-ptr-record-result.png]]

- 這題反過來從 IP 查 `PTR` record，看這個位址反解回哪個 domain。

```text
inlanefreight.com
```

### What is the full domain returned when you query the mail records for facebook.com?

![[assets/information-gathering-main-09-dns-mx-facebook.png]]

- 查 `facebook.com` 的 `MX` record，結果會列出負責收信的 mail host。

```text
smtpin.vvv.facebook.com
```

## Subdomain

![[assets/information-gathering-main-10-subdomain-section-overview-01.png]]

- 這一段是在做子網域枚舉，先把可能的 subdomain 跑出來。

![[assets/information-gathering-main-11-subdomain-section-overview-02.png]]

- 再從枚舉結果裡篩出有效的項目。

![[assets/information-gathering-main-12-subdomain-result-my-inlanefreight.png]]

- 最後確認到的有效子網域是 `my.inlanefreight.com`。

```text
my.inlanefreight.com
```

## DNS Zone Transfers

![[assets/information-gathering-main-13-zone-transfer-section-overview.png]]

- 這一段重點是測試目標 DNS 伺服器是否允許做 zone transfer。
- 一旦 `AXFR` 成功，通常就能一次拿到整個 zone 裡的主機名稱與 IP。

### After performing a zone transfer for the domain inlanefreight.htb on the target system, how many DNS records are retrieved from the target system's name server? Provide your answer as an integer, e.g, 123.

![[assets/information-gathering-main-14-zone-transfer-results.png]]

- 先看整份 zone transfer 回傳結果，這題要的是總共列出多少筆 DNS records。

```text
22
```

### Within the zone record transferred above, find the ip address for ftp.admin.inlanefreight.htb. Respond only with the IP address, eg 127.0.0.1

![[assets/information-gathering-main-14-zone-transfer-results.png]]

- 同一份 `AXFR` 輸出裡，直接找 `ftp.admin.inlanefreight.htb` 這筆主機記錄即可。

```text
10.10.34.2
```

### Within the same zone record, identify the largest IP address allocated within the 10.10.200 IP range. Respond with the full IP address, eg 10.10.200.1

![[assets/information-gathering-main-14-zone-transfer-results.png]]

- 一樣在同一份 zone transfer 結果中，找出 `10.10.200.x` 這段裡最大的位址。

```text
10.10.200.14
```

## Virtual Hosts

![[assets/information-gathering-main-15-virtual-hosts-section-overview.png]]

- 這一段是在做 vhost 枚舉，重點不是只看 IP，而是看同一台 Web 服務背後掛了哪些虛擬主機。

### Brute-force vhosts on the target system. What is the full subdomain that is prefixed with "web"? Answer using the full domain, e.g. "x.inlanefreight.htb"

![[assets/information-gathering-main-16-virtual-hosts-gobuster-command.png]]

- 先用 `gobuster vhost` 對目標做虛擬主機爆破。

```text
gobuster vhost -u http://<ip>:<port> -w </path/to/wordlists> --append-domain -t 30
```

![[assets/information-gathering-main-17-virtual-hosts-gobuster-output.png]]

- 從輸出裡先收集 `web` 開頭的候選 vhost。

![[assets/information-gathering-main-19-virtual-hosts-http-400-response.png]]

- 直接用這些名稱存取時先遇到 `400`，表示還需要把 hostname 正確綁到目標 IP。

![[assets/information-gathering-main-18-virtual-hosts-hosts-file-entry.png]]

- 把 IP 和 hostname 加進 `/etc/hosts` 後，就能正常驗證結果。

```text
web17611.inlanefreight.htb
```

### Brute-force vhosts on the target system. What is the full subdomain that is prefixed with "vm"? Answer using the full domain, e.g. "x.inlanefreight.htb"

- 用同一輪 vhost 枚舉結果往下找 `vm` 開頭的項目。

```text
vm5.inlanefreight.htb
```

### Brute-force vhosts on the target system. What is the full subdomain that is prefixed with "br"? Answer using the full domain, e.g. "x.inlanefreight.htb"

- 同樣從枚舉結果裡找 `br` 開頭的項目。

```text
browse.inlanefreight.htb
```

### Brute-force vhosts on the target system. What is the full subdomain that is prefixed with "a"? Answer using the full domain, e.g. "x.inlanefreight.htb"

- 再從同一份輸出中找 `a` 開頭的 vhost。

```text
admin.inlanefreight.htb
```

### Brute-force vhosts on the target system. What is the full subdomain that is prefixed with "su"? Answer using the full domain, e.g. "x.inlanefreight.htb"

- 最後找 `su` 開頭的結果。

```text
support.inlanefreight.htb
```

## Fingerprinting

![[assets/information-gathering-main-20-fingerprinting-section-overview.png]]

- 這一段是從 response headers、目錄結構和常見 CMS 痕跡去判斷 Web 技術棧。

### Determine the Apache version running on app.inlanefreight.local on the target system. (Format: 0.0.0)

![[assets/information-gathering-main-21-fingerprinting-http-headers.png]]

- 直接看 HTTP headers，就能從 `Server` 欄位拿到 Apache 版本資訊。

```text
2.4.41
```

### Which CMS is used on app.inlanefreight.local on the target system? Respond with the name only, e.g., WordPress.

![[assets/information-gathering-main-22-fingerprinting-dirsearch-command.png]]

- 先對 `app.inlanefreight.local` 跑 `dirsearch`，看有沒有 CMS 常見目錄。

```text
dirsearch -u http://app.inlanefreight.local -x 301,403
```

![[assets/information-gathering-main-23-fingerprinting-dirsearch-output.png]]

- 從掃描結果裡找可疑路徑，再往下確認是不是特定 CMS。

![[assets/information-gathering-main-24-fingerprinting-joomla-indicator.png]]

- 最後從頁面或目錄痕跡可以判斷這套 CMS 是 Joomla。

```text
Joomla
```

### On which operating system is the dev.inlanefreight.local webserver running in the target system? Respond with the name only, e.g., Debian.

![[assets/information-gathering-main-21-fingerprinting-http-headers.png]]

- 同一組 HTTP headers 裡，除了 Apache 版本外，也能看到作業系統資訊。

```text
Ubuntu
```

## Creepy Crawlies

### After spidering inlanefreight.com, identify the location where future reports will be stored. Respond with the full domain, e.g., files.inlanefreight.com.

```text
python3 ReconSpider.py http://inlanefreight.com
cat results.json
```

![[assets/information-gathering-main-25-reconspider-results-json.png]]

- 對站台做 spidering 後，回頭看 `results.json`，從收集到的連結或資源位置裡找 future reports 會存放的地方。

```text
inlanefreight-comp133.s3.amazonaws.htb
```

## Web Archives

- 這一段都靠 Web Archive / Wayback Machine 回到指定時間點，再直接看當時頁面上的文字。

### How many Pen Testing Labs did HackTheBox have on the 8th August 2018? Answer with an integer, eg 1234.

![[assets/information-gathering-main-28-web-archives-htb-labs-query.png]]

- 先把時間切到 `2018-08-08` 附近的快照。

![[assets/information-gathering-main-29-web-archives-htb-labs-count.png]]

- 再從當時頁面上找 Pen Testing Labs 的數量。

```text
74
```

### How many members did HackTheBox have on the 10th June 2017? Answer with an integer, eg 1234.

![[assets/information-gathering-main-30-web-archives-htb-members-count.png]]

- 同樣切到 `2017-06-10` 的快照，直接看頁面上顯示的會員數。

```text
3054
```

### Going back to March 2002, what website did the facebook.com domain redirect to? Answer with the full domain, eg http://www.facebook.com/

![[assets/information-gathering-main-26-web-archives-facebook-query.png]]

- 先回到 `2002-03` 左右的快照。

![[assets/information-gathering-main-27-web-archives-facebook-redirect.png]]

- 從當時的轉址頁面可以直接看到 facebook.com 轉去的網站。

```text
http://site.aboutface.com/
```

### According to the paypal.com website in October 1999, what could you use to "beam money to anyone"? Answer with the product name, eg My Device, remove the ™ from your answer.

![[assets/information-gathering-main-31-web-archives-paypal-palm-organizer.png]]

- 切到 `1999-10` 的 PayPal 頁面後，直接看那句宣傳文案在講什麼產品。

```text
Palm 0rganizer
```

### Going back to November 1998 on google.com, what address hosted the non-alpha "Google Search Engine Prototype" of Google? Answer with the full address, eg http://google.com

![[assets/information-gathering-main-32-web-archives-google-prototype.png]]

- 在 `1998-11` 的 Google 快照裡，可以直接看到 "Google Search Engine Prototype" 掛在哪個位址。

```text
http://google.stanford.edu/
```

### Going back to March 2000 on www.iana.org, when exacty was the site last updated? Answer with the date in the footer, eg 11-March-99

![[assets/information-gathering-main-33-web-archives-iana-last-updated.png]]

- 回到 `2000-03` 的快照後，往頁面 footer 找 last updated 日期。

```text
17-December-99
```

### According to wikipedia.com snapshot taken on February 9, 2003, how many articles were they already working on in the English version? Answer with the number they state without any commas, e.g., 100000, not 100,000.

![[assets/information-gathering-main-34-web-archives-wikipedia-article-count.png]]

- 切到 `2003-02-09` 的 Wikipedia 快照，直接看英文版正在處理的文章數。

```text
104155
```
