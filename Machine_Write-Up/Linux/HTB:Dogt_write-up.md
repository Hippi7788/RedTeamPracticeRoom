## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Dog
2. 難度:Easy
3. IP:10.129.231.223

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmapn掃描開放埠，從詳細掃描跟漏洞腳本掃描可以得到很多目錄
```bash
sudo nmap -sT -Pn --min-rate 5000 10.129.231.223 -oA portscan/ports
sudo nmap -sT -Pn -sV -sC -O -p99,22,80 10.129.231.223 -oA portscan/detail
sudo nmap -sU --top-ports 20 10.129.231.223 -oA portscan/udp
sudo nmap --script=vuln -p22,80 10.129.231.223 -oA portscan/vuln
```

<img width="801" height="243" alt="螢幕擷取畫面 2026-04-21 221511" src="https://github.com/user-attachments/assets/befc8591-24ad-4dc5-b4fc-f1f3152a65a0" />

<img width="1124" height="734" alt="螢幕擷取畫面 2026-04-21 221521" src="https://github.com/user-attachments/assets/e28fe448-3b71-4f03-bc13-2c4ae04002b6" />

<img width="659" height="632" alt="螢幕擷取畫面 2026-04-21 221532" src="https://github.com/user-attachments/assets/51b2b6ac-ac34-4a90-baa2-a130dfeb6b4e" />

<img width="863" height="760" alt="螢幕擷取畫面 2026-04-21 222018" src="https://github.com/user-attachments/assets/561bce4d-f331-40dc-aa4e-b1712b5e2c70" />

使用whatweb了解80/HTTP的訊息，可得特殊標頭

```bash
whatweb http://10.129.231.223
```

<img width="1899" height="105" alt="螢幕擷取畫面 2026-04-21 221941" src="https://github.com/user-attachments/assets/9773e060-b7eb-4b61-bcf8-02f2d0a7d44a" />

觀看HTTP主頁與各分頁，以及原始碼

<img width="1546" height="855" alt="螢幕擷取畫面 2026-04-21 222029" src="https://github.com/user-attachments/assets/d346cebb-e5ec-4836-b8c1-002bc7b785b5" />

<img width="769" height="384" alt="螢幕擷取畫面 2026-04-21 222218" src="https://github.com/user-attachments/assets/3f2cbc79-ac79-497c-ac60-d84704d21ab5" />

<img width="1262" height="718" alt="螢幕擷取畫面 2026-04-21 222243" src="https://github.com/user-attachments/assets/07b3f743-d9c5-4f35-b530-a9c699fa484d" />

觀看robots.txt，robots.txt是一個存放在網站根目錄下的純文字檔案，專門用來告訴搜尋引擎爬蟲（如 Googlebot）網站中哪些頁面可以抓取，哪些則禁止存取

<img width="671" height="823" alt="螢幕擷取畫面 2026-04-21 222458" src="https://github.com/user-attachments/assets/70aead9d-4b66-4e98-a7ac-192cf30371c2" />


### 2.2 枚舉(Enumeration)

使用Gobuster枚舉目錄

```bash
sudo gobuster dir -u http://10.129.231.223 -w /usr/share/wordlists/dirb/common.txt
```

<img width="800" height="654" alt="螢幕擷取畫面 2026-04-21 222553" src="https://github.com/user-attachments/assets/230296e2-93ee-4394-9906-e56a4c78e2f8" />

探索/.git目錄和其他目錄

<img width="886" height="663" alt="螢幕擷取畫面 2026-04-21 222606" src="https://github.com/user-attachments/assets/26d4fec8-89d9-4200-b520-584e2c6bdeb9" />

<img width="799" height="526" alt="螢幕擷取畫面 2026-04-21 223401" src="https://github.com/user-attachments/assets/e13b4c76-2571-49d8-a7d5-8b3c4e1c9a6a" />

<img width="794" height="461" alt="螢幕擷取畫面 2026-04-21 223537" src="https://github.com/user-attachments/assets/b62b6734-17b8-4a6d-9a7b-0d445bcad511" />

使用git-dumper下載/.git的內容

```bash
git-dumper http://10.129.231.223/.git/ git_dump/
```

<img width="549" height="94" alt="螢幕擷取畫面 2026-04-21 223217" src="https://github.com/user-attachments/assets/d6c99330-9def-4cef-9d5e-19e815080a46" />

<img width="917" height="146" alt="螢幕擷取畫面 2026-04-21 223621" src="https://github.com/user-attachments/assets/488f8512-cea8-464f-87ee-a813c733878b" />

探索文件後，在setting.php中找到一組資料庫用憑證

<img width="796" height="499" alt="螢幕擷取畫面 2026-04-21 223844" src="https://github.com/user-attachments/assets/fddc433d-dfe9-4032-a51d-82cb1e9a3eda" />

但這個憑證不能登入HTTP，有可能是
1. 使用者名稱不同但密碼重用
2. 資料庫專用
3. 兔子洞
通常密碼重用的可能性很高，可以先找找看使用者名稱

<img width="891" height="588" alt="螢幕擷取畫面 2026-04-21 224116" src="https://github.com/user-attachments/assets/fa54992b-c460-4f86-b650-992e1d48be05" />

從testing.info找到Backdrop CMS的版本訊息

<img width="857" height="302" alt="螢幕擷取畫面 2026-04-21 224318" src="https://github.com/user-attachments/assets/06cd4497-11b6-47b3-94a5-e34df6659943" />

也找到相關的公開漏洞訊息

<img width="1002" height="439" alt="螢幕擷取畫面 2026-04-21 224952" src="https://github.com/user-attachments/assets/e2c7481f-699e-4ede-b5d1-ba83284838d5" />

這個腳本可以幫忙生成壓縮包，獲得HTTP控制權後可以上傳並呼叫，獲得WebShell，但前提是要有控制權，所以還是要找使用者名稱

<img width="928" height="157" alt="螢幕擷取畫面 2026-04-21 225032" src="https://github.com/user-attachments/assets/cb970fb9-04d0-44d2-9e94-d8dfaf865192" />

我用grep試圖找找，我發現網站可以透過Email登入，在/About中了解到xxx@dog.htb這個後綴

<img width="1287" height="256" alt="螢幕擷取畫面 2026-04-21 230156" src="https://github.com/user-attachments/assets/9f2ed8cb-63a1-4e25-9a0e-4ab8d119bb58" />

使用grep順利找到一個疑似可用的名稱

<img width="1907" height="159" alt="螢幕擷取畫面 2026-04-21 230226" src="https://github.com/user-attachments/assets/49041be3-b569-4edd-a3eb-fb1024ee7203" />


### 2.3 初始存取(Initial Access)

我使用這個名稱，加上剛才的資料庫憑證的密碼順利登入網站

<img width="1915" height="807" alt="螢幕擷取畫面 2026-04-21 230351" src="https://github.com/user-attachments/assets/2ebcd4b4-18a9-4151-8af0-01ade97da009" />

根據漏洞腳本的說明，我找到上傳的頁面

<img width="1431" height="822" alt="螢幕擷取畫面 2026-04-21 230442" src="https://github.com/user-attachments/assets/91655f44-21af-448e-8e54-3ee921a18a3b" />

<img width="1360" height="746" alt="螢幕擷取畫面 2026-04-21 230705" src="https://github.com/user-attachments/assets/d58d4ad2-13e0-45b5-9818-7ea89792e863" />

但我發現腳本給我的格式不對，網站有限制副檔名

<img width="688" height="730" alt="螢幕擷取畫面 2026-04-21 231026" src="https://github.com/user-attachments/assets/082a0d46-db57-4937-bf93-0500b6c5aef0" />

我檢查了zip檔的內容，就是在shell這個目錄裡的兩個文件

```bash
zipinfo shell.zip
```

<img width="641" height="300" alt="螢幕擷取畫面 2026-04-21 231444" src="https://github.com/user-attachments/assets/6fa64602-9703-4d1c-b58d-6bd16a8259a1" />

我重新把shell目錄做成tar檔

```bash
tar -cvf shell.tar shell
tar -tvf shell.tar
```
-c (create)：建立新的歸檔檔案。
-v (verbose)：在螢幕上顯示正在處理的檔案清單，讓你了解進度。
-f (file)：指定要建立的歸檔檔案名稱（注意：-f 之後必須緊接檔名）。
-x (extract)：解開檔案。
-t (list) ：列出歸檔檔案的內容。
-z：使用gzip壓縮，副檔名通常為 .tar.gz。
-j：使用bzip2壓縮，副檔名通常為 .tar.bz2。

<img width="642" height="228" alt="螢幕擷取畫面 2026-04-21 231656" src="https://github.com/user-attachments/assets/480befdd-2ec2-4204-bfc7-9db996ceb041" />

重新上傳後跳出成功提示

<img width="1006" height="750" alt="螢幕擷取畫面 2026-04-21 232438" src="https://github.com/user-attachments/assets/f4e20ecc-6beb-44eb-9ba9-1d1f0c127f8d" />

<img width="1105" height="523" alt="螢幕擷取畫面 2026-04-21 233213" src="https://github.com/user-attachments/assets/3e2cb2ac-5ca1-47d5-b89e-5b826d68a262" />

根據腳本提示找到WebShell，這個WebShell持續時間很短，若被重製，可以重新上傳後重新整理

<img width="1056" height="410" alt="螢幕擷取畫面 2026-04-21 233451" src="https://github.com/user-attachments/assets/1a6950ad-a2fa-4991-bf61-a5c7402aca99" />

開啟nc監聽
```bash
nc -lvnp 1234
```

<img width="352" height="186" alt="螢幕擷取畫面 2026-04-21 233612" src="https://github.com/user-attachments/assets/b340a692-6319-4202-85e8-70916b27b868" />

在WebShell連結反彈Shell到nc監聽

<img width="1023" height="113" alt="螢幕擷取畫面 2026-04-21 234128" src="https://github.com/user-attachments/assets/99c88ec1-fdca-47c8-a5f4-117eb470acf4" />

獲得系統訪問權

<img width="741" height="214" alt="螢幕擷取畫面 2026-04-21 234048" src="https://github.com/user-attachments/assets/0e7c34e3-d88e-469e-856f-e71b2699f2f2" />

### 2.4 橫向移動(Lateral Movement)

在家目錄中有兩個使用者

<img width="534" height="138" alt="螢幕擷取畫面 2026-04-21 234227" src="https://github.com/user-attachments/assets/8617e224-bbcc-4436-9b43-21239e11fa36" />

查看/etc/passwd也是這兩個使用者擁有bash

<img width="815" height="761" alt="螢幕擷取畫面 2026-04-21 234253" src="https://github.com/user-attachments/assets/2ddec980-2ebe-4f3e-9884-d96a13ddb30c" />

嘗試密碼重用，成功獲得使用者shell

擁有密碼的情況下一定要嘗試密碼重用，有時會有奇效

<img width="349" height="120" alt="螢幕擷取畫面 2026-04-21 234355" src="https://github.com/user-attachments/assets/960fcc44-4999-40b7-af2a-a4d7af236986" />

使用ssh連接，獲得更好的交互性

<img width="756" height="792" alt="螢幕擷取畫面 2026-04-21 234505" src="https://github.com/user-attachments/assets/09d3039b-a3cc-4825-8449-6251bb584032" />

獲得user.txt

<img width="338" height="199" alt="螢幕擷取畫面 2026-04-21 234536" src="https://github.com/user-attachments/assets/d4ab2ee7-9f25-4b46-822b-e91b69c2910b" />



### 2.5 權限提升(Privilege Escalation)

查看sudo權限，使用者可以無密碼使用bee工具

<img width="1065" height="146" alt="螢幕擷取畫面 2026-04-21 234602" src="https://github.com/user-attachments/assets/4ad52af0-676e-4d05-b1e1-f0ac3dae83da" />

bee是專為Backdrop CMS開發的命令行工具，類似於Drupal中的Drush，主要功能：
1. 執行Cron任務。
2. 清除緩存。
3. 下載並安裝Backdrop核心。
4. 下載、啟用或禁用模組和主題。
5. 查看網站資訊及可用更新。

bee可以配合eval參數跑PHP程式碼，但必須要在BackdropCMS的安裝目錄下，也可以使用--root參數設定目錄

<img width="1697" height="374" alt="螢幕擷取畫面 2026-04-21 235101" src="https://github.com/user-attachments/assets/6b0317a1-db9b-4772-897d-bbdc2de05b57" />

<img width="743" height="97" alt="螢幕擷取畫面 2026-04-21 235118" src="https://github.com/user-attachments/assets/73625ec1-ee9b-43d2-abc7-23ec35be7da4" />

<img width="1110" height="768" alt="螢幕擷取畫面 2026-04-21 234859" src="https://github.com/user-attachments/assets/b5139b7e-eab4-4f73-9f23-7d55af442d0c" />

先移動到/var/www/html目錄，或使用--root設定路徑，再使用sudo權限配合eval參數，用PHP構建提權邏輯
```bash
cd /var/www/html
sudo bee eval 'system("bash")'
```

<img width="576" height="85" alt="螢幕擷取畫面 2026-04-21 234959" src="https://github.com/user-attachments/assets/310960ba-b9c9-4e36-95d5-65366a91f3bb" />


### 2.6 最終成果(Impact)

獲得root.txt

<img width="945" height="480" alt="螢幕擷取畫面 2026-04-21 235657" src="https://github.com/user-attachments/assets/d354d044-8a72-42c7-8db4-6288f29a3064" />



---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
