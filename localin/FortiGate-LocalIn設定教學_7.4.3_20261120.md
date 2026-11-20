# FortiGate 7.4.3 Local-in Policy 設定教學（保護 SSL VPN 與 IPsec 入口）

> [!WARNING]
> **測試環境與免責**：本文在 lab 的 FortiGate 60E（FortiOS 7.4.3 build2573）上實測，僅供參考，不是 Fortinet 官方文件。請先在自己的測試環境驗證，套用到正式環境的風險由使用者自負。
>
> 文中 IP、帳號、主機名稱都已換成範例值：內網用 `192.168.10.x`，公網用 RFC 5737 文件網段（`198.51.100.x`、`203.0.113.x`）；`8.8.8.8`、`168.95.1.1` 是公共 DNS，保留原值當 GeoIP 查詢範例。
>
> 文中的「收集器」是搭配使用的一支 syslog 收集程式（不在本 repo）：接收 FortiGate 的 syslog → 依規則判定要封鎖的來源 IP → 產生一份純文字 IP 清單 → FortiGate 用 threat feed（External Connector）定期拉取。observe 模式只記錄、不產生清單；enforce 模式才把 IP 寫進清單。任何能產生同樣格式清單的工具都可以替代。

## 一分鐘摘要

**做完的效果**：SSL VPN 埠和 IKE（UDP 500／4500）只接受台灣來源；外部惡意 IP 清單 `ATTACKER-FEED` 裡的 IP 一律丟掉；其他國家連登入頁都看不到。固定 IP 的 site-to-site 對端另外放行，不受影響。

**要花多久**：設定約 30～60 分鐘。之後要觀察幾天，確認收集器沒誤判，才請程式切到 enforce。

**最大風險**：
- 「最後一條全擋」一生效，海外使用者（含出差的自己人）和沒先放行的 site-to-site 對端會立刻斷。
- 如果你平常是「先連 SSL VPN 再管 FortiGate」，寫錯會把自己鎖在門外。

**動手前要準備**：
1. console 線或其他不經 WAN 的管理路徑。
2. 組態備份（`execute backup config ...` 或 GUI 下載），存到本機。
3. 維護時段，並先通知可能在海外的使用者。
4. 下面第 6 節「退回方法」先複製到記事本，換好名稱再動手。

> [!NOTE]
> 本文 CLI 依官方文件與 KB 改寫，**2026-11-20 已在實機 FortiGate-60E v7.4.3 build2573 跑過一輪並還原**（實機畫面見 [圖文版](FortiGate-LocalIn設定教學_7.4.3_20261120圖文版.md)）。實測發現的 12 處更正已改進本文，每處標「2026-11-20 實機驗證」。
>
> **仍未實測**：規則 20「放行台灣」（lab 來源是私有位址，查不到國家）、從國外 IP 被規則 90 擋、SSL VPN 登入失敗（39426）觸發封鎖（見 5.3 末段）、`detect-unknown-esp`、`IKE-TCP-4500`、多 WAN／SD-WAN。
>
> 介面 `wan1`、SSL VPN 埠 `10443`、`<收集器IP>` 都是範例，請換成現場的（lab 的 WAN 是 `internal`）。

---

## 1. 這次會建的東西

| 名稱 | 種類 | 用途 |
|---|---|---|
| `GEO-TW` | 地區位址物件 | 代表台灣的 IP |
| `ATTACKER-FEED` | External Connector（Threat Feed，IP） | 收集器產生的惡意 IP 清單 |
| `S2S-PEERS` | 位址群組 | 固定 IP 的 site-to-site 對端 |
| `SSLVPN-10443` | 自訂服務 | SSL VPN 監聽埠 |
| local-in 5／10／20／90 | local-in 規則 | 放對端 → 擋清單 → 放台灣 → 其餘擋 |

規則只寫 `SSLVPN-10443` 和 `IKE` 兩種服務，**不寫 ALL**。寫 ALL 會連 BGP／OSPF、HA、DNS 回應、FortiGuard 更新一起擋掉。

---

## 2. 事前檢查（只看不改）

每條都先跑，把輸出存下來，退回時要對照。

### 2.1 SSL VPN 聽哪個埠、哪個介面

```
show vpn ssl settings
```

- 看 `set port`（例 10443）和 `set source-interface`（例 wan1）。
- **`show` 不顯示預設值**（2026-11-20 實機驗證）：7.4.3 的 SSL VPN 預設埠就是 10443，所以 `show vpn ssl settings` 看不到 `set port` 時代表 10443。要看實際值：

```
show full-configuration vpn ssl settings | grep port
show full-configuration vpn ssl settings | grep source-interface
diagnose sys tcpsock | grep 10443
```

- FortiOS 的 `grep` **不支援 `-e`**，一次只能一個關鍵字（2026-11-20 實機驗證：`grep: invalid option -- 'e'`）。
- 最後一行有輸出（`process=.../sslvpnd`）才代表真的在聽。設了埠但沒有任何 `srcintf ssl.root` 的防火牆政策時，sslvpnd 不會聽（2026-11-20 實機驗證）。
- 如果 `source-interface` 是 loopback、外面靠 VIP 進來：**local-in 管不到**，本文不適用，要改用一般防火牆 policy 擋（見第 7 節）。
- 順便看有沒有 `set source-address`。那是 SSL VPN 自己的來源限制，可以和 local-in 並存，但兩邊都要記得維護。

### 2.2 管理介面是不是也在 WAN

```
show system interface wan1
get system global | grep admin-sport
```

- 看 `allowaccess` 有沒有 https／ssh／telnet。
- 如果 `admin-sport` 和 SSL VPN 埠一樣，限制 SSL VPN 埠就等於連管理一起限制。先確認兩者不同。
- 誘餌埠（第 5 節）要能運作，wan1 的 `allowaccess` **不能有 ssh、telnet**，22／23／3389 也不能有 VIP 轉進去。

### 2.3 有沒有既有的 local-in

```
show firewall local-in-policy
diagnose firewall iprope list 100001
```

- 有舊規則就記下編號，新規則編號不要撞到。
- local-in 由上往下比對，第一條吻合就決定結果。

### 2.4 IPsec 有哪些合法對端

```
show vpn ipsec phase1-interface
get system settings | grep ike-port
diagnose vpn ike gateway list
```

- 有 `set remote-gw x.x.x.x` 的，就是固定 IP 對端。**全部抄下來**，放進第 3.3 步的 `S2S-PEERS`。
  - `show` **不顯示 `set type static`**（預設值），所以只會看到 `set remote-gw`（2026-11-20 實機驗證）。看到 `set type dynamic` 或 `ddns` 才是另外兩種。
- 對端在海外（例如雲端日本區、國外分公司）一定要先放行，否則會被「只放台灣」擋掉。
- 對端用 DDNS（`remote-gw-ddns` 或 `type ddns`）的，要另建 FQDN 位址物件放進群組。
- `set type dynamic` 是撥入型（FortiClient IPsec）。這些人只要在台灣就能連，出國就連不上。
- `ike-port` 若不是 500，第 3.4 步要另建服務物件。

### 2.5 IKE 有沒有聽 TCP 4500

```
diagnose sys tcpsock | grep 4500
```

- 7.4.2 起，只要設了任一條 IPsec，FortiGate 會多聽 TCP 4500（IKE over TCP）。內建的 `IKE` 服務只含 UDP 500／4500。
- 有看到在聽，可以照第 3.4 步的選用做法一起限制。

### 2.6 `detect-unknown-esp` 現在的值

```
show full-configuration system settings | grep detect-unknown-esp
```

- 只是記下來，**這一步不要改**。原因見第 4 節。

### 2.7 syslogd2 有沒有人在用

```
show log syslogd2 setting
```

- 若已經 `status enable` 送給別的伺服器，改用 `syslogd3`，不要蓋掉。
- `show` 只顯示非預設值；原本 disable 時會是空的 `config ... end`。要看清楚用 `show full-configuration log syslogd2 setting`（2026-11-20 實機驗證）。

### 2.8 local-in log 現在的值（新增，2026-11-20 實機驗證）

```
show full-configuration log setting | grep local-in
show full-configuration log memory filter
```

- 記下 `local-in-deny-unicast` 的原值。官方預設是 disable，但實機這台**原本就是 enable**。第 6.3 步要還原成這裡記下的值，不能一律 disable。
- 第二行看 `local-traffic`。沒硬碟的機型若是 `disable`，local-in deny 不會進記憶體 log，只能從 syslog 看（見 5.4）。

---

## 3. 逐步設定（只能 CLI）

7.4.3 的 local-in 只能用 CLI 建。GUI 要到 Feature Visibility 打開 Local In Policy 才看得到，而且只能看。

順序原則：**先建物件 → 再建放行規則 → 擋清單 → 最後才開全擋**。全擋先建成停用，測過再啟用。

### 3.1 地區物件

```
config firewall address
    edit "GEO-TW"
        set type geography
        set country "TW"
    next
end
```

驗證：

```
show firewall address GEO-TW
diagnose geoip ip2country <一個台灣使用者的公網 IP>
```

- 第二行要回 TW。回錯國家，代表 GeoIP 資料庫有誤，先處理再往下。
- 沒有 FortiGuard 授權也查得到（2026-11-20 實機驗證：`168.95.1.1 - Taiwan`、`8.8.8.8 - United States`）。私有位址回 `Reserved`，不屬於任何國家，所以內網來源不會吻合 `GEO-TW`。

### 3.2 外部惡意 IP 清單

```
config system external-resource
    edit "ATTACKER-FEED"
        set type address
        set resource "http://<收集器IP>:8907/feed/attackers.txt"
        set username "<feed 帳號>"
        set password "<feed 密碼>"
        set refresh-rate 5
    next
end
```

- `refresh-rate` 單位分鐘，預設 5，範圍 1～43200。
- 用的是 HTTP，Basic 驗證的帳密是明文傳。收集器只放內網或走 VPN 抓，不要經網際網路。
- 收集器在 observe 模式時清單是空的。**空清單不會擋任何人**，所以這時放進規則是安全的。

驗證：

```
execute update-external-resource ATTACKER-FEED
diagnose sys external-address-resource list
```

- **2026-11-20 實機驗證**：帶名稱的 `diagnose sys external-address-resource list ATTACKER-FEED` 在 7.4.3 **沒有任何輸出**，已從本文拿掉。改用不帶名稱的 `list`，看 `ATTACKER-FEED` 那一行，例如 `name:ATTACKER-FEED, uuid-idx:857, num of ipv4/ipv6 ranges:0/0, used:no`。
- `num of ipv4/ipv6 ranges` 就是筆數。`used:no` 是還沒有規則引用，建完 local-in 後變 `used:yes`。
- 第一行叫 FortiGate 立刻拉一次，不用等 `refresh-rate`。
- 或 GUI：Security Fabric > External Connectors > `ATTACKER-FEED` > View Entries。
- observe 時應該是 0 筆；狀態不應該是錯誤。
- `show` 不顯示 `set refresh-rate 5`（預設值），看不到是正常的（2026-11-20 實機驗證）。

### 3.3 site-to-site 對端群組

每個對端一個 /32 物件，再收成群組。下面以兩個對端為例，IP 換成第 2.4 步抄到的。

```
config firewall address
    edit "S2S-PEER-01"
        set subnet <對端1 IP> 255.255.255.255
    next
    edit "S2S-PEER-02"
        set subnet <對端2 IP> 255.255.255.255
    next
end
config firewall addrgrp
    edit "S2S-PEERS"
        set member "S2S-PEER-01" "S2S-PEER-02"
    next
end
```

- 完全沒有固定 IP 對端，就跳過這步，也跳過 3.5 的第 5 條。

### 3.4 服務物件

```
config firewall service custom
    edit "SSLVPN-10443"
        set tcp-portrange 10443
    next
end
```

- `IKE` 是內建服務（UDP 500、4500），直接用。
- 選用：第 2.5 步看到 TCP 4500 在聽，而且沒有任何 tunnel 走 TCP，可以一起限制：

```
config firewall service custom
    edit "IKE-TCP-4500"
        set tcp-portrange 4500
    next
end
```

  用了的話，下面每條規則的 `set service` 都加上 `"IKE-TCP-4500"`。

### 3.5 local-in 規則

```
config firewall local-in-policy
    edit 5
        set intf "wan1"
        set srcaddr "S2S-PEERS"
        set dstaddr "all"
        set action accept
        set service "IKE"
        set schedule "always"
        set comments "S2S peers first"
    next
    edit 10
        set intf "wan1"
        set srcaddr "ATTACKER-FEED"
        set dstaddr "all"
        set action deny
        set service "SSLVPN-10443" "IKE"
        set schedule "always"
        set comments "block attacker feed"
    next
    edit 20
        set intf "wan1"
        set srcaddr "GEO-TW"
        set dstaddr "all"
        set action accept
        set service "SSLVPN-10443" "IKE"
        set schedule "always"
        set comments "allow Taiwan"
    next
    edit 90
        set intf "wan1"
        set srcaddr "all"
        set dstaddr "all"
        set action deny
        set service "SSLVPN-10443" "IKE"
        set schedule "always"
        set status disable
        set comments "deny rest - enable after test"
    next
end
```

為什麼這樣排：

- **第 5 條在最上面**：固定對端最重要。就算收集器誤判把對端 IP 放進清單，tunnel 也不會斷。
- **第 10 條在台灣之前**：清單裡的台灣 IP 也要擋。
- **第 20 條一定要寫 `set action accept`**：local-in 的 action 預設是 deny。沒寫就變成擋台灣。
  - 2026-11-20 實機驗證：第 10、90 條打了 `set action deny`，`show firewall local-in-policy` 卻不顯示這行，證實 deny 是預設值。看到規則沒有 `set action` 那一行，就是 deny。
- **第 90 條必須有**：local-in 沒有隱含全擋。沒吻合任何一條的流量，會照原本 SSL VPN／IKE 的行為放進來。
- 第 90 條先 `status disable`，做完第 3.6 步再開。
- 有兩個以上 WAN，`set intf` 寫多個，例如 `"wan1" "wan2"`。

### 3.6 測試後才啟用全擋

先從台灣測（第 5 節的方法），SSL VPN 和 site-to-site 都正常，再啟用：

```
config firewall local-in-policy
    edit 90
        set status enable
    next
end
```

- 一啟用就從國外測一次。有問題立刻 `set status disable` 回去。

### 3.7 送 syslog 給收集器

```
config log syslogd2 setting
    set status enable
    set server "<收集器IP>"
    set mode udp
    set port 5514
    set format default
end
y
```

- **最後的 `y` 是回答確認問題**（2026-11-20 實機驗證）。port 不是 514 時，`end` 會跳：

```
Port 5514 is different from default port 514.
Confirm to use port 5514 instead?
Do you want to continue? (y/n)
```

  回答 `y` 後顯示 `Port set to 5514`。**沒回答時 status 和 server 已經生效，只有 port 沒改（還是 514）**，收集器就收不到。用腳本或貼上整段時，確認最後那行 `y` 有被送出。
- 驗證：`show log syslogd2 setting` 要看到 `set port 5514`。
- `mode` 預設 udp、`format` 預設 default，寫出來只是讓設定一眼看得懂。
- 收集器要走 VPN 才到得了，加 `set interface-select-method specify` 和 `set interface "<介面>"`，或 `set source-ip`。
- local-in 的 log 屬於 local traffic。`syslogd2 filter` 的 `local-traffic` 預設 enable，通常不用改。確認：`show full-configuration log syslogd2 filter`。

### 3.8 打開 local-in deny 的 log

```
config log setting
    set local-in-deny-unicast enable
end
```

- 官方預設是關的，但**現場可能已經開了**（2026-11-20 實機驗證：這台原本就是 enable，這步是空操作）。先照 2.8 記原值。
- 不開，收集器看不到誘餌埠（22、23、3389）被打的紀錄。
- 7.4.3 只能這樣全域開。「每條 local-in 各自開 log」要 7.6.0 起。
- **log 量會明顯變多**：WAN 每天被掃的封包都會記一筆。沒硬碟的小機型只記在記憶體，會把其他 log 擠掉。官方沒給數字。
- 開了之後 local-in 規則擋掉的，也會留下帶 policy ID 的紀錄（依 KB 推論，未逐項實測）。
- 2026-11-20 實機驗證：誘餌埠 23 被打時，收集器收到 logid `0001000014` 的 local-in deny log，log 帶 `devid`（序號）與 `devname`。收集器稽核紀錄記下的欄位是 subtype local、action deny、目的埠 23、服務 TELNET（這是收集器 JSON 稽核紀錄的寫法，不是 FortiGate syslog 原文）。

### 3.9 收集器先觀察，再切 enforce

1. 程式維持 observe 模式。這時 `ATTACKER-FEED` 是空的，第 10 條不擋任何人。
2. 觀察幾天，看程式「本來會封」的 IP 清單裡有沒有自己人：醫院或分公司的出口 IP、廠商遠端維護 IP、site-to-site 對端、雲端服務。
3. 確認沒有誤判，再請程式切 enforce。
4. 切完到 FortiGate 看清單筆數有沒有上來（3.2 的驗證指令）。**update 完一定再看一次筆數**：2026-11-20 實機驗證時有一次收集器已回 200 且有內容，FortiGate 仍是 0 筆，再 update 一次才上來（原因不明，只遇到一次）。
5. 同次實測也確認：FortiGate **不會**把清單裡的私有 IP 濾掉。

---

## 4. `detect-unknown-esp`：先不要改

- 官方 KB 207821 說：7.2.4 之後要先 `set detect-unknown-esp disable`，local-in 才擋得到 IKE。**這是單一 KB 的說法，手冊沒有對應說明。**
- 其他 KB 對這個設定的說法互相矛盾：KB 249721 說 7.2.4 預設 disable；也有 KB 說預設 enable，並說關掉不是好的資安做法。
- 這個設定開著時，FortiGate 會在 local-in 之前先丟掉不認識 SPI 的 ESP 封包。關掉等於少一層檢查。

**建議做法**：

1. 不改它，照第 3 節設定完。
2. 從國外對 UDP 500 測試（第 5.3 節），看第 90 條的命中數有沒有增加。
3. 有擋到就維持現狀。
4. 擋不到，才在維護時段、lab 先試過後，考慮關掉：

```
config system settings
    set detect-unknown-esp disable
end
```

   關掉後要確認所有既有 tunnel 正常、VPN event log 沒有異常。

---

## 5. 驗證方法

### 5.1 規則有沒有載入核心

```
show firewall local-in-policy
diagnose firewall iprope list 100001
```

- 100001 是 local-in 的規則群組。
- **停用的規則不會出現在 `iprope list 100001`**（2026-11-20 實機驗證：第 90 條停用時只看得到 5、10、20），這是正常的。
- 清單規則的來源顯示成 `source external ip pool(1): <uuid-idx>`，地區規則顯示 `source country(1): TW`（2026-11-20 實機驗證）。
- 輸出若出現 `flag3 (40): truncated`，代表位址太多被截斷（見第 7 節）。

### 5.2 每條規則的命中數

```
diagnose firewall iprope show 100001 10
diagnose firewall iprope show 100001 20
diagnose firewall iprope show 100001 90
```

- 會顯示封包數、位元組數；**命中過才會多出 `hit count` 和 `first hit／last hit`**（2026-11-20 實機驗證）。沒命中過的規則只有 pkts／bytes，不是指令壞掉。
- 歸零：`diagnose firewall iprope clear 100001 <編號>`。重開機不會歸零。

### 5.3 從國外測試

找一台國外 IP 的機器（例如國外區域的雲端主機）。**不要用會轉址到台灣的手機 VPN**，先查那台的出口 IP：在 FortiGate 上跑 `diagnose geoip ip2country <那台的 IP>`，確認不是 TW。

在國外那台：

```
curl -vk --connect-timeout 5 https://<WAN IP>:10443/
nc -vz -w 5 <WAN IP> 10443
sudo nmap -sU -p 500,4500 <WAN IP>
```

- 預期：`curl`／`nc` 逾時（被丟棄，不是被拒絕）。
- **FortiGate 被 local-in 擋時不回 RST**（2026-11-20 實機驗證：sniffer 只看到同一個 SYN 重送 5 次，沒有任何回應）。用戶端中間若有代理或路由器（例如 VPN 子網路由器），用戶端可能顯示 `Connection refused`，那是中間設備回的，不代表 FortiGate 放行或拒絕。以 FortiGate 端 sniffer 和命中數為準。
- UDP 掃描結果本來就不準（open|filtered），以 FortiGate 端的命中數和 log 為準。
- 這三個是一般 Linux 工具，**不是 Fortinet 官方指令**。

同時在 FortiGate 抓封包和看判斷：

```
diagnose sniffer packet wan1 'host <國外IP> and port 10443' 4 10
```

```
diagnose debug reset
diagnose debug flow filter clear
diagnose debug flow filter addr <國外IP>
diagnose debug flow filter dport 10443
diagnose debug flow show iprope enable
diagnose debug flow show function-name enable
diagnose debug console timestamp enable
diagnose debug flow trace start 20
diagnose debug enable
```

- 預期看到 `checked gnum-100001 policy-90` 和 `iprope_in_check() check failed on policy 90, drop`。
- 2026-11-20 實機驗證：預期訊息一字不差。實際順序是：

```
checked gnum-100001 policy-90, ret-matched, act-accept
policy-90 is matched, act-drop
iprope_in_check() check failed on policy 90, drop
```

  第一行的 `act-accept` **不是放行**，只是「比對吻合」；要看下一行 `policy-90 is matched, act-drop`。
- 測完一定要關：`diagnose debug disable`、`diagnose debug reset`。用 `diagnose debug info` 確認 `debug output: disable`。

**誘餌埠測試（2026-11-20 實機驗證）**：從測試端 `nc -vz -w 3 <WAN IP> 23`（23 必須不在 WAN 的 allowaccess 裡），FortiGate 會記一筆 local-in deny（logid `0001000014`），收集器據此封鎖。一次連線會因 SYN 重送記多筆。

**SSL VPN 登入失敗測試（2026-11-20 實機驗證：這樣測不可行）**：用假帳密 POST `/remote/logincheck`，FortiGate 不會產生登入失敗的 39426。原因：lab 只有一條 `srcintf ssl.root` 的 deny 政策（為了讓 sslvpnd 聽埠），登入頁回 403，debug 顯示 `rmt_web_access_check:804 access failed, uri=[/remote/logincheck],ret=4103`，sslvpnd 在驗帳密前就拒絕。要測這條，要有放行的 ssl.root 政策加上使用者群組。**此路徑仍未實測。**

### 5.4 看 log

- GUI：Log & Report > Local Traffic。local-in deny 的 log ID 是 `0001000014`。
- 收集器端：`sudo tcpdump -ni any udp port 5514`，確認 FortiGate 的 syslog 有到。
- CLI 看 log（2026-11-20 實機驗證可用的寫法）：

```
execute log filter reset
execute log filter device memory
execute log filter category traffic
execute log filter field subtype local
execute log filter field dstport 10443
execute log filter view-lines 10
execute log display
execute log filter reset
```

  - `category` 可用名稱（`traffic`、`event`）。看 SSL VPN 事件：`category event` 加 `field subtype vpn`。
  - **`view-lines` 小於 5 會報錯**（`Command fail. Return code -651`），至少給 5。
- **沒硬碟的機型可能在記憶體 log 完全看不到 local-in deny**（2026-11-20 實機驗證）：若 `show full-configuration log memory filter` 的 `local-traffic` 是 `disable`，local-in deny 不進記憶體 log，上面的指令會回 `0 logs found`，但 syslog 那邊照常收得到。這時以收集器或 syslog 伺服器為準。不建議為了看 log 去打開記憶體的 local-traffic（會把其他 log 擠掉）。

### 5.5 外部清單

```
execute update-external-resource ATTACKER-FEED
diagnose sys external-address-resource list
```

- 看 `ATTACKER-FEED` 那一行的 `num of ipv4/ipv6 ranges`。帶名稱的 `list ATTACKER-FEED` 在 7.4.3 沒有輸出（2026-11-20 實機驗證），不要用。
- 切 enforce 後筆數應該大於 0，且與收集器產出的檔案大致相符。還是 0 就再 update 一次（見 3.9）。

---

## 6. 退回方法（先寫好再動手）

照順序做。規則要先刪，物件才刪得掉（被引用的物件刪不了）。

### 6.1 最快止血：只停用全擋

```
config firewall local-in-policy
    edit 90
        set status disable
    next
end
```

- 海外使用者馬上恢復。其他規則留著不影響。（2026-11-20 實機：停用指令已執行成功；停用後的重連測試沒有留下紀錄，視為**未實測**）

### 6.2 完整拿掉

```
config firewall local-in-policy
    delete 90
    delete 20
    delete 10
    delete 5
end
config firewall addrgrp
    delete "S2S-PEERS"
end
config firewall address
    delete "S2S-PEER-01"
    delete "S2S-PEER-02"
    delete "GEO-TW"
end
config system external-resource
    delete "ATTACKER-FEED"
end
config firewall service custom
    delete "SSLVPN-10443"
end
```

- 有建 `IKE-TCP-4500` 就再加 `delete "IKE-TCP-4500"`。
- 2026-11-20 實機驗證：照這個順序刪一次成功，刪完 `show firewall local-in-policy` 為空、各物件 `entry is not found in table`。

### 6.3 log 與 syslog 還原

```
config log setting
    set local-in-deny-unicast <第 2.8 步記下的原值>
end
config log syslogd2 setting
    set status enable
    unset server
    unset port
    set status disable
end
```

- **第一段不能一律寫 `disable`**（2026-11-20 實機驗證）：這台原本就是 enable，一律 disable 會把原設定改壞。原本就是 enable 的，第一段整段不用跑。
- **第二段為什麼要先 enable 再 disable**（2026-11-20 實機驗證）：
  - 只下 `set status disable` 也會跳 `Port 5514 is different from default port 514 ... (y/n)`，因為 port 還是 5514。
  - 先 disable 之後，server／port 變成隱藏欄位，`show` 和 `get` 都看不到，`unset server` 回 `command parse error`（-61），清不掉。
  - 在同一個區塊裡「enable → unset server → unset port → disable」，實測沒有錯誤、也不跳 y/n，還原後 `show full-configuration log syslogd2 setting` 只剩 `set status disable`，跟動手前一樣。
- 第 2.7 步若 syslogd2 原本就有在用，**不要**跑第二段，照當初存下的輸出還原。

### 6.4 若有改過 `detect-unknown-esp`

```
config system settings
    set detect-unknown-esp <第 2.6 步記下的原值>
end
```

### 6.5 最後手段

- 用事前備份的組態還原（`execute restore config ...`，會重開機）。

---

## 7. 已知限制

- **管不到 VIP**：SSL VPN 用 loopback＋VIP 發布時，local-in 無效，要在 WAN→loopback 的防火牆 policy 上方加 deny。
- **ISDB 要 7.4.4 起**：7.4.3 的 local-in 不能用 `Malicious-Malicious.Server` 這類 ISDB 物件當來源。
- **7.6.1 起出廠就有預設 local-in**：用 ISDB 擋惡意 IP 與 Tor。升版後記得對照，避免和本文規則重複或衝突。
- **SSL VPN tunnel mode 在 7.6.3 起移除**（所有機型）；7.6.0 起 40F、60F／61F 完全沒有 SSL VPN。升版前先規劃改 IPsec。
- **每條 local-in 的位址上限**：KB 183653 說單條超過 10172 個位址，後面的不會生效，也看不到警告。Threat feed 本身可裝到 131072 筆，KB 沒說 feed 算不算在內（未查證）。清單請收集器控制在數千筆內，並定期看第 5.1 步有沒有 `truncated`。
- **feed 連不到時**：官方 KB 說沿用舊清單；但重開機後是否還在，兩篇 KB 說法相反（未查證，以實測為準）。最壞情況是清單變空＝不擋，但第 20／90 條的台灣限制仍有效。
- **SD-WAN**：wan1 若是 SD-WAN 成員，升到 7.4.6、7.4.7、7.6.1 時，用介面名寫的 local-in 可能被升級刪掉，要改綁 SD-WAN zone 重建（KB 367001）。
- **出國的自己人會被擋**：要臨時放行，就在第 10 條之後加一條 accept，來源是他當地的 IP，用完刪掉。
- **GeoIP 會錯**：少數台灣 IP 可能被判成別的國家。遇到用 `diagnose geoip ip2country` 查，必要時請 Fortinet 修正或用 override。

---

## 8. 附錄：主張與官方出處

| 主張 | 適用版本 | 出處 |
|---|---|---|
| local-in 可用位址、群組、geography、IP threat feed；只能 CLI 建 | 7.4.3 | [Admin Guide 7.4.3 Local-in policy](https://docs.fortinet.com/document/fortigate/7.4.3/administration-guide/363127) |
| local-in 參數預設：action=deny、status=enable | 7.4.0 CLI 參考 | [CLI Ref 7.4.0 local-in-policy](https://docs.fortinet.com/document/fortigate/7.4.0/cli-reference/295620) |
| threat feed 進 local-in 要 7.2.4／7.4.0 起 | 7.2.4+、7.4.0+ | [KB 94707](https://community.fortinet.com/fortigate-3/technical-tip-external-threat-list-threat-feed-blocked-via-the-firewall-ipv4-policy-94707) |
| external-resource 參數：username／password（HTTP Basic）、refresh-rate 預設 5、名稱最長 35 字 | 7.4.0 CLI 參考 | [CLI Ref 7.4.0 external-resource](https://docs.fortinet.com/document/fortigate/7.4.0/cli-reference/72620) |
| threat feed 上限 10 MB 或 131072 筆 | 7.4.3 | [Admin Guide 7.4.3 Threat feeds](https://docs.fortinet.com/document/fortigate/7.4.3/administration-guide/9463) |
| 空 threat feed 不吻合任何流量 | 未標版本 | [KB 219525](https://community.fortinet.com/fortigate-3/technical-tip-fortigate-behavior-when-external-threat-feed-list-is-empty-219525) |
| feed 連線失敗沿用舊清單；`diagnose sys external-address-resource list` | 6.2+ | [KB 112156](https://community.fortinet.com/fortigate-3/technical-tip-threat-feed-list-behavior-when-connection-failed-between-fortigate-and-threat-feed-url-112156) |
| local-in 擋 SSL VPN、不影響 VIP | 7.x | [KB 222845](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Restricting-Allowing-access-to-the-FortiGate-SSL/ta-p/222845) |
| 限制 IKE 只給特定對端的 local-in 範例；VIP 優先於 local-in | 未標版本 | [KB 132821](https://community.fortinet.com/fortigate-3/technical-tip-allow-ipsec-vpn-ports-and-protocol-access-to-specific-ip-addresses-only-132821) |
| 內建 IKE 服務含 UDP 500／4500；可自訂 ike-port | 7.0+ | [KB 101733](https://community.fortinet.com/fortigate-3/technical-tip-configure-custom-ike-port-between-two-fortigates-101733) |
| 7.4.2 起 IKE 聽 TCP 4500 | 7.4.2+ | [KB 342618](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Network-scans-show-TCP-4500-on-the-FortiGate-even/ta-p/342618) |
| 擋 IKE 要關 `detect-unknown-esp` | 7.2.4+（單一 KB） | [KB 207821](https://community.fortinet.com/fortigate-3/technical-tip-unable-to-block-ike-negotiation-with-threat-feed-list-207821) |
| `detect-unknown-esp` 開啟時在 local-in 前丟未知 SPI；稱預設 disable | 7.2.4 | [KB 249721](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Detect-Unknown-SPI-before-local-in-policy/ta-p/249721) |
| iprope 群組 100001、命中數與歸零指令 | 7.0.4+ | [KB 397472](https://community.fortinet.com/t5/FortiGate/Technical-Tip-How-to-collect-local-in-policy-s-hit-count/ta-p/397472) |
| debug flow 判讀 local-in 吻合／丟棄 | 未標版本 | [KB 338679](https://community.fortinet.com/t5/FortiGate/Technical-Tip-How-to-validate-if-the-traffic-matches-a-local-in/ta-p/338679) |
| 單條 local-in 超過 10172 位址會截斷 | 7.x | [KB 183653](https://community.fortinet.com/fortigate-3/technical-tip-adding-more-than-10172-addresses-in-a-local-in-policy-does-not-work-as-expected-183653) |
| `diagnose geoip ip2country` | 多版本 | [KB 93014](https://community.fortinet.com/fortigate-3/technical-tip-commands-to-verify-geoip-information-and-troubleshoot-geoip-database-93014) |
| `local-in-deny-unicast` 預設 disable | 7.4.0 CLI 參考 | [CLI Ref 7.4.0 log setting](https://docs.fortinet.com/document/fortigate/7.4.0/cli-reference/441620) |
| 每條 local-in 單獨開 log | 7.6.0+ | [Admin Guide 7.6.7](https://docs.fortinet.com/document/fortigate/7.6.7/administration-guide/263651) |
| syslogd setting：mode 預設 udp、port 預設 514、format 預設 default | 7.4.2 CLI 參考 | [CLI Ref 7.4.2 syslogd2 setting](https://docs.fortinet.com/document/fortigate/7.4.2/cli-reference/418620) |
| syslogd filter：local-traffic 預設 enable（syslogd2 未單獨查證） | 7.4.2 CLI 參考 | [CLI Ref 7.4.2 syslogd filter](https://docs.fortinet.com/document/fortigate/7.4.2/cli-reference/416620) |
| ISDB 當 local-in 來源 | 7.4.4+ | [Admin Guide 7.4.4](https://docs.fortinet.com/document/fortigate/7.4.4/administration-guide/363127) |
| 7.6.1 起預設 local-in 擋惡意 ISDB | 7.6.1+ | [New Features 7.6](https://docs.fortinet.com/document/fortigate/7.6.0/new-features/695423/enhanced-security-with-default-local-in-policy-7-6-1) |
| SSL VPN 機型支援與 tunnel mode 移除 | 7.4.8／7.6.0／7.6.3 | [KB 359125](https://community.fortinet.com/t5/FortiGate/Technical-Tip-SSL-VPN-support-on-FortiGate-models/ta-p/359125) |
| SD-WAN 成員介面的 local-in 升版被刪 | 7.4.6／7.4.7／7.6.1 | [KB 367001](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Local-in-Policy-is-Missing-after-upgrading-to-v7-4/ta-p/367001) |
| loopback＋VIP 替代做法、SSL VPN 最佳實務 | 7.4.3 | [Admin Guide 7.4.3 SSL VPN best practices](https://docs.fortinet.com/document/fortigate/7.4.3/administration-guide/947829) |

### 未查證清單

- `get system global | grep admin-sport`、`show vpn ssl settings`、`diagnose vpn ike gateway list`、`diagnose sniffer packet`：常用指令，本次沒逐條找出處。
- ~~`execute log filter`／`execute log display` 的過濾參數~~ → 2026-11-20 實機驗證，見 5.4。
- `syslogd2 filter` 的預設值（只查到 `syslogd filter` 那頁）。
- threat feed 的筆數是否計入單條 local-in 的 10172 上限。
- feed 連不到時重開機後清單是否保留。
- `local-in-deny-unicast` 是否涵蓋「明確 deny 規則」擋掉的流量（KB 有帶 policy ID 的 log 範例，但未在 7.4.3 確認）。
- 7.4.3 的 `set intf` 能否直接填 SD-WAN zone。

## 相關文件

- [圖文版](FortiGate-LocalIn設定教學_7.4.3_20261120圖文版.md) — 同一套設定的實機畫面對照
- 作者另有一份前期研究筆記（含 DoS、automation stitch 的分析；stitch 自動封鎖本文刻意不採用），未收錄於本 repo。
