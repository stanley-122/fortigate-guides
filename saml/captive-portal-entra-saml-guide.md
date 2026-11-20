# FortiGate Captive Portal + Entra ID SAML 設定教學

> [!NOTE]
> **測試環境**：FortiGate 60E，FortiOS **v7.4.12 build2902**；FortiAP 231F（bridge mode）；Microsoft Entra ID 免費版。
> 本文基於 2026-07-27 實機實作，每一步都在真機上跑過並驗證成功。
> 成功證據：`diagnose firewall auth list` 顯示使用者 UPN、群組比對成功、流量正常通過。
>
> 文中的租戶、GUID、帳號、IP 都已換成示範值（`<TENANT-ID>`、`contoso`、`192.168.20.x` 等）。

> [!WARNING]
> 本文是個人實測紀錄，**僅供參考，使用風險自負**。不同 FortiOS 版本的預設值與指令可能不同，正式環境請先在測試機驗證。

相關文件：[圖文版（GUI 截圖對照）](captive-portal-entra-saml-illustrated.md)、[SAML 原理與 FortiGate 實作](saml-concepts-fortigate.md)

---

## 一、這套架構在做什麼

使用者連上網路 → 拿到 IP → 開網頁被攔截 → 導向 Microsoft 登入 → 通過後才能上網。
身分驗證交給 Entra ID（等同公司的 M365 帳號），FortiGate 只負責攔截、驗簽、比對群組、放行。

```
① 使用者連上 SSID（或插網線）
        ↓
② AP／交換器把流量打上 VLAN tag，送到 FortiGate
        ↓
③ FortiGate 的 VLAN 介面發 DHCP，使用者拿到 IP
        ↓
④ 介面上的 Captive Portal 攔下未認證流量
        ↓
⑤ 導向 Entra ID → 使用者登入（含 MFA）
        ↓
⑥ Entra 回傳 Assertion（含 username 與 group GUID）
        ↓
⑦ FortiGate 用 IdP 憑證驗簽 → 比對群組 GUID → 放行
```

**適用**：訪客網路、BYOD、不想自建 RADIUS 但已有 M365 的環境。
**不適用**：需要在 L2 接入階段就分派 VLAN 的場景（那要 802.1X，SAML 做不到，
理由見 [SAML 原理與 FortiGate 實作](saml-concepts-fortigate.md) 第六章）。

---

## 二、前置需求

| 項目 | 說明 |
|---|---|
| FortiOS | 7.4.x（本文以 7.4.12 實測）|
| Entra ID 授權 | **免費版即可**，不需要 P1／P2 |
| 對外連線 | FortiGate 必須連得到 `login.microsoftonline.com` |
| 權限 | Entra 全域管理員、FortiGate super_admin |
| 網路 | 使用者所在 VLAN 的**閘道必須是 FortiGate**（portal 才攔得到）|

---

## 三、Part A — Entra ID 側設定

### A-1 建立 Enterprise Application

1. <https://entra.microsoft.com> → **應用程式** → **企業應用程式** → **新增應用程式**
2. **建立您自己的應用程式** → 命名（例：`FortiGate-WiFi-Portal`）
3. 選 **整合您在圖庫中找不到的任何其他應用程式 (非資源庫)** → 建立

> ⚠ **WiFi Portal 與 VPN 要各建一個 App**，兩者的 Entity ID 不同，不能共用。

### A-2 設定 SAML 單一登入

進 App → **單一登入** → 選 **SAML** → 編輯「**基本 SAML 設定**」：

| 欄位 | 值 |
|---|---|
| 識別碼 (實體識別碼) | `https://<FortiGate認證位址>:1003/saml/metadata/` |
| 回覆 URL (ACS) | `https://<FortiGate認證位址>:1003/saml/login/` |

`<FortiGate認證位址>` = 使用者端連得到的 FortiGate 位址，用該 VLAN 的介面 IP 或 FQDN。
本文以 `192.168.20.1`（測試 VLAN 的閘道 IP）為例。

> ★ **Entra 只驗格式（必須 `https://` 開頭），不驗可達性。** 私有 IP、`.local` 網域都收——
> 因為 SAML 是 front-channel 協定，IdP 從不主動連線 SP，Assertion 全由瀏覽器轉送。
> 但 `http://` 一定被拒。

> ★ **要新增第二組時，點欄位下方的「新增識別碼」／「新增回覆 URL」連結**，
> 會多出一列空白欄位。**不要直接改既有那列**——那是覆蓋不是新增（實測踩過）。
> 多組並存是允許的，一台 App 可以同時服務 lab 與正式環境。

> ★ **儲存後回到概覽頁，只會顯示標記為「預設」的那一筆。** 新增的第二筆要點「編輯」
> 進去才看得到。看到概覽是舊值不代表沒存成功。

### A-3 屬性與宣告（★ 最容易錯的地方）

編輯「**屬性與宣告**」，確認有這兩項：

| 宣告名稱 | 來源 | 命名空間 |
|---|---|---|
| `username` | `user.userprincipalname` | **必須留空** |
| `group` | 安全性群組（送出 GUID） | **必須留空** |

**群組宣告的設定路徑**：
屬性與宣告 → **新增群組宣告** → 選「**安全性群組**」→ 展開「**進階選項**」
→ 勾「**自訂群組宣告的名稱**」→ 名稱填 `group`、**命名空間留空**

> ★★ **命名空間一定要留空。** 填了會變成
> `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/xxx` 這種長 URI，
> FortiGate 只認短名稱，抓不到就會認證失敗。

### A-4 取得簽章憑證

**方法一（建議，最不會抄錯）**：直接從公開的 federation metadata 端點抓，
不需登入，順便拿到 entityID 與 SSO URL：

```bash
TENANT="<你的租戶ID>"
APPID="<你的AppID>"
curl -fsS "https://login.microsoftonline.com/${TENANT}/federationmetadata/2007-06/federationmetadata.xml?appid=${APPID}" -o entra-meta.xml

# 取出憑證轉成 PEM
B64=$(grep -o '<X509Certificate>[^<]*' entra-meta.xml | head -1 | sed 's|<X509Certificate>||')
echo "$B64" | openssl base64 -d -A | openssl x509 -inform DER -out entra-idp.pem

# 順便印出 IdP 端點（照抄，不要自己拼）
grep -o 'entityID="[^"]*"' entra-meta.xml | head -1
grep -o 'SingleSignOnService[^/]*HTTP-Redirect[^/]*Location="[^"]*"' entra-meta.xml | head -1
```

**方法二**：App → 單一登入 → SAML 憑證 → 下載「**憑證 (Base64)**」

### A-5 指派使用者與群組

App → **使用者和群組** → 新增使用者/群組 → 指派要能登入的群組。

然後到 **群組** → 找到該群組 → 複製 **物件識別碼 (Object ID)**，
這個 GUID 等一下要填進 FortiGate。

> ⚠ **記下的是 Object ID（GUID），不是群組名稱。** 這是整合失敗最常見的原因。

### A-6 MFA（視需求）

免費版租戶預設開啟「**安全性預設值**」，會強制所有使用者註冊 MFA。

要關閉：**身分識別** → **概觀** → **屬性** → **管理安全性預設值** → 已停用

> ⚠ 全租戶生效，包含管理員帳號。免費版無法只對特定帳號豁免
> （細緻豁免需要 Entra ID P1 的條件式存取）。正式租戶不建議關。

---

## 四、Part B — FortiGate 側設定

> **推送順序很重要**：憑證 → 介面/DHCP → SSID → SAML → 群組 → 政策 → **最後才掛 portal**。
> 順序顛倒會出現 `entry not found in datasource`，或是產生「使用者已被攔住但 exempt
> 政策還不存在」的空窗期。

### B-0 匯入 IdP 憑證（第一步，其他設定會引用它）

**GUI**：系統 → 憑證 → 匯入 → **遠端憑證** → 選 A-4 的 PEM 檔 → 命名 `REMOTE_Cert_Entra`

**CLI**（可用 SSH，實測成功）：

```
config vpn certificate remote
edit "REMOTE_Cert_Entra"
set remote "<貼上 entra-idp.pem 全文，含開頭與結尾那兩行>"
next
end
```

> ⚠ **不要走 REST API**。實測 `cmdb/vpn.certificate/remote` 會**回 success 但憑證是空的**
> （靜默失敗），症狀是 portal 的 `/saml` 端點自我迴圈。GUI 或 CLI 都可以，REST 不行。
>
> ⚠ 匯入後**無法直接驗證內容**——`remote` 欄位是 write-only，讀回來永遠是空字串。
> 只能靠實測：若登入後 log 出現 `ret=101 (Signature element not found.)` 就是憑證有問題。

### B-1 VLAN 介面

```
config system interface
edit "vlan-saml"
set vdom "root"
set interface "<接交換器/AP 的實體介面>"
set vlanid 100
set mode static
set ip 192.168.20.1 255.255.255.0
set allowaccess ping https
set role lan
set alias "SAML-Test"
next
end
```

> ★ `allowaccess` **必須含 https**，否則使用者連不到認證頁。
> ★ Role 必須是 `lan` 或 `undefined`，captive portal 才設得上去。
> ★ **bridge mode 的 AP**：父介面是「**接交換器的實體介面**」。
>   （tunnel mode 才掛在 SSID 介面底下，兩者不可混用）

### B-2 DHCP

```
config system dhcp server
edit 10
set interface "vlan-saml"
set default-gateway 192.168.20.1
set netmask 255.255.255.0
set dns-service default
config ip-range
edit 1
set start-ip 192.168.20.100
set end-ip 192.168.20.200
next
end
next
end
```

> ⚠ `dns-service default` 模式下**不能**同時寫 `set dns-server1`，會 parse error。
> 要指定 DNS 得先 `set dns-service specify`。

### B-3 無線 SSID（bridge mode）

```
config wireless-controller vap
edit "SAML-Test"
set ssid "SAML-Test"
set security wpa2-only-personal
set passphrase "<PSK>"
set local-bridging enable
set vlanid 100
set schedule "always"
next
end
```

**綁定到 AP 的 radio（不綁不會廣播）**：

```
config wireless-controller wtp-profile
edit "<你的 profile 名稱>"
config radio-1
set vap-all manual
set vaps "SAML-Test"
end
config radio-2
set vap-all manual
set vaps "SAML-Test"
end
next
end
```

> ★★ **`vap-all tunnel`（預設值）只會自動綁 tunnel mode 的 SSID。**
> bridge mode 的 SSID 用預設值時**完全不會廣播**，而且沒有任何錯誤訊息。
> 必須改成 `manual` 再用 `set vaps` 手動指定。實測踩過，症狀是 AP 上線但搜不到 SSID。

驗證是否在廣播：

```
diagnose wireless-controller wlac -d vap
```
要看到 `wlan=SAML-Test ... vlan_id=100`。

### B-4 SAML server

```
config user saml
edit "entra-portal"
set entity-id "https://192.168.20.1:1003/saml/metadata/"
set single-sign-on-url "https://192.168.20.1:1003/saml/login/"
set single-logout-url "https://192.168.20.1:1003/saml/logout/"
set idp-entity-id "https://sts.windows.net/<租戶ID>/"
set idp-single-sign-on-url "https://login.microsoftonline.com/<租戶ID>/saml2"
set idp-single-logout-url "https://login.microsoftonline.com/<租戶ID>/saml2"
set idp-cert "REMOTE_Cert_Entra"
set user-name "username"
set group-name "group"
set digest-method sha256
next
end
```

> ★ SP 路徑含 `/saml/` 這一層。**IPsec VPN 用的是 `/remote/saml/`，不要混用。**
> ★ `entity-id` 與 `single-sign-on-url` 必須和 Entra 的「識別碼」「回覆 URL」**逐字元一致**（含結尾斜線）。

### B-5 使用者群組

```
config user group
edit "entra-staff"
set member "entra-portal"
config match
edit 1
set server-name "entra-portal"
set group-name "<Entra 群組的 Object ID GUID>"
next
end
next
end
```

> ★★ 填 **GUID**，不是群組名稱。

### B-6 認證通用設定

```
config system global
set auth-https-port 1003
end

config user setting
set auth-cert "Fortinet_Factory"
set auth-secure-http enable
set auth-timeout 480
end

config firewall auth-portal
set portal-addr "192.168.20.1"
end
```

> ★★ **`auth-secure-http` 預設是 `disable`，一定要開。**
> 不開的話 portal 走 HTTP，而 Entra 只接受 `https://` 的回覆 URL，整個流程會失敗。實測踩過。
>
> ★ **`auth-timeout` 單位是「分鐘」不是秒**，預設只有 **5 分鐘**，太短。
>   上限 1440（24 小時）。預設計時方式是 `idle-timeout`（閒置計時，有流量就歸零）。
>   要「登入後固定 N 小時」才加 `set auth-timeout-type hard-timeout`。
>
> ★ `portal-addr` 設定重導向用的位址。不設的話 FortiGate 用介面 IP 導向，
>   可能與 Entra 的回覆 URL 對不上。

### B-7 防火牆政策（★ exempt 在前、認證在後）

```
config firewall address
edit "saml-ms-login-1"
set type fqdn
set fqdn "login.microsoftonline.com"
next
edit "saml-ms-login-2"
set type fqdn
set fqdn "aadcdn.msftauth.net"
next
edit "saml-ms-login-3"
set type fqdn
set fqdn "aadcdn.msauth.net"
next
edit "saml-ms-login-4"
set type fqdn
set fqdn "login.live.com"
next
end

config firewall addrgrp
edit "saml-ms-endpoints"
set member "saml-ms-login-1" "saml-ms-login-2" "saml-ms-login-3" "saml-ms-login-4"
next
end

config firewall policy
edit 101
set name "saml-exempt-dns"
set srcintf "vlan-saml"
set dstintf "<對外介面>"
set srcaddr "all"
set dstaddr "all"
set service "DNS"
set action accept
set schedule "always"
set captive-portal-exempt enable
set nat enable
next
edit 102
set name "saml-exempt-entra"
set srcintf "vlan-saml"
set dstintf "<對外介面>"
set srcaddr "all"
set dstaddr "saml-ms-endpoints"
set service "HTTPS"
set action accept
set schedule "always"
set captive-portal-exempt enable
set nat enable
next
edit 100
set name "saml-authenticated"
set srcintf "vlan-saml"
set dstintf "<對外介面>"
set srcaddr "all"
set dstaddr "all"
set service "ALL"
set action accept
set schedule "always"
set groups "entra-staff"
set nat enable
next
move 101 before 100
move 102 after 101
end
```

> ★ **必須用 FQDN 位址物件**：`login.microsoftonline.com` 是 anycast 多 IP，寫死 IP 一定會漏。
> ★ **DNS 也要 exempt**，否則 FQDN 物件解不出來、使用者也查不到登入網域。
> ★★ **FortiOS 照「政策清單排序 Seq#」比對，不是 policy ID。**
>   在已運作的機器上新增，政策會被追加到清單**最底部**，可能排在既有 catch-all 之後
>   而永遠命中不到。推完必須 `show firewall policy` 確認實際位置，必要時用 `move` 調整。

### B-8 掛上 Captive Portal（★ 最後一步）

```
config system interface
edit "vlan-saml"
set security-mode captive-portal
set security-groups "entra-staff"
next
end
```

**無瀏覽器裝置的豁免**（印表機、IoT、醫療儀器——這些永遠過不了 portal）：

```
config firewall address
edit "no-browser-devices"
set subnet 192.168.20.240 255.255.255.240
next
end

config user security-exempt-list
edit "no-browser"
config rule
edit 1
set srcaddr "no-browser-devices"
next
end
next
end

config system interface
edit "vlan-saml"
set security-exempt-list "no-browser"
next
end
```

> ⚠ **介面層 captive portal 會強制該介面上「所有」裝置認證。**
> 上線前務必盤點無瀏覽器裝置並全部納入 exempt，否則上線當天會出事。
> 這類裝置需設固定 IP（放在 DHCP range 之外）。
>
> ⚠ **介面層的 `security-exempt-list` 才擋得住介面層攔截**；
> policy 的 `captive-portal-exempt` 只在流量命中該條政策時才作用，兩者不能互相取代。

---

## 五、Part C — 交換器側（★ 這步漏了會完全不通）

**bridge mode 的 AP，客戶端流量走實體網路**，必須經過路徑上每一台交換器：

- **AP 上聯埠**：trunk，native/untagged = AP 管理 VLAN，**tagged = 所有客戶端 VLAN**
- **FortiGate 對接埠**：trunk，tagged = 所有客戶端 VLAN

> ★★ **這是本次實作實際踩到的坑。** 症狀：客戶端關聯 AP 成功、FortiGate 也顯示
> `vlan_id=100` 正確，但**拿不到 IP（拿到 169.254.x.x APIPA）**，
> 且 FortiGate 上抓封包**完全抓不到該客戶端的任何流量**。
> 原因就是中間交換器沒帶 tag，封包在半路被丟掉。
>
> **判斷方法**：`diagnose sniffer packet <實體介面> 'ether host <客戶端MAC>' 4 0 a`
> 若 0 個封包 → 流量根本沒到 FortiGate → 往交換器 trunk 方向查，不要浪費時間查 SAML。

（tunnel mode 則不受此限——流量走 CAPWAP 封裝直接回 FortiGate，
但 VLAN 子介面要改掛在 SSID 介面底下。）

---

## 六、驗收

### 成功的判準（唯一標準）

```
diagnose firewall auth list
```

要看到類似：

```
192.168.20.100, wifiuser@contoso.onmicrosoft.com
    src_mac: xx:xx:xx:xx:xx:xx
    expire: 3600, allow-idle: 3600
    server: entra-portal          ← SAML 認證來源
    group_name: entra-staff       ← 群組 GUID 比對成功
    packets: in 15232 out 9222    ← 流量正常通過
```

> ★ **白畫面 ≠ 失敗。** 認證成功後 FortiOS 會 303 導回使用者原本要去的網址；
> 若該網址本來就連不上，瀏覽器會顯示白畫面或無法連線。
> **一律以 `diagnose firewall auth list` 為準，不要看瀏覽器畫面下結論。**

### 常用診斷指令

```bash
# 重測前清掉卡住的 session（失敗的登入嘗試會卡住 portal）
diagnose firewall auth clear

# SAML 除錯（測完務必 disable）
diagnose debug application samld -1
diagnose debug enable
diagnose debug disable

# 無線客戶端狀態（看有沒有關聯上、VLAN 對不對、拿到什麼 IP）
diagnose wireless-controller wlac -d sta

# SSID 是否在廣播
diagnose wireless-controller wlac -d vap

# FQDN 物件有沒有解出來
diagnose firewall fqdn list

# 政策實際排序
show firewall policy
```

### 分層排查法（照這個順序，不要跳）

| 症狀 | 查哪裡 |
|---|---|
| 搜不到 SSID | `vap-all` 是否為 manual、`vaps` 有沒有綁 |
| 連上但拿不到 IP（169.254.x.x）| **交換器 trunk**、DHCP server status |
| 拿到 IP 但沒被導向登入頁 | 介面 `security-mode`、政策排序 |
| 導向後停在 Microsoft 錯誤頁 | Entra 的回覆 URL／識別碼是否逐字元相符 |
| 登入成功但沒放行 | 群組 GUID、`user-name`／`group-name` 宣告名稱 |
| log 出現 `ret=101` | IdP 憑證沒匯入成功 |

---

## 七、本次實作踩到的坑（總表）

| # | 坑 | 症狀 | 解法 |
|---|---|---|---|
| 1 | 交換器沒帶 tag | 拿不到 IP、FortiGate 抓不到任何客戶端封包 | AP 上聯埠與 FortiGate 對接埠都要 trunk |
| 2 | `vap-all tunnel` 不綁 bridge SSID | AP 上線但搜不到 SSID，無錯誤訊息 | 改 `manual` + `set vaps` |
| 3 | `auth-secure-http` 預設 disable | portal 走 HTTP，Entra 拒收 | `set auth-secure-http enable` |
| 4 | `auth-timeout` 預設 5 分鐘 | 測到一半就過期 | 單位是分鐘，調 480 |
| 5 | Entra 新增 URL 時覆蓋既有值 | lab 設定被蓋掉 | 點「新增識別碼／回覆 URL」連結，別改既有列 |
| 6 | Entra 概覽只顯示「預設」那筆 | 以為沒存成功 | 進「編輯」才看得到全部 |
| 7 | 憑證走 REST 匯入 | 回 success 但憑證是空的（靜默失敗）| 用 GUI 或 CLI |
| 8 | 政策順序看 Seq# 不是 ID | 新政策排在 catch-all 之後永不命中 | `show firewall policy` 確認並 `move` |
| 9 | `set type default`（NTP）| 非法 enum，推送中斷 | 只有 `fortiguard` / `custom` |
| 10 | 抓包 filter 對 tagged 封包不匹配 | 誤判成沒流量 | 用 `vlan` filter 或 `ether host <MAC>` |

---

## 八、正式部署前要調整的

| 項目 | 測試值 | 正式建議 |
|---|---|---|
| `auth-timeout` | 60 分鐘 | **480～720 分鐘**（一個班次不被打斷）|
| `auth-cert` | `Fortinet_Factory` | **CA 簽發的信任憑證**（否則每個使用者都看到憑證警告）|
| SP 位址 | 介面 IP | **FQDN**（多 VLAN 時只需一組 Entra 回覆 URL）|
| 無瀏覽器裝置 | 預留 /28 | **實際盤點後決定範圍** |
| MFA | 視測試需要關閉 | **正式租戶保持開啟** |

### 使用者體驗的兩個已知抱怨點

1. **必須主動開瀏覽器才會觸發轉導。** 開機後直接用 Outlook、LINE 等桌面 App 的人，
   流量會被擋住但看不到任何提示，只覺得「網路壞了」。
2. **裝置休眠會累積 idle 時間。** 午休一小時回來可能要重新認證——
   但通常不用重打帳密（瀏覽器仍有 Entra SSO session，會自動跳轉通過）。

---

## 九、實測環境參考值

> 以下是本次測試環境的值，**正式部署一律換成你自己的值**。

| 項目 | 值 |
|---|---|
| 租戶 ID | `<TENANT-ID>` |
| Enterprise App | `FortiGate-WiFi-Portal`（App ID `<APP-ID>`）|
| 群組 Object ID | `<GROUP-OBJECT-ID>`（群組名 `wifi-users`）|
| 測試帳號 | `wifiuser@contoso.onmicrosoft.com` |
| FortiGate | 60E v7.4.12 build2902 |
| FortiAP | 231F，bridge mode |
| 測試 VLAN | VLAN 100，`192.168.20.0/24`，SSID `SAML-Test` |
