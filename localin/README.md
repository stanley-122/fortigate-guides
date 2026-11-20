# FortiGate 7.4.3 Local-in Policy：保護 SSL VPN 與 IPsec 入口

用 local-in policy 讓 FortiGate 的 SSL VPN 埠與 IKE（UDP 500／4500）只接受台灣來源，並搭配外部惡意 IP 清單（threat feed）自動封鎖掃描者。

> [!WARNING]
> 測試環境：lab 的 FortiGate 60E、FortiOS 7.4.3（build2573）。內容僅供參考，不是 Fortinet 官方文件；請先在自己的測試環境驗證，套用到正式環境的風險由使用者自負。

## 檔案

| 檔案 | 用途 |
|---|---|
| [FortiGate-LocalIn設定教學_7.4.3_20261120.md](FortiGate-LocalIn設定教學_7.4.3_20261120.md) | 純文字版：完整 CLI、事前檢查、驗證、退回方法與官方出處，部署時照著做 |
| [FortiGate-LocalIn設定教學_7.4.3_20261120圖文版.md](FortiGate-LocalIn設定教學_7.4.3_20261120圖文版.md) | 圖文版：每一步的實機 CLI 輸出（轉成圖片），教學與交接用 |
| `images/` | 圖文版用的 31 張圖 |

## 閱讀前先知道

- 文中標「2026-11-20 實機驗證」的是在 lab 實機跑過的；標「未實測」「未查證」的就是還沒驗證，請自行確認。
- 主要未實測項目：規則「只放台灣」（lab 來源是私有位址）、國外來源被全擋規則擋下、SSL VPN 登入失敗觸發封鎖、`detect-unknown-esp`、IKE over TCP 4500、多 WAN／SD-WAN。
- IP、帳號、主機名稱都已換成範例值：內網 `192.168.10.x`，公網用 RFC 5737 文件網段（`198.51.100.x`、`203.0.113.x`）。
- 圖片是把實機 CLI 文字輸出重新渲染成圖，不是螢幕截圖；除了去識別化替換外，文字逐字取自實機紀錄。
- 文中的「收集器」是另一支 syslog 收集程式（不在本 repo），作用是收 syslog → 判定要封鎖的 IP → 產生純文字 IP 清單給 FortiGate 的 threat feed 拉取。任何能產生同樣格式清單的工具都可以替代。
