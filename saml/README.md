# FortiGate Captive Portal + Microsoft Entra ID SAML

用 Microsoft Entra ID（原 Azure AD）當 SAML IdP，讓 FortiGate 的 Captive Portal（WiFi／有線 VLAN）改走 Microsoft 帳號登入的實作筆記。

> [!WARNING]
> 個人實測紀錄，**僅供參考，使用風險自負**。正式環境請先在測試機驗證。

## 測試環境

| 項目 | 版本／說明 |
|---|---|
| FortiGate | FortiGate-60E，FortiOS v7.4.12 build2902 |
| FortiAP | 231F，bridge mode |
| IdP | Microsoft Entra ID 免費版 |
| 多 VLAN 驗證 | FortiGate VM，FortiOS 7.2.0 |

## 文件

| 文件 | 內容 |
|---|---|
| [設定教學（純文字版）](captive-portal-entra-saml-guide.md) | CLI 指令、推送順序、10 個踩坑總表 |
| [圖文版](captive-portal-entra-saml-illustrated.md) | Entra 入口網站與 FortiGate GUI 截圖逐步對照、多 VLAN 做法 |
| [SAML 原理與 FortiGate 實作](saml-concepts-fortigate.md) | 三角色、信任鏈、Assertion 解剖、為何 802.1X 吃不到 SAML、IPsec VPN + SAML |

## 去識別化說明

文件與截圖中的識別資訊都已換成示範值：

- 租戶 ID、應用程式 ID、物件 ID、群組 ID → `<TENANT-ID>`、`<APP-ID>`、`<OBJECT-ID>`、`<GROUP-OBJECT-ID>`
- 憑證指紋 → `<THUMBPRINT>`；SAML 簽署憑證到期日、企業應用程式建立日 → `<憑證到期日>`／`<建立日>`（截圖上為 `<到期日>`／`<建立日>`）
- 租戶網域、帳號 → `contoso.onmicrosoft.com`、`admin@contoso.com`、`wifiuser@contoso.onmicrosoft.com`
- 內網 IP → `192.168.20.0/24`（VLAN 100）、`192.168.21.0/24`（VLAN 10）、`192.168.22.0/24`（VLAN 20）等示範網段；IdP 模擬主機用 RFC 5737 文件網段 `198.51.100.0/24`
- MAC → `xx:xx:xx:xx:xx:xx`

截圖上的遮罩是不透明色塊，再寫上示範值；部分截圖有裁切，文中會註明。
