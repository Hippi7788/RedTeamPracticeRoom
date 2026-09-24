這裡為您整理一份「列出 ➡️ 排隊 ➡️ 做事」三步驟公式的實用辦公範例表格。
在 PowerShell 中，只要調整這三塊積木，就能組合出各種不同的自動化功能：

| 實用情境 | 1. 先列出（目標） | 2. 排隊（過濾/傳送） | 3. 再做事（執行動作） | 完整一鍵指令 |
|---|---|---|---|---|
| 批次建資料夾 (您這次的任務) | Get-ChildItem -Directory (列出所有子資料夾) | | ForEach-Object { | New-Item -Path $_.FullName -Name "新資料夾" -ItemType Directory } | Get-ChildItem -Directory | ForEach-Object { New-Item -Path $_.FullName -Name "新資料夾" -ItemType Directory } |
| 文字檔大搜查 (找出包含特定關鍵字的檔案) | Get-ChildItem -File -Filter "*.txt" (只列出所有純文字檔) | | | Select-String -Pattern "機密項目" (在每份檔案內容中搜尋關鍵字) | Get-ChildItem -File -Filter "*.txt" | Select-String -Pattern "機密項目" |
| 桌面大掃除 (把散落的圖片歸類到專用資料夾) | Get-ChildItem -File (列出桌面上所有檔案) | | Where-Object { $_.Extension -in '.jpg','.png' } | (排隊檢查：只要副檔名是圖片的) | Move-Item -Destination "D:\我的圖片\" (把留下來的圖片通通搬走) | Get-ChildItem -File | Where-Object { $_.Extension -in '.jpg','.png' } | Move-Item -Destination "D:\我的圖片\" |
| 抓出肥大檔案 (找出資料夾中大於 100MB 的檔案) | Get-ChildItem -File -Recurse (包含所有子資料夾深層搜尋) | | Where-Object { $_.Length -gt 100MB } | (排隊檢查：只要檔案大小大於 100MB) | Select-Object Name, Length (把這些肥大檔案的名字和大小顯示出來) | Get-ChildItem -File -Recurse | Where-Object { $_.Length -gt 100MB } | Select-Object Name, Length |
| 批次加開頭 (幫所有相片加上拍攝年份) | Get-ChildItem -File -Filter "*.jpg" (只列出所有 JPG 圖片) | | ForEach-Object { | Rename-Item $_.FullName -NewName ("2026_" + $_.Name) } | Get-ChildItem -File -Filter "*.jpg" | ForEach-Object { Rename-Item $_.FullName -NewName ("2026_" + $_.Name) } |

------------------------------
## 💡 觀察表格的小發現：

   1. 傳送帶 | 的妙用：有時候隊伍只需要排一次（例如批次建資料夾），有時候隊伍可以排兩次（例如桌面大掃除：先過濾出圖片，再集體搬家）。
   2. ForEach-Object 與 Where-Object 的差別：
   * ForEach-Object（每個人都要做）：像是發發財金，排隊過來的每個人都要執行後面的動作。
      * Where-Object（符合條件才能過）：像是海關檢查，只有符合特定條件（例如大於 100MB、是圖片）的人才能留下來走到下一步。
   
看完了這些例子，哪一個情境最貼近您平常在電腦上的工作痛點呢？如果您有興趣，我們可以針對其中一個例子，進一步聊聊它在 PowerShell 底層是怎麼抓取檔案資訊的！

