# The Memory Subsystem

## 1. DRAM Organization

### Channel, Rank, Bank, Row and Column

SRAM / DRAM 的用途見[第一單元](01_How_a_System_Works.md)。本章聚焦 DRAM 內部組織如何影響存取。

| 層次 | 意義 |
| --- | --- |
| Channel | 一組記憶體介面，提供傳輸資源 |
| Rank | 配合回應的一組 DRAM chips，形成所需資料寬度 |
| Bank | 可分別管理 row 狀態的陣列 |
| Row / Column | 在 bank 內選取資料的位置 |

不同 banks 可重疊部分內部操作，但不代表各自擁有獨立的外部資料線；共用 channel 的資料傳輸仍受該介面容量限制。

### Row Buffer and Commands

DRAM 儲存的電荷需要定期 refresh。ACTIVATE 啟動 row，sense amplifiers 感測並恢復資料；這些電路形成 row buffer，再由 column 存取選擇資料。**Row buffer 在 DRAM 端，不是 CPU cache。**

PRECHARGE 關閉目前 row 並將 bitlines 準備好，供下一次 ACTIVATE 使用；「關閉」是這一步的效果，不是需要另外插入的一個通用 CLOSE 命令。資料恢復不是到 PRECHARGE 才開始，仍需遵守相關時序。

![DRAM 存取狀態與命令](https://raw.githubusercontent.com/ray13542/soc_note/main/assets/unit05/dram_access.png)

### Simplified First-Data Latency

自訂例子：tRP = 10 ns、tRCD = 15 ns、tCL = 15 ns。

| 起始狀態 | 主要命令 | 到第一筆資料的簡化延遲 |
| --- | --- | --- |
| 所需 row 已開啟：row hit | READ | tCL = 15 ns |
| Bank 已關閉且準備好 | ACTIVATE → READ | tRCD + tCL = 30 ns |
| 同 bank 開著另一個 row：row conflict | PRECHARGE → ACTIVATE → READ | tRP + tRCD + tCL = 40 ns |

假設第一個命令已可合法發出，忽略其他限制與等待。這不是完整 CPU load latency，也未計入 queue、interconnect、refresh 或整段資料傳輸；實際還有其他 DRAM timing 要求。

### Address Mapping

自訂例子：兩筆都落在同 bank 不同 rows，可能要切 row；若分散到不同 banks，則有機會重疊部分準備工作。Controller 的 address mapping 決定地址如何對應 channel / rank / bank / row / column，因此要依實際存取模式評估。

### Key Takeaways

* DRAM 延遲取決於 bank / row 狀態，不是固定常數。
* Row conflict 的主要順序為 PRECHARGE → ACTIVATE → READ。
* Bank 並行不等於外部資料介面完全獨立。
* **關鍵字：Channel、Rank、Bank、Row Buffer、ACTIVATE、PRECHARGE、Row Hit、Row Conflict。**

### Source

* [ChipVerify — DRAM Organization](https://chipverify.com/soc-design/dram-organization)
* 時序數值、存取案例與圖解為自訂教學整理。

## 2. Memory Controllers and Scheduling

### Controller vs PHY

```text
CPU / DMA → Interconnect
 → Controller：queue、地址映射、bank 狀態、timing、scheduler
 → PHY：高速實體訊號的驅動與取樣
 → DRAM
```

Controller 把存取要求轉成合法 DRAM 命令，安排 refresh，並依協定回傳資料及回應；PHY 處理實體介面。

### Scheduling and Row Locality

FCFS 依到達順序服務。FR-FCFS 優先考慮可發出的工作，典型策略偏好可服務的 row hits；仍須符合 timing、ordering 與 QoS。

![排程減少 row 切換](https://raw.githubusercontent.com/ray13542/soc_note/main/assets/unit05/dram_scheduling.png)

自訂例子：目前 Row A 開啟，queue 順序為 B、A、A，且三筆獨立、允許重排。

* FCFS：切到 B，服務第一筆，再切回 A 服務其餘兩筆。
* 偏好 row hits：先完成 A、A，再切到 B，少一次 row 切換。

若新的 A 請求不斷到來，B 可能 starvation。可用 aging 提高久候請求的優先程度，並配合有界服務政策。QoS 與 round robin 等原理見[第四單元](04_Interconnect_and_Data_Movement.md)，此處不重複展開。

### Page Policy

這裡的 page 指 DRAM row 管理，不是作業系統的 virtual-memory page。

| 政策 | 行為與取捨 |
| --- | --- |
| Open-page | 留 row 開啟，期待後續 row hit；換 row 時可能先需 precharge |
| Closed-page | 存取後安排關閉，可能提前完成 precharge，但放棄後續 row hit |
| Adaptive | 依存取模式調整，增加判斷與狀態管理 |

沒有一種政策對所有流量都最佳；closed-page 也不代表實際每筆 latency 完全相同。

### Write Queue and Correct Data

自訂例子：DRAM[X] = 10，write queue 有尚未寫入的 X = 99；接著有一筆**必須看到這次寫入結果**的 read X。

```text
Read X
  ├─ 從 write queue 取得符合順序的最新資料 → 回傳 99
  └─ 等相關 write 完成，再讀 DRAM        → 回傳 99
```

不能直接回傳舊值 10。實作還需處理部分 byte writes、重疊範圍與多筆寫入的順序。重新排程可以改變執行安排，不能改變規定的資料結果。

### Key Takeaways

* Controller 排程並遵守 DRAM 時序；PHY 處理實體訊號。
* Row-hit 排程可能提升效率，但需防止其他請求長期被跳過。
* Page policy 決定何時保留或關閉 row；scheduler 決定服務順序。
* Read 要考慮待處理 writes，維持要求的資料順序。
* **關鍵字：Memory Controller、PHY、FCFS、FR-FCFS、Page Policy、Write Queue、Ordering。**

### Source

* [ChipVerify — Memory Controllers and Scheduling](https://chipverify.com/soc-design/memory-controllers)
* A/B 排程、write queue 數值案例與圖解為自訂教學整理。

## 3. Memory Bandwidth Budgets

待學習。

## 4. On-Chip SRAM, Scratchpads and System Caches

待學習。
