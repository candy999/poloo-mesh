# poloo-mesh 開發指引

OpenWrt 多 AP 漫遊常駐程式（ucode）。規格以 `docs/DESIGN.md` 為準；動手前先讀它，改設計時同步更新它並在第 15 節記一筆。

## 不能違反的原則

1. **永遠不踢**：不送 Deauth／Disassoc；BTM 一律 `disassociation_imminent: false`、不帶斷線計時。
2. **永遠不呼叫 hostapd 的 `notify_response`**：程式卡住時 hostapd 必須照常運作。
3. 學習只能在 `DESIGN.md` 第 7.2 節的範圍內調整；連續兩次、退避、每日上限不能由學習改變。
4. 不支援 11k／11v 的裝置完全不碰。
5. Flash 寫入每天最多 4 次；其他狀態只寫 `/tmp`。
6. `mode = observe` 時不送任何 Beacon Request 或 BTM。

## 開發方式

- 依 `DESIGN.md` 第 13 節的階段做，一次一個階段；階段 0（硬體確認）的結果寫進 `docs/HARDWARE.md`。
- hostapd ubus 的方法名稱與欄位依版本而定，不要憑記憶，以 `ubus -v list 'hostapd.*'` 與實際通知內容為準。
- 演算法（分群、距離、評分、衰減）放在不依賴 ubus／socket 的獨立模組，才能在電腦上用離線重播測試。
- 參數集中在一個檔案，名稱與 `DESIGN.md` 第 12 節一致。
- 每個建議都要寫決定紀錄：裝置、位置、候選、訊號差、用了哪段經驗。

## 命名

```
套件：poloo-mesh
LuCI：luci-app-poloo-mesh
常駐程式：poloo-meshd
設定檔：/etc/config/poloo-mesh
ubus：poloo-mesh
UDP／TCP 埠：27330
```

## 文件語言

文件與註解用繁體中文；程式識別字用英文。
