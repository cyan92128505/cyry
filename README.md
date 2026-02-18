# cyry

# 無線分離式軌跡球鍵盤：從零開始的完整開發指南

**ZMK + Seeed XIAO nRF52840 的無線分離式 40% ortholinear 軌跡球鍵盤專案，在 2026 年已具備成熟的技術基礎可以實現。** ZMK 主線已於 2024 年 12 月合併 pointing device 支援（PR #2477），PAW3222 感測器擁有由日本鍵盤社群維護的完善驅動模組，XIAO nRF52840 是 ZMK 官方支援的開發板。台灣可透過 iCShop、蝦皮、JLCPCB 與 AliExpress 取得絕大多數零件，**預估總成本約 NT$3,000–5,500（不含外殼 3D 列印）**。以下是完整的架構設計、分階段開發路線圖與台灣採購指南。

---

## 一、硬體架構與核心技術選型

### 主控板：Seeed XIAO nRF52840

每半邊各使用一片 XIAO nRF52840（ZMK board identifier：`seeeduino_xiao_ble`）。這是 ZMK 官方支援的開發板，搭載 Nordic nRF52840 晶片（ARM Cortex-M4F @ 64MHz）、Bluetooth 5.0、USB-C，以及**內建 BQ25101 鋰電池充電管理晶片**——直接焊接 LiPo 電池到背面的 BAT+/BAT- 焊盤即可，USB 插入時自動充電，拔除後自動切換為電池供電。

**關鍵限制是 GPIO 數量：標準版僅有 11 個 GPIO（D0–D10）**。若矩陣使用 4 行 × 5 列就佔掉 9 個 pin，軌跡球的 SPI 介面需要 4 個 pin（SCK、MOSI/MISO 共用、CS、IRQ），將面臨嚴重的 pin 不足問題。解決方案有三：

- **使用 XIAO nRF52840 Plus**（新版本，背面有額外的 castellated pad，總計 **20 個多功能 GPIO**），iCShop 售價約 NT$485
- 在非軌跡球側使用標準版即可（11 pin 足夠 4×5 矩陣 + 少量備用）
- 使用 IO 擴展器（如 MCP23017）或移位暫存器減少矩陣佔用的 pin 數

### 分離式通訊：BLE Central-Peripheral 模型

ZMK 的分離式架構採用 **Central（中央）/ Peripheral（周邊）模型**。預設左半為 Central（執行 keymap 邏輯、與主機連線），右半為 Peripheral（透過 BLE GATT 傳送按鍵事件至 Central）。兩半自動配對，無需手動操作。**額外延遲約 3.75ms（平均），最差 7.5ms**——對打字完全可接受。

由於軌跡球在右半（Peripheral 側），指標事件需透過 BLE 傳輸至 Central。自 PR #2477 合併後，ZMK 主線已原生支援 peripheral 側的 pointing device，使用 `zmk,input-split` 裝置機制。另一種策略是**將右半設為 Central**（在 Kconfig 中設定 `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`），讓軌跡球直接連接到主控端，減少延遲。

### 軌跡球模組：PAW3222 光學感測器

**PAW3222 是日本無線鍵盤社群的主流選擇**——torabo-tsuki 全系列、cool642tb、DYA Dash、roBa 等專案全部採用此感測器。它是 PixArt 出品的低功耗光學滑鼠感測器，透過 SPI 介面通訊，CPI 範圍 608–4826，非常適合電池驅動的軌跡球應用。

ZMK 驅動模組方面，有三個選擇：

- **sekigon-gonnoc/zmk-driver-paw3222**：原版驅動，穩定可靠，由 torabo-tsuki 作者 Sekigon 維護
- **nuovotaka/zmk-driver-paw3222-alpha**：增加了基於 layer 的輸入模式切換（移動/捲動/精確模式）
- **tyama1531-lab/zmk-driver-paw3222-alpha2**：進一步增加了 behavior-based 模式切換、執行時 CPI 調整、旋轉補償功能

推薦先從 sekigon-gonnoc 原版開始，穩定運作後再考慮升級到 alpha2 版本。整合方式是在 `west.yml` 中加入模組引用，無需 fork ZMK 本體。

**SPI 接線需要 4 個 GPIO**：SCK、MOSI/MISO（PAW3222 使用單線 SDIO，所以 MOSI 和 MISO 接同一根線）、CS（Chip Select）、IRQ（動作中斷）。Devicetree overlay 配置範例：

```dts
&spi0 {
  status = "okay";
  compatible = "nordic,nrf-spim";
  cs-gpios = <&gpio0 PIN_CS GPIO_ACTIVE_LOW>;
  trackball: trackball@0 {
    compatible = "pixart,paw3222";
    reg = <0>;
    spi-max-frequency = <2000000>;
    irq-gpios = <&gpio0 PIN_IRQ GPIO_ACTIVE_LOW>;
  };
};
```

### 電池與電源管理

XIAO nRF52840 的 BQ25101 PMIC 提供內建的 LiPo 充電功能（充電電流 50mA 或 100mA），因此**不需要額外的 TP4056 充電模組**。直接將 3.7V LiPo 電池焊接到 XIAO 背面的 BAT+/BAT- 焊盤即可。建議加裝 SPDT 滑動開關控制電源通斷。

ZMK 內建了閒置休眠模式（`CONFIG_ZMK_SLEEP=y`），在深度睡眠下 nRF52840 的電流低至 **~5µA**。使用 110mAh（301230 規格）電池可維持數週使用；250mAh（502030）則可達數月。ZMK 官網的 Power Profiler（zmk.dev/power-profiler）可精確估算電池壽命。

有線模式：當 USB-C 連接時，XIAO 自動從 USB 供電並為電池充電。**即使沒有電池，純 USB-C 連接也能正常運作**——這滿足了「沒電池時可用有線模式」的需求。

---

## 二、NuPhy Air60 零件重用的現實考量

**這是本專案最需要提前釐清的相容性問題。** NuPhy Air60 使用的是 **Gateron Low-profile Mechanical（KS-33）** 軸體——這既不是標準 Cherry MX、也不是 Kailh Choc v1/v2，而是 NuPhy/Gateron 的專有低矮軸格式。

**軸體重用的困難**：KS-33 的引腳間距與安裝孔位和 MX/Choc 均不相容。若要重用這些軸體，必須在 PCB 設計時使用 KS-33 專用 footprint，這意味著現有開源鍵盤 PCB 設計幾乎都無法直接套用。KiCad 中的 KS-33 footprint 資源較少，需要自行測量繪製或尋找社群資源。

**鍵帽重用的可能性較高**：NuPhy 的低矮鍵帽使用類 MX 十字軸心（cross stem），理論上可安裝在 Kailh Choc v2 軸體上（Choc v2 也採用 MX 十字軸心）。但需注意 Choc v2 的整體高度與 KS-33 不同，鍵帽與外殼之間的間隙可能需要調整。

**務實建議**：考慮改用 **Kailh Choc v2 軸體**（MX 十字軸心相容，可沿用 NuPhy 鍵帽）搭配 Choc v2 hotswap 插座設計 PCB。這樣既能重用鍵帽，又能使用日本社群已驗證的 PCB 設計模式（cool642tb-mini 即使用 Choc v2）。

---

## 三、參考鍵盤分析與設計啟發

### 最直接的參考：torabo-tsuki OM 與 cool642tb

在所有參考鍵盤中，**torabo-tsuki OM**（Ortholinear Mini）和 **cool642tb(-mini)** 與本專案的需求最為吻合——兩者都是 ortholinear 格子佈局、無線分離式、PAW3222 軌跡球、ZMK 韌體。

**torabo-tsuki OM**（GitHub: sekigon-gonnoc/torabo-tsuki-om）採用 **17mm 鍵距**的方正格子佈局（比標準 19mm 更緊湊）、19mm 軌跡球、AA 乾電池供電、ZMK 韌體。PCB 設計檔案與 3D 列印外殼檔案皆已開源。它由 Sekigon 設計——即 PAW3222 ZMK 驅動的作者本人，因此 PAW3222 整合的參考價值極高。

**cool642tb-mini**（原版 GitHub: telzo2000/cool642tb）同樣是 42 鍵 ortholinear、17mm 鍵距、25mm 軌跡球、XIAO BLE + ZMK。特點是使用 AAA 乾電池（每側 2 顆），刻意避開 LiPo 以簡化設計。在遊舍工房（Yushakobo）售價 ¥32,500。

**DYA Dash**（GitHub: cormoran/dya-dash-keyboard）則帶來了創新的觸控感測器模式切換和磁鐵吸附收納設計，對腰掛使用場景特別有參考價值。它支援 ZMK Studio 即時改鍵，使用 AAA 電池。

**roBa**（GitHub: kumamuk-git/roBa）是社群最受歡迎的專案（265 星），42 鍵 column-staggered 設計，含水平旋轉編碼器，GPLv3 完全開源，build guide 完整，適合學習 ZMK 分離式鍵盤的整體結構。

**LiNEA40** 作為外型參考最合適——40 鍵低矮無線分離式，使用 XIAO BLE，支援 ZMK Studio。但硬體設計未公開，組裝指南在 note.com 上有詳細圖文教學。

### 各專案技術對照

| 特性 | torabo-tsuki OM | cool642tb | DYA Dash | roBa | LiNEA40 |
|------|----------------|-----------|----------|------|---------|
| 佈局 | Ortholinear 17mm | Ortholinear 17mm | Semi-staggered | Column-staggered | Ortholinear |
| 鍵數 | ~40 | 42 | ~36-40 | 42 | 40 |
| MCU | nRF52840 系列 | XIAO BLE | nRF52840 系列 | nRF52840 系列 | XIAO BLE |
| 感測器 | PAW3222 | PAW3222 | PAW3222 | 軌跡球 | PAW3222 |
| 電池 | AA ×1/側 | AAA ×2/側 | AAA ×1/側 | LiPo | LiPo |
| 開源 PCB | ✅ | ✅ (原版) | ✅ (CC BY-NC) | ✅ (GPL-3.0) | ❌ |
| 韌體 | ZMK | ZMK | ZMK | ZMK | ZMK |

---

## 四、分階段開發路線圖

### 第一階段：基礎學習與巨集鍵盤（第 1–3 週）

**目標**：在麵包板上完成一個 4 鍵巨集鍵盤，學會焊接、ZMK 基本配置與 GitHub Actions 建置流程。

- 購買焊接練習套件（LED 閃爍套件），練習基本穿孔焊接
- 用 1 片 XIAO nRF52840 + 4 個按鍵開關 + 4 個 1N4148 二極體，在麵包板上搭建 2×2 矩陣
- 建立個人的 `zmk-config` GitHub repo，學會透過 GitHub Actions 自動建置 UF2 韌體
- 學會雙擊 XIAO Reset 鍵進入 Bootloader（出現 `XIAO-SENSE` 磁碟），拖放 .uf2 刷入韌體
- **學習資源**：ubiqueiot.com 的 XIAO ZMK macropad 教學、zmk.dev/docs/development/hardware-integration/new-shield

**所需技能**：基礎穿孔焊接、Git 基本操作、鍵盤矩陣掃描原理（了解二極體為何防止 ghosting）

### 第二階段：單半鍵盤有線原型（第 4–6 週）

**目標**：手工接線或在洞洞板上完成一半鍵盤，透過 USB 連線驗證所有按鍵。

- 設計 4×5 或 3×5+拇指鍵的矩陣佈局（建議先從 20 鍵的純方格開始）
- 使用 1N4148（穿孔）二極體，COL2ROW 方向（陰極帶朝向 ROW 線）
- 建立完整的 ZMK shield 定義：`Kconfig.shield`、`Kconfig.defconfig`、`.dtsi`（共用 devicetree）、`_left.overlay`、`.keymap`
- 測試所有按鍵、設計 layer（數字層、符號層、導航層）
- **學習資源**：golem.hu/guide/keyboard-matrix/、ZMK New Shield Guide

### 第三階段：軌跡球整合（第 7–9 週）

**目標**：在右半整合 PAW3222 軌跡球模組，驗證游標移動與捲動功能。

- 取得 PAW3222 breakout 模組（Sekigon 的 14mm 模組最理想，或自行打板）+ 25mm 軌跡球
- 接線 SPI：SCK、SDIO（MOSI/MISO 共用）、CS、IRQ → XIAO 的 GPIO
- 在 `west.yml` 加入 `sekigon-gonnoc/zmk-driver-paw3222` 模組
- 配置 devicetree overlay 的 SPI 與感測器節點
- 啟用 `CONFIG_ZMK_POINTING=y` 和 `CONFIG_PAW3222=y`
- 調整 CPI 靈敏度、測試 move/scroll 模式切換
- **學習資源**：sekigon-gonnoc/zmk-driver-paw3222 README、torabo-tsuki OM 的 overlay 配置

### 第四階段：無線分離與電池整合（第 10–12 週）

**目標**：兩半無線運作，含電池供電。

- 組裝第二半（左半鏡像，不含軌跡球）
- 焊接 LiPo 電池到兩片 XIAO 的 BAT+/BAT- 焊盤，加裝電源開關
- 配置 ZMK 分離韌體：左半 `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`、右半為 peripheral
- 分別刷入 `_left.uf2` 與 `_right.uf2`
- 測試 BLE 配對（兩半之間 + 與主機）、多裝置 BLE profile
- 配置省電：`CONFIG_ZMK_SLEEP=y`、idle timeout
- **注意**：首次啟用 pointing 功能後，主機端需要刪除配對重新連線（HID descriptor 快取問題）

### 第五階段：PCB 設計與最終組裝（第 13–20 週）

**目標**：設計自訂 PCB、打樣、焊接 SMD 元件、完成成品。

- 學習 KiCad 基礎（或使用 Ergogen v4 參數化佈局產生器），參考 flatfootfox.com 的 Ergogen 教學系列
- 設計可翻轉 PCB（同一片 PCB 用於左右兩半，降低打樣成本）
- 在右半預留 SPI header 給軌跡球模組
- 下單 JLCPCB（5 片約 US$2–7 + 運費 US$5–15）
- 設計 3D 列印外殼，含腰帶扣環
- 焊接 SMD 元件（SOD-123 二極體、hotswap 插座）
- **KiCad 資源**：keyboard-kicad-lib（Eymeric65，含可翻轉 footprint 與影片教學）、ai03 鍵盤 KiCad 元件庫

---

## 五、台灣零件採購完整指南

### MCU：Seeed XIAO nRF52840

| 管道 | 價格 | 備註 |
|------|------|------|
| **iCShop**（icshop.com.tw）| NT$485/片（Plus 版）| 台灣最可靠的來源，滿 NT$1,000 免運 |
| Seeed Studio 官網直購 | ~US$9.90 + 運費 | 中國倉庫發貨，運費 US$5–15 |
| Mouser Taiwan（mouser.tw）| ~NT$350–400 | 全球分銷商，交期穩定 |
| 蝦皮 | NT$400–550 | 搜尋「XIAO nRF52840」，台灣賣家可快速到貨 |
| AliExpress | ~NT$280–350 | 最便宜但需 2–4 週，注意真偽 |

**建議**：軌跡球側購買 Plus 版（20 個 GPIO），非軌跡球側用標準版即可。兩片合計約 **NT$800–970**。

### PAW3222 感測器模組

**這是最難取得的零件。** PAW3222 不像 PMW3360 有廣泛的模組化產品。取得方式：

- **Sekigon 的 14mm breakout 模組**：在 BOOTH（nogikes.booth.pm）上販售，是日本鍵盤社群的標準方案。需從日本寄送（可用 Buyee 代購服務）。此模組已在 torabo-tsuki、cool642tb 等專案中大量驗證。GitHub 開源 PCB 設計：sekigon-gonnoc/small-mouse-sensor-module
- **自行打板**：使用 Sekigon 開源的 breakout PCB 設計檔案，在 JLCPCB 打樣。PAW3222 裸晶片可在 AliExpress/淘寶搜尋取得（~NT$30–80），但需要 SMD 焊接能力與透鏡組件
- **替代方案 PMW3610**：超低功耗感測器，有 `inorichi/zmk-pmw3610-driver` ZMK 驅動，AliExpress 上模組較易取得。但社群驗證程度不及 PAW3222

### PCB 打樣

**JLCPCB**（jlcpcb.com）是首選：5 片雙層 100×100mm 起價 **US$2**，運費到台灣 US$1–15（依速度），最快 3–5 天到達。**重要提醒**：台灣進口需透過「EZ WAY 易利委」APP 完成實名認證，收件人姓名、身分證號與手機需與 APP 註冊一致。

PCBWay 品質略優但價格稍高（~US$5 起）。台灣本地打樣服務（如捷特 Jetpcb、嵌揚 EmbPCB）速度快但成本顯著較高，適合急件。社群維護的 PCB 廠商清單：hackmd.io/@openlabtaipei/rygQ7dcQBI。

### 電池與電源

- **LiPo 電池（301230 / 502030）**：蝦皮搜尋「301230 鋰電池」或 AliExpress 購買，每顆 NT$30–100
- **SPDT 滑動開關（MSK-12C02）**：AliExpress 或 iCShop，每個 NT$3–10
- **TP4056 充電模組**：XIAO 已內建充電，通常不需要。若需備用，iCShop 約 NT$35–50

### 軌跡球球體與軸承

- **25mm 球體**：AliExpress 搜尋「25mm resin ball trackball」，NT$30–100；或 ELECOM M-B25RD 替換球（Amazon.co.jp）
- **34mm 球體**：Perixx PERIPRO-303（Amazon.co.jp 約 ¥990）、蝦皮搜尋「軌跡球 替換球 34mm」
- **陶瓷軸承球（ZrO₂）**：AliExpress 搜尋「zirconia ceramic ball 2.5mm」，搭配 3D 列印的 BTU 座，NT$30–80/包
- 軌跡球外殼可參考 Charybdis 的開源 BTU 座設計

### 電子元件與鍵盤零件

| 零件 | 來源 | 價格 |
|------|------|------|
| 1N4148 二極體（穿孔/SMD） | iCShop / 蝦皮 / AliExpress | NT$15–30/100 個 |
| Kailh Choc v2 hotswap 插座 | **ErgoTaiwan**（ergotaiwan.tw）| ~NT$3–5/個 |
| FPC 連接器 | Mouser TW / AliExpress | NT$5–15/個 |
| 電阻、電容 | 光華商場 / 廣華電子 / iCShop | NT$0.5–2/個 |
| Kailh Choc v2 軸體 | ErgoTaiwan / 蝦皮 / AliExpress | ~NT$3–8/個 |

**台灣鍵盤 DIY 重點店家**：**ErgoTaiwan**（ergotaiwan.tw）專營分離式鍵盤零件，是台灣最相關的供應商。**光華商場**的電子材料行可即時購買電阻、二極體、連接器等基礎零件。

### 建議採購策略

**第一批（中國直購，合併運費）**：JLCPCB PCB 打樣 + AliExpress 的 1N4148、hotswap 插座、LiPo 電池、陶瓷軸承球、軌跡球球體。預估 NT$1,500–2,500，2–4 週到貨。

**第二批（台灣本地，快速取得）**：iCShop 的 XIAO nRF52840（×2）+ ErgoTaiwan 的軸體/插座 + 光華商場的零散元件。預估 NT$1,200–2,000，1–3 天到貨。

**第三批（日本代購）**：Sekigon 的 PAW3222 14mm breakout 模組（BOOTH nogikes 商店），透過 Buyee 或樂天代購。或自行打板取代。

---

## 六、ZMK 技術整合細節與已知限制

### Pointing device 支援現況

**ZMK 主線已完整支援 pointing device**，不再需要任何 fork。2024 年 12 月 PR #2477 合併後，split peripheral 側的 pointing device 也已原生支援。關鍵配置項：

```
CONFIG_ZMK_POINTING=y    # 啟用 pointing 功能
CONFIG_PAW3222=y          # 啟用 PAW3222 驅動
CONFIG_SPI=y              # 啟用 SPI 匯流排
```

在 split peripheral 側使用軌跡球時，需在 devicetree 中配置 `zmk,input-split` 裝置，讓 peripheral 的指標事件透過 BLE GATT 傳送至 central，再由 `zmk,virtual-device` 重新發射並經 `zmk,input-listener` 處理。torabo-tsuki LP 的韌體 repo（sekigon-gonnoc/zmk-keyboard-torabo-tsuki-lp）是最完整的 split + PAW3222 配置範例。

### 已知問題與注意事項

**GPIO 不足**是使用 XIAO 標準版時最常遇到的問題。4 行 × 5 列矩陣需 9 pin，軌跡球 SPI 需 4 pin，加總 13 pin 超過標準版的 11 pin。使用 XIAO Plus 版（20 pin）是最直接的解決方案。

**XIAO BLE 的 mouse emulation 問題**（GitHub issue #2941）曾報告 `&mmv`、`&msc` 等模擬滑鼠按鍵在 XIAO BLE 上不作用（在 nice!nano 上正常）。此 issue 已關閉，推測已修復，但建議使用最新版 ZMK。

**BLE 配對穩定性**：有社群報告約 12.5% 的 XIAO nRF52840 單元在 split 配對上出現問題，可能與品管波動有關。建議多購買 1–2 片備用。首次刷入支援 pointing 的韌體後，主機端**必須刪除舊的藍牙配對並重新連線**，因為 HID descriptor 會被快取。

### 推薦的 GitHub Repo 結構

```
my-keyboard/
├── README.md
├── LICENSE.md                    # 硬體：CERN-OHL-P-2.0；韌體：MIT
├── .github/workflows/build.yml  # GitHub Actions 自動建置
├── pcb/                          # KiCad 設計檔案
│   ├── keyboard.kicad_pro / .kicad_sch / .kicad_pcb
│   ├── gerbers/                  # 生產用 Gerber 檔
│   └── bom.csv
├── firmware/                     # ZMK 配置
│   ├── config/boards/shields/my_keyboard/
│   │   ├── Kconfig.shield / Kconfig.defconfig
│   │   ├── my_keyboard.dtsi      # 共用 devicetree
│   │   ├── my_keyboard_left.overlay / _right.overlay
│   │   ├── my_keyboard.keymap
│   │   └── my_keyboard.zmk.yml
│   ├── build.yaml / west.yml
├── case/                         # 3D 列印外殼 (STL + 原始檔)
│   └── belt_mount.stl            # 腰帶扣環設計
└── docs/
    ├── build_guide.md
    └── images/
```

---

## 七、腰掛使用場景的特殊設計考量

將鍵盤掛在腰部兩側供育兒時使用，是一個非常獨特的需求，需要特別注意幾個面向。

**全無線是必要的**——BLE 分離式不需要兩半之間的線纜，消除了抱小孩時被線纜勾住的風險。ZMK 的 BLE split 正好符合此需求。

**重量控制**方面，每半的目標是 **60g 以下**（含電池）。XIAO nRF52840 約 3g，PCB + 軸體 + 鍵帽約 30–40g，110mAh LiPo 約 3g，3D 列印外殼約 10–15g，總計 50–60g 完全可行。低矮軸（Choc v1/v2）比 MX 軸更適合——整體厚度更薄，掛在腰側更不突兀。

**固定機制**建議使用 3D 列印的腰帶夾（belt clip）或 MOLLE 織帶環。考慮 **Fidlock 磁扣**實現快拆——需要抱小孩時一秒拆下。外殼設計需避免尖角，使用圓角以免刮傷衣物或小孩。

**鍵盤鎖定功能**至關重要：實作一個「鎖定 layer」（所有鍵 = `&none`），透過特定組合鍵觸發，防止非打字時的誤觸。ZMK 的 combo 功能可以設定雙鍵同按啟動鎖定。

**單手模式**也值得考慮：設計一個 layer 允許單手基本打字（類似 Miryoku 的單手模式），在需要一手抱小孩時仍能簡單操作。

---

## 八、入門者焊接與電路學習路徑

### 必備工具

| 工具 | 建議品項 | 預算 |
|------|---------|------|
| 烙鐵 | **Pinecil**（社群最愛）或 TS101 攜帶型 | NT$700–2,000 |
| 焊錫 | 63/37 有鉛含松香芯 0.8mm（Kester 品牌佳） | NT$200–300 |
| 助焊劑 | 松香助焊劑筆或膏 | NT$100–200 |
| 吸錫器 | 手動吸錫泵 + 銅編織帶（吸錫線） | NT$100–200 |
| 三用電表 | 基本數位電表（需有導通蜂鳴功能） | NT$300–600 |
| 輔助夾具 | 第三隻手（helping hands）| NT$200–400 |
| 工作墊 | 矽膠耐熱墊 | NT$200–300 |

### 關鍵電路概念

了解以下概念即可開始，不需要完整的電子學背景：

**鍵盤矩陣**：將 N 個鍵排列成 R×C 矩陣，只需 R+C 個 GPIO 即可掃描所有按鍵。MCU 依序對每列施加電壓，讀取每行是否有電流——如果有，代表該列該行交叉處的按鍵被按下。**二極體的作用**是防止 ghosting（三鍵以上同時按壓時電流逆流產生的假訊號），確保電流只能單向流動。所有二極體方向必須一致（COL2ROW：陰極帶朝向行線）。

**SPI 協定**：PAW3222 透過 SPI 與 MCU 通訊。SPI 是 4 線同步串列協定——SCK（時鐘）、MOSI（主出從入）、MISO（主入從出）、CS（晶片選擇，低電位啟用）。PAW3222 的特殊之處是 MOSI 和 MISO 共用同一根線（SDIO），簡化為 3 線 + CS。

---

## 結論與核心建議

本專案的技術可行性已充分驗證——日本鍵盤社群在 2024–2025 年間已有多個幾乎相同規格的成功案例（torabo-tsuki OM、cool642tb-mini），且核心軟體棧（ZMK pointing device + PAW3222 driver）已進入主線穩定版。

**最關鍵的三個決策點**：第一，**選擇 XIAO nRF52840 Plus**（而非標準版）作為軌跡球側的 MCU，一勞永逸解決 GPIO 不足問題。第二，**放棄重用 NuPhy Air60 軸體**（KS-33 不相容），改用 Kailh Choc v2（可沿用鍵帽），或直接購入新的 Choc v1 軸體搭配低矮鍵帽。第三，**以 torabo-tsuki OM 或 cool642tb 的開源設計為起點修改**，而非從零設計——這將大幅減少 PCB 設計和韌體配置的工作量。

從巨集鍵盤原型到最終成品，預估 **16–20 週**是合理的開發週期。最大的挑戰不在焊接（穿孔焊接門檻不高），而在 ZMK 的 devicetree 配置與 KiCad PCB 設計——這兩項技能各需約 2–3 週的學習曲線，但有充足的社群教學資源與開源範例可參考。