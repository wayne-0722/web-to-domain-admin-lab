# Web to Active Directory 作品集摘要

## 作品定位

本作品是一個 Web to Active Directory 的滲透測試實驗，模擬攻擊者從外部 Web 服務取得初始存取權後，透過憑證外洩、憑證重用、橫向移動與內網 Pivot，最終取得 Domain Admin 權限的完整攻擊鏈。

此作品重點不只是展示單一漏洞利用，而是呈現以下能力：

- 從 Web 漏洞延伸評估企業內網風險
- 手動驗證漏洞可利用性與實際影響
- 建立完整 attack path 並保留測試證據
- 將技術發現整理成弱點描述、根因、影響與修補建議
- 在受控實驗環境中重現真實企業常見弱點組合

## 一分鐘說明

這份作品是自建的 Web to AD 攻擊鏈 lab。起點是 Ubuntu 上的 DVWA，先利用 Command Injection 取得 reverse shell，接著在 Web 主機中發現外洩的 `.env` 憑證。由於該服務帳號被重複使用，可透過 SMB 驗證登入一台雙網卡 Windows 工作站，並確認該主機同時能連到外網與內網。後續將此工作站作為 pivot 節點往內網 Domain Controller 前進，透過憑證擷取與 hashcat 離線破解，最終驗證 Domain Admin 權限。

此作品呈現完整滲透測試流程：初始存取、憑證存取、橫向移動、內網枚舉、權限提升、影響評估與修補建議。

## 測試範圍

| 角色 | 系統 | IP |
| --- | --- | --- |
| Attacker | Kali Linux | 192.168.203.131 |
| Web Server | Ubuntu / DVWA | 192.168.203.130 |
| Pivot Host | Windows 10 dual-homed workstation | 192.168.203.132 / 192.168.88.110 |
| Domain Controller | Windows Server 2022 | 192.168.88.100 |

## 攻擊路徑

```text
External Attacker
-> Command Injection on DVWA
-> Reverse Shell on Ubuntu
-> Exposed credential in .env
-> SMB login to Windows workstation
-> Pivot through dual-homed host
-> Domain Controller access
-> LSASS credential extraction
-> Hash cracking
-> Domain Admin compromise
```

## 技術亮點

### 1. Web 漏洞利用

- 針對 DVWA Command Injection 功能進行測試
- 使用命令分隔符確認後端存在 OS command execution
- 建立 reverse shell 取得初始 foothold

對應證據：

- `evidence/core/command-injection-request-response.png`
- `evidence/core/reverse-shell.png`

### 2. 敏感資訊與憑證風險

- 從 Web 主機檔案系統發現 `.env` 設定檔
- 取得服務帳號與密碼
- 判斷此弱點不只是資訊外洩，還可能造成後續橫向移動

對應證據：

- `evidence/core/exposed-env-file.png`

### 3. 橫向移動

- 使用取得的帳密測試 SMB 驗證
- 成功登入 Windows 工作站
- 確認服務帳號存在權限過高與憑證重用問題

對應證據：

- `evidence/core/win10-smb-pwned.png`

### 4. 內網 Pivot 與網路分段問題

- 透過遠端指令確認 Windows 工作站為 dual-homed host
- 該主機同時連接外網與內網，成為攻擊者進入內網的橋接點
- 驗證可從該主機連線至 Domain Controller

對應證據：

- `evidence/core/win10-ipconfig.png`
- `evidence/core/win10-ping-dc.png`

### 5. AD 風險擴大

- 針對內網與 Domain Controller 進行服務確認
- 取得 LSASS 中的雜湊資料
- 使用 hashcat 進行離線破解
- 驗證取得 Domain Admin 權限

對應證據：

- `evidence/core/lsass-dump.png`
- `evidence/core/hashcat-cracking.png`
- `evidence/core/dc-smb-pwned.png`

## 能力對應

| 能力面向 | 本作品呈現方式 |
| --- | --- |
| Web 安全測試 | Command Injection 驗證、payload 測試、reverse shell |
| Linux 基礎 | Web 目錄探索、檔案權限與設定檔檢查 |
| Windows / AD 基礎 | SMB 驗證、網域帳號、Domain Controller、LSASS |
| 內網滲透 | 橫向移動、pivot host、內外網路徑判斷 |
| 工具使用 | nmap、crackmapexec、impacket、hashcat |
| 報告能力 | 攻擊鏈、證據截圖、影響說明、修補建議 |
| 風險理解 | 憑證重用、權限過高、雙網卡、網路分段不足 |

## 專案文件

- `README.zh-TW.md`: 中文專案總覽
- `README.md`: 英文專案總覽
- `report/Report.zh-TW.md`: 中文完整報告
- `report/Report.en.md`: 英文完整報告
- `evidence/core/`: 精簡後的核心證據
- `evidence/raw/`: 原始測試截圖與過程證據
