# gen-flow

自動掃描 ASIC 專案目錄，產生各階段所需的 run scripts 與 `file.f`，省去每次手動撰寫的重複工作。

## 安裝

```bash
git clone https://github.com/akira2963753/Gen-Flow-Skill.git ~/.claude/skills/gen-flow
```

安裝完成後即可直接使用，不需要其他設定。

## 使用方式

在專案根目錄下呼叫：

```
/gen-flow
```

## 支援的專案結構

```
project/
├── 00_TESTBED/     ← Testbench
├── 01_RTL/         ← RTL 設計檔
├── 02_SYN/         ← Synthesis
└── 03_GATESIM/     ← Gate-level Simulation
```

資料夾名稱不符合預設命名也沒關係，執行時會詢問是否建立或指定其他名稱。

## 會產生哪些檔案

| 檔案 | 說明 |
|------|------|
| `01_RTL/01_run` | VCS RTL simulation 執行腳本 |
| `01_RTL/file.f` | RTL filelist（含 package、design、testbench） |
| `02_SYN/02_run` | Design Compiler 執行腳本 |
| `02_SYN/file.f` | Synthesis filelist（僅 RTL，不含 testbench） |
| `02_SYN/syn16.tcl` | TSMC 16nm synthesis TCL 範本（含時序約束） |
| `02_SYN/syn90.tcl` | TSMC 90nm synthesis TCL 範本（含時序約束） |
| `03_GATESIM/03_run` | VCS gate-sim 執行腳本（含 SDF back-annotation） |
| `03_GATESIM/file.f` | Gate-sim filelist（含 netlist、testbench、cell lib） |

## 支援製程

| 製程 | Cell Library |
|------|-------------|
| TSMC 16nm (ADFP) | `/usr/cad/ADFP/.../N16ADFP_StdCell.v` |
| TSMC 90nm | `/usr/cad/CBDK_TSMC90GUTM_Arm_f1.0/.../tsmc090.v` |

不論選擇哪種製程，兩份 TCL 範本都會一併複製到 `02_SYN/`，方便日後切換。

## 執行流程

採用「先掃描、再問完、後執行」三段式架構，所有問題集中在一輪確認，問完後全自動執行。

```
Phase 0：掃描所有目錄與檔案（純讀，不寫入）
  │
  ▼
Phase 1：依掃描結果，一次問完所有問題
  │  ├─ Q0：覆蓋策略（全部覆蓋 / 逐一詢問 / 全部跳過）
  │  ├─ Q1：00_TESTBED 目錄（不存在時）
  │  ├─ Q2：TB 檔案選擇（多個時）
  │  ├─ Q3：01_RTL 目錄（不存在時）
  │  ├─ Q4：Top Design 選擇（多個 RTL 時）
  │  ├─ Q5：02_SYN 目錄（不存在時）
  │  ├─ Q6：製程 / TCL 選擇
  │  ├─ Q7：03_GATESIM 目錄（不存在時）
  │  ├─ Q8：SDF Annotation 方式
  │  └─ Q9：Cell Library（SYN 被跳過時）
  │
  ▼
Phase 2：執行所有寫入（不再詢問）
  ├─ A：建立 TESTBED 目錄
  ├─ B：01_RTL → 產生 01_run + file.f
  ├─ C：02_SYN → 複製 TCL 範本、更新設計名稱、產生 02_run + file.f
  └─ D：03_GATESIM → 產生 03_run + file.f
```

## 注意事項

- `01_RTL/` 為必要目錄，若目錄為空會中止流程
- `02_SYN/`、`03_GATESIM/` 可選擇跳過，只產生需要的階段
- TCL 範本來源：`~/.claude/skills/gen-flow/resource/dc/`
- 若要新增製程支援，將對應 TCL 放入 `resource/dc/` 並更新 SKILL.md 的製程對應表
