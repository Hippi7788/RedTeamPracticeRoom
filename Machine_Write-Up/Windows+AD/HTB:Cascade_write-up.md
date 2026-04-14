## 1. 執行摘要 (Executive Summary)

### 測試範圍與目標：
1. 靶機名稱:Cascade
2. 難度:Medium
3. IP:10.129.211.241

### 主要發現：
1. 漏洞 ：（風險等級：）
2. 漏洞 ：（風險等級：）

### 攻擊鏈摘要：

### 潛在影響：

### 修復建議：

---

## 2. 技術發現與攻擊路徑詳述 (Findings)

### 2.1 資訊蒐集(Reconnaissance)

使用Nmap掃描開放埠，除了AD環境預設開放的以外並未有特殊服務

```bash
sudo nmap -sT -Pn --min-rate 800 -p- 10.129.211.241 -oA portscan/ports
sudo nmap -sU --top-ports 20 10.129.211.241 -oA portscan/udp
sudo nmap -sT -Pn -sV -sC -O -p$ports 10.129.211.241 -oA portscan/detail
sudo nmap --script=vuln -p$ports 10.129.211.241 -oA portscan/vuln
```

<img width="689" height="584" alt="螢幕擷取畫面 2026-04-13 204418" src="https://github.com/user-attachments/assets/47642811-c921-444d-9de6-fb8cb17d8b7a" />

<img width="596" height="588" alt="螢幕擷取畫面 2026-04-13 204429" src="https://github.com/user-attachments/assets/b7e2b5a3-9984-4af9-992e-978bf0a4866e" />




### 2.2 枚舉(Enumeration)

### 2.3 初始存取(Initial Access)
### 2.4 橫向移動(Lateral Movement)
### 2.5 權限提升(Privilege Escalation)
### 2.6 最終成果(Impact)

---

## 3. 學習回顧 (Lessons Learned)

### 成功的部分：
### 浪費時間的部分：
### 新知識點：
### 與實戰對應：
