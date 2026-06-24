# Pipelined MIPS-Lite CPU(五級管線化處理器)

> 以 Verilog 從零實作的五級管線化(5-stage pipeline)MIPS-Lite CPU,涵蓋 ALU、桶式移位器、移位相加乘法器與暫存器堆,並以 Icarus Verilog 進行模擬驗證。
> 中原大學(CYCU)資訊工程 — 計算機組織(Computer Organization)課堂專案。

![Language](https://img.shields.io/badge/HDL-Verilog-blue)
![Simulator](https://img.shields.io/badge/sim-Icarus%20Verilog-green)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 問題(這個專案做什麼)

本專案實作一顆 **MIPS-Lite 指令集的管線化 CPU**,目的是把計算機組織課程中的「處理器資料路徑與控制」從紙上設計落實成可模擬、可驗證的硬體描述。

處理器採用經典的 **五級管線(5-stage pipeline)**架構:

```
IF(取指) → ID(解碼/讀暫存器) → EX(執行/ALU) → MEM(記憶體存取) → WB(寫回)
```

各級之間以管線暫存器(`IF_ID` / `ID_EX` / `EX_MEM` / `MEM_WB`)隔開,讓多道指令能同時在不同階段中流動。整體頂層模組為 `final_cpu/mips_Pipeline.v`,並由測試平台 `final_cpu/tb_Pipeline.v` 載入指令/資料/暫存器初始內容後驅動時脈進行模擬。

### 支援的指令

依測試平台與控制單元(`control_single.v`、`alu_ctl.v`、`tb_Pipeline.v`)的解碼邏輯,可驗證支援下列指令:

| 類型 | 指令 |
| --- | --- |
| R-type 算術/邏輯 | `ADD`、`SUB`、`AND`、`OR`、`SLT`、`SLL` |
| R-type 乘法相關 | `MULTU`、`MFHI`、`MFLO` |
| I-type | `ADDIU`、`LW`、`SW`、`BEQ` |
| J-type | `J`、`JAL` |
| 其他 | `NOP`(以 `SLL $0,$0,0` 表示) |

> 原始 README 將其描述為「13 道 MIPS 指令」。實際解碼表中可辨識的指令略多於此(視是否將 `MFHI`/`MFLO`/`NOP` 各別計入而定),本表以程式碼中實際解碼到的指令為準。

---

## 方法與技術(底層 / 數位設計重點)

這是一個**底層數位設計**題材,重點不在高階演算法,而在於用結構化(structural)Verilog 把資料路徑的每個元件親手搭出來:

- **五級管線資料路徑**:`IF_ID` / `ID_EX` / `EX_MEM` / `MEM_WB` 四個管線暫存器在每個 `posedge clk` 把控制訊號與資料逐級往下傳,並支援同步 reset 清空。
- **自製結構化 ALU**(`ALU .v`):以 8 個 4-bit 的 `ALU4` 串接成 32-bit 漣波進位(ripple-carry)運算單元,支援 `AND / OR / ADD / SUB / SLT / SLL`;減法/比較透過 `isinvert` 對 B 運算元做補數處理。
- **桶式移位器(barrel shifter)**(`Shifter.v`):由 `SLL1 / SLL2 / SLL4 / SLL8 / SLL16` 五級級聯,依 shamt 各位元決定是否位移,實作 0~31 位元的邏輯左移。
- **移位相加乘法器**(`MULTU.v` + `HiLo.v`):32 次「判斷 + 位移相加」迭代完成 32×32 → 64-bit 無號乘法,結果存入 `HiLo` 暫存器,再由 `MFHI` / `MFLO` 透過 `mux3` 選擇讀回高/低 32 位。
- **暫存器堆與記憶體**:`reg_file.v`(32 個暫存器)、`memory.v`(1KB、byte-addressable、Little-Endian;同時作為指令記憶體與資料記憶體實例化)。
- **控制單元**:`control_single.v` 依 opcode 產生主控制訊號(RegDst、ALUSrc、MemtoReg、Branch、Jump、R31toReg、JaltoReg…),`alu_ctl.v` 依 ALUOp 與 funct 再細分出 ALU 實際運算與乘法/HiLo 選擇訊號。
- **PC 與分支/跳躍邏輯**:`reg32.v`(PC 暫存器)、`add32.v`(PC+4 / 分支位址加法)、`sign extend.v`(立即值符號擴展),搭配多組 `mux2` / `mux3` 完成分支、跳躍、JAL 寫回 `$31` 等位址選擇。

> 設計風格刻意以「小元件 → 組裝成大元件」的結構化描述為主(例如全加器 `FA.v`、`add32.v`、`ALU4` 等),貼近數位邏輯課程中由閘級往上堆疊的訓練方式。

---

## 結果

- 完成一顆可在 Icarus Verilog 上模擬執行的五級管線 MIPS-Lite CPU。
- 隨附一段測試程式(`instr_mem.txt`),涵蓋 `LW / BEQ / ADD / SUB / OR / J / SW` 等指令的混合流程,並提供 `data_mem.txt`、`reg.txt` 作為記憶體與暫存器初始值。
- 模擬時測試平台會逐 cycle 印出 PC 與目前解碼到的指令名稱,並把波形輸出成 `mips_Pipeline.vcd` 供 GTKWave 等工具觀察管線各級訊號。

### 範例輸出(節錄,實際數值依模擬環境而定)

執行模擬後,終端機會出現類似下列逐 cycle 的訊息(指令名稱由測試平台解碼印出):

```
          0, PC: ...
          0, LW
          1, PC: ...
          1, BEQ
          ...
          n, writing data: Mem[  24] <=  ...
         16, End of Simulation
```

> 上方為輸出格式示意;確切的 PC 值、記憶體寫入值與 cycle 數請以你本機 `vvp` 的實際輸出為準。

---

## 如何 build 與執行

### 需求

- [Icarus Verilog](http://iverilog.icarus.com/)(提供 `iverilog` 編譯器與 `vvp` 模擬器)
- (選用)[GTKWave](https://gtkwave.sourceforge.net/) 用於檢視 `mips_Pipeline.vcd` 波形

### 步驟

模擬會以 `$readmemh` 讀取 `instr_mem.txt`、`data_mem.txt`、`reg.txt`,因此**請在 `final_cpu` 目錄下執行**,確保這些初始化檔案能被找到。

> 注意:部分原始碼檔名含有空白(例如 `ALU .v`、`sign extend.v`),在指令列中務必用引號包住,或直接讓 iverilog 編譯整個資料夾。

最簡單的做法是編譯資料夾內所有 `.v` 檔:

```bash
cd final_cpu

# 編譯所有 Verilog 原始碼,頂層測試平台為 tb_Pipeline
iverilog -o mips_Pipeline.vvp *.v

# 執行模擬
vvp mips_Pipeline.vvp
```

若想明確指定頂層模組,可加上 `-s tb_Pipeline`:

```bash
iverilog -s tb_Pipeline -o mips_Pipeline.vvp *.v
vvp mips_Pipeline.vvp
```

### 檢視波形(選用)

```bash
gtkwave mips_Pipeline.vcd
```

### 修改測試程式

- `instr_mem.txt`:指令記憶體,每行 1 byte、以十六進位填寫,採 **Little-Endian** 排列。
- `data_mem.txt`:資料記憶體初始值(同樣每行 1 byte、十六進位)。
- `reg.txt`:32 個暫存器的初始值(每行一個暫存器值)。
- 模擬週期數可在 `tb_Pipeline.v` 的 `parameter cycle_count` 調整。

---

## 已知限制

- **無硬體 hazard 處理**:目前的管線**沒有**實作 forwarding(資料前遞)、hazard detection 或 stall/flush 邏輯。資料/控制相依需靠在測試程式中**手動插入 `NOP`** 來避免,程式設計時必須自行排開危障。
- **分支/跳躍在本實作中綁在一起**:`J` 與 `BEQ` 的控制訊號共用部分路徑(例如 `J`、`JAL` 同時拉高 `Branch` 與 `Jump`),分支判斷使用組合邏輯比較且未做延遲槽/沖刷處理,撰寫測試程式時需留意控制危障。
- **乘法器為固定 32 次迭代展開**:`MULTU.v` 把移位相加展開成 32 段固定程式碼,屬無號乘法;有號乘法、除法、溢位旗標等均未實作。
- **記憶體容量有限**:資料/指令記憶體為 1KB(`mem_array[0:1023]`),且採位元組定址,大型程式需自行擴充。
- **檔名含空白字元**:`ALU .v`、`sign extend.v` 等檔名帶有空白,在某些工具鏈或腳本中需特別處理。
- **部分檔頭註解沿用「Single Cycle」字樣**:`tb_Pipeline.v` / `control_single.v` 等檔頭註解仍寫著 single-cycle,但實際資料路徑為管線化版本,閱讀時以程式碼為準。
- **授權條款未指定**:repo 中未包含 LICENSE 檔。

---

## 專案結構

```
Pipelined-CPU-Design/
└── final_cpu/
    ├── mips_Pipeline.v      # 頂層:五級管線資料路徑
    ├── tb_Pipeline.v        # 測試平台(頂層 testbench)
    ├── IF_ID.v / ID_EX.v / EX_MEM.v / MEM_WB.v   # 管線暫存器
    ├── control_single.v     # 主控制單元(依 opcode)
    ├── alu_ctl.v            # ALU 控制(依 ALUOp/funct)
    ├── ALU .v / ALU4.v / ALU1.v / FA.v / add32.v # ALU 與加法器元件
    ├── Shifter.v / SLL1.v / SLL2.v / SLL4.v / SLL8.v / SLL16.v # 桶式移位器
    ├── MULTU.v / HiLo.v     # 無號乘法器與 Hi/Lo 暫存器
    ├── reg_file.v / reg32.v # 暫存器堆 / 32-bit 暫存器(含 PC)
    ├── memory.v             # 1KB byte-addressable 記憶體(指令/資料)
    ├── mux2.v / mux3.v      # 多工器
    ├── sign extend.v        # 符號擴展
    ├── instr_mem.txt        # 指令記憶體初始內容(Little-Endian, hex)
    ├── data_mem.txt         # 資料記憶體初始內容
    └── reg.txt              # 暫存器初始值
```

---

## 架構圖

> TODO:架構圖連結(原始 README 引用之 GitHub 使用者上傳圖片連結請確認後填入)。
> 詳細設計說明可參考 repo 內的 `期末計組.pptx` 與 `計算機組織 文字報告.docx`。

---

## 作者與分工

中原大學計算機組織課堂分組專案,分工概述(取自原始 README):

- 撰寫 Multiplier、繪製架構圖、協助 Shifter 與波形圖。
- 陳柏宇:撰寫 ALU、Shifter、心得與重點說明、繪圖。
- 吳添聖:撰寫 Shifter、ALU control、繪圖。
- 廖彥蓉:修正報告格式。
