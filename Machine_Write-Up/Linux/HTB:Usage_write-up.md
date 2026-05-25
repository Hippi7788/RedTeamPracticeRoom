## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Usage
2. 難度:Easy
3. IP:10.129.178.188

### 主要發現：
1. 漏洞 ：利用密碼重設功能存在錯誤盲注獲得網頁管理員憑證（風險等級：High）
2. 漏洞 ：利用舊版本公開漏洞任意檔案上傳導致遠端命令執行（風險等級：Critical）
3. 漏洞 ：.monitrc配置檔案敏感資訊明文洩露導致橫向移動（風險等級：High）
4. 漏洞 ：Sudo權限配置不當與7za通配符參數注入獲得root權限（風險等級：High）

### 攻擊鏈摘要：

透過配置/etc/hosts解析出admin隱藏後台，利用密碼重設處的SQL盲注破解出後台管理員密碼，登入Laravel-Admin後台。透過Burp Suite篡改上傳副檔名繞過檢查，成功上傳PHP Webshell並執行反彈Shell建立初始會話。讀取隱藏檔.monitr 內的明文密碼，藉由SSH憑證對撞成功切換至另一名用戶，並分析sudo特權可執行檔，利用7za *的萬用字元參數注入漏洞，繞過安全性限制成功導出id_rsa金鑰晉升為root權限。

### 潛在影響：

1. 完全失去主機控制權：攻擊者取得最高管理員（root）權限與 SSH 私鑰，可自由存取、修改或刪除系統內的所有檔案與日誌。
2. 敏感資料外洩：資料庫內的所有帳號雜湊遭導出，可能導致更廣泛的內部跨平台憑證對撞攻擊。
3. 供應鏈與內部網路跳板：受駭伺服器可被用作攻擊企業內部網路（Intranet）其他機敏資產的跳板。


### 修復建議：

1. 全面將資料庫查詢改為參數化查詢，禁止直接拼接SQL指令，以根除SQLi。
2. 立即將laravel-admin升級至最新的安全修補版本，並加強後端檔案上傳的檢查機制。
3. 定期使用強雜湊演算法重新將密碼加密，並實施強密碼複雜度策略。
4. 嚴格遵循最小權限原則，禁止在設定檔中明文寫入密碼。
5. 審查/etc/sudoers配置，限縮免密碼執行特定指令的範圍。
6. 在腳本或程式中呼叫外部指令時，嚴禁使用萬用字元 *。應改用絕對路徑、明確的檔案清單，或在參數結尾加上--符號防止檔案名被解析為系統參數。


---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.178.188 -oA portscan/ports
sudo nmap -sU --top-ports 20 19.129.178.188 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p99,80,22 10.129.1781.88 -oA portscan/detial
sudo nmap --script=vuln -p22,80 10.129.178.188 -oA portscan/vuln
```

<img width="686" height="256" alt="螢幕擷取畫面 2026-05-22 215322" src="https://github.com/user-attachments/assets/178cc4c9-050c-483a-9a99-9051a8fb4f8c" />

<img width="608" height="590" alt="螢幕擷取畫面 2026-05-22 215308" src="https://github.com/user-attachments/assets/0192473b-204d-4e4a-8170-2ee935b99aaa" />

<img width="1120" height="474" alt="螢幕擷取畫面 2026-05-22 215455" src="https://github.com/user-attachments/assets/4f9dc4b9-6182-468c-be80-e297776af969" />


<img width="624" height="301" alt="螢幕擷取畫面 2026-05-22 220013" src="https://github.com/user-attachments/assets/b1d253bd-e29b-45ef-854b-ed001004d28a" />

將詳細訊息掃描中得到的子域名寫進/etc/hosts

<img width="522" height="244" alt="螢幕擷取畫面 2026-05-22 215547" src="https://github.com/user-attachments/assets/81dbf7cd-87b2-4ee0-a176-18738f579f30" />

使用whatweb調查web的訊息

```bash
whatweb 10.129.178.188
```

<img width="1915" height="161" alt="螢幕擷取畫面 2026-05-22 215731" src="https://github.com/user-attachments/assets/7ae466b1-d472-44cd-991a-cf80fae73b1d" />

web主頁是登入頁

<img width="1465" height="665" alt="螢幕擷取畫面 2026-05-22 215759" src="https://github.com/user-attachments/assets/573ccdf5-3077-41e1-bd4e-877e6438923d" />

從原始碼來看貌似有些資料庫查詢的語句，可以往注入去考慮

<img width="849" height="472" alt="螢幕擷取畫面 2026-05-22 215939" src="https://github.com/user-attachments/assets/e1ec4b5b-650b-4627-807c-c688c1d4c4b0" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破子目錄

```bash
sudo gobuster dir -u http://usage.htb -w /usr/share/wordlists/dirb/common.txt --exclude-length 206,162
```

<img width="982" height="512" alt="螢幕擷取畫面 2026-05-22 220528" src="https://github.com/user-attachments/assets/a068bd22-1b6d-4933-b138-9962f36d6f91" />

如果創建新帳號再登入會進入靜態頁面

<img width="1385" height="854" alt="螢幕擷取畫面 2026-05-22 221136" src="https://github.com/user-attachments/assets/84c16fc5-4bc7-48c2-9fa4-5dc7b1d24451" />

將admin.usage.htb加入/etc/hosts將會看到另一個登入頁

<img width="512" height="274" alt="螢幕擷取畫面 2026-05-22 222814" src="https://github.com/user-attachments/assets/5b8b3e92-8d54-4740-8a32-61fc20876201" />

<img width="1023" height="645" alt="螢幕擷取畫面 2026-05-22 222825" src="https://github.com/user-attachments/assets/6e3c08af-2748-4784-90fc-52ad747dda1c" />

### 2.3 初始存取(Initial Access)

找到重設帳號處對SQLi Payload有反應

<img width="1290" height="346" alt="螢幕擷取畫面 2026-05-22 222952" src="https://github.com/user-attachments/assets/9f9afdfc-63d3-46ec-b54d-01169bd64cac" />

<img width="1320" height="534" alt="螢幕擷取畫面 2026-05-22 222903" src="https://github.com/user-attachments/assets/3e231212-0896-44d7-adb6-77070641a60d" />

<img width="1346" height="459" alt="螢幕擷取畫面 2026-05-22 222926" src="https://github.com/user-attachments/assets/12354f85-8cea-4ae0-b88a-3b85b629ec2f" />

<img width="1350" height="447" alt="螢幕擷取畫面 2026-05-22 223350" src="https://github.com/user-attachments/assets/7d296cae-1f88-4dc8-8564-760c84ebd540" />

這裡是基於錯誤的盲注，直接上sqlmap

```bash
sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch
sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch --dbs
sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch -D usage_blog --tables
sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch -D usage_blog -T admin_users --dump
```

<img width="785" height="282" alt="螢幕擷取畫面 2026-05-22 230354" src="https://github.com/user-attachments/assets/8f09b521-594d-47d8-b111-9c9ba8b96785" />

<img width="1040" height="258" alt="螢幕擷取畫面 2026-05-22 231110" src="https://github.com/user-attachments/assets/09d2683a-27f9-4b1e-a783-17c9899fcf95" />

導出管理員密碼哈希後存成文件，使用john破解

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="573" height="143" alt="螢幕擷取畫面 2026-05-22 231342" src="https://github.com/user-attachments/assets/fd6ffdd1-2dc7-4829-80e3-2e8b7716fc63" />

<img width="842" height="232" alt="螢幕擷取畫面 2026-05-22 231437" src="https://github.com/user-attachments/assets/2e2ff050-aa0e-4da0-8f2b-2bb0177f3625" />

登入成功

<img width="1919" height="851" alt="螢幕擷取畫面 2026-05-22 231902" src="https://github.com/user-attachments/assets/34cee566-fac3-416a-8550-ef006f1329e8" />

管理頁面是基於laravel-admin 1.8.17，可以找到公開的檔案上傳漏洞，再上傳管理員大頭貼時可以擷取請求，並且更換副檔名為php腳本，實現遠端程式碼執行

<img width="1154" height="433" alt="螢幕擷取畫面 2026-05-22 232330" src="https://github.com/user-attachments/assets/f0b75950-28fa-449a-98e0-31cd3b6baad2" />

<img width="1709" height="712" alt="螢幕擷取畫面 2026-05-22 232405" src="https://github.com/user-attachments/assets/1a375e65-4667-4aac-b0d6-3fe4e03f9c88" />

<img width="1769" height="736" alt="螢幕擷取畫面 2026-05-22 232712" src="https://github.com/user-attachments/assets/e72b7115-eae9-48c5-b9d7-5397c98cc0e9" />

建立webshell，並且更名為shell.php.jpg

<img width="316" height="132" alt="螢幕擷取畫面 2026-05-23 000225" src="https://github.com/user-attachments/assets/80d10721-41e6-4ab8-9c10-39685e37be30" />

找到大頭貼上傳頁面

<img width="1363" height="867" alt="螢幕擷取畫面 2026-05-22 233426" src="https://github.com/user-attachments/assets/5353102a-2e5c-4a59-a6a4-00e3d5c3ef0e" />

上傳shell.php.jpg，並且使用Burp Suite攔截POST請求

<img width="1904" height="768" alt="螢幕擷取畫面 2026-05-23 000814" src="https://github.com/user-attachments/assets/34f01592-4f29-4a5f-bc30-5c6c13047a05" />

將POST請求中的檔名更改為shell.php

<img width="1228" height="311" alt="螢幕擷取畫面 2026-05-23 001007" src="https://github.com/user-attachments/assets/4b10c6ec-2b6a-4982-9949-dd8a204adfd5" />

對著圖中紅圈處右鍵開啟新標籤可以直達呼叫頁面

<img width="1904" height="768" alt="螢幕擷取畫面 2026-05-23 000814" src="https://github.com/user-attachments/assets/8d41375e-f244-450e-9b3d-68ded242b5a7" />

PoC成功

<img width="1034" height="331" alt="螢幕擷取畫面 2026-05-23 001355" src="https://github.com/user-attachments/assets/f567d0a9-72ce-4b61-968d-caa65e28619e" />

開啟本地監聽，使用rlwrap增強交互性

<img width="321" height="167" alt="螢幕擷取畫面 2026-05-22 233950" src="https://github.com/user-attachments/assets/54ae63e1-24ca-44c7-a464-06ba6987b848" />

注入反彈shell payload

<img width="1178" height="355" alt="螢幕擷取畫面 2026-05-23 001447" src="https://github.com/user-attachments/assets/e29aa829-a38a-42b2-9e5f-5d34c17a4fdb" />

獲得初始控制權

<img width="861" height="301" alt="螢幕擷取畫面 2026-05-23 001502" src="https://github.com/user-attachments/assets/c2bb880b-890b-4b88-a765-275dad9aac20" />

獲得user.txt

<img width="649" height="305" alt="螢幕擷取畫面 2026-05-23 001534" src="https://github.com/user-attachments/assets/09a52f94-abba-402b-9985-800bd06c20b1" />

### 2.4 橫向移動(Lateral Movement)

在家目錄下有許多隱藏檔案，Monit是一個小型開源實用程序，用於管理和監控Unix系統。Monit可以執行自動維護和修復，並在出現錯誤時執行有意義的因果操作。

<img width="625" height="327" alt="螢幕擷取畫面 2026-05-23 001702" src="https://github.com/user-attachments/assets/7fd1f908-8173-4a78-a2e4-ed9029ef3db2" />

在.monitrc檔案中發現疑似明文密碼，先儲存起來

<img width="593" height="518" alt="螢幕擷取畫面 2026-05-23 001737" src="https://github.com/user-attachments/assets/5291de63-d7f8-43ce-b49c-fc7fc18091f7" />

<img width="273" height="145" alt="螢幕擷取畫面 2026-05-23 001926" src="https://github.com/user-attachments/assets/1f5a230e-7dd8-4963-9197-51438b5669d9" />

對撞ssh成功

<img width="668" height="793" alt="螢幕擷取畫面 2026-05-23 001857" src="https://github.com/user-attachments/assets/d2aabb03-c105-457d-a2e6-fc2ec254d466" />

### 2.5 權限提升(Privilege Escalation)

查詢sudo權力，可無須密碼使用某檔案

<img width="1157" height="125" alt="螢幕擷取畫面 2026-05-23 001955" src="https://github.com/user-attachments/assets/ee248498-02f9-4ce2-a528-de7235fcc091" />

這是一個可執行檔

<img width="1902" height="68" alt="螢幕擷取畫面 2026-05-23 002053" src="https://github.com/user-attachments/assets/0ba31143-49be-4337-ae02-849b84622e7d" />

以sudo權限執行後會讓使用者選擇3個選項，可以直接輸入

<img width="519" height="137" alt="螢幕擷取畫面 2026-05-23 002119" src="https://github.com/user-attachments/assets/f82c117a-2814-4b8d-b96d-3fb645768e2f" />

選擇1的話，會跑7za，選擇2或3則沒有特別的反饋

<img width="1491" height="422" alt="螢幕擷取畫面 2026-05-23 002218" src="https://github.com/user-attachments/assets/0c9c5812-3947-4038-bade-d68ef4d21bfd" />

<img width="522" height="151" alt="螢幕擷取畫面 2026-05-23 002229" src="https://github.com/user-attachments/assets/35b2e8e0-8931-4b72-939b-c325009767cb" />

<img width="593" height="164" alt="螢幕擷取畫面 2026-05-23 002241" src="https://github.com/user-attachments/assets/086040cb-00f6-4252-ad4b-44a4642c3a2d" />

使用strings調查可執行檔中的可讀字串，找到目錄/var/www/html和7za的執行命令語句

<img width="645" height="790" alt="螢幕擷取畫面 2026-05-23 002459" src="https://github.com/user-attachments/assets/e98a989a-f348-45c1-be5a-7728b053a075" />

7za的萬用字元*有注入漏洞，總共有兩個方式，一是以檔名進行命令注入，二是利用軟連結達成任意檔案讀取，這裡實測選二，因為靶機中並不是真的sudo 7za，而是限定了語句

1. 移動到/var/www/html目錄
2. 在目錄下建立空檔案@proof
3. 建立了一個名為proof的軟連結，指向Root的SSH私鑰
4. 以@開頭的字串代表列表檔案，7za讀取到@proof後會認為這是一個列表，接著嘗試打開一個叫做proof的文字檔，並讀取裡面每一行的路徑來進行壓縮
5. 因為proof是一個指向/root/.ssh/id_rsa的軟連結，且7za此時是以sudo權限執行，所以它能夠成功跨越權限阻礙，讀取到/root/.ssh/id_rsa的內容
6. 7za開始將id_rsa裡面的文字逐行當作「檔案路徑」來讀取
7. 由於私鑰的內容是像 -----BEGIN OPENSSH PRIVATE KEY----- 或加密字串，根本不是合法的作業系統檔案路徑，7za在嘗試尋找這些「檔名」時會全部失敗
8. 7za會噴出大量的錯誤訊息，在訊息中會直接寫出id_rsa的內容
9. 如果不是要求root shell的話，可以直接讀取root.txt

另外，我嘗試過讀取/etc/shadow，但讀取失敗，因為包含了大量的特殊符號與空格/換行，7za內部會直接判定「這個Listfile格式錯誤」或直接崩潰跳出，所以只能夠過SSH提權

```bash
touch @proof
ln -fs /root/.ssh/id_rsa proof
sudo /usr/bin/usage_management
```

<img width="1116" height="766" alt="螢幕擷取畫面 2026-05-23 005709" src="https://github.com/user-attachments/assets/b7ba4a63-b830-4ac4-84db-b1fec78838af" />

將讀取到的金鑰內容存成文件，刪除掉錯誤訊息後給予600的權限

<img width="293" height="122" alt="螢幕擷取畫面 2026-05-23 005941" src="https://github.com/user-attachments/assets/e2fc1cb8-a2ab-4995-8614-0458a23a2bb3" />

透過ssh登入成功

<img width="1188" height="752" alt="螢幕擷取畫面 2026-05-23 005956" src="https://github.com/user-attachments/assets/08ea9784-232a-40e6-9f4e-ccc8d5db6add" />


### 2.6 最終成果(Impact)

獲得root.txt

<img width="1065" height="499" alt="螢幕擷取畫面 2026-05-23 010042" src="https://github.com/user-attachments/assets/c949c4b9-2872-4005-883a-082307433838" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 提權方面非常有趣，應該先了解7za再去構建提權思路。
2. webshell上傳功能來源於開發者對使用者的過度信任，認為POST請求只會來自於前端UI，有點像業務邏輯漏洞。

### 浪費時間的部分：

1. 上傳功能花了一點時間才想到如此繞過方式。
2. SQLi真的跑太慢。

### 新知識點：

1. 關於7za的萬用字元提權方式。

### 與實戰對應：

1. 上傳功能類似於業務邏輯漏洞，這種對前端UI的信任在漏洞賞金企畫中很常見。
