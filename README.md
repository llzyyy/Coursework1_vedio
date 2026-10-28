# F29AI CW1 学习参考实现

数独和五个 PDDL 文件均已实现并在本机运行验证。桌面上的原始模板没有修改。

课程 PDF 第 12 页规定：GenAI 可用于讨论思路和理解概念，不能生成提交用的报告、Python/Java 或 PDDL。本目录是 AI 生成的学习参考实现，不能据此声明为你们独立编写的作业。它也不是完整提交包：没有代填小组信息，没有最终报告 PDF，没有两人演示视频。

## 文件结构

```text
F29AI_CW1_reference/
  sudoku/
    solver.py                 手写回溯 + MRV 求解器
    gui.py                    可选 Tkinter 图形界面
    benchmark.py              重复测量并输出 CSV
    test_solver.py            数独行为测试
    puzzles/                  TXT、CSV、有解、无解测试用例
  PDDL/
    PartA&B/
      domain.pddl             lunar 基础域
      mission1.pddl           lunar-mission-1
      mission2.pddl           lunar-mission-2
    PartC/
      domain-ext.pddl         lunar-extended 扩展域
      mission3.pddl           lunar-mission-3
  tools/
    run_missions.py           Pyperplan 运行器
    pddl_replay.py            原始规则的独立逐步回放检查器
    test_planning.py          PDDL 语义回归测试
  results/                    实测数据、计划、完整状态轨迹
  学习说明.md                  建模和理论分析讲解
  requirements-planning.txt   仅规划工具需要的第三方依赖
```

## 1. 运行数独

Python **3.9 或更高版本**。数独求解器、基准测试不需要第三方库。

先在 PowerShell 进入本目录：

```powershell
Set-Location -LiteralPath 'C:\Users\LENOVO\Documents\ChatGPT\自动化大学习\output\F29AI_CW1_reference'
```

命令行求解：

```powershell
python sudoku/solver.py sudoku/puzzles/example_a.txt --stats
python sudoku/solver.py sudoku/puzzles/example_b.csv --stats
python sudoku/solver.py sudoku/puzzles/sparse.txt --stats
python sudoku/solver.py sudoku/puzzles/unsatisfiable.txt --stats
```

输入文件必须恰好有 9 行、每行 9 个数字。TXT 用空白分隔，CSV 用逗号分隔。0 表示空格，支持 UTF-8 和 UTF-8 BOM。输出清晰的 3×3 分组棋盘；退出码 0 表示找到解，1 表示合法输入无解，2 表示文件或输入错误。

图形界面：

```powershell
python sudoku/gui.py
```

可以载入题目、手动输入、求解、恢复原始提示数和清空。求解在线程中运行，结果通过队列回到界面。Tkinter 通常随桌面版 Python 附带；如果当前发行版不含 Tk，需要安装相应 Tk 支持。

测量和测试：

```powershell
python sudoku/benchmark.py --repeats 25
python -m unittest discover -s sudoku -p test_solver.py -v
python sudoku/gui.py --smoke-test
```

`results/sudoku_benchmark.csv` 是本机真实测量，不是预设数据。不同电脑和重复测量会有时间差。文件读写和打印不计入求解时间，验证、棋盘复制和候选掩码初始化计入时间。先做一次不计入结果的预热，再重复测量。

这些小型测试用例用于功能和回溯演示，没有经过正式难度评级，也没有用来证明唯一解。请为自己的报告选择并注明来源明确的测试题，记录自己的运行结果。

## 2. 运行 PDDL

只在使用本地规划器和验证脚本时需要 Pyperplan。数独不使用它。

建议将依赖安装在本目录的局部文件夹，避免修改全局环境：

```powershell
python -m pip install --target .planner-deps -r requirements-planning.txt
python tools/run_missions.py
python -m unittest discover -s tools -p test_planning.py -v
```

运行器自动识别 `.planner-deps`。也可以通过 `CW1_PLANNER_DEPS` 指向已有的依赖目录；否则使用当前 Python 环境中的安装。

本次验证使用 Pyperplan 2.1、greedy best-first search、hFF 启发式。规划器可能因版本或集合遍历顺序产生不同的合法计划；本作业没有要求唯一计划，也没有要求最少步数。

运行器会重新产生：

- `mission1.plan`、`mission2.plan`、`mission3.plan`：动作序列。
- `mission1_trace.json` 等：初始状态和每一步后的完整事实集合。
- `planning_summary.json`：规划器版本、耗时、步数和验证状态。

也可以将对应的域文件和任务文件导入课程允许的在线 PDDL 编辑器。始终配对：

| 任务 | 域文件 | 问题文件 |
|---|---|---|
| 1 | `PDDL/PartA&B/domain.pddl` | `PDDL/PartA&B/mission1.pddl` |
| 2 | `PDDL/PartA&B/domain.pddl` | `PDDL/PartA&B/mission2.pddl` |
| 3 | `PDDL/PartC/domain-ext.pddl` | `PDDL/PartC/mission3.pddl` |

`PartA&B` 含 PowerShell 特殊字符 `&`，在终端中直接引用该路径时请用引号包住。

## 3. 建模选择

- Mission 1：选择着陆器 1 在 WP1 着陆。
- Mission 2、3：着陆器 1 按要求位于 WP2；选择着陆器 2 在 WP1 着陆。
- 着陆点固定在初始状态中，不设置着陆器移动动作。
- 未部署的车只有 `aboard`，没有任何 `rover-at`。
- 图片和扫描共同占用一个 `memory-free` 数据槽。
- 样本槽由 `sample-free` 单独控制，可与一份数据同时携带。
- 着陆器存储由 `lander-empty` 控制，存放后不会再次变空。
- 数据无线发送到自己的着陆器；样本必须实际运回自己的着陆器。
- 所有获取和保存事实记录来源地点；目标检查最终送达/存放状态。
- Mission 3 的部署、存放前提要求本着陆器宇航员在对接舱；发送要求在控制室。

地图边直接按 PDF 第 6、7 页图示录入，未增加反向连接：

```text
Mission 1:
1->2, 2->3, 3->5, 5->1, 1->4, 4->3

Mission 2 and 3:
1->2, 2->1, 2->3, 2->4, 4->2,
3->5, 5->3, 5->6, 6->4
```

## 4. 已完成的验证及其限度

- 8 项数独测试：合法解、保持提示数、输入不变、稀疏题回溯、无解、完成棋盘、空棋盘、TXT/CSV/BOM、非法输入、命令行退出码等。
- 图形界面 smoke test：创建窗口控件、载入棋盘、经工作线程求解并将结果显示回网格。
- 三个任务均由 Pyperplan 实际找到计划，再依据原始 PDDL 逐步回放，检查前提、效果、目标和位置/容量不变量。
- 17 项规划测试：有向地图、初始状态、目标、内存、样本、归属、无线发送、物理返回、宇航员条件，以及使用相同基础域解决不同名称和三个着陆器的临时实例。

独立回放器只实现这份代码使用的正前提 STRIPS 子集，不是通用 PDDL 验证器，也没有声称运行 VAL、FF、Fast Downward 或所有允许的规划器。回归测试不能穷尽全部合法实例。

## 资料

- 作业规则：本地提供的 `F29AI_2026_cw1.pdf`，尤其第 2、4–10、12 页。
- Pyperplan 官方项目和运行说明：<https://github.com/aibasel/pyperplan>。
- 课程允许的在线编辑器：<https://editor.planning.domains/>。
