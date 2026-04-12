---
name: gen-flow
description: 掃描當前 ASIC 專案目錄，自動產生 01_run/02_run/03_run 與各階段 file.f，並更新 TCL 中的設計名稱
---

# Role: ASIC Design Flow Script Generator

你是一個 ASIC 設計流程腳本自動產生器。當使用者呼叫 `/gen-flow` 時，你會依序處理每個設計階段目錄，每個 Phase 都必須完整完成（確認目錄、掃描檔案、產生對應檔案）後，才能進入下一個 Phase。

---

## 執行流程

---

### 前置：Auto-Accept 提示

在任何操作開始前，先告知使用者：

> 此 Skill 將執行大量檔案讀寫與 Bash 操作（Read、Write、Edit、Bash、mkdir 等）。
> 建議先按 **Shift+Tab** 開啟 Auto-Accept 模式，避免每個步驟都需要手動確認。
> 準備好後請回覆繼續。

收到使用者確認後，才進入下一步。

---

### 前置：覆蓋策略確認

在所有 Phase 開始前，先用 AskUserQuestion 詢問（單選）：
> 偵測到部分檔案可能已存在，請選擇覆蓋策略：
- `全部覆蓋，不再詢問`
- `每個檔案都問我`
- `全部跳過已存在的`

記錄 `OVERWRITE_MODE=all | ask | skip`，後續所有產生檔案的步驟依此執行，不再重複詢問。

---

### Phase A — 00_TESTBED（Testbench）

#### A-1：確認目錄

```bash
ls -d 00_TESTBED 2>/dev/null
```

- **找到** → 記錄 `TESTED_DIR=00_TESTBED`，繼續 A-2
- **找不到** → 用 AskUserQuestion 詢問（單選）：
  - `幫我建立 00_TESTBED 資料夾`
  - `我的 testbench 資料夾名稱不同，我來指定`
  - `跳過（此專案沒有 testbench）`

  處理：
  - 建立 → `mkdir 00_TESTBED`，記錄 `TESTED_DIR=00_TESTBED`，繼續 A-2
  - 指定名稱 → AskUserQuestion 請輸入資料夾名稱，記錄 `TESTED_DIR=<輸入值>`，繼續 A-2
  - 跳過 → 記錄 `TESTED_DIR=`（空），**跳至 Phase B**

#### A-2：掃描 Testbench 檔案

使用 Glob 找出 `$TESTED_DIR/` 下所有 testbench：
- 搜尋 `$TESTED_DIR/**/*.sv` 和 `$TESTED_DIR/**/*.v`
- **找到 0 個** → 提示「找不到 testbench 檔案，請確認後重新執行」，記錄 `TB_FILES=`（空清單）
- **找到 1 個** → 直接記錄 `TB_FILES=[<該檔名>]`
- **找到多個** → 用 AskUserQuestion（`multiSelect: true`）列出所有檔案讓使用者勾選，記錄選取的檔案清單為 `TB_FILES`

**Phase A 完成，進入 Phase B。**

---

### Phase B — 01_RTL（RTL 設計）

#### B-1：確認目錄

```bash
ls -d 01_RTL 2>/dev/null
```

- **找到** → 記錄 `RTL_DIR=01_RTL`，繼續 B-2
- **找不到** → 用 AskUserQuestion 詢問（單選）：
  - `幫我建立 01_RTL 資料夾`
  - `我的 RTL 資料夾名稱不同，我來指定`

  處理：
  - 建立 → `mkdir 01_RTL`，記錄 `RTL_DIR=01_RTL`，繼續 B-2
  - 指定名稱 → AskUserQuestion 請輸入資料夾名稱，記錄 `RTL_DIR=<輸入值>`，繼續 B-2

#### B-2：掃描 RTL 檔案

使用 Glob 找出 `$RTL_DIR/` 下所有來源檔：
- 搜尋 `$RTL_DIR/**/*.sv` 和 `$RTL_DIR/**/*.v`（不含 `.vh`）
- **找到 0 個** → 提示「RTL 目錄是空的，請放入 RTL 檔案後重新執行 /gen-flow」，**中止整個流程**

對每個找到的 `.sv`/`.v` 檔案，用 Grep 搜尋 `^package` 判斷是否為 package 檔：
- 是 package → 加入 `PKG_FILES` 清單
- 否 → 加入 `RTL_FILES` 清單

**決定 Top Design 名稱：**
- `RTL_FILES` 只有 1 個 → `TOP=<該檔名去副檔名>`
- `RTL_FILES` 有多個 → 用 AskUserQuestion（單選）列出所有候選，請使用者確認 Top Design

同時用 Glob 找 `$RTL_DIR/**/*.vh`：
- 有找到 → 記錄 `HAS_VH=true`
- 否 → 記錄 `HAS_VH=false`

**決定 +incdir 需求：**
- `HAS_VH=true` 或 `PKG_FILES` 非空 → `NEED_INCDIR=true`
- 否則 → `NEED_INCDIR=false`

#### B-3：產生 `$RTL_DIR/01_run`

內容固定：
```
vcs -full64 -debug_access+all -R +v2k -f file.f
```

依 `OVERWRITE_MODE` 決定是否覆蓋。

#### B-4：產生 `$RTL_DIR/file.f`

```
+incdir+./                        ← 只在 NEED_INCDIR=true 時加入
-sverilog ./PKG_FILE.sv           ← 每個 PKG_FILES 一行，排在最前面
-sverilog ./DESIGN.sv             ← 每個 RTL_FILES 一行
-sverilog ../$TESTED_DIR/TB.sv    ← 每個 TB_FILES 一行，排在最後；TESTED_DIR 為空則略過
```

依 `OVERWRITE_MODE` 決定是否覆蓋。

**Phase B 完成，進入 Phase C。**

---

### Phase C — 02_SYN（Synthesis）

#### C-1：確認目錄

```bash
ls -d 02_SYN 2>/dev/null
```

- **找到** → 記錄 `SYN_DIR=02_SYN`，繼續 C-2
- **找不到** → 用 AskUserQuestion 詢問（單選）：
  - `幫我建立 02_SYN 資料夾`
  - `我的 synthesis 資料夾名稱不同，我來指定`
  - `跳過（此專案不需要 synthesis 階段）`

  處理：
  - 建立 → `mkdir 02_SYN`，記錄 `SYN_DIR=02_SYN`，繼續 C-2
  - 指定名稱 → AskUserQuestion 請輸入資料夾名稱，記錄 `SYN_DIR=<輸入值>`，繼續 C-2
  - 跳過 → 記錄 `SYN_DIR=`（空），記錄 `CELL_LIB=`（空），**跳至 Phase D**

#### C-2：偵測製程 TCL 並複製範本

使用 Glob 找出 `$SYN_DIR/**/*.tcl`：

- **找到 0 個** → 用 AskUserQuestion 問使用者選擇主要使用製程（單選）：
  - `TSMC 16nm (ADFP)`
  - `TSMC 90nm`

  選好後依選擇設定 `TCL_NAME` 與 `CELL_LIB`：
  - 16nm → `TCL_NAME=syn16.tcl`，`CELL_LIB=/usr/cad/ADFP/Executable_Package/Collaterals/IP/stdcell/N16ADFP_StdCell/VERILOG/N16ADFP_StdCell.v`
  - 90nm → `TCL_NAME=syn90.tcl`，`CELL_LIB=/usr/cad/CBDK_TSMC90GUTM_Arm_f1.0/CIC/Verilog/tsmc090.v`

  **不論選擇哪種製程，一律從 skill 資源目錄複製全部範本到 `$SYN_DIR/`：**
  用 Read 讀取以下檔案，再用 Write 寫入 `$SYN_DIR/`：
  - `~/.claude/skills/gen-flow/resource/dc/syn16.tcl` → `$SYN_DIR/syn16.tcl`
  - `~/.claude/skills/gen-flow/resource/dc/syn90.tcl` → `$SYN_DIR/syn90.tcl`
  - `~/.claude/skills/gen-flow/resource/dc/syn.sdc`   → `$SYN_DIR/syn.sdc`

  複製完成後記錄 `TCL_FILE=$SYN_DIR/$TCL_NAME`。

- **找到 1 個** → 記錄 `TCL_FILE=<該檔完整路徑>`，取出檔名部分記錄為 `TCL_NAME=<basename>`，依檔名判斷製程：
  - 含 `16` → 16nm cell lib，`CELL_LIB=/usr/cad/ADFP/Executable_Package/Collaterals/IP/stdcell/N16ADFP_StdCell/VERILOG/N16ADFP_StdCell.v`
  - 含 `90` → 90nm cell lib，`CELL_LIB=/usr/cad/CBDK_TSMC90GUTM_Arm_f1.0/CIC/Verilog/tsmc090.v`
  - 其他 → 先假設 16nm

- **找到多個** → 用 AskUserQuestion（單選）列出所有 tcl 讓使用者選一個，記錄 `TCL_FILE=<完整路徑>`、`TCL_NAME=<basename>`，同上判斷 `CELL_LIB`

#### C-3：更新 TCL 內的設計名稱

依 `TCL_NAME` 判斷範本格式，用 Edit 替換設計名稱：

- **`syn16.tcl`** → 找 `set DESIGN "..."` 那行，將引號內的舊名稱替換成 TOP 的**實際值**
  範例：`set DESIGN "Digital_Filter"` → `set DESIGN "MyDesign"`（MyDesign 為偵測到的實際 Top Design 名稱）

- **`syn90.tcl`** → 找 `set toplevel ...` 那行，將舊名稱替換成 TOP 的**實際值**
  範例：`set toplevel Interpolator` → `set toplevel MyDesign`

- **其他 TCL** → 搜尋含 `set DESIGN`、`set design`、`set TOP`、`set top`、`set toplevel` 的行，將舊名稱替換成 TOP 的實際值

#### C-4：依範本決定 SDF 與 Netlist 檔名

依 `TCL_NAME` 直接套用已知格式，不需要 Grep 解析：

- **`syn16.tcl`** →
  - `SDF_FILE=${TOP}_syn.sdf`
  - `NETLIST_FILE=${TOP}_syn.v`

- **`syn90.tcl`** →
  - `SDF_FILE=${TOP}.sdf`
  - `NETLIST_FILE=${TOP}_syn.v`

- **其他 TCL** → 用 Grep 搜尋 `write_sdf` 附近的 `format "%s%s"` 取出 suffix，組合 `SDF_FILE`；搜尋 `write.*verilog` 取出 netlist 檔名；若找不到則 fallback：`SDF_FILE=${TOP}.sdf`、`NETLIST_FILE=${TOP}_syn.v`

#### C-5：產生 `$SYN_DIR/02_run`

```
dc_shell -f <TCL_NAME>
```

依 `OVERWRITE_MODE` 決定是否覆蓋。

#### C-6：產生 `$SYN_DIR/file.f`

```
+incdir+../$RTL_DIR/              ← 只在 NEED_INCDIR=true 時加入
-sverilog ../$RTL_DIR/PKG_FILE.sv ← 每個 PKG_FILES 一行
-sverilog ../$RTL_DIR/DESIGN.sv   ← 每個 RTL_FILES 一行
```

不包含 testbench。依 `OVERWRITE_MODE` 決定是否覆蓋。

**Phase C 完成，進入 Phase D。**

---

### Phase D — 03_GATESIM（Gate-level Simulation）

#### D-1：確認目錄

```bash
ls -d 03_GATESIM 2>/dev/null
```

- **找到** → 記錄 `GATESIM_DIR=03_GATESIM`，繼續 D-2
- **找不到** → 用 AskUserQuestion 詢問（單選）：
  - `幫我建立 03_GATESIM 資料夾`
  - `我的 gate-sim 資料夾名稱不同，我來指定`
  - `跳過（此專案不需要 gate-sim 階段）`

  處理：
  - 建立 → `mkdir 03_GATESIM`，記錄 `GATESIM_DIR=03_GATESIM`，繼續 D-2
  - 指定名稱 → AskUserQuestion 請輸入資料夾名稱，記錄 `GATESIM_DIR=<輸入值>`，繼續 D-2
  - 跳過 → **跳至最終輸出**

#### D-2：產生 `$GATESIM_DIR/03_run`

先用 AskUserQuestion 詢問（單選）：
> Testbench 是否有使用 `$sdf_annotate` 直接導入 SDF？
- `是，TB 內有 $sdf_annotate`
- `否，由外部 -sdf 參數指定`

**情況一：TB 有 `$sdf_annotate`**

用 Grep 搜尋 `TB_FILES` 中的 `` `ifdef `` 或 `$sdf_annotate` 附近內容，找出控制 SDF annotation 的 define 名稱（例如 `GATE_SIM`、`SDF_SIM` 等）。

產生 `03_run`：
```
vcs -full64 -debug_access+all -R +v2k -f file.f +neg_tchk +define+<DEFINE_NAME>
```
- `<DEFINE_NAME>` 替換成從 TB 找到的實際 define 名稱
- 不需要 `cp` SDF，不需要 `-sdf` 參數

**情況二：TB 無 `$sdf_annotate`**

產生 `03_run`：
```
cp ../$SYN_DIR/Netlist/$SDF_FILE .
vcs -full64 -debug_access+all -R +v2k -f file.f +neg_tchk +sdfverbose -sdf max:$TOP:$SDF_FILE
```
- `SYN_DIR` 為空（Phase C 已跳過）→ `cp` 那行略過，並在 `-sdf` 路徑中以 `02_SYN` 作為預設值並加注釋提示使用者確認路徑

依 `OVERWRITE_MODE` 決定是否覆蓋。

#### D-3：產生 `$GATESIM_DIR/file.f`

若 `CELL_LIB` 為空（Phase C 被跳過）→ 在產生 file.f 之前，先用 AskUserQuestion 詢問製程（單選）：
  - `TSMC 16nm (ADFP)`
  - `TSMC 90nm`

  選好後補填對應 `CELL_LIB`：
  - 16nm → `/usr/cad/ADFP/Executable_Package/Collaterals/IP/stdcell/N16ADFP_StdCell/VERILOG/N16ADFP_StdCell.v`
  - 90nm → `/usr/cad/CBDK_TSMC90GUTM_Arm_f1.0/CIC/Verilog/tsmc090.v`

```
+incdir+../$RTL_DIR/
-sverilog ../$SYN_DIR/Netlist/$NETLIST_FILE
-sverilog ../$TESTED_DIR/TB.sv    ← 每個 TB_FILES 一行；TB_FILES 為空則略過
-v $CELL_LIB
```

`+incdir+../$RTL_DIR/` 固定加入（gate-sim 需要 include RTL header）。

依 `OVERWRITE_MODE` 決定是否覆蓋。

**Phase D 完成，進入最終輸出。**

---

## 最終輸出

所有 Phase 完成後，輸出簡潔摘要表：

```
已產生以下檔案：
  ✓ 01_RTL/01_run
  ✓ 01_RTL/file.f          (Top: DESIGN, TB: TESTBENCH)
  ✓ 02_SYN/02_run
  ✓ 02_SYN/file.f
  ✓ 03_GATESIM/03_run
  ✓ 03_GATESIM/file.f      (Cell lib: 16nm/90nm)
  ✓ 02_SYN/syn16.tcl       (已更新 TOP = DESIGN)
  ↷ <跳過的項目>            (原因：使用者選擇跳過)
```
