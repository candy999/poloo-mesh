# PolooMesh 報路

> 報路（pò-lōo）：台語「告訴人家路怎麼走」。只告訴，走不走由對方決定。

讓 OpenWrt 的多台 AP 互相照應，替每支手機報路。插上同一個區網就自動找到彼此，記住每支手機在家裡每個位置「實際上哪台最好用」，再用 802.11k/v/r 告訴它往哪走。

- **零設定**：插上同一個區網，AP 就會自動找到彼此
- **只報路、永遠不踢**：不用訊號門檻斷線，手機不想換就退避
- **從經驗學習**：避開會斷線、會來回彈的 AP，不只看訊號強弱
- **退得回**：一個開關就回到只觀察

搭配任何有線或 mesh 回程使用，**本身不負責回程連線**（不是 802.11s／batman-adv／EasyMesh 回程）。

Experience-based Wi-Fi roaming for OpenWrt. Suggests, never kicks.

## 狀態

🚧 設計階段，還沒有程式碼。

| 文件 | 內容 |
|---|---|
| [`docs/DESIGN.md`](docs/DESIGN.md) | 設計規格 v1（開發以這份為準） |
| [`docs/DISCUSSION.md`](docs/DISCUSSION.md) | 討論紀錄與每個決定的理由 |
| [`docs/reference/padavan-twr2-protocol.md`](docs/reference/padavan-twr2-protocol.md) | padavan-tw 已實作的 AP 間溝通協定（TWR2） |
| [`docs/reference/fingerprint-spec-draft-v0.md`](docs/reference/fingerprint-spec-draft-v0.md) | 指紋規格草案 v0（已被 v1 取代，保留參考） |

## 需求（預定）

- OpenWrt 23.05 以上（ucode 與 hostapd ubus）
- 完整版 wpad（`wpad-mbedtls` 或 `wpad-openssl`），`wpad-basic-*` 不支援 11k
- 不要與 usteer、DAWN 同時使用
