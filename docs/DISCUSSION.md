# 討論紀錄（2026-11-25）

這份是 PolooMesh 從構想到命名的討論整理，保留每個決定的理由。正式規格見 `DESIGN.md`。

---

## 1. OpenWrt 原生的 k/v/r 支援，與 usteer 怎麼判斷切換

### 1.1 核心觀念

**最終決定要不要切換 AP 的是客戶端**。k/v/r 只幫忙：
- **11k**：告訴客戶端附近有哪些 AP（鄰居報告），也能請客戶端回報它量到的各 AP 訊號（Beacon Report）。
- **11v**：AP 用 BSS Transition Management（BTM）「建議」客戶端換到別台。
- **11r**：換過去時快速重新認證（Fast Transition）。

AP 要主動判斷「該不該叫客戶端換」，得另外跑常駐程式（OpenWrt 上是 usteer 或 DAWN）。

### 1.2 OpenWrt 設定重點

- 需要完整版 wpad：`wpad-basic-mbedtls` 不支援 11k，`wpad-mbedtls` 三個都支援（24.10）。
- 每台 AP 的 SSID、密碼、`mobility_domain` 要相同：

```
option ieee80211r '1'
option mobility_domain 'a1b2'
option ft_over_ds '0'
option ft_psk_generate_local '1'
option ieee80211k '1'
option rrm_neighbor_report '1'
option rrm_beacon_report '1'
option bss_transition '1'
option wnm_sleep_mode '1'
```

### 1.3 usteer 的判斷流程（參考，PolooMesh 刻意不同）

1. 客戶端在目前 AP 的訊號掉到 `roam_scan_snr` 以下 → 用 11k Beacon Request 請客戶端掃描回報。
2. 別台比目前強超過 `signal_diff_threshold` → 用 11v BTM 建議換過去。
3. 客戶端照做時靠 11r 快速認證。
4. 不理會的話，`roam_kick_delay` 逾時後**踢掉**（PolooMesh 不做這一步）。

官方建議最小設定：`roam_scan_snr '-65'`、`signal_diff_threshold '8'`。陷阱：`signal_diff_threshold` 要大於 0 才會引導；`roam_scan_snr` 一定要設才會觸發掃描。

### 1.4 實務提醒

- 有些裝置開了 k/v/r 反而變差；基本的漫遊（發射功率調低、頻道錯開、覆蓋不要重疊太多）沒調好之前，加上 k/v/r 也幫不上忙。
- 舊裝置連不上多半是 11r 造成的，可以另開一個沒有 11r 的 SSID（padavan-tw 的 `_IoT` 網路做法）。

---

## 2. padavan-tw TWR2 規格的檢討（`reference/padavan-twr2-protocol.md`）

**結論**：設計可行。單台 D3 已跑通「問 → 回報 → 建議 → 回覆」；沒驗證的是多台實際環境與各家手機行為，不是協定本身。移植到 OpenWrt 比 padavan 簡單，hostapd ubus 的方法都現成。

**建議修改**：
1. **混用時 11r 位元會誤導手機**：7.1 節「兩邊都有 `r` 就設 MD 位元」，但 padavan 與 OpenWrt 之間 FT 不通，手機會先做 FT、失敗才退回完整認證，有些會卡住。→ `R` 行尾加 MDID 與 FT 做法，相同才設。
2. **OpenWrt 上 11r 不需要 R0KH／R1KH**：PSK 用 `ft_psk_generate_local '1'`，各 AP 自己推金鑰，也沒有 padavan 第 8 節預設金鑰的風險。第 14 節表格最後一列可以改。
3. **`R` 行要多帶 Neighbor Report hex**：OpenWrt 的 `bss_transition_request`、`rrm_nr_set` 吃 hex 元素，`rrm_beacon_req` 要 op class。把 `rrm_nr_get_own` 的值接在 `R` 行最後（4.5 節允許行尾加欄位）。`rrm_nr_get_own` 偶爾回 `Not found`，要快取。
4. **群播是最大的實務風險**：IGMP snooping 沒查詢者、Mesh／無線回程、群播轉單播，報告都可能默默消失；而 7.3 節確認換過去完全靠收到報告。→ 廣播退回（第 12 節提案 1）提前做，最省事是一開始就群播＋廣播一起送。
5. **能被引導的裝置可能比預期少**：很多手機不回或回空的 Beacon Report，照「裝置說了算」就完全不會被引導。不是 bug，但要有心理準備。
6. **小問題**：報告簽章（12 節提案 3）要加序號或時間戳防重放；OpenWrt 上不要和 usteer／DAWN 同時跑。

建議順序：padavan 上先補廣播退回與 11r 位元判斷 → CR6608 刷好後做兩台實測 → 再移植 OpenWrt。

---

## 3. 「訊號指紋地圖」的構想

**問題**：能不能讓 AP 之間溝通、產生訊號指紋地圖，輔助裝置精準切換？

**回答**：可以，但不做成真正的室內地圖，而是「經驗表」：記下每次建議過去之後結果是好是壞，以後同樣狀況照經驗處理。

**指紋不必由 AP 自己量**：手機只會被服務 AP 聽到，別台在不同頻道收不到它的資料封包；手機掃描時常用隨機 MAC，AP 對不回是誰。最可靠的指紋就是 11k Beacon Report——手機自己量到的各 AP 訊號。

**做法**：把位置分成「區」，每區記下結果；成功不只看有沒有換過去，還要換過去後穩定、沒跳回來。

**經驗夠了以後**：只量經驗中最好的那台的頻道，確認後直接建議（少量測、省電、少打擾）；常來回彈的區標成「不要動」；加上訊號下降趨勢可以提早處理；經驗不足照固定規則。

**限制**：環境會變，舊經驗要淡化；不同手機天線差很多，經驗要分裝置；不支援 11k／11v 的裝置一樣幫不上忙；家裡只有 2、3 台 AP 時，好處是少跳、少量測，不會有明顯的換手速度提升。

**順序**：先多台實測，同時把 Beacon Report 結果與事後有沒有跳回來寫進紀錄；兩週後看資料再決定要不要開預測。

這個構想後來寫成 `reference/fingerprint-spec-draft-v0.md`。

---

## 4. 指紋規格 v0 的檢討（`reference/fingerprint-spec-draft-v0.md`）

**結論**：可以照著做，底線守得住（學習只在限定範圍內微調、冷啟動照固定規則、上線前先跑 shadow）。以下 8 點已全部改進 `DESIGN.md` v1（對照表見 `DESIGN.md` 第 15 節）。

1. **不要開 `notify_response`**：v0 說要先呼叫才收得到探測通知，這是錯的——不開也收得到，只是 hostapd 不等回覆。開了之後 hostapd 每個探測請求都等訂閱者回覆（最多 100 ms），官方文件警告訂閱者沒回時 hostapd 會停止回應探測請求。程式卡住，AP 就從手機清單上消失。
2. **指紋缺值要分兩種**：7.2 節「先量前三名頻道」與 5.2 節「沒出現補 −95」衝突，「沒量」被當成「聽不到」，產生假位置。→ 樣本記下量了哪些頻道，距離只比兩邊都量過的頻道。
3. **ubus 對照漏了 BTM 回覆**：同一個訂閱會收到 `bss-transition-response`（含狀態碼）。
4. **協調者改成「模型跟著裝置走」**：5.1 節已經每台裝置各自分群，不需要全家一致的位置編號。模型放在服務 AP，換 AP 時新 AP 從 `gone` 知道來源，直接要模型；AP 重開機就向全區網詢問。協調者選舉、快照、ack、500 筆佇列全部拿掉，決策也不用等 `loc` 回覆。
5. **學習實際能學到的比預期少**：不探索時替代 BSS 的經驗很難累積到把握 ≥ 3，「放寬」很少觸發；q 大多落在 0.75 附近，差到 0.2 幾乎只會來自掉線或來回彈。實際上學到的是「避開這個位置常掉線、常來回彈的 BSS」——仍然有價值。→ shadow 模式量「雙經驗位置數」。
6. **評分修正**：出門斷 Wi-Fi 不算壞（只算 60 秒內又連回來的斷線）；來回彈片段不受「閒置不評分」排除。
7. **第一版可以砍**：AP 指紋、對照表、探測請求訂閱；有流量時不送 Beacon Request（手機量測要離開頻道，會頓一下）。
8. **開工前確認 `ucode-mod-socket`**：以及能不能加入群播群組；不能就直接用廣播。

---

## 5. 命名

**過程**：ShirubeRoam（しるべ，路標）→ 想要台味：Sikloo（熟路）、Tshualoo（𤆬路）、Loopai（路牌）、Shunfeng（順風）、Tudigong（土地公）、Hitping（彼爿）→ 想用 Mesh 結尾 → 燈塔 LighthouseMesh（但 Lighthouse 在 GitHub 已被 Google、以太坊佔用）→ 台味「指路」：**PolooMesh（報路）**、TshualooMesh、KilooMesh、SunlooMesh、TautinMesh（鬥陣）、MazuMesh。

**定案**：`poloo-mesh`，對外顯示「PolooMesh 報路」。
- 報路（pò-lōo）：台語「告訴人家路怎麼走」。只告訴，走不走由對方決定——對應「只報路、永遠不踢」。
- 用小寫連字號：跟 OpenWrt 套件命名習慣一致（`sqm-scripts`、`luci-app-sqm`），repo 名等於套件名；GitHub 會把連字號當分詞，搜 `poloo` 或 `mesh` 都找得到。

**關於 Mesh 這個詞**：嚴格說這不是 mesh 開發。mesh 包含「回程」（AP 之間怎麼連、封包怎麼轉送）與「漫遊協調」兩件事，本專案只做後者。所以簡介第一段一定要寫清楚「不負責回程連線」。

**命名對齊**：

```
套件：poloo-mesh
LuCI：luci-app-poloo-mesh
常駐程式：poloo-meshd
設定檔：/etc/config/poloo-mesh
ubus：poloo-mesh
```

**GitHub About（英文）**：

```
Poloo (報路, "telling the way" in Taiwanese) — experience-based Wi-Fi roaming for OpenWrt. APs on the same LAN find each other with zero config, learn per device and per location which AP actually works best, then point clients there via 802.11k/v/r. Suggests, never kicks. Works over any wired or mesh backhaul; does not set up the backhaul itself.
```

**Topics**：

```
openwrt hostapd wifi roaming mesh band-steering 80211k 80211v 80211r ucode
```

---

## 參考資料

- [OpenWrt：Setting up usteer and band-steering](https://openwrt.org/docs/guide-user/network/wifi/usteer)
- [OpenWrt：Wifi Roaming](https://openwrt.org/docs/guide-user/network/wifi/roaming)
- [OpenWrt：Setting up DAWN](https://openwrt.org/docs/guide-user/network/wifi/dawn)
- [hostapd ubus 文件（notify_response、bss_transition_request 等）](https://git.openwrt.org/33e7f7c0285619c3be58ae7db5617bea9938a46a)
- [hostapd：新增 bss-transition-response 通知](https://git.openwrt.org/a3de42e72ce349c14263737c5d6ec7b00397cf14)
- [hostapd：bss_transition_request 方法的 patch](https://lists.openwrt.org/pipermail/openwrt-devel/2021-October/036645.html)
- [OpenWrt 論壇：rrm_nr_get_own 偶爾 Not found](https://forum.openwrt.org/t/git-actions-for-issues-does-not-work-sometimes/214985)
- [OpenWrt 論壇：Proper configuration of 802.11k and 802.11v](https://forum.openwrt.org/t/proper-configuration-of-802-11k-and-802-11v/152994)
- [OpenWrt 論壇：k/v/r 不一定讓漫遊變好](https://forum.openwrt.org/t/802-11kvr-roam-will-not-actively-trigger-roaming/180349)
