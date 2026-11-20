# FortiGate 實作教學（lab 實測）

FortiGate 設定的實作筆記，每篇都在 lab 實機跑過，並標明哪些步驟還沒實測。

| 主題 | 內容 |
|---|---|
| [localin/](localin/) | 用 local-in policy 保護 SSL VPN／IKE 入口，並搭配 threat feed 自動封鎖掃描者（FortiOS 7.4.3） |
| [saml/](saml/) | Captive Portal 串 Microsoft Entra ID SAML 登入，另有 SAML 原理說明 |

## 使用前先知道

- 這些是個人的 lab 實測紀錄，**不是 Fortinet 或 Microsoft 的官方文件**。內容僅供參考，套用到正式環境請先自行測試，風險由使用者自負。
- 文中的 IP、帳號、主機名稱、租戶資訊都已換成範例值：內網用 `192.168.x.x`，公網用 RFC 5737 文件網段（`192.0.2.x`、`198.51.100.x`、`203.0.113.x`），網域用 `contoso`。
- 標「未實測」或「未查證」的內容，請自行確認後再使用。

## 授權

本 repo 的文字與圖片採用 [CC BY 4.0](LICENSE) 授權：可以轉載、修改，但要註明出處。
