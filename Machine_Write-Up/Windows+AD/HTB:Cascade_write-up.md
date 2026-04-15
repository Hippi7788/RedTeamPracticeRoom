## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Cascade
2. 難度:Medium
3. IP:10.129.211.241

### 主要發現：
1. 漏洞 ：敏感資訊外洩與硬編碼憑證（風險等級：Critical）
2. 漏洞 ：透過權限配置過高的非管理員使用者在未被正確清理的刪除文件中取得硬編碼密碼（風險等級：High）
3. 漏洞 ：透過已刪除帳戶的硬編碼密碼與主網域管理員帳戶相同獲得管理員憑證（風險等級：High）

### 攻擊鏈摘要：

匿名枚舉LDAP，獲得一組可用於SMB服務的憑證，並從SMB服務中獲得TightVNC註冊表備份，解密該備份後，取得了可用於WinRM遠程登入的憑證。該用戶擁有對.NET可執行檔的存取權限，反編譯並分析原始程式碼後，發現加密金鑰和IV硬編碼在檔案中，解密可獲得另一組憑證，該帳戶屬於AD Recycle Bin群組，能夠查看已刪除的Active Directory物件，檢查已刪除物件後獲得其中一個已刪除的使用者帳戶和硬編碼密碼，該密碼可重複使用以作為主網域管理員登入。

### 潛在影響：

1. 透過解密的VNC與WinRM憑證，攻擊者可在內部網路自由移動並竊取關鍵業務數據。
2. 攻擊者獲得網域管理員權限，可操控所有使用者帳號、存取機密資料或癱瘓整個網路架構。
3. 攻擊者可能建立後門帳號或修改GPO，即便重設已知遭洩密碼也難以徹底根除。


### 修復建議：

1. 禁止在程式碼或備份檔案中硬編碼金鑰與密碼。
2. 停用LDAP的匿名訪問。
3. 導入密鑰管理系統。
4. 嚴格限制能存取AD Recycle Bin的使用者清單，僅限必要之系統管理員。
5. 禁止跨帳號重複使用密碼，並定期清理已刪除物件中的敏感屬性資訊。
6. 對.NET程式進行混淆處理，增加反編譯難度。
7. 定期掃描SMB分享資料夾，清理含有敏感資訊的備份檔。


---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，除了AD環境預設開放的以外並未有特殊服務
1. 用穩定的TCP協議進行全掃描，這步需要完整性，不能有漏，最低速率要參考網速，可以先ping靶機測試
2. 針對UDP協議掃描，只掃前20個常用埠就行
3. 詳細訊息掃描，探測服務版本、使用默認腳本並且掃描作業系統
4. 用漏洞腳本進行掃描，基本上AD環境不會有什麼結果，但還是要做，不能放棄可能有的攻擊面

```bash
sudo nmap -sT -Pn --min-rate 800 -p- 10.129.211.241 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.211.241 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.211.241 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.211.241 -oA portscan/vuln
```

<img width="689" height="584" alt="螢幕擷取畫面 2026-04-13 204418" src="https://github.com/user-attachments/assets/47642811-c921-444d-9de6-fb8cb17d8b7a" />

<img width="596" height="588" alt="螢幕擷取畫面 2026-04-13 204429" src="https://github.com/user-attachments/assets/b7e2b5a3-9984-4af9-992e-978bf0a4866e" />

<img width="1898" height="778" alt="螢幕擷取畫面 2026-04-13 204836" src="https://github.com/user-attachments/assets/b19676cb-90c9-4706-a27b-94b6fad754e9" />


<img width="1245" height="623" alt="螢幕擷取畫面 2026-04-13 204946" src="https://github.com/user-attachments/assets/950a4afd-592d-4a74-9b96-cfd71aa8febb" />

紀錄掃描到的網域名稱

<img width="513" height="260" alt="螢幕擷取畫面 2026-04-13 205046" src="https://github.com/user-attachments/assets/98676fa8-26c6-47bc-8e94-0cc0bc52b82e" />

在紅隊視角中，時間偏差問題常常成為AD滲透或橫向移動中的隱形障礙，尤其是利用Kerberos或NTLM等需要用到時間戳的協議的時候，一旦目標機器和攻擊機有時間偏差，就會導致驗證失敗或在系統中留下異常日誌。因為Windows對於這種協議的時間偏差只有很小的容忍範圍，通常在五分鐘左右。

ntpdate走NTP(Network Time Protocol)協議，123 UDP port，不太會留下系統日誌或被管理員注意到，但若UDP123沒開放則無法使用。

還有net命令中的時間調整(time set)命令可做，但沒有任何返回消息，走的是SMB(445,135,139)的協議，可能留下系統日誌。

```bash
sudo ntpdate 10.129.211.241
```

<img width="786" height="90" alt="螢幕擷取畫面 2026-04-13 205117" src="https://github.com/user-attachments/assets/f9691fa6-6307-4312-be92-7c48e524f734" />



### 2.2 枚舉(Enumeration)

使用smbclient枚舉SMB服務，這個工具可以模仿正常使用者的流量，比較不會被管理員注意

```bash
smbclient -L //10.129.211.241 -N
smbclient -L //10.129.211.241 -U "guest" -N
```

<img width="828" height="271" alt="螢幕擷取畫面 2026-04-13 205236" src="https://github.com/user-attachments/assets/4fc754a8-e95d-4ad1-8dcd-7941c71bb341" />

rpcclient可匿名枚舉出使用者名稱，這個工具也模仿正常使用者的流量，但一般要求權限比較高，例如IPC$可讀的情況，可能rpcclient無法調出使用者SID，impacket-rpcclient卻可以，所以通常是先用rpcclient去看，如果有特殊情況rpcclient不好用的時候，再上impacket

```bash
rpcclient -U "" 10.129.211.241 -N
```

<img width="671" height="580" alt="螢幕擷取畫面 2026-04-13 205434" src="https://github.com/user-attachments/assets/a4833c79-70ed-4c1e-9a10-077960491601" />

將使用者名稱儲存成文件

<img width="755" height="341" alt="螢幕擷取畫面 2026-04-13 205638" src="https://github.com/user-attachments/assets/3f4b8808-1820-4b96-9a0b-cd9ff9cc7be5" />

嘗試AS-REP Roasting，並未發現有開啟不須預驗證的使用者，請事先同步時間

```bash
impacket-GetNPUsers cascade.local/ -usersfile username -format hashcat -no-pass -dc-ip 10.129.211.241 -outputfile asrep_hash
```

<img width="960" height="403" alt="螢幕擷取畫面 2026-04-13 205932" src="https://github.com/user-attachments/assets/3ef7544b-87ae-435d-9aec-b719b9b510d6" />

使用nxc做密碼噴灑，他是crackmapexec的繼承者，可以自動化內網安全評估與橫向移動

```bash
nxc smb 10.129.211.241 -u username -p username --continue-on-success
```

<img width="1654" height="279" alt="螢幕擷取畫面 2026-04-13 210250" src="https://github.com/user-attachments/assets/afba6c09-e4ab-4ac7-a043-50715e9173e8" />

我無法利用使用者名單獲得密碼，所以我回頭來枚舉LDAP，LDAP是一種用於在IP網路中存取與管理「目錄服務」資料的開放標準網路協定，它最常用於企業內部的集中式身份驗證與帳號管理

使用ldapsearch枚舉LDAP服務，這也是相對來說隱蔽性比較好的工具，一樣可以模仿正常使用者流量，因為訊息非常多，我將結果保存成文件後使用grep查詢

```bash
ldapsearch -x -H ldap://10.129.211.241 -b 'DC=CASCADE,DC=LOCAL' > ldap_ano
```

<img width="729" height="497" alt="螢幕擷取畫面 2026-04-13 211333" src="https://github.com/user-attachments/assets/ce01ef82-85a7-4c9a-a82d-0377e85c7441" />

我添加了過濾條件，指定類別為人

```bash
ldapsearch -x -H ldap://10.129.211.241 -b 'DC=CASCADE,DC=LOCAL' '(objectClass=person)' > ldap_per
```

<img width="953" height="70" alt="螢幕擷取畫面 2026-04-13 211457" src="https://github.com/user-attachments/assets/099d9006-e8c9-4e6e-b87e-90305e4fca0a" />

配合grep搜尋後，我找到疑似密碼的字串，推測為歷史遺留密碼，可能未被清除。

<img width="337" height="680" alt="螢幕擷取畫面 2026-04-13 211725" src="https://github.com/user-attachments/assets/a95acc60-21e8-4ded-a2f3-db92b87a7bdc" />

搜尋關鍵字前幾行，我順利找到使用者名稱

<img width="668" height="639" alt="螢幕擷取畫面 2026-04-13 211919" src="https://github.com/user-attachments/assets/42ea0f18-455c-4929-bafb-97a0e10325dc" />

疑似為Base64編碼，解碼後取得明文密碼。

此類弱點常見於：
1. 過時系統遷移
2. 測試帳號未清除
3. 自訂屬性儲存敏感資訊

<img width="336" height="93" alt="螢幕擷取畫面 2026-04-13 212125" src="https://github.com/user-attachments/assets/8b7550ac-0476-419e-89cc-53939c81a6f2" />

但此憑證並不能利用WinRM登入

取得憑證時，一定要優先嘗試可否遠端登入，在TCP5985有開的情況下，evil-winrm肯定是第一優先要測試的，若無法登入，SMB服務可能有內部文件洩漏，也是優先度較高的

<img width="1180" height="250" alt="螢幕擷取畫面 2026-04-13 212340" src="https://github.com/user-attachments/assets/5ceac52b-7536-4a44-9263-a53798ad6f12" />


### 2.3 初始存取(Initial Access)

這個憑證可以透過SMB協議登入，我使用smbclient登入

<img width="842" height="333" alt="螢幕擷取畫面 2026-04-13 212546" src="https://github.com/user-attachments/assets/08f7833f-7bc2-43b3-8f91-1eb1cca05a3a" />

共享資料夾中，只有Data可讀且有檔案，我將檔案全部拷貝下來

<img width="748" height="614" alt="螢幕擷取畫面 2026-04-13 213613" src="https://github.com/user-attachments/assets/f2a96b72-6275-4a36-b463-73ebb8ad49f1" />

首先使用firefox開啟.html檔

```bash
firefox Meeting_Notes_june_2018.html &
```

<img width="428" height="138" alt="螢幕擷取畫面 2026-04-13 214041" src="https://github.com/user-attachments/assets/af17dee2-3e4e-4030-81f9-e204682fc619" />

信中說有一使用者帳號TempAdmin被刪除，且其密碼與管理員一致

<img width="1393" height="636" alt="螢幕擷取畫面 2026-04-13 214052" src="https://github.com/user-attachments/assets/6eacf63b-6a18-4b30-9156-8edb96e11388" />

<img width="1417" height="387" alt="螢幕擷取畫面 2026-04-13 214146" src="https://github.com/user-attachments/assets/486891da-3976-47ba-aa96-756e79c62720" />

開啟一日誌檔，找到一使用者名稱

<img width="1526" height="354" alt="螢幕擷取畫面 2026-04-13 214346" src="https://github.com/user-attachments/assets/6b1625b1-b79f-435d-abd2-96709833360e" />

另一日誌檔有伺服器名稱

<img width="664" height="597" alt="螢幕擷取畫面 2026-04-13 214449" src="https://github.com/user-attachments/assets/1c7eae5e-43a8-43f8-987c-fcfb40d521a4" />

開啟.reg檔，在Windows作業系統中，.reg是「登錄檔編輯器匯入檔」的副檔名，它是一個純文字檔案，專門用來批次修改Windows的「登錄檔」。

在其中找到加密後的密碼

<img width="388" height="724" alt="螢幕擷取畫面 2026-04-13 214617" src="https://github.com/user-attachments/assets/ee7a44c2-0753-4c98-b364-ec9963ceb191" />

從檔名裡，可以了解到VNC這個服務，我使用vncpwd來破解密碼

<img width="1656" height="767" alt="螢幕擷取畫面 2026-04-13 214754" src="https://github.com/user-attachments/assets/c904ed8b-1da2-40a0-8ef4-5257a5287170" />

<img width="1731" height="792" alt="螢幕擷取畫面 2026-04-13 215017" src="https://github.com/user-attachments/assets/21a63e98-f194-481f-b5f6-9b8d6be5f034" />

經典VNC實作使用DES演算法，但它犯了密碼學的大忌：使用固定的挑戰金鑰。VNC伺服器在存儲密碼時，並不是進行不可逆的Hash，而是用一個內建在程式碼裡的固定金鑰對用戶密碼進行DES加密。這意味著只要拿到加密後的密文，任何人都可以用同一組固定金鑰將其完美還原回明文。

大多數VNC密碼解密工具不直接接受文字字串，而是要求讀取一個二進位檔案，使用以下命令將密碼轉為二進位檔案
```bash
echo -n "6bcf2a4b6e5aca0f" | xxd -r -p > vnc.pass
```
1. -n：不要換行，避免換行符破壞結構
2. xxd：是一個處理十六進位與二進位轉換的工具
3. -r (reverse)：將十六進位轉回二進位
4. -p (plain)：告訴xxd輸出是純粹的十六進位連續字串，沒有地址或ASCII欄位

<img width="517" height="125" alt="螢幕擷取畫面 2026-04-13 215438" src="https://github.com/user-attachments/assets/58b9aab3-5969-4c51-8aeb-23a63dad71c1" />

回頭去SMB協議裡，可以看到檔案來自於s.smith的目錄中，我認為這就是密碼配對的使用者名稱

<img width="712" height="86" alt="螢幕擷取畫面 2026-04-13 215908" src="https://github.com/user-attachments/assets/137b2eee-3d96-4afe-879d-c107955c0923" />

透過WinRM登入成功

<img width="1223" height="296" alt="螢幕擷取畫面 2026-04-13 220022" src="https://github.com/user-attachments/assets/5102b54a-1966-48e0-bf2d-c02ff0d771a6" />

獲得user.txt

<img width="618" height="329" alt="螢幕擷取畫面 2026-04-13 220214" src="https://github.com/user-attachments/assets/7d96d0d5-8a78-4bf7-884d-8fe9b023f7ef" />

### 2.4 橫向移動(Lateral Movement)

檢查使用者權限和組

<img width="658" height="714" alt="螢幕擷取畫面 2026-04-13 220704" src="https://github.com/user-attachments/assets/c0a6130b-63bb-428a-acf6-674272535ba5" />

Audit Share剛才在SMB中看過，我去那個資料夾看看

<img width="735" height="476" alt="螢幕擷取畫面 2026-04-13 221105" src="https://github.com/user-attachments/assets/2d7ff66c-4f71-4bcf-9a1a-19439da26a6c" />

<img width="702" height="324" alt="螢幕擷取畫面 2026-04-13 221106" src="https://github.com/user-attachments/assets/73dfcabd-53c1-4db7-97f5-d72f332a32ca" />

將資料夾中的檔案拷貝

<img width="1012" height="153" alt="螢幕擷取畫面 2026-04-13 221543" src="https://github.com/user-attachments/assets/cc3364e2-7913-4355-a857-821b0b9e5dc5" />

使用sqlite3檢查db檔，我獲得一組加密密碼，看起來像base64但無法正常解密，應該是透過特別的偏移加密，Key需要找出來

<img width="1263" height="350" alt="螢幕擷取畫面 2026-04-13 221806" src="https://github.com/user-attachments/assets/5b2913fd-c17e-4c20-81ec-6cbd62fe9722" />

這裡只要把.exe跟.dll檔拿去用dnSpy反編譯就好了，它是一款針對.NET框架開發的開源反編譯器，被許多開發者譽為神級工具，主要因為它能在完全沒有原始碼的情況下，讓使用者直接閱讀、修改甚至逐行偵錯.NET程式集


我當時想練習monodis，在實際打靶中盡量使用dnSpy，因為monodis的主要功能是將.NET的程式集反組譯回CIL代碼，而非C#，在不是進行極底層的研究或沒有圖形介面的情況下「完全沒好處」，這裡看看就好，最主要就是從.exe跟.dll反編譯出KEY跟IV

CIL是.NET的中間語言，雖然看起來像組合語言，但邏輯結構非常清晰，幾乎可以完整對應回原始碼

可以使用--typedef參數列出該檔案中定義的所有類型

```bash
monodis --typedef CascCrypto.dll
```

<img width="918" height="284" alt="螢幕擷取畫面 2026-04-13 222734" src="https://github.com/user-attachments/assets/b4e9f5de-3849-4450-8b27-224fbeea9953" />

可以再用--method確認函數邏輯，但我暫時不這麼做，因為在靶機中，Key跟IV不外乎就是硬編碼或外部傳入，只要不是使用函數邏輯去生成，或是調用環境變數、金鑰管理服務等「正確做法」，都可以直接查硬編碼，如果是外部傳入就查別的檔案的硬編碼，如果各檔案的硬編碼都沒有，再回頭來看函數邏輯

我用--userstrings參數來列出所有硬編碼字串，發現一組字串是符合格式的

```bash
monodis --userstrings CascCrypto.dll
```

<img width="418" height="187" alt="螢幕擷取畫面 2026-04-13 224741" src="https://github.com/user-attachments/assets/683eefda-c1c3-452e-ab9f-0b036750d8fe" />

接著我查.exe檔的硬編碼，也有一組符合格式

```bash
monodis --userstrings CascAudit.exe
```

<img width="925" height="612" alt="螢幕擷取畫面 2026-04-13 224940" src="https://github.com/user-attachments/assets/b7c982e0-aed3-4f31-a772-b7864354eaef" />

使用CyberChef解密

<img width="1277" height="663" alt="螢幕擷取畫面 2026-04-13 225243" src="https://github.com/user-attachments/assets/dd5125ce-050c-488d-994b-9af5043a59fa" />

橫向移動成功

<img width="1200" height="268" alt="螢幕擷取畫面 2026-04-13 225518" src="https://github.com/user-attachments/assets/b4d872b4-e9cf-427c-a34b-d6efb293f76c" />

【注意，實戰時請直接使用dnSpy或ILSpy反編譯，這裡展示的硬編碼是非常嚴重的資安問題，一般不太會有這種事，通常是C#中會有個邏輯，這個邏輯會告訴我KEY怎麼來的，然後再去挖那個來源，例如說環境變數或什麼設備或服務生成再導入的，硬編碼最近只有在AI寫的程式中會有，所以直接用dnSpy或ILSpy看精美的C#比較清楚】

### 2.5 權限提升(Privilege Escalation)

我檢查了當前使用者的權限和屬組，AD Recycle Bin是2008年引入的功能，用於還原意外刪除的AD物件，通常需手動啟用，能保留被刪除物件的完整屬性。

記得剛才有一封電子郵件提到有一使用者帳號TempAdmin被刪除，且其密碼與管理員一致，我打算去看看這個使用者可不可以挖到密碼

<img width="639" height="754" alt="螢幕擷取畫面 2026-04-13 225558" src="https://github.com/user-attachments/assets/37f4a625-cabb-4f4a-8803-74cdb865dd80" />


我檢查網域中所有已刪除的物件

```powershell
Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects
```

<img width="1356" height="549" alt="螢幕擷取畫面 2026-04-13 225659" src="https://github.com/user-attachments/assets/5832b0a0-f4a4-450f-86bc-e400a63ea4d2" />

在其中我找到TempAdmin的相關資料

<img width="1066" height="159" alt="螢幕擷取畫面 2026-04-13 225713" src="https://github.com/user-attachments/assets/6076e4ff-08bf-41a9-8a6b-1abd7d1d6cdc" />

我檢查這個帳戶的詳細訊息

```powershell
Get-ADObject -filter { SAMAccountName -eq "TempAdmin" } -includeDeletedObjects -property *
```

<img width="951" height="747" alt="螢幕擷取畫面 2026-04-13 225906" src="https://github.com/user-attachments/assets/cd41301b-709a-4be5-a60d-44db4a5961b3" />

解碼base64

<img width="430" height="138" alt="螢幕擷取畫面 2026-04-13 225945" src="https://github.com/user-attachments/assets/54ad70d1-a9d3-4c29-a092-8230f1051978" />

嘗試登入Administrator成功

<img width="1189" height="254" alt="螢幕擷取畫面 2026-04-13 230025" src="https://github.com/user-attachments/assets/a509486a-f929-45ce-9a2b-8453d46b4f96" />


### 2.6 最終成果(Impact)

獲得root.txt

<img width="645" height="479" alt="螢幕擷取畫面 2026-04-13 230135" src="https://github.com/user-attachments/assets/5f2b01f2-9173-47b4-807f-83264b461dd4" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 相對來說繞了一點，需要兩次解密，確實有點Medium的難度，SMB憑證、初次存取憑證、橫向移動憑證、被刪除憑證和管理員憑證，最好每次獲得憑證就存成文件管理。
2. 成功練習monodis，這是難得的機會。

### 浪費時間的部分：

1. 並沒有特別卡住的地方，只有練習monodis時花了一點時間了解。

### 新知識點：

1. 關於VNC解密的方法
2. 關於monodis的使用方法

### 與實戰對應：

1. 再次強調，實戰時請直接使用dnSpy或ILSpy反編譯。

