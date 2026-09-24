# 💡 PowerShell「列出 ➡️ 排隊 ➡️ 做事」自動化筆記

PowerShell 的核心靈魂是 **Pipeline（管線機制，符號為 `|`）**。
運作邏輯非常像工廠的傳送帶：**「先列出所有目標、讓它們排隊傳送、最後對每個人動手做事」**。

---

## 🛠️ 三步驟核心公式解密

* **`|`（管線）**：串接指令的傳送帶。
* **`$_`**：代表「目前正在傳送帶上排隊的那一個物件」。
* **`ForEach-Object`**：**每個人都要做**。排隊過來的每個人，都要執行大括號 `{}` 裡的動作。
* **`Where-Object`**：**符合條件才能過**。像海關檢查，只有符合條件（例如大小、副檔名）的物件才能繼續走下去。

---

## 📋 實用辦公自動化對照表

| 實用情境 | 1. 先列出（目標） | 2. 排隊（過濾或傳送） | 3. 再做事（執行動作） |
| :--- | :--- | :--- | :--- |
| **批次建資料夾** | `Get-ChildItem -Directory`<br>(列出子資料夾) | 傳送給所有人做 | 在其底下新建子資料夾 |
| **文字檔大搜查** | `Get-ChildItem -File -Filter "*.txt"`<br>(列出純文字檔) | 直接傳送 | 在每份檔案內搜尋關鍵字 |
| **桌面大掃除** | `Get-ChildItem -File`<br>(列出所有檔案) | 檢查：只要 `.jpg` 或 `.png` | 把留下來的檔案搬家 |
| **抓出肥大檔案** | `Get-ChildItem -File -Recurse`<br>(深層搜尋所有檔案) | 檢查：檔案大小大於 `100MB` | 顯示這些檔案的名字與大小 |
| **批次加開頭** | `Get-ChildItem -File -Filter "*.jpg"`<br>(列出 JPG 圖片) | 傳送給所有人做 | 幫檔案名稱前面加上年份 |

---

## 💻 完整一鍵複製指令

### 1. 批次建資料夾 (各自新增三個資料夾)
```powershell
Get-ChildItem -Directory | ForEach-Object { New-Item -Path \$_.FullName -Name "新資料夾" -ItemType Directory }
```

### 2. 文字檔大搜查 (找出包含特定關鍵字的檔案)
```powershell
Get-ChildItem -File -Filter "*.txt" | Select-String -Pattern "機密項目"
```

### 3. 桌面大掃除 (把散落的圖片歸類到專用區)
```powershell
Get-ChildItem -File | Where-Object { \$_.Extension -in '.jpg','.png' } | Move-Item -Destination "D:\我的圖片\"
```

### 4. 抓出肥大檔案 (找出大於 100MB 的檔案)
```powershell
Get-ChildItem -File -Recurse | Where-Object { \$_.Length -gt 100MB } | Select-Object Name, Length
```

### 5. 批次加開頭 (幫所有相片加上拍攝年份)
```powershell
Get-ChildItem -File -Filter "*.jpg" | ForEach-Object { Rename-Item \(_.FullName -NewName ("2026_" + \)_.Name) }
```
