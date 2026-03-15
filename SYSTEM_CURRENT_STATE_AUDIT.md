# 系统现状审计

## 文档目的

本文档基于当前仓库源码、配置和测试脚本的完整静态审阅结果，给出 Open Interface 当前系统的真实状态图、六层能力映射、关键瓶颈与工程风险。本文档描述的是“当前实现”，不是目标架构。

## 审阅范围

本次审阅覆盖以下内容：

- `app/` 下全部 Python 源码
- `tests/` 下全部 Python 测试与校验脚本
- `app/resources/context.txt`
- `app/resources/old-context.txt`
- `README.md`
- `app/README.md`
- `build.py`
- `requirements.txt`
- `.github/workflows/pylint.yml`
- `.python-version`
- `.gitignore`

未将图片、媒体素材、`.venv/`、`.git/` 视为系统实现主体。

## 一句话结论

当前系统已经形成了一个可运行的闭环式桌面 Agent MVP：

- 感知、规划、定位、执行、验证、记忆恢复六层均已具备初版实现
- 但感知语义密度、定位精度、验证语义理解、恢复策略显式化程度仍明显不足
- 同时，系统已经不只是 Agent 内核，还叠加了多会话持久化、统一时间线 UI、配置中心热加载等产品层能力

更准确地说，当前系统状态不是“原型前夜”，而是“已闭环、可演示、可迭代，但核心智能能力仍处于第一版”。

## 总体架构图

```text
用户输入
  -> app/ui.py
  -> app/app.py
  -> app/core.py
     -> app/llm.py
        -> app/models/*
        -> app/utils/screen.py
        -> app/resources/context.txt
     -> app/interpreter.py
     -> app/verifier.py
     -> app/agent_memory.py
     -> app/session_store.py
  -> UI 时间线 / 状态反馈 / 会话持久化
```

## 六层结构与真实实现映射

| 层级 | 主要文件 | 当前状态判断 |
| --- | --- | --- |
| 感知层 | `app/utils/screen.py` | 已有截图、anchor 检测、标注图、`screen_state`，但仍是弱语义视觉块方案 |
| 模型规划层 | `app/resources/context.txt` `app/llm.py` `app/models/*` | 已切成单步闭环规划，输入信息比旧版丰富，但高层语义仍不足 |
| 定位层 | `app/interpreter.py` `app/models/model.py` | 已形成 anchor 优先、百分比兜底、像素兼容换算，但精定位能力仍弱 |
| 执行层 | `app/interpreter.py` | 结构最稳定，承担统一参数归一化、坐标映射、动作执行、执行日志写入 |
| 验证层 | `app/verifier.py` | 已有图像差分验证闭环，但仍是视觉变化验证，不具备文字/语义理解 |
| 记忆与恢复层 | `app/agent_memory.py` `app/core.py` | 已能记录近期失败与坏 anchor，但恢复策略主要依赖模型提示，不是显式恢复引擎 |

## 与你原始描述的对比结论

你的六层总结整体正确，但有 4 个需要校正的地方：

1. 模型层拿到的信息已经不只是“截图 + 粗环境”，而是包含会话历史、anchors、`screen_state`、屏幕尺寸、agent memory。
2. 恢复层不是完整策略系统，而是“轻量记忆 + prompt 驱动恢复”。
3. 当前系统已经明显包含产品化支撑子系统，不只是六层 Agent 内核。
4. 测试体系中已有一部分脚本与当前接口出现漂移，说明系统在快速演进，但测试同步不足。

## 第一层：感知层现状

### 主要实现

核心在 `app/utils/screen.py`：

- `get_screenshot()`：直接通过 `pyautogui.screenshot()` 采集当前屏幕
- `_detect_anchor_boxes()`：对截图做灰度、边缘检测、连通块遍历，生成候选 anchor box
- `_build_anchor_metadata()`：为每个 anchor 生成中心点百分比、宽高百分比、bbox 百分比、交互分数
- `_annotate_image()`：输出带红框和编号的标注图
- `_build_screen_state()`：把 anchor 集合压缩成结构化摘要

### 当前能力

- 能给模型提供“可点击区域候选”
- 能提供统一的视觉标注图，降低模型直接猜像素坐标的风险
- 能生成基础结构化视图 `screen_state`
- 已考虑最少 anchor 数量不足时的网格兜底

### 当前局限

- 没有 OCR
- 没有图标分类
- 没有控件类型识别
- 没有 Accessibility 树或系统级 UI 元素解析
- `screen_state` 仍然是由 anchor 派生的摘要，不是真正的语义场景理解
- 密集工具栏、小图标、桌面散点图标等场景会天然受限

### 审计结论

这一层已经从“纯截图”进化到“截图 + 弱结构化视觉候选”，但距离真正意义上的 UI 感知还差一大步。

## 第二层：模型规划层现状

### 主要实现

- `app/resources/context.txt`：定义单步闭环 Agent 合约、坐标约束、安全规则
- `app/llm.py`：组装上下文、会话历史、屏幕信息、运行时设置
- `app/models/gpt5.py`
- `app/models/gpt4v.py`
- `app/models/gpt4o.py`
- `app/models/openai_computer_use.py`
- `app/models/model.py`

### 当前能力

- 已明确约束每轮最多返回一个可执行步骤
- 已将会话历史拼接进用户请求
- 已把 `visual_anchors`、`screen_state`、`agent_loop`、逻辑分辨率、截图分辨率一并传给模型
- 已支持多模型后端与 OpenAI-compatible 路由
- 已在 prompt 层引导模型规避近期失败 anchor 和重复失败动作

### 当前局限

- `screen_state` 语义密度低，模型仍然主要依赖视觉理解截图本身
- 规划层没有显式世界模型或任务树，只是单步反应式闭环
- 多模型适配路径较多，行为一致性依赖 prompt 和后处理，不是统一策略内核
- JSON 解析与容错仍偏工程兜底，不是强结构协议执行

### 审计结论

规划层已经进入“单步探索式 Agent”阶段，这是重要升级；但它仍属于“感知不足时的经验型规划”，不是高可靠推理调度器。

## 第三层：定位层现状

### 主要实现

核心在 `app/interpreter.py` 和 `app/models/model.py`：

- 首选 `target_anchor_id`
- 其次 `x_percent` / `y_percent`
- 再次兼容 `x` / `y` 的归一化或截图像素换算
- 统一转换到本机逻辑像素后执行

### 当前能力

- 禁止模型直接依赖裸像素坐标作为主协议
- 已把模型输出与本地屏幕逻辑分辨率做统一映射
- 可把模型给出的截图像素坐标转换成百分比/逻辑坐标
- 支持 coordinate debug 信息回写执行日志

### 当前局限

- anchor 本身仍然是粗候选区域，不是精确控件中心
- 没有二次精定位，例如局部裁剪放大、局部 OCR、模板匹配、热点 refinement
- 没有针对小控件、高密度区域的专门策略
- 当前定位准确率上限本质受感知层质量约束

### 审计结论

定位层已经明显优于“纯猜像素”，但仍属于“粗定位 + 坐标映射”，还不是精定位系统。

## 第四层：执行层现状

### 主要实现

核心在 `app/interpreter.py`：

- 统一解释函数名与参数
- 坐标动作归一化
- `pyautogui` 键鼠动作执行
- 执行日志写入 `execution_logs`
- 运行状态消息回传 UI

### 当前能力

- 支持 click、move、drag、write、press、scroll、sleep 等动作
- 统一了 anchor / 百分比 / 像素兼容输入
- 保留最后一次执行快照，供验证层读取
- 失败时能落库并输出调试信息

### 当前局限

- 依赖 `pyautogui`，平台/权限/焦点都可能影响稳定性
- 执行前有 `command` warm-up 行为，存在副作用风险
- 没有更强的前台窗口确认、焦点确认、执行前后系统状态保护
- 没有动作级回滚能力

### 审计结论

执行层是当前最稳的一层，短期不应成为主重构对象，除非上层需求明确要求更多动作类型或更强安全护栏。

## 第五层：验证层现状

### 主要实现

核心在 `app/verifier.py`：

- 计算全局图像差分比例
- 计算动作中心局部区域差分比例
- 按动作类型套用阈值分类为 `passed` / `failed` / `uncertain`

### 当前能力

- 已形成动作后验证闭环
- 能区分“明显变化”“无变化”“变化不确定”
- 对移动类、文本类、点击类采用不同判断策略
- 验证结果已反哺 memory

### 当前局限

- 不理解文字内容变化
- 不理解窗口标题变化
- 不理解弹窗语义变化
- 不理解焦点变化、选中状态变化、输入框光标状态
- 阈值法在局部微变化和系统动效场景中容易误判

### 审计结论

这一层的方向是正确的，但当前本质仍是图像差分器，不是语义验证器。

## 第六层：记忆与恢复层现状

### 主要实现

- `app/agent_memory.py`：维护最近动作、最近失败、不可靠 anchor、连续验证失败数
- `app/core.py`：在每步执行后记录成功/失败，并在连续验证失败达到阈值后停止任务
- `app/resources/context.txt`：要求模型基于近期失败改换动作或目标

### 当前能力

- 能避免明显重复点击同一坏 anchor
- 能把近期失败上下文暴露给模型
- 能对连续验证失败进行 stop-loss

### 当前局限

- 没有显式的恢复策略树
- 没有“换观察方式 / 换区域 / 换动作 / 退回上一步”的程序化恢复框架
- 失败记忆仍是短期、轻量、单请求内为主
- 记忆主要服务于 prompt，而不是独立决策模块

### 审计结论

这层已经能阻止最明显的盲打循环，但还不具备成熟恢复系统应有的系统性和可解释性。

## 六层之外的真实系统能力

当前仓库里还存在三块非常重要的支撑能力，不能忽略：

### 1. 会话与时间线持久化

`app/session_store.py` 已实现：

- `sessions`
- `messages`
- `execution_logs`
- `app_state`

系统已经支持：

- 多会话
- 当前会话恢复
- 消息历史
- 执行步骤时间线
- 最近活跃会话恢复

### 2. UI 产品层

`app/ui.py` 已不是简单输入框，而是：

- 左侧会话列表
- 右侧统一时间线
- 独立运行状态区
- 配置窗口
- 国际化切换
- 主题切换
- 中断/重试入口

### 3. 配置中心与热加载

`app/utils/settings.py` 和 `app/core.py` 已支持：

- SQLite 化配置中心
- 旧 `settings.json` 迁移
- 配置校验
- 请求超时、模型、语言、主题等设置
- 保存后热加载 LLM 运行时设置

这说明当前系统已经具备“Agent MVP + 产品骨架”的双重属性。

## 当前主要瓶颈清单

### P0 级瓶颈

1. 感知语义太弱
   - 没 OCR
   - 没控件识别
   - 没图标理解
   - 没窗口语义抽取

2. 精定位能力不足
   - anchor 粗框对小热点天然不稳
   - 缺少局部精修策略

3. 验证层无法做语义成功判定
   - 只能看差分
   - 看不懂文字、标题、弹窗意图

4. 恢复策略显式化不足
   - 主要依赖 prompt 提醒模型自行换招

### P1 级瓶颈

5. 多模型适配路径分散
   - `gpt5`、`gpt4o` assistant、`gpt4v`、`computer_use_preview` 行为风格不完全统一

6. 测试体系与当前实现有漂移
   - 一部分测试已落后于接口签名
   - 一部分回归脚本更像历史红灯遗留物

7. 执行安全护栏仍偏轻
   - 焦点确认不足
   - 窗口上下文约束不足
   - warm-up 键可能引入副作用

### P2 级瓶颈

8. 缺少长期记忆/跨请求策略复用

9. 缺少任务级抽象
   - 目前更像“单步 reactive loop”，不是“任务分解 + 子目标管理”

10. 缺少更系统的指标体系
   - 成功率、误点率、恢复成功率、验证误判率等尚未内建

## 当前测试与验证现状

本次实际运行到的检查结果如下：

- `python tests/verify_visual_agent_mvp.py`：通过
- `python tests/verify_gpt5_reasoning.py`：通过
- `python tests/verify_request_timeout_diagnosis.py`：通过
- `python tests/coordinate_mapping_test.py`：失败 1 项
- `.venv/bin/python tests/request_runtime_control_check.py`：失败
- `.venv/bin/python tests/multi_session_regression_check.py`：6 组通过 4 组，失败 2 组
- `.venv/bin/python -m pytest tests/test_config_center_red.py -q`：6 项通过

### 对失败的判断

当前失败并不全部代表生产实现有问题，主要分为两类：

1. 测试脚本落后于当前接口
   - 例如假 LLM 未接收 `request_context`
   - 例如假 Interpreter 未实现验证链路所需快照接口

2. 架构语义已变化但断言未更新
   - 例如配置中心已 SQLite 化，但部分脚本仍按 `settings.json` 路径做断言

因此，当前测试状态可以总结为：

- 系统不是“没有测试”
- 但测试体系需要一次系统性梳理，区分现行回归测试、历史红灯测试、已过时测试

## 工程风险判断

### 短期风险

- 小控件误点
- 无语义验证导致“动作已执行但未真正成功”
- 多模型切换时行为不一致
- 测试对演进速度跟进不足

### 中期风险

- 如果继续只加 prompt、不升级感知和验证，成功率会很快遇到平台上限
- 恢复逻辑若继续只靠模型自发策略，复杂桌面流程会出现不稳定长尾

## 最终判断

你对系统的六层判断方向是正确的，可以作为后续重构讨论的基础。

但如果要更准确地描述当前系统，建议使用下面这句话：

> 当前系统已经完成单步闭环桌面 Agent MVP，并补齐了会话持久化、统一时间线 UI、配置中心热加载等产品骨架；但感知仍是弱语义视觉块方案，定位仍偏粗，验证仍是图像差分，恢复仍主要依赖 prompt 驱动，因此整体仍处于“可运行、可演示、可持续迭代，但核心智能能力仍是一版”的阶段。
