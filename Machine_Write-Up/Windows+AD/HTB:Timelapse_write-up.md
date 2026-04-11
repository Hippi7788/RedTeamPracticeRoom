## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Timelapse
2. 難度:Easy
3. IP:10.129.227.113

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，並沒有除了AD環境中常用的以外的特殊服務，要注意TCP5986(WinRM)是開放的，這是5985的SSL版，後續在使用evil-winrm連線時可能要使用-S做處理

並且，在詳細訊息掃描中有暴露出兩個域名

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.227.113 -oA portscan/ports
sudo nmap -sU -Pn --top-ports 20 10.129.227.113 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.227.113 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.227.113 -oA portscan/vuln
```

<img width="691" height="535" alt="螢幕擷取畫面 2026-04-10 214350" src="https://github.com/user-attachments/assets/91d02d24-d32c-49ed-9b2b-de48b321a7d4" />

<img width="634" height="577" alt="螢幕擷取畫面 2026-04-10 214357" src="https://github.com/user-attachments/assets/aa2a5ec2-5a5a-411f-88e2-ca17b10a02f4" />

<img width="1051" height="762" alt="螢幕擷取畫面 2026-04-10 214847" src="https://github.com/user-attachments/assets/9ff75973-99d1-4329-8763-4993e6076909" />

<img width="1270" height="656" alt="螢幕擷取畫面 2026-04-10 214903" src="https://github.com/user-attachments/assets/b9ae5fe1-9534-458b-a23c-e7067e5f50c1" />


我用了文書處理工具整理開放埠

```bash
ports=$(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',')
```

<img width="784" height="209" alt="螢幕擷取畫面 2026-04-10 214518" src="https://github.com/user-attachments/assets/1544816a-136f-42c1-816d-152ac2b05aa5" />

優先處理時間同步，面對AD環境滲透時，時間同步很重要，若是在進行Kerberoas等與時間戳相關的協議的攻擊的話，時間差的容忍度只有不到五分鐘

因為UDP123有開放，我使用ntpdate時間同步，NTP比較不會被管理員監控，從這裡走比較隱蔽

```bash
sudo ntpodate 10.129.227.113
```

<img width="800" height="103" alt="螢幕擷取畫面 2026-04-10 215053" src="https://github.com/user-attachments/assets/12fbddd6-0e40-4d7f-8f67-a44f136b9bff" />


將掃描到的域名寫入/etc/hosts

<img width="530" height="247" alt="螢幕擷取畫面 2026-04-10 215157" src="https://github.com/user-attachments/assets/65a3be01-1c72-4cd0-9c96-9d7deab3843f" />



### 2.2 枚舉(Enumeration)

使用smbclient匿名枚舉SMB協議，這個工具模擬現實使用者連線，相對來說比較正常

```bash
smbclient -L //10.129.227.113 -N
```

<img width="824" height="280" alt="螢幕擷取畫面 2026-04-10 215259" src="https://github.com/user-attachments/assets/38f959d4-7448-4fdc-a7a9-1bf597f4ba83" />

SMB協議暴露出若干資料夾，嘗試連接後只有Shares允許我讀

```bash
smbclient //10.129.227.113/shares -N
```

<img width="744" height="239" alt="螢幕擷取畫面 2026-04-10 220020" src="https://github.com/user-attachments/assets/396b1560-2577-46d9-9840-c05cf0002623" />

將其中的所有資料都提取出來

<img width="711" height="198" alt="螢幕擷取畫面 2026-04-10 220138" src="https://github.com/user-attachments/assets/ffff98f3-0bb9-4c32-97d2-a78e87f4fae6" />

<img width="1062" height="198" alt="螢幕擷取畫面 2026-04-10 220213" src="https://github.com/user-attachments/assets/ef97a325-d84d-4f1e-b780-ed7c2b24ac89" />

### 2.3 初始存取(Initial Access)

其中的word檔是關於LASP的說明文件，並沒有透漏什麼訊息，我嘗試解壓縮關於winrm的備份資料，但需要密碼

<img width="520" height="108" alt="螢幕擷取畫面 2026-04-10 220453" src="https://github.com/user-attachments/assets/232a2562-67e3-413d-9215-4f6ce618291e" />

我使用zip2john將zip檔的哈希提取出來，*2john是John the Ripper的輔助工具，使用*2john兩階段破解是在破解密碼的領域裡是穩定性和效率最好的方式之一。

當一個檔案被加密時，系統會根據使用者設定的密碼，透過特定的演算法（如傳統的PKZIP串流加密或更現代的AES加密）產生一段加密數據。zip2john會掃描ZIP檔案的標頭和元數據，找出儲存在其中的加密參數，並將這些複雜的二進位數據轉換成一種John the Ripper能識別的文字格式（$zip2$）。

```bash
zip2john winrm_backup.zip > winrm_backup.zip.hash
```

<img width="1359" height="589" alt="螢幕擷取畫面 2026-04-10 220600" src="https://github.com/user-attachments/assets/a110c236-90ea-4032-897a-11b1e7eef443" />

使用john破解密碼哈希，得到明文密碼

<img width="919" height="202" alt="螢幕擷取畫面 2026-04-10 220705" src="https://github.com/user-attachments/assets/df6700ba-f84d-44f2-a9e4-32c644d1c05b" />

使用密碼獲得pfx檔，PFX是一種副檔名為.pfx或.p12的加密檔案格式，用於在單一檔案中捆綁數位憑證、私密金鑰和證書鏈。它基於PKCS#12標準，常由Microsoft IIS等系統使用，透過密碼保護在不同伺服器間安全傳輸個人身分。通常使用OpenSSL為轉換工具。

<img width="670" height="192" alt="螢幕擷取畫面 2026-04-10 220814" src="https://github.com/user-attachments/assets/1ca1c50a-2439-42e5-89cb-20c6b35b36a4" />

可以在網路上搜到利用方法

<img width="868" height="598" alt="螢幕擷取畫面 2026-04-10 221120" src="https://github.com/user-attachments/assets/0570612f-2655-440a-a7f3-63bb0cf28702" />


<img width="1569" height="823" alt="螢幕擷取畫面 2026-04-10 221220" src="https://github.com/user-attachments/assets/20313a79-e383-41ce-9870-85e0bb3d1369" />

嘗試使用openssl提取密鑰，但這個pfx檔也設置了密碼

<img width="771" height="93" alt="螢幕擷取畫面 2026-04-10 221354" src="https://github.com/user-attachments/assets/5d11f70f-93de-421a-acf7-da242ac6c30b" />

我再次使用pfx2john提取哈希，嘗試破解密碼

<img width="790" height="583" alt="螢幕擷取畫面 2026-04-10 221454" src="https://github.com/user-attachments/assets/c394daa2-a960-407a-9df2-f67e0957b0c5" />

<img width="897" height="239" alt="螢幕擷取畫面 2026-04-10 221700" src="https://github.com/user-attachments/assets/a46b7eee-0eb8-4a01-b603-03bbdd9a5c63" />

成功提取金鑰，並且我設置了簡單的PEM密碼為1234，-nocerts是不提取證書的意思

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacyy_dev_auth.pfx.key
```

<img width="762" height="113" alt="螢幕擷取畫面 2026-04-10 221801" src="https://github.com/user-attachments/assets/08634909-137a-491e-b15f-8b145f7268f9" />

<img width="612" height="763" alt="螢幕擷取畫面 2026-04-10 221818" src="https://github.com/user-attachments/assets/e410c1cf-519a-4fb5-a9c7-47cd744e8afe" />

我將金鑰匯出成無密碼版本

```bash
openssl rsa -in legacyy_dev_auth.pfx.key -out legacyy_dev_auth.pfx.key.rsa
```

<img width="720" height="677" alt="螢幕擷取畫面 2026-04-10 222309" src="https://github.com/user-attachments/assets/2e31f31b-e192-4c77-9a2d-5caa24cfac91" />

繼續使用openssl提取證書，-clcerts -nokeys是提取證書、不要金鑰的意思，可以使用--help了解

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacyy_dev_auth.pfx.cer
```

<img width="827" height="571" alt="螢幕擷取畫面 2026-04-10 222618" src="https://github.com/user-attachments/assets/983329a9-8566-43ab-933d-189ba6ecbed2" />

使用evil-winrm嘗試登入，記得使用-S利用SSL加密，在這裡的截圖可以顯示我用錯檔案，不是用不需要密碼的.rsa，而是用最初設定了PEM密碼為1234的.key，所以後續經常會看到evil-winrm一直要求我輸入密碼1234，這是我的失誤，因為我的網路其實不太穩，我不太願意重連，所以就將錯就錯用下去了

<img width="1180" height="322" alt="螢幕擷取畫面 2026-04-10 222810" src="https://github.com/user-attachments/assets/dece639f-442d-4b2f-adc7-c15fa91d5990" />

獲得user.txt

<img width="551" height="296" alt="螢幕擷取畫面 2026-04-10 222917" src="https://github.com/user-attachments/assets/5a0e1acd-d0b2-491d-95b2-492e9da57de0" />


### 2.4 橫向移動(Lateral Movement)

我簡單的檢查了當前使用者，發現使用者屬於Development組，只要是在這個組裡，一定要優先檢查歷史紀錄，有機會獲得未清理乾淨的明文憑證

<img width="670" height="739" alt="螢幕擷取畫面 2026-04-10 223127" src="https://github.com/user-attachments/assets/5c239f3e-6129-48c8-96b8-ac7fd884e06b" />

檢查歷史紀錄，獲得一組明文憑證

<img width="1305" height="224" alt="螢幕擷取畫面 2026-04-10 223603" src="https://github.com/user-attachments/assets/cf71790b-d368-470e-8d5f-657d8d0037a9" />

將憑證記錄到文件中

<img width="365" height="137" alt="螢幕擷取畫面 2026-04-10 223704" src="https://github.com/user-attachments/assets/69d111b8-777b-4e55-98fa-e6c8a0bf063d" />

使用憑證橫向移動成功

<img width="1181" height="305" alt="螢幕擷取畫面 2026-04-10 223838" src="https://github.com/user-attachments/assets/0ccd5d3a-eb8c-411d-83bc-6975314bd875" />



### 2.5 權限提升(Privilege Escalation)

簡單檢查了使用者，發現這個使用者屬於LAPS_Readers組

<img width="653" height="729" alt="螢幕擷取畫面 2026-04-10 223951" src="https://github.com/user-attachments/assets/247ce991-2823-47ff-8836-58f404c6caf9" />

LAPS是由 Microsoft提供的安全解決方案，專門用於自動化、管理與保護已加入網域之電腦的本機管理員帳戶密碼，LAPS會定期隨機生成每台受控電腦的本機管理員密碼，並將其加密儲存在AD中該電腦物件的屬性中，透過讓每台電腦擁有唯一的本機管理員密碼，有效防止攻擊者利用一台受害電腦的憑據橫向入侵網域內的其他電腦。

而使用者屬於LAPS_Readers組，該使用者將具備閱讀受控電腦密碼的權限，微軟官方說明也有關於這個機制的介紹

<img width="962" height="794" alt="螢幕擷取畫面 2026-04-10 224214" src="https://github.com/user-attachments/assets/5aea1b45-de75-4c60-8e09-cbd85b9f3489" />

只是當前靶機好像沒有微軟所介紹的指令

<img width="972" height="181" alt="螢幕擷取畫面 2026-04-10 224410" src="https://github.com/user-attachments/assets/196ae6cc-4482-4fc0-8d01-2301ba47500e" />

雖然指令不存在，但這個機制和權限是存在的，所以我只要檢查這台機器的所有屬性，其中應該會有當前使用者能讀的密碼

<img width="1208" height="758" alt="螢幕擷取畫面 2026-04-10 224905" src="https://github.com/user-attachments/assets/5deec0d0-79b9-4436-b704-653f19e7e0d7" />

存在ms-Mcs-AdmPwd，因為當前使用者已經在LAPS_Readers組裡了，若是不能讀，也可以利用LAPS_Readers組的權限透過Set-AdmPwdReadPasswordPermission指令將讀取ms-Mcs-AdmPwd的權限委派給特定的OU

<img width="862" height="377" alt="螢幕擷取畫面 2026-04-10 224926" src="https://github.com/user-attachments/assets/ebfc099b-9cbe-4836-8d89-7e8cd3094d41" />

使用明文密碼登入管理員憑證

<img width="1187" height="303" alt="螢幕擷取畫面 2026-04-10 225057" src="https://github.com/user-attachments/assets/2ba5977b-5073-4d01-b50f-380049935065" />



### 2.6 最終成果(Impact)

獲得root.txt

<img width="644" height="403" alt="螢幕擷取畫面 2026-04-10 225248" src="https://github.com/user-attachments/assets/7ca60318-8b77-49dd-b8f0-225dbff15c32" />



---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. Development組一定要優先查閱歷史資料。
2. 一邊查閱資料，一邊利用剛學到的知識，讓我學到很多。

### 浪費時間的部分：

1. 閱讀資料時間有點長，這台靶機並沒有特別複雜。

### 新知識點：

1. PFX是我不太熟的領域，一邊搜索相關知識，一邊攻擊，讓我學到很多。
2. LAPS相關以前在TryHackMe學過，但利用是第一次，所以我也是一邊查閱資料，一邊打。

### 與實戰對應：

1. 我認為做滲透測試不可能碰到的服務或工具都是剛好知道的，所以查閱資料、現場學習的能力非常重要。
2. 如果是工具，就要學會看--help；如果是服務或協議，要學會Google看官方說明。

