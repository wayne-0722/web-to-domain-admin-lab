# Web to Active Directory 滲透測試報告

## 文件資訊

| 項目 | 內容 |
| --- | --- |
| 專案名稱 | Web to Active Directory Compromise Lab |
| 測試類型 | Web 應用程式與內網滲透測試情境模擬 |
| 測試環境 | 隔離實驗室環境 |
| 主要目標 | 驗證 Web 漏洞是否可被串聯成 Active Directory 網域入侵 |
| 報告語言 | 繁體中文 |

> 本報告為作品集用途，所有測試皆於自建、隔離且授權的實驗環境中執行。報告中的帳號、密碼與 IP 皆屬於 lab 環境，不代表真實組織資產。

## Executive Summary

本次測試模擬外部攻擊者針對一台部署 DVWA 的 Web Server 進行滲透測試，並評估單一 Web 漏洞是否可能進一步影響內部 Active Directory 環境。

測試結果顯示，攻擊者可透過 Web 應用程式中的 Command Injection 取得 Ubuntu Web Server 的初始存取權。取得 shell 後，攻擊者於 Web 目錄中發現暴露的 `.env` 設定檔，內含可用的網域服務帳號憑證。由於該帳號存在憑證重用與權限過高問題，攻擊者可透過 SMB 登入一台同時連接外網與內網的 Windows 工作站。

該 Windows 工作站具備雙網卡設定，形成內外網之間的橋接節點，使攻擊者能從原本只接觸 Web Server 的位置進一步進入內網。後續測試中，攻擊者成功識別 Domain Controller、取得 LSASS 中的憑證雜湊，並透過離線破解取得高權限帳號密碼，最終成功驗證 Domain Admin 權限。

本攻擊鏈代表從外部 Web 漏洞擴大為完整網域控制的嚴重風險。主要根因並非單一漏洞，而是多項控制缺失同時存在：Web 輸入驗證不足、敏感設定檔暴露、憑證重用、服務帳號權限過高、雙網卡主機造成網路分段失效，以及缺乏對橫向移動與憑證存取行為的偵測。

## Risk Rating

| 風險項目 | 嚴重性 | 影響 |
| --- | --- | --- |
| Command Injection 導致遠端命令執行 | Critical | 取得 Web Server shell |
| `.env` 敏感資訊暴露 | High | 取得可重用服務帳號憑證 |
| 服務帳號憑證重用與權限過高 | Critical | 橫向移動至 Windows 工作站 |
| Dual-homed host 造成網路分段失效 | High | 從外部區段進入內網 |
| LSASS 憑證擷取與弱密碼 | Critical | 取得 Domain Admin 權限 |

整體風險評估：**Critical**

理由：攻擊者可由外部 Web 服務開始，逐步擴大權限並取得 Active Directory Domain Admin 權限，代表機密性、完整性與可用性皆受到重大影響。

## Scope

| 角色 | 系統 | IP 位址 | 說明 |
| --- | --- | --- | --- |
| Attacker | Kali Linux | 192.168.203.131 | 攻擊端 |
| Web Server | Ubuntu / DVWA | 192.168.203.130 | 初始目標 |
| Pivot Host | Windows 10 Pro | 192.168.203.132 / 192.168.88.110 | 雙網卡工作站 |
| Domain Controller | Windows Server 2022 | 192.168.88.100 | AD 網域控制器 |

網路區段：

- 外網區段：192.168.203.0/24
- 內網區段：192.168.88.0/24

## Methodology

本次測試依照常見滲透測試流程進行，包含：

1. 資產與服務識別
2. Web 漏洞驗證
3. 初始存取
4. 敏感資訊搜尋
5. 憑證驗證
6. 橫向移動
7. 內網路徑確認
8. Active Directory 枚舉
9. 憑證存取與權限提升
10. 影響評估與修補建議

本報告以可重現證據為主，每一個重大結論皆對應至 `evidence/core` 或 `evidence/raw` 中的截圖與測試紀錄。

## Tools and Techniques

| 類別 | 工具 / 技術 | 用途 |
| --- | --- | --- |
| Web 測試 | Browser、手動 payload 驗證 | 驗證 Command Injection 與回應結果 |
| 初始存取 | Reverse shell | 取得 Web Server shell |
| 服務識別 | nmap | 探測外網與內網服務 |
| Windows / AD 測試 | crackmapexec | SMB 驗證、遠端指令、憑證擷取 |
| AD 工具 | impacket | AD 相關驗證與枚舉 |
| 憑證測試 | hashcat | NTLM hash 離線破解 |
| 系統枚舉 | ipconfig、arp、nslookup | 確認網路介面、路由與 DC 可達性 |

## Severity Definition

| 嚴重性 | 定義 |
| --- | --- |
| Critical | 可導致系統完全控制、Domain Admin compromise、大量資料外洩或關鍵服務中斷 |
| High | 可造成未授權存取、橫向移動、敏感資訊暴露或重要系統受影響 |
| Medium | 需搭配其他條件才能擴大影響，但仍可能造成安全風險 |
| Low | 直接影響有限，通常屬於資訊揭露、設定弱點或強化建議 |

## Attack Path Overview

<p align="center">
  <img src="../evidence/core/attack-path.png" alt="Attack Path">
</p>

攻擊路徑如下：

```text
External Attacker
-> Web Command Injection
-> Reverse Shell on Ubuntu
-> Exposed .env Credential
-> SMB Authentication to Windows Workstation
-> Pivot via Dual-homed Host
-> Domain Controller Access
-> LSASS Credential Extraction
-> Hash Cracking
-> Domain Admin Compromise
```

## Findings

## Finding 1: Command Injection 導致遠端命令執行

| 欄位 | 內容 |
| --- | --- |
| 嚴重性 | Critical |
| 受影響資產 | Ubuntu / DVWA，192.168.203.130 |
| 弱點類型 | OS Command Injection |
| 影響 | 攻擊者可於 Web Server 上執行任意系統命令 |

### Description

DVWA 的命令執行功能未正確限制使用者輸入，導致攻擊者可透過命令分隔符將額外系統指令傳入後端執行。此弱點使攻擊者能從 Web 應用程式層級跨越至作業系統層級，進一步取得 shell 存取權。

### Evidence

Command Injection request / response：

<p align="center">
  <img src="../evidence/core/command-injection-request-response.png" alt="Command Injection Evidence">
</p>

Reverse shell：

<p align="center">
  <img src="../evidence/core/reverse-shell.png" alt="Reverse Shell Evidence">
</p>

### Impact

成功利用後，攻擊者可：

- 執行系統指令
- 讀取 Web Server 本機檔案
- 探索 Web 根目錄與設定檔
- 建立 reverse shell
- 作為後續內網攻擊的 initial foothold

### Root Cause

- 使用者輸入未做嚴格白名單驗證
- 後端直接將輸入傳遞給系統命令
- 缺乏命令執行時的參數化處理或安全 API

### Recommendation

- 避免直接呼叫 shell 執行使用者輸入
- 若必須呼叫系統命令，應使用安全 API 並固定允許的參數
- 對輸入值採用白名單驗證，例如僅允許合法 IP 或 hostname 格式
- 以最低權限帳號執行 Web 服務
- 於 WAF、EDR 或系統稽核中監控可疑命令執行行為

## Finding 2: Web Server 暴露敏感設定檔與有效憑證

| 欄位 | 內容 |
| --- | --- |
| 嚴重性 | High |
| 受影響資產 | Ubuntu / DVWA，192.168.203.130 |
| 弱點類型 | Sensitive Information Disclosure |
| 影響 | 攻擊者可取得服務帳號憑證並進行橫向移動 |

### Description

取得 Web Server shell 後，於 Web 目錄中發現 `.env` 設定檔與備份資料。設定檔包含資料庫或服務使用之帳號密碼，且該帳號可用於 Windows / AD 環境驗證。

此發現將原本的 Web Server 入侵擴大為身分憑證風險。若此類憑證同時可用於其他系統，攻擊者即可跳脫原本 Web Server 的影響範圍，進一步嘗試登入內部資產。

### Evidence

<p align="center">
  <img src="../evidence/core/exposed-env-file.png" alt="Exposed .env Evidence">
</p>

觀察到的敏感資訊類型：

- 網域服務帳號：`lab\svc-sql`
- 對應密碼：已於報告文字中遮罩，完整內容僅保留於 lab 證據截圖
- 備份檔案中出現相同帳號線索

### Impact

成功取得憑證後，攻擊者可：

- 嘗試登入其他 Windows 主機
- 測試 SMB、RDP、WinRM 等服務
- 進行憑證重用攻擊
- 以合法帳號身分繞過部分邊界防護

### Root Cause

- 敏感設定檔放置於 Web 可讀或容易被 Web 服務帳號讀取的位置
- 設定檔與備份檔案缺乏適當權限控管
- 憑證未依用途隔離

### Recommendation

- 移除 Web 目錄中的 `.env`、備份檔與任何含密碼檔案
- 使用 secrets manager 或環境層級的安全設定管理方式
- 限制 Web 服務帳號對設定檔的讀取範圍
- 對已暴露帳號立即重設密碼
- 檢查該帳號於所有系統中的登入紀錄與權限配置

## Finding 3: 服務帳號憑證重用與權限過高導致橫向移動

| 欄位 | 內容 |
| --- | --- |
| 嚴重性 | Critical |
| 受影響資產 | Windows 10 Workstation，192.168.203.132 |
| 弱點類型 | Credential Reuse / Excessive Privilege |
| 影響 | 攻擊者可由 Web Server 擴大至 Windows 工作站 |

### Description

攻擊者使用於 Web Server 發現的服務帳號憑證，對外網 Windows 工作站進行 SMB 驗證。測試結果顯示該帳號可成功登入，且具備足夠權限執行遠端操作。

這表示服務帳號不僅用於應用程式或資料庫情境，也被配置為可登入 Windows 主機的高權限帳號。此狀況使單一憑證外洩可快速擴大為內部主機控制。

### Evidence

<p align="center">
  <img src="../evidence/core/win10-smb-pwned.png" alt="Windows SMB Pwned Evidence">
</p>

### Impact

攻擊者可：

- 使用合法帳號登入 Windows 工作站
- 執行遠端命令
- 讀取系統資訊與網路設定
- 取得新的攻擊位置
- 進一步向內網與 Domain Controller 前進

### Root Cause

- 服務帳號密碼於多個系統重用
- 服務帳號具備互動式登入或遠端管理權限
- 未落實最小權限原則
- 未限制服務帳號的允許登入主機

### Recommendation

- 每個服務使用獨立帳號與獨立密碼
- 禁止服務帳號互動式登入
- 限制服務帳號可登入的主機與服務
- 移除不必要的 local admin 或遠端管理權限
- 對高風險服務帳號導入密碼輪替與監控

## Finding 4: Dual-homed Host 造成網路分段失效

| 欄位 | 內容 |
| --- | --- |
| 嚴重性 | High |
| 受影響資產 | Windows 10 Workstation，192.168.203.132 / 192.168.88.110 |
| 弱點類型 | Network Segmentation Weakness |
| 影響 | 攻擊者可透過外網主機進入內網 |

### Description

Windows 工作站同時連接外網與內網，具備兩個網路介面。攻擊者在取得該工作站存取權後，可透過該主機確認內網路由與 Domain Controller 連線能力。

此架構使原本應受網路分段保護的內網資產，間接受到外網 Web 入侵事件影響。即使 Web Server 本身沒有直接連到內網，攻擊者仍可利用 dual-homed host 作為 pivot 節點。

### Evidence

工作站網路設定：

<p align="center">
  <img src="../evidence/core/win10-ipconfig.png" alt="Windows ipconfig Evidence">
</p>

連線至 Domain Controller：

<p align="center">
  <img src="../evidence/core/win10-ping-dc.png" alt="Windows Ping DC Evidence">
</p>

### Impact

攻擊者可：

- 從外部區段跨入內網區段
- 對內網服務進行探測
- 接觸 Domain Controller
- 繞過原本預期的邊界隔離

### Root Cause

- 工作站同時連接不同安全區域
- 缺乏對跨區主機的存取控制
- 網路分段僅存在於拓樸上，未落實流量控管

### Recommendation

- 避免一般工作站同時連接不同安全區域
- 對必須跨區的主機建立嚴格 firewall policy
- 限制外網區段主機對內網的可達性
- 於網路層監控跨區 SMB、RDP、LDAP、Kerberos 等流量
- 定期檢查雙網卡與異常路由設定

## Finding 5: LSASS 憑證擷取與弱密碼導致 Domain Admin Compromise

| 欄位 | 內容 |
| --- | --- |
| 嚴重性 | Critical |
| 受影響資產 | Windows Workstation / Domain Controller |
| 弱點類型 | Credential Dumping / Weak Password |
| 影響 | 攻擊者成功取得 Domain Admin 權限 |

### Description

在取得 Windows 工作站控制權後，攻擊者擷取 LSASS 中的憑證資料，取得多組 NTLM hash。後續透過離線破解取得高權限帳號密碼，並成功驗證可登入 Domain Controller。

此結果代表攻擊者已從單一 Web 漏洞擴大到完整 AD 網域控制。若發生於真實環境，攻擊者可建立網域帳號、部署惡意程式、竊取資料、修改群組原則或中斷關鍵服務。

### Evidence

LSASS credential dump：

<p align="center">
  <img src="../evidence/core/lsass-dump.png" alt="LSASS Dump Evidence">
</p>

Hash cracking：

<p align="center">
  <img src="../evidence/core/hashcat-cracking.png" alt="Hashcat Evidence">
</p>

Domain Controller 驗證：

<p align="center">
  <img src="../evidence/core/dc-smb-pwned.png" alt="Domain Controller Pwned Evidence">
</p>

### Impact

取得 Domain Admin 權限後，攻擊者可：

- 控制所有加入網域的主機
- 建立、刪除或修改網域帳號
- 存取敏感資料與共享資料夾
- 部署 persistence 機制
- 修改 Group Policy
- 停用安全控管或影響服務可用性

### Root Cause

- Windows 主機允許高權限憑證留存在 LSASS
- 高權限帳號密碼可被離線破解
- 缺乏 Credential Guard 或等效保護
- 缺乏對憑證擷取行為的 EDR 偵測

### Recommendation

- 啟用 Windows Defender Credential Guard
- 限制 Domain Admin 帳號登入一般工作站
- 使用分層管理模型，例如 Tier 0 / Tier 1 / Tier 2
- 對管理員帳號使用長密碼與 MFA
- 監控 LSASS 存取、可疑 dump 行為與大量 authentication event
- 定期檢查高權限帳號登入紀錄

## MITRE ATT&CK Mapping

| 階段 | 技術 | 說明 |
| --- | --- | --- |
| Initial Access | Exploit Public-Facing Application | 透過 Web Command Injection 取得初始存取 |
| Execution | Command and Scripting Interpreter | 執行系統命令與 shell |
| Credential Access | Unsecured Credentials | 從 `.env` 取得憑證 |
| Lateral Movement | Remote Services: SMB | 使用 SMB 登入 Windows 工作站 |
| Discovery | System Network Configuration Discovery | 透過 ipconfig 確認雙網卡 |
| Discovery | Remote System Discovery | 確認 Domain Controller 可達性 |
| Credential Access | OS Credential Dumping | 擷取 LSASS 憑證資料 |
| Credential Access | Brute Force / Password Cracking | 離線破解 NTLM hash |
| Privilege Escalation | Valid Accounts | 使用高權限帳號驗證 Domain Admin 權限 |

## Business Impact

若此攻擊鏈發生於真實企業環境，可能造成：

- 機密資料外洩
- 內部系統遭橫向移動與批次入侵
- 網域帳號與權限遭竄改
- 端點與伺服器遭植入後門
- 安全監控或防毒系統遭停用
- 關鍵服務中斷
- 事件應變與復原成本大幅增加

本案例顯示，Web 應用程式弱點不應只視為單一主機風險。當憑證管理、權限控管與網路分段同時失效時，外部 Web 服務可能成為攻擊者進入 AD 核心環境的入口。

## Prioritized Remediation Plan

### Immediate Actions

1. 修補 Command Injection，避免使用者輸入進入系統命令。
2. 移除 Web 目錄中的 `.env`、備份檔與任何敏感資訊。
3. 立即重設已暴露服務帳號與管理員帳號密碼。
4. 停用或限制 `svc-sql` 等服務帳號的互動式登入與遠端登入權限。
5. 檢查 Domain Admin 帳號是否曾登入一般工作站。

### Short-term Improvements

1. 重新設計服務帳號權限，落實最小權限。
2. 移除不必要的雙網卡主機，或加入嚴格 firewall policy。
3. 對內外網區段建立明確 ACL 與流量監控。
4. 部署或調整 EDR 偵測 LSASS 存取、遠端命令執行與 lateral movement。
5. 對重要帳號導入密碼長度要求與密碼輪替。

### Long-term Improvements

1. 建立 AD 分層管理模型。
2. 導入 secrets management 流程。
3. 建立定期弱點掃描與滲透測試流程。
4. 導入 SIEM 關聯分析，偵測異常登入與跨區移動。
5. 建立 incident response playbook，涵蓋憑證外洩與 AD compromise 情境。

## Detection Opportunities

| 攻擊階段 | 可觀察行為 | 建議偵測方式 |
| --- | --- | --- |
| Command Injection | Web request 中出現命令分隔符、系統指令或異常參數 | WAF log、Web access log、EDR process telemetry |
| Reverse Shell | Web Server 對外建立異常連線 | Firewall log、NetFlow、EDR network event |
| Credential Discovery | Web 服務帳號讀取 `.env`、備份檔或非預期設定檔 | Linux auditd、file integrity monitoring |
| SMB Lateral Movement | 同一服務帳號登入多台主機 | Windows Event ID 4624、4627、4672 |
| Remote Command Execution | 透過 SMB service 建立遠端執行行為 | Windows Event ID 7045、4688、Sysmon |
| LSASS Access | 非預期程序存取 LSASS memory | EDR alert、Sysmon Event ID 10 |
| Domain Admin Use | Domain Admin 登入一般工作站或非管理主機 | AD logon event、SIEM correlation rule |

## Retest Criteria

完成修補後，建議以以下條件進行 retest：

- Web 參數無法再執行任意系統命令
- Web Server 不存在可讀取的 `.env` 或備份憑證檔
- 外洩服務帳號無法登入 Windows 工作站
- 一般工作站無法同時橋接外網與內網，或跨區流量已被限制
- LSASS 存取行為會被 EDR 或 Windows 事件紀錄偵測
- Domain Admin 帳號未出現在一般工作站的登入紀錄中

## Conclusion

本次測試成功證明，單一 Web Command Injection 弱點在缺乏縱深防禦的環境中，可能被串聯成完整 Active Directory 入侵。

攻擊鏈的核心問題包含：

- Web 應用程式輸入驗證不足
- 敏感憑證暴露於 Web Server
- 服務帳號憑證重用
- 服務帳號權限過高
- 雙網卡主機破壞網路分段
- 高權限憑證保護不足

本案例的防禦重點不應只停留在修補 Web 漏洞，而應同時改善憑證管理、權限控管、網路分段與偵測能力。若只修補 Command Injection，仍可能留下其他可被攻擊者利用的路徑；若只更換密碼但不改善帳號權限與網路架構，也可能在下一次憑證外洩後再次發生橫向移動。

因此，建議採取 defense-in-depth 策略，將 Web 安全、主機安全、AD 安全與網路安全視為同一條攻擊路徑上的連續控制點。

## Appendix: Evidence Index

| 證據 | 檔案 |
| --- | --- |
| 攻擊路徑圖 | `evidence/core/attack-path.png` |
| Command Injection request / response | `evidence/core/command-injection-request-response.png` |
| Reverse shell | `evidence/core/reverse-shell.png` |
| `.env` 憑證暴露 | `evidence/core/exposed-env-file.png` |
| Windows SMB 驗證成功 | `evidence/core/win10-smb-pwned.png` |
| Windows 雙網卡設定 | `evidence/core/win10-ipconfig.png` |
| Windows 連線至 DC | `evidence/core/win10-ping-dc.png` |
| LSASS dump | `evidence/core/lsass-dump.png` |
| Hashcat cracking | `evidence/core/hashcat-cracking.png` |
| Domain Controller 驗證成功 | `evidence/core/dc-smb-pwned.png` |
