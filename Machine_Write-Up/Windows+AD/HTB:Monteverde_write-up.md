## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Monteverde
2. 難度:Medium
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

使用Nmap掃描開放埠，並未發現除了AD常用服務之外的特殊服務

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.228.111 -oA portscan/ports
sudo nmap -sU -Pn --top-ports 20 10.129.228.111 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.228.111 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.228.111 -oA portscan/vuln
```

<img width="719" height="591" alt="螢幕擷取畫面 2026-04-09 153905" src="https://github.com/user-attachments/assets/c83d4cb6-a512-43b0-b429-f01c9d46b85f" />

<img width="672" height="565" alt="螢幕擷取畫面 2026-04-09 153916" src="https://github.com/user-attachments/assets/2dc502e2-d15b-47a6-bee7-191b9b3c282c" />

<img width="1295" height="749" alt="螢幕擷取畫面 2026-04-09 154423" src="https://github.com/user-attachments/assets/f894dda3-4fdd-41f4-b94b-402dfc904212" />

<img width="1384" height="680" alt="螢幕擷取畫面 2026-04-09 154445" src="https://github.com/user-attachments/assets/926d2586-845d-43c2-a9be-eed2c23943f7" />

我用了文書處理工具整理開放埠

```bash
ports=$(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',')
```

<img width="828" height="213" alt="螢幕擷取畫面 2026-04-09 154036" src="https://github.com/user-attachments/assets/b344fbb1-3835-464c-8f31-42e1d1835ee7" />

因為UDP123有開放，我使用ntpdate時間同步，NTP比較不會被管理員監控，從這裡走比較隱蔽

```bash
sudo ntpdate 10.129.228.111
```

<img width="788" height="74" alt="螢幕擷取畫面 2026-04-09 154454" src="https://github.com/user-attachments/assets/b49c04a5-4aa2-4d46-9957-a1b86888b9e5" />

將掃描到的網域寫入/etc/hosts

<img width="522" height="248" alt="螢幕擷取畫面 2026-04-09 154602" src="https://github.com/user-attachments/assets/9152d362-c1ff-4877-ba38-b2f5cd1a014e" />

### 2.2 枚舉(Enumeration)

使用smbclient匿名枚舉SMB協議，這個工具模擬現實使用者連線，相對來說比較正常

SMB匿名枚舉時，「不使用名稱」和使用「訪客(guest)」是不同的，肯定兩邊都要試

```bash
smbclient -L //10.129.228.111 -N
smbclient -L //10.129.228.111 -U "Guest"
```

<img width="834" height="273" alt="螢幕擷取畫面 2026-04-09 154939" src="https://github.com/user-attachments/assets/2cd4a5a0-bed1-4a4f-bf36-5f5501cc54cc" />

使用rpcclient枚舉RPC協議，這個工具模仿正常使用者連線，跟smbclient一樣都是優先使用的

```bash
rpcclient -U "" -N 10.129.228.111
```

<img width="374" height="284" alt="螢幕擷取畫面 2026-04-09 155039" src="https://github.com/user-attachments/assets/67d4ee5e-ef0c-4108-86fe-b6a684ce4c4b" />

<img width="557" height="357" alt="螢幕擷取畫面 2026-04-09 155154" src="https://github.com/user-attachments/assets/bc7e43a2-b3ef-4359-be09-e6b3602b4abe" />

我將使用者名單存成文件

<img width="344" height="306" alt="螢幕擷取畫面 2026-04-09 155655" src="https://github.com/user-attachments/assets/d949a6ab-0265-4960-8b80-9b4a9a41ee49" />

<img width="792" height="241" alt="螢幕擷取畫面 2026-04-09 155704" src="https://github.com/user-attachments/assets/5200b39a-d6b7-4c58-b89f-de0a92718824" />

直接嘗試AS-REP Roasting，並未獲得結果

```bash
impacket-GetNPUsers megabank.local/ -usersfile username -format hashcat -no-pass -dc-ip 10.129.228.111 -outputfile asrep_hash
```

<img width="1191" height="283" alt="螢幕擷取畫面 2026-04-09 160050" src="https://github.com/user-attachments/assets/523c330d-0256-4659-b74d-4178e09a6e33" />

繼續針對RPC枚舉，調查本地群組

<img width="522" height="439" alt="螢幕擷取畫面 2026-04-09 161325" src="https://github.com/user-attachments/assets/ce970cd7-e308-48ef-88ae-8c8f4d02b6b4" />

我針對Remote Management Users群組枚舉，獲得一個使用者的詳細訊息

<img width="680" height="573" alt="螢幕擷取畫面 2026-04-09 161339" src="https://github.com/user-attachments/assets/e6c4450c-75f2-4cc6-bf04-eb6d8109570c" />

### 2.3 初始存取(Initial Access)

我沒有其他方法獲取憑證，只能透過相對暴力的手段，首先我嘗試密碼噴灑。

密碼噴灑時，一定要從有關聯的關鍵字所組成的小字典開始，否則流量太大可能被系統阻擋，我準備了兩個字典，一是使用者名稱復用，也就是username:username；第二是使用cupp這個社交工程工具，因為已經了解到mhope的一些資料，我把這些資料做成字典

<img width="801" height="549" alt="螢幕擷取畫面 2026-04-09 162401" src="https://github.com/user-attachments/assets/263f478b-751e-4e4e-a1ee-c75ae2fbab98" />

<img width="591" height="537" alt="螢幕擷取畫面 2026-04-09 163046" src="https://github.com/user-attachments/assets/1cdc44d3-2260-4fc6-9586-4da32ae94084" />

使用nxc做密碼噴灑，nxc的前身是crackmapexec，是一款功能強大的開源網路服務漏洞利用與評估工具，支持多協議的密碼噴灑、哈希傳遞及Kerberos票據攻擊

```bash
nxc smb 10.129.228.111 -u username -p username --continue-on-success
```

<img width="736" height="211" alt="螢幕擷取畫面 2026-04-09 163503" src="https://github.com/user-attachments/assets/03c9032e-b1f8-49cf-a892-e047d5186087" />

<img width="1107" height="271" alt="螢幕擷取畫面 2026-04-09 163515" src="https://github.com/user-attachments/assets/622f04f3-daa6-45e8-bebc-b6763fa3d285" />

找到一組憑證，我將它存成文件

<img width="304" height="147" alt="螢幕擷取畫面 2026-04-09 163550" src="https://github.com/user-attachments/assets/9d43c2f5-7ecf-4190-8035-cafdec7ffc11" />

經過時間同步後我先嘗試Kerberos Roasting，但並未烤出有關聯的SPN帳號

```bash
impacker-GetUserSPNs -dc-ip 10.129.228.111 MEGABANK.LOCAL/SABatchJobs:SABatchJobs
```

<img width="804" height="122" alt="螢幕擷取畫面 2026-04-09 164212" src="https://github.com/user-attachments/assets/75f8a834-b0b0-4f11-9854-8fd21eeaa136" />

使用smbclient嘗試登入成功

```bash
smbclient //10.129.228.111/user$ -U "MEGABANK.LOCAL/SABatchJobs%SABatchJobs"
```

<img width="758" height="268" alt="螢幕擷取畫面 2026-04-09 164318" src="https://github.com/user-attachments/assets/dc16f57c-e682-4009-9ebc-82760b90af39" />



### 2.4 橫向移動(Lateral Movement)

我深入mhope的家目錄搜尋文件，因為他是Remote Management Users群組的成員

<img width="978" height="447" alt="螢幕擷取畫面 2026-04-09 164416" src="https://github.com/user-attachments/assets/b6304d32-7a06-4365-8bc1-6e321e72ded4" />

使用xmlint格式化展開.xml文件

<img width="825" height="367" alt="螢幕擷取畫面 2026-04-09 164448" src="https://github.com/user-attachments/assets/cff5981e-6950-42f2-ba25-f5f8d4d13528" />

將獲得的憑證存成文件

<img width="384" height="165" alt="螢幕擷取畫面 2026-04-09 164557" src="https://github.com/user-attachments/assets/8cb257b7-b353-4a5b-b231-749c4832a081" />

使用evil-winrm連線成功，若是TCP5985/5986有開放，優先使用evil-winrm，因為這款工具隱蔽性和完整性相對較好

<img width="954" height="292" alt="螢幕擷取畫面 2026-04-09 164642" src="https://github.com/user-attachments/assets/ef238a2b-ca5b-4f3c-b653-3e2b355dc8ae" />

獲得user.txt

<img width="594" height="313" alt="螢幕擷取畫面 2026-04-09 164710" src="https://github.com/user-attachments/assets/1514f6f2-46f0-4640-8fe5-c53a44be4f45" />


### 2.5 權限提升(Privilege Escalation)

經過簡單枚舉，我發現當前使用者身處Azure Admins群組，我想嘗試針對這個群組提權，因為這裡經常有密碼殘留或是公開漏洞

<img width="1441" height="328" alt="螢幕擷取畫面 2026-04-09 164902" src="https://github.com/user-attachments/assets/c514ed16-b63c-4d00-8e37-3396a3e39a7a" />

家目錄裡也有相關的資料夾存在，從檔名推斷，應該是本機和AD環境同步的Azure AD Connect服務

<img width="603" height="436" alt="螢幕擷取畫面 2026-04-09 165031" src="https://github.com/user-attachments/assets/5ad7cb3b-f546-4093-8eaf-10c0dfb0783f" />

<img width="763" height="292" alt="螢幕擷取畫面 2026-04-09 165057" src="https://github.com/user-attachments/assets/71c9c7c6-3eb1-4888-b9e3-df5e0275ccc8" />

我確認了Azure AD Connect服務啟動中

<img width="562" height="120" alt="螢幕擷取畫面 2026-04-09 170735" src="https://github.com/user-attachments/assets/95a12df3-b012-4814-bbc7-ce225af91b00" />

也從Program Files中確認了Azure AD Connect

<img width="911" height="583" alt="螢幕擷取畫面 2026-04-09 173452" src="https://github.com/user-attachments/assets/8e7e895b-a22f-4d9f-8574-a1fdfd08b632" />

Azure AD Connect(ADSync)是微軟提供的工具，用於將本地AD的資料（帳號、群組、密碼雜湊）同步到雲端的Azure AD。他會存在一個MSOL帳號，是自動生成的網域帳號，負責從AD中讀取所有變更，權限很大。

漏洞的邏輯是這樣的
1. 資料庫竊取：直接連線到SQL Express，從mms_server_configuration提取解密所需的鹽值與ID。
2. 密文獲取：從mms_management_agent提取加密後的XML配置檔案。
3. 本地解密：利用系統內建的mcrypt.dll調用加密API，結合提取出的鹽值，將加密的組態還原為明文XML。
4. 資訊解析：從解密後的XML中解析出MSOL_帳號的明文密碼。

而這一切因為mhope屬於Azure Admins群組，這意味著擁有解密同步密碼的權限，使攻擊成為可能。

我從網路上尋找PoC腳本，網路上存在的腳本有幾個都需要小修一下

<img width="1233" height="844" alt="螢幕擷取畫面 2026-04-09 174001" src="https://github.com/user-attachments/assets/5a63121a-c1a6-4b4c-be3f-1c7d61a0e07b" />


開啟python伺服器託管腳本

<img width="552" height="67" alt="螢幕擷取畫面 2026-04-09 175444" src="https://github.com/user-attachments/assets/6cdbbc1b-bfa3-4e2f-9251-2a07e1962e89" />

使用腳本，IEX命令是Invoke-Expression的別名，是遠端安裝/執行腳本的命令，PowerShell會解析並執行該字串所代表的命令，實戰中會被AMSI掃瞄並攔截

<img width="933" height="137" alt="螢幕擷取畫面 2026-04-09 175454" src="https://github.com/user-attachments/assets/f1ded211-206a-4f2f-b9ad-07e80a9c16fd" />

使用管理員憑證登入成功

<img width="1184" height="276" alt="螢幕擷取畫面 2026-04-09 175543" src="https://github.com/user-attachments/assets/4606ea7a-fb3c-43a0-b699-8da0e5ec7f78" />


### 2.6 最終成果(Impact)

獲得root.txt

<img width="650" height="429" alt="螢幕擷取畫面 2026-04-09 175659" src="https://github.com/user-attachments/assets/4ca50de8-f9a6-4030-aa29-d4f1897e54a1" />



---

## 3. 學習回顧 (Lessons Learned)

1. 靶機整體難度不高，攻擊鏈清晰，我認為在Easy跟Medium之間。
2. 進行密碼噴灑時一定要優先嘗試弱密碼，不可以一上來就放大字典，會被系統攔截。
3. 提權其實不難，只是不太熟悉Azure，為了先搞懂漏洞原理，花了一點時間閱讀相關的資料。


