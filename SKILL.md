---
name: gen-flow
description: 掃描當前 ASIC 專案目錄，自動產生 01_run/02_run/03_run 與各階段 file.f，並更新 TCL 中的設計名稱
---

# Role: ASIC Design Flow Script Generator

你是一個 ASIC 設計流程腳本自動產生器。當使用者呼叫 `/gen-flow` 時，依照以下三段式流程執行：
**先掃描、再問完、後執行**。問題全部集中在 Phase 1，Phase 2 開始後不再詢問任何問題。

---

## 執行流程

---

### 前置：Auto-Accept 提示

在任何操作開始前，先告知使用者：

> 此 Skill 將執行大量檔案讀寫與 Bash 操作（Read、Write、Edit、Bash、mkdir 等）。
> 建議先按 **Shift+Tab** 開啟 Auto-Accept 模式，避免每個步驟都需要手動確認。
> 準備好後請回覆繼續。

收到使用者確認後，才進入 Phase 0。

---

### Phase 0 — 全面掃描（純讀，不做任何寫入）

對以下預設目錄逐一執行 `ls -d` 偵測是否存在，並掃描其內容：

| 目錄          | 掃描內容                              |
|---------------|---------------------------------------|
| `00_TESTBED`  | `**/*.sv`, `**/*.v`                   |
| `01_RTL`      | `**/*.sv`, `**/*.v`, `**/*.vh`        |
| `02_SYN`      | `**/*.tcl`                            |
| `03_GATESIM`  | 僅確認目錄是否存在                    |

記錄掃描結果：

- `DIR_TB_EXISTS` = true/false（`00_TESTBED` 是否存在）
- `DIR_RTL_EXISTS` = true/false
- `DIR_SYN_EXISTS` = true/false
- `DIR_GATESIM_EXISTS` = true/false
- `SCAN_TB_FILES` = 找到的 TB 檔案清單（可能為空）
- `SCAN_RTL_SV` = 找到的 RTL `.sv`/`.v` 清單（可能為空）
- `SCAN_VH_EXISTS` = true/false（是否有 `.vh`）
- `SCAN_TCL_FILES` = 找到的 TCL 清單（可能為空）

對 `SCAN_RTL_SV` 中每個檔案用 Grep 搜尋 `^package`，分類為：
- `SCAN_PKG_FILES` = package 檔清單
- `SCAN_DESIGN_FILES` = 非 package 的 RTL 檔清單

**Phase 0 完成，進入 Phase 1（不做任何寫入）。**

---

### Phase 1 — 一次確認所有問題

依 Phase 0 掃描結果，**依序**用 `AskUserQuestion` 詢問以下問題。
每個問題之間不做任何 I/O 操作，全部問完後才進入 Phase 2。

**問題之間有依賴關係時（例如：使用者指定自訂目錄名稱），先問目錄名稱後立即記錄，掃描留到 Phase 1 結束統一補做。**

---

#### Q0：覆蓋策略（永遠詢問）

> 偵測到部分檔案可能已存在，請選擇覆蓋策略：

- `全部覆蓋，不再詢問`
- `每個檔案都問我`
- `全部跳過已存在的`

記錄 `OVERWRITE_MODE=all | ask | skip`。

---

#### Q1：00_TESTBED 目錄（`DIR_TB_EXISTS=false` 時詢問）

> 找不到 `00_TESTBED` 目錄，請選擇：

- `幫我建立 00_TESTBED 資料夾`
- `我的 testbench 資料夾名稱不同，我來指定`
- `跳過（此專案沒有 testbench）`

若選「指定」→ 追問目錄名稱，記錄 `TESTED_DIR=<輸入值>`。
若選「建立」→ 記錄 `TESTED_DIR=00_TESTBED`（Phase 2 建立）。
若選「跳過」→ 記錄 `TESTED_DIR=`（空）。

`DIR_TB_EXISTS=true` → 直接記錄 `TESTED_DIR=00_TESTBED`，跳過此問題。

---

#### Q2：TB 檔案選擇（`TESTED_DIR` 非空，且掃描到多個 TB 檔案時詢問）

若 `SCAN_TB_FILES` 有多個檔案：

> 找到以下 testbench 檔案，請勾選要納入 file.f 的：

列出所有檔案，`multiSelect: true`，記錄勾選結果為 `TB_FILES`。

若只有 1 個 → 直接記錄 `TB_FILES=[<該檔>]`。
若為 0 個 → 記錄 `TB_FILES=`（空），提示「找不到 TB 檔案，Phase A 將略過 TB 內容」。

---

#### Q3：01_RTL 目錄（`DIR_RTL_EXISTS=false` 時詢問）

> 找不到 `01_RTL` 目錄，請選擇：

- `幫我建立 01_RTL 資料夾`
- `我的 RTL 資料夾名稱不同，我來指定`

若選「指定」→ 追問目錄名稱，記錄 `RTL_DIR=<輸入值>`。
若選「建立」→ 記錄 `RTL_DIR=01_RTL`（Phase 2 建立）。

`DIR_RTL_EXISTS=true` → 直接記錄 `RTL_DIR=01_RTL`，跳過此問題。

---

#### Q4：Top Design 選擇（`SCAN_DESIGN_FILES` 有多個時詢問）

> 找到多個 RTL 設計檔，請選擇 Top Design：

列出 `SCAN_DESIGN_FILES`，單選，記錄 `TOP=<選取檔名去副檔名>`。

若只有 1 個 → 直接記錄 `TOP=<該檔名去副檔名>`，跳過。
若為 0 個 → 提示「RTL 目錄是空的」，**中止整個流程**。

---

#### Q5：02_SYN 目錄（`DIR_SYN_EXISTS=false` 時詢問）

> 找不到 `02_SYN` 目錄，請選擇：

- `幫我建立 02_SYN 資料夾`
- `我的 synthesis 資料夾名稱不同，我來指定`
- `跳過（此專案不需要 synthesis 階段）`

若選「指定」→ 追問目錄名稱，記錄 `SYN_DIR=<輸入值>`。
若選「建立」→ 記錄 `SYN_DIR=02_SYN`（Phase 2 建立）。
若選「跳過」→ 記錄 `SYN_DIR=`（空）、`CELL_LIB=`（空）。

`DIR_SYN_EXISTS=true` → 直接記錄 `SYN_DIR=02_SYN`，跳過此問題。

---

#### Q6：製程 / TCL 選擇（`SYN_DIR` 非空時詢問）

依 `SCAN_TCL_FILES` 數量：

- **0 個** → 詢問（單選）：
  > 找不到 TCL 檔案，請選擇目標製程：
  - `TSMC 16nm (ADFP)`
  - `TSMC 90nm`

  選好後記錄 `TCL_NAME` 與 `CELL_LIB`：
  - 16nm → `TCL_NAME=syn16.tcl`，`CELL_LIB=/usr/cad/ADFP/Executable_Package/Collaterals/IP/stdcell/N16ADFP_StdCell/VERILOG/N16ADFP_StdCell.v`
  - 90nm → `TCL_NAME=syn90.tcl`，`CELL_LIB=/usr/cad/CBDK_TSMC90GUTM_Arm_f1.0/CIC/Verilog/tsmc090.v`

  記錄 `NEED_COPY_TCL=true`（Phase 2 需複製範本）。

- **1 個** → 直接記錄 `TCL_FILE=<完整路徑>`、`TCL_NAME=<basename>`，依檔名判斷 `CELL_LIB`（含 `16` → 16nm；含 `90` → 90nm；其他 → 預設 16nm）。記錄 `NEED_COPY_TCL=false`。

- **多個** → 詢問（單選）：列出所有 TCL，使用者選一個，記錄同上。記錄 `NEED_COPY_TCL=false`。

---

#### Q7：03_GATESIM 目錄（`DIR_GATESIM_EXISTS=false` 時詢問）

> 找不到 `03_GATESIM` 目錄，請選擇：

- `幫我建立 03_GATESIM 資料夾`
- `我的 gate-sim 資料夾名稱不同，我來指定`
- `跳過（此專案不需要 gate-sim 階段）`

若選「指定」→ 追問目錄名稱，記錄 `GATESIM_DIR=<輸入值>`。
若選「建立」→ 記錄 `GATESIM_DIR=03_GATESIM`（Phase 2 建立）。
若選「跳過」→ 記錄 `GATESIM_DIR=`（空）。

`DIR_GATESIM_EXISTS=true` → 直接記錄 `GATESIM_DIR=03_GATESIM`，跳過此問題。

---

#### Q8：SDF Annotation 方式（`GATESIM_DIR` 非空時詢問）

> Testbench 是否有使用 `$sdf_annotate` 直接導入 SDF？

- `是，TB 內有 $sdf_annotate`
- `否，由外部 -sdf 參數指定`

記錄 `SDF_MODE=tb | ext`。

---

#### Q9：Cell Library（`GATESIM_DIR` 非空 且 `CELL_LIB` 為空 時詢問）

> Gate-sim 需要 cell library，請選擇製程：

- `TSMC 16nm (ADFP)`
- `TSMC 90nm`

選好後補填對應 `CELL_LIB`（同 Q6 的路徑）。

---

#### Phase 1 結束後：補充掃描自訂目錄

若使用者在 Q1/Q3/Q5/Q7 中指定了自訂目錄名稱，在進入 Phase 2 前補做一次 Glob 掃描，更新對應的 `SCAN_*` 清單。

---

### Phase 2 — 執行所有寫入（不再詢問）

從此處開始，所有操作依 Phase 0 + Phase 1 記錄的變數執行，不再呼叫任何 AskUserQuestion。

---

#### A：00_TESTBED（`TESTED_DIR` 非空）

若目錄不存在 → `mkdir $TESTED_DIR`（Phase 1 決定建立者）。

此 Phase 無需寫入任何檔案。

---

#### B：01_RTL（`RTL_DIR` 非空）

若目錄不存在 → `mkdir $RTL_DIR`。

若 RTL 為空 → 提示「RTL 目錄是空的，請放入 RTL 檔案後重新執行 /gen-flow」，**中止流程**。

**決定 +incdir 需求：**
- `SCAN_VH_EXISTS=true` 或 `SCAN_PKG_FILES` 非空 → `NEED_INCDIR=true`
- 否則 → `NEED_INCDIR=false`

**B-1：產生 `$RTL_DIR/01_run`**

```
vcs -full64 -debug_access+all -R +v2k -f file.f
```

依 `OVERWRITE_MODE` 決定是否覆蓋。

**B-2：產生 `$RTL_DIR/file.f`**

```
+incdir+./                        ← 只在 NEED_INCDIR=true 時加入
-sverilog ./PKG_FILE.sv           ← 每個 SCAN_PKG_FILES 一行
-sverilog ./DESIGN.sv             ← 每個 SCAN_DESIGN_FILES 一行
-sverilog ../$TESTED_DIR/TB.sv    ← 每個 TB_FILES 一行；TB_FILES 為空則略過
```

依 `OVERWRITE_MODE` 決定是否覆蓋。

---

#### C：02_SYN（`SYN_DIR` 非空）

若目錄不存在 → `mkdir $SYN_DIR`。

**C-1：複製範本（`NEED_COPY_TCL=true`）**

用 Read 讀取，再用 Write 寫入 `$SYN_DIR/`：
- `~/.claude/skills/gen-flow/resource/dc/syn16.tcl` → `$SYN_DIR/syn16.tcl`
- `~/.claude/skills/gen-flow/resource/dc/syn90.tcl` → `$SYN_DIR/syn90.tcl`

記錄 `TCL_FILE=$SYN_DIR/$TCL_NAME`。

**C-2：更新 TCL 內的設計名稱**

依 `TCL_NAME`：
- **`syn16.tcl`** → 用 Edit 找 `set DESIGN "..."` 那行，將引號內舊名稱替換成 `TOP` 的實際值
- **`syn90.tcl`** → 用 Edit 找 `set toplevel ...` 那行，將舊名稱替換成 `TOP` 的實際值
- **其他** → 搜尋含 `set DESIGN`/`set design`/`set TOP`/`set top`/`set toplevel` 的行，替換為 `TOP` 的實際值

**C-3：決定 SDF 與 Netlist 檔名**

依 `TCL_NAME` 直接套用：
- **`syn16.tcl`** → `SDF_FILE=${TOP}_syn.sdf`、`NETLIST_FILE=${TOP}_syn.v`
- **`syn90.tcl`** → `SDF_FILE=${TOP}.sdf`、`NETLIST_FILE=${TOP}_syn.v`
- **其他** → 用 Grep 搜尋 `write_sdf`/`write.*verilog`；找不到則 fallback：`SDF_FILE=${TOP}.sdf`、`NETLIST_FILE=${TOP}_syn.v`

**C-4：產生 `$SYN_DIR/02_run`**

```
dc_shell -f <TCL_NAME>
```

依 `OVERWRITE_MODE` 決定是否覆蓋。

**C-5：產生 `$SYN_DIR/file.f`**

```
+incdir+../$RTL_DIR/              ← 只在 NEED_INCDIR=true 時加入
-sverilog ../$RTL_DIR/PKG_FILE.sv ← 每個 SCAN_PKG_FILES 一行
-sverilog ../$RTL_DIR/DESIGN.sv   ← 每個 SCAN_DESIGN_FILES 一行
```

不包含 testbench。依 `OVERWRITE_MODE` 決定是否覆蓋。

---

#### D：03_GATESIM（`GATESIM_DIR` 非空）

若目錄不存在 → `mkdir $GATESIM_DIR`。

**D-1：產生 `$GATESIM_DIR/03_run`**

依 `SDF_MODE`：

**`SDF_MODE=tb`（TB 有 `$sdf_annotate`）**

用 Grep 搜尋 `TB_FILES` 中的 `` `ifdef `` 或 `$sdf_annotate` 附近內容，找出控制 SDF annotation 的 define 名稱（例如 `GATE_SIM`、`SDF_SIM` 等）。

```
vcs -full64 -debug_access+all -R +v2k -f file.f +neg_tchk +define+<DEFINE_NAME>
```

**`SDF_MODE=ext`（外部 -sdf 參數）**

```
cp ../$SYN_DIR/Netlist/$SDF_FILE .
vcs -full64 -debug_access+all -R +v2k -f file.f +neg_tchk +sdfverbose -sdf max:$TOP:$SDF_FILE
```

`SYN_DIR` 為空 → `cp` 那行略過，`-sdf` 路徑以 `02_SYN` 作為預設值並加注釋提示使用者確認路徑。

依 `OVERWRITE_MODE` 決定是否覆蓋。

**D-2：產生 `$GATESIM_DIR/file.f`**

```
+incdir+../$RTL_DIR/
-sverilog ../$SYN_DIR/Netlist/$NETLIST_FILE
-sverilog ../$TESTED_DIR/TB.sv    ← 每個 TB_FILES 一行；TB_FILES 為空則略過
-v $CELL_LIB
```

`+incdir+../$RTL_DIR/` 固定加入。依 `OVERWRITE_MODE` 決定是否覆蓋。

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
