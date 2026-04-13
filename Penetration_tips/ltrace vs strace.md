ltrace 與 strace 的主要區別在於它們監控的程式層級不同：



* 監控對象
* strace: 追蹤進程執行的系統呼叫 (System Calls)。這是應用程式與作業系統核心 (Kernel) 之間的溝通行為，例如讀取檔案 (read)、開啟網路連線 (connect)。
* ltrace: 追蹤進程呼叫的共用函式庫 (Library Calls)。這是應用程式呼叫 .so 檔案（如 libc）中函式的行為，例如字串處理 (strlen) 或記憶體配置 (malloc)。
* 層級關係
* 一個「函式庫呼叫」內部可能會觸發多個「系統呼叫」。例如，你在程式裡呼叫 printf() (ltrace 會抓到)，其底層會透過系統呼叫 write() (strace 會抓到) 來輸出文字到螢幕。 
* 這兩個命令的輸出都是標準錯誤，如果要grep或儲存成文件的話記得先重定向，strace ls > file 2>&1或ltrace sysinfo 2>&1 | grep "popen"


功能與特性對照

| 特性 | strace | ltrace |
|---|---|---|
| 主要功能 | 監控核心層級互動 | 監控使用者層級動態庫呼叫 |
| 典型範例 | open, read, write, fork | printf, malloc, strcmp |
| 效能影響 | 較大，因為每次系統呼叫都要切換上下文 | 較小，主要針對動態連結函式 |
| 關聯性 | ltrace -S 可以同時顯示系統呼叫 | 無法直接替代 strace |

