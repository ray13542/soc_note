# Clocks, Resets and Power

## 1. Clock Architecture and CDC

### Clock Relationships

Reference clock 可經 PLL、divider 產生 CPU、interconnect 和周邊時脈；always-on timer 需確認休眠時仍有有效時脈。不同頻率不必然代表 asynchronous：若相位與頻率關係已知、且正確建模，可用 STA 分析；無法依賴固定關係的 crossing 才需適當 CDC 設計。

CDC 基礎見[第一單元](01_How_a_System_Works.md)，本章補充實作與實體限制。

| 傳遞內容 | 常用方式與限制 |
| --- | --- |
| 持續維持的單一 bit | 多級 synchronizer 降低 metastability 傳出的機率 |
| 短暫事件 | 握手或合適的 toggle 協定；必須限制事件間距 |
| 多 bit 控制資料 | 穩定資料配合握手，不能各 bit 獨立同步後直接拼接 |
| 連續資料流 | Async FIFO，搭配 full / empty 與 backpressure |

例如 2 ns pulse 進入週期 10 ns 的 domain，2FF 仍可能完全沒取樣到；降低 metastability 風險不等於保證接收到事件。

### Async FIFO and Conservative Flags

資料存於雙埠儲存體；read / write pointer 各自在自己的 domain 更新，轉成 Gray code 後跨域。對方看到的是延後的 pointer，因此 full / empty 的解除可能較保守。有限深度不能解決長期平均寫入率大於讀出率，仍需 backpressure。

![Gray pointer 的跨域路徑與 full 解除](https://raw.githubusercontent.com/ray13542/soc_note/main/assets/unit07/gray_cdc.png)

### Registered Gray Pointer: RTL Fragment

以下是示意片段，省略 storage、full / empty 產生與完整 reset 協定；不是可直接使用的 FIFO。Reset release 需各 domain 安全處理，並協調 FIFO 狀態。

```systemverilog
// 每次成功讀取，binary pointer 前進一步。
assign rbin_next  = rbin + (r_en && !r_empty);
assign rgray_next = (rbin_next >> 1) ^ rbin_next;

always_ff @(posedge rclk or negedge rrst_n)
  if (!rrst_n) begin
    rbin  <= '0;
    rgray <= '0;
  end else begin
    rbin  <= rbin_next;
    rgray <= rgray_next;
  end

// Attribute 語法與效果需依工具確認。
(* ASYNC_REG = "TRUE" *) logic [PTR_W-1:0] rq1, rq2;
always_ff @(posedge wclk or negedge wrst_n)
  if (!wrst_n) begin
    rq1 <= '0;
    rq2 <= '0;
  end else begin
    rq1 <= rgray;
    rq2 <= rq1;
  end
// 寫端判斷使用 rq2，不使用 rq1。
```

先 register Gray code，避免 binary-to-Gray 組合轉換的 glitch 直接跨域。相鄰合法更新只改一 bit，前提是 pointer 每次只前進一步；目的端可能跳過中間狀態，不要求連續兩次取樣值也只差一 bit。

### Why Gray Still Needs Timing Constraints

自訂反例：來源每 2 ns 最多更新一次。

| 來源更新 | 改變的 bit | 路徑延遲 | 抵達第一級 D 的時間 |
| --- | --- | --- | --- |
| t = 0：000 → 001 | bit 0 | 3 ns | 3 ns |
| t = 2：001 → 011 | bit 1 | 0.2 ns | 2.2 ns |

接收端輸入會先看到 010，再變成 011，更新順序被線路延遲打亂。Gray 的 RTL 性質不足以保證實體電路保持相同順序。

約束重點是 **source Gray register → destination 第一級 synchronizer**。依來源最短更新間隔與實體餘裕限制延遲／skew；兩個方向要各自分析，不能一律用較慢接收時脈的週期。

以下為 Vivado XDC 示意，假設來源最快 2 ns 更新一次，教學上選 1.5 ns 預算。數值與 netlist 名稱必須依設計調整，並非通用 signoff 設定。

```tcl
# 示意 synthesized pin 名稱；確認集合涵蓋所有 pointer bits。
set gray_q [get_pins -hierarchical -regexp {.*u_fifo/rgray_reg\[[0-9]+\]/Q}]
set sync1_d [get_pins -hierarchical -regexp {.*u_fifo/rq1_reg\[[0-9]+\]/D}]
set_max_delay -datapath_only 1.5 -from $gray_q -to $sync1_d
set_bus_skew 1.5 -from $gray_q -to $sync1_d
```

* `set_max_delay -datapath_only` 限制個別資料路徑的延遲；`set_bus_skew` 限制各路徑抵達時間的相對差異。語法依工具而異，並非所有 SDC 工具都支援 `set_bus_skew`。
* 第一級 → 第二級是同一個目的 clock domain，保留正常 timing analysis；靠近放置、避免中間組合邏輯，讓 metastability 有足夠 settling time。
* Vivado 的廣泛 `set_clock_groups -asynchronous` 或 `set_false_path` 可能蓋掉 max-delay 約束。需安排適當的逐路徑例外並檢查 ignored / overridden exceptions，不能認為寫入檔案就生效。
* 檢查集合是否為空、bit 數是否完整，再看 post-route timing、bus-skew 與 CDC 報告。Bus-skew assertion 與一般 timing exception 的作用不同。

### Reducing Full-Release Delay

讀端釋放空間 → read pointer 更新 → 寫端 sync1 → sync2 → full 判斷 → 寫入重新允許。

| 方法 | 效果與限制 |
| --- | --- |
| 消除 sync2 後多餘 pipeline | 可降低 flag / 控制延遲；仍需保持同步安全與時序正確 |
| 檢討 synchronizer 級數 | 例如 3 級改 2 級，必須先以元件、頻率與 settling time 評估 MTBF |
| 增加 FIFO 深度或 staging buffer | 可吸收流量突發、減少 full 停頓；不縮短 pointer 跨域拍數 |
| 更換為其他已驗證架構 | 屬於架構重設計，需另外驗證 CDC 與 flag 行為 |

**不能直接使用第一級輸出去提早解除 full。** 第一級可能 metastable；不是說它經常發生，而是它尚未提供設計要求的可靠性。也不能憑猜測讀端將釋放空間就先寫入。

面試回答可先區分「減少回報延遲」與「降低滿載停頓」：先檢查安全同步之後是否多了控制拍數，再評估深度與流量；若調整同步級數，需提出 MTBF 依據。

### Clock Gating and Switching

直接用 `clk & enable`，若 enable 在 clock 高電位時改變，可能產生短脈衝。ASIC 使用適當 ICG，FPGA 使用專用 clock / enable 資源；切換 clock source 也需適當 glitch-free 機制。

### Key Takeaways

* Synchronizer、事件不遺失、多 bit 一致性，是不同設計問題。
* Gray code 需要正確 RTL，也需要來源到第一級的實體延遲／skew 保證。
* Full 保守解除是可靠性代價；增加深度改善停頓，不改變同步拍數。
* **關鍵字：Clock Domain、CDC、Metastability、Gray Code、Bus Skew、MTBF、Async FIFO、ICG。**

### Sources

* [ChipVerify — Clock Architecture and CDC](https://chipverify.com/soc-design/clock-architecture-and-cdc)
* [AMD — Bus Skew Constraints](https://docs.amd.com/r/2022.2-English/ug903-vivado-using-constraints/About-Bus-Skew-Constraints)
* [AMD — Overridden Max Delay Datapath Only](https://docs.amd.com/r/2022.2-English/ug906-vivado-design-analysis/TIMING-24-Overridden-Max-Delay-Datapath-Only)
* RTL、約束、數值案例與圖解為教學示意，需依實際設計與工具驗證。

## 2. Reset Architecture and Reset Domain Crossings

待學習。

## 3. Power Domains and Power Management

待學習。

## 4. Dynamic Voltage and Frequency Scaling

待學習。
