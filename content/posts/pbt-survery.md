+++
date = '2026-03-18T09:31:59+01:00'
draft = false
title = '基于性质测试（Property-based Testing）及其与大语言模型结合的调研报告'
+++

**调研日期**: 2025年3月  
**关键词**: Property-based Testing, QuickCheck, Hypothesis, Large Language Models, 自动测试生成

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [Property-based Testing 技术基础](#2-property-based-testing-技术基础)
3. [主流 PBT 框架对比分析](#3-主流-pbt-框架对比分析)
4. [LLM 与 Property-based Testing 结合研究](#4-llm-与-property-based-testing-结合研究)
5. [LLM 测试生成技术综述](#5-llm-测试生成技术综述)
6. [未来研究方向](#6-未来研究方向)
7. [参考文献与资源](#7-参考文献与资源)

---

## 1. 执行摘要

### 研究背景

随着软件系统复杂度的不断提升，传统的示例驱动测试（Example-based Testing, EBT）在覆盖边界条件和极端场景方面存在明显局限性。基于性质测试（Property-based Testing, PBT）作为一种更高级的测试方法论，通过定义程序应满足的通用性质，自动生成大量随机输入进行验证，能够有效发现传统测试难以覆盖的边界缺陷。

与此同时，大语言模型（LLM）在代码生成领域的快速发展为软件测试带来了新的机遇。将 LLM 与 PBT 技术相结合，有望实现更智能、更全面的自动化测试生成。

### 核心发现

1. **PBT 技术优势显著**: PBT 通过自动生成大规模测试输入，能够发现传统测试遗漏的边界条件和性能问题，测试覆盖率更高。

2. **LLM 与 PBT 结合效果突出**: 最新研究表明，使用 PBT 作为 LLM 代码生成的验证引擎（如 PGS 框架），相比传统 TDD 方法可提升 23.1%-37.3% 的 pass@1 指标。

3. **PBT 与 EBT 具有互补性**: 实验数据显示，PBT 和 EBT 在边界检测方面各有优势，单独使用检测率为 68.75%，组合使用可达 81.25%。

4. **工业级部署已取得成功**: Meta 的 TestGen-LLM 工具已在 Instagram 和 Facebook 平台成功部署，73% 的测试改进建议被工程师接受并投入生产。

---

## 2. Property-based Testing 技术基础

### 2.1 定义与核心理念

**Property-based Testing（基于性质测试）** 是一种测试方法论，其核心理念是：开发者定义程序应满足的**通用性质（Properties）**，而非编写具体的输入-输出示例。测试框架自动生成大量随机输入，验证这些性质是否在所有情况下都成立。

Hypothesis 作者 David R. MacIver 对 PBT 的精确定义：

> Property-based testing is the construction of tests such that, when these tests are fuzzed, failures in the test reveal problems with the system under test that could not have been revealed by direct fuzzing of that system.

### 2.2 工作原理

PBT 框架的核心工作流程包含以下关键组件：

#### 2.2.1 生成器（Generators/Arbitraries）

生成器负责产生随机测试数据。优秀的生成器应具备：
- 支持各种数据类型（基本类型、复合类型、自定义类型）
- 可配置的数据范围和约束条件
- 能够生成边界值和极端情况

```python
# Hypothesis 示例：生成各种类型的测试数据
from hypothesis import strategies as st

# 基本类型
st.integers()      # 整数
st.text()          # 字符串
st.lists(st.integers())  # 整数列表

# 带约束的生成器
st.integers(min_value=0, max_value=100)
st.text(min_size=1, max_size=50)
```

#### 2.2.2 性质（Properties）

性质是程序应满足的不变式或约束条件。常见的性质模式包括：

| 性质类型 | 描述 | 示例 |
|---------|------|------|
| **逆运算** | 正向操作后逆向操作应恢复原状 | `decode(encode(x)) == x` |
| **幂等性** | 多次操作结果相同 | `sort(sort(list)) == sort(list)` |
| **结合律** | 操作顺序不影响结果 | `(a + b) + c == a + (b + c)` |
| **不变式** | 操作前后某些属性保持不变 | 排序后列表长度不变 |
| **元性质** | 基于参考实现的对比 | 与标准库实现结果一致 |

```python
# Hypothesis 性质测试示例
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_reverse_is_involution(lst):
    """反转两次应得到原列表"""
    assert list(reversed(list(reversed(lst)))) == lst

@given(st.lists(st.integers()))
def test_sort_is_idempotent(lst):
    """排序是幂等操作"""
    assert sorted(sorted(lst)) == sorted(lst)
```

#### 2.2.3 收缩（Shrinking）

当测试失败时，PBT 框架会自动尝试找到**最小的失败案例**，这一过程称为收缩。收缩大大简化了调试过程：

```
失败案例: [47, 23, 89, 12, 56, 34, 78, 91, 45, 67]
收缩后:   [1, 2]  # 最小失败案例
```

### 2.3 与传统示例测试的对比

| 维度 | 示例驱动测试 (EBT) | 基于性质测试 (PBT) |
|------|-------------------|-------------------|
| **测试定义** | 具体输入-输出对 | 通用性质/不变式 |
| **测试数量** | 少量手工编写的测试 | 自动生成大量测试 |
| **边界覆盖** | 依赖开发者经验 | 自动探索边界 |
| **调试难度** | 失败案例可能复杂 | 自动收缩简化案例 |
| **维护成本** | 高（需要维护大量测试） | 较低（性质定义简洁） |
| **学习曲线** | 低 | 中等 |
| **适用场景** | 业务逻辑验证 | 算法、库函数、协议实现 |

### 2.4 应用场景与优势

**适用场景**:
- 算法实现验证（排序、搜索、加密等）
- 协议解析器测试（JSON、CSV、自定义格式）
- 编译器和解释器测试
- 数据库操作测试
- 并发和分布式系统测试

**主要优势**:
1. **发现隐藏缺陷**: 自动探索开发者难以预见的边界条件
2. **提高测试信心**: 大规模随机测试提供更强的覆盖保证
3. **文档化规格**: 性质定义本身就是程序的规格说明
4. **减少维护成本**: 一条性质测试替代多条示例测试

---

## 3. 主流 PBT 框架对比分析

### 3.1 QuickCheck (Haskell) - 原始框架

**历史背景**: QuickCheck 由 Chalmers 大学的 Koen Claessen 和 John Hughes 于 1999 年开发，是 PBT 的开创性工作。

**核心特性**:
- 约 300 行代码的轻量级实现
- 与 Haskell 类型系统深度集成
- 支持 Monad 测试（处理副作用）
- 基于类型的自动生成器派生

```haskell
-- QuickCheck 示例
import Test.QuickCheck

-- 性质定义
prop_reverse_involution :: [Int] -> Bool
prop_reverse_involution xs = reverse (reverse xs) == xs

-- 运行测试
-- quickCheck prop_reverse_involution
-- +++ OK, passed 100 tests.
```

**局限性**:
- 开源版本缺乏状态机测试支持
- 不支持并行测试
- 商业版本（QuviQ QuickCheck）功能更完善

### 3.2 Hypothesis (Python) - 高级特性

**简介**: Hypothesis 是 Python 生态中最成熟的 PBT 框架，由 David R. MacIver 开发维护。

**核心特性**:
- 智能测试用例生成（基于覆盖率反馈）
- 内置丰富的生成器策略
- 与 pytest 无缝集成
- 支持 Stateful Testing（状态机测试）
- 优秀的错误报告和收缩机制

```python
# Hypothesis 完整示例
from hypothesis import given, strategies as st, settings
from hypothesis.stateful import RuleBasedStateMachine, rule

# 基本性质测试
@given(st.lists(st.integers(), min_size=1))
@settings(max_examples=1000)
def test_max_property(lst):
    """最大值应大于等于列表中所有元素"""
    m = max(lst)
    for x in lst:
        assert m >= x

# 状态机测试（测试有状态系统）
class DictStateMachine(RuleBasedStateMachine):
    def __init__(self):
        super().__init__()
        self.dict = {}
    
    @rule(key=st.integers(), value=st.integers())
    def set_value(self, key, value):
        self.dict[key] = value
    
    @rule(key=st.integers())
    def get_value(self, key):
        # 性质：设置的值应该能被获取
        if key in self.dict:
            assert self.dict[key] == self.dict.get(key)

TestDictStateMachine = DictStateMachine.TestCase
```

**优势**:
- 收缩机制优于 QuickCheck
- 自动发现边界情况
- 与 Python 生态（Django、pandas 等）良好集成

### 3.3 fast-check (JavaScript/TypeScript) - 前端生态

**简介**: fast-check 是 JavaScript/TypeScript 生态中最流行的 PBT 框架。

**核心特性**:
- TypeScript 原生支持
- 支持异步测试
- 内置 Model-based Testing
- 种子控制确保可重现性
- 丰富的 Arbitrary（生成器）库

```typescript
// fast-check 示例
import fc from 'fast-check';

// 性质测试
describe('Array properties', () => {
  it('reverse is involution', () => {
    fc.assert(
      fc.property(fc.array(fc.integer()), (arr) => {
        const reversed = [...arr].reverse();
        return [...reversed].reverse().every((v, i) => v === arr[i]);
      })
    );
  });
});

// 带配置的测试
it('handles large inputs', () => {
  fc.assert(
    fc.property(
      fc.array(fc.integer(), { minLength: 0, maxLength: 10000 }),
      (arr) => {
        // 测试逻辑
      }
    ),
    { numRuns: 1000 }  // 运行1000次
  );
});
```

### 3.4 框架特性对比表

| 特性 | QuickCheck (Haskell) | Hypothesis (Python) | fast-check (JS/TS) |
|------|---------------------|---------------------|-------------------|
| **首次发布** | 1999 | 2013 | 2017 |
| **状态机测试** | ❌ (开源版) | ✅ | ✅ |
| **并行测试** | ❌ (开源版) | ❌ | ⚠️ (有限支持) |
| **收缩机制** | ✅ | ✅✅ (优秀) | ✅ |
| **异步支持** | ✅ | ✅ | ✅ |
| **学习曲线** | 中 | 中 | 低 |
| **文档质量** | 良好 | 优秀 | 优秀 |
| **社区活跃度** | 中 | 高 | 高 |

### 3.5 选型建议

| 场景 | 推荐框架 | 理由 |
|------|---------|------|
| **Python 项目** | Hypothesis | 生态最成熟，功能最完善 |
| **前端/Node.js** | fast-check | TypeScript 支持，API 友好 |
| **Haskell 项目** | QuickCheck | 原生支持，类型系统集成 |
| **需要并行测试** | PropEr (Erlang) / quickcheck-state-machine (Haskell) | 少数支持并行测试的开源框架 |
| **Java/Kotlin** | jqwik 或 Kotest | JVM 生态最佳选择 |
| **Go 语言** | Gopter 或 Rapid | Go 社区主流选择 |

---

## 4. LLM 与 Property-based Testing 结合研究

### 4.1 PGS (Property-Generated Solver) 框架

**论文来源**: "Use Property-Based Testing to Bridge LLM Code Generation and Validation" (2025)

#### 4.1.1 核心思想

PGS 框架创新性地将 PBT 作为 LLM 代码生成迭代过程的核心验证引擎，解决了传统 TDD 方法的关键问题：

**传统 TDD 方法的问题**:
1. 测试用例生成与代码生成可能共享相同偏差
2. 生成准确的测试预言（oracle）比代码生成本身更难
3. 现有方法优先代码覆盖率而非语义正确性

**PGS 的解决方案**:
- 不验证具体的输入-输出对，而是验证高层性质/不变式
- 定义性质通常比预测精确输出简单得多
- 性质捕获关键正确性特征，无需精确的输入-输出映射

#### 4.1.2 架构设计

PGS 采用双智能体架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    PGS 框架架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌─────────────┐              ┌─────────────┐            │
│    │  Generator  │◄────────────►│   Tester    │            │
│    │   (生成器)   │              │  (测试器)    │            │
│    └──────┬──────┘              └──────┬──────┘            │
│           │                            │                    │
│           │ 生成候选代码               │ 定义性质            │
│           │                            │ 生成测试输入        │
│           │                            │ 执行验证            │
│           │                            │ 提供反馈            │
│           ▼                            ▼                    │
│    ┌─────────────────────────────────────────────┐         │
│    │              迭代优化循环                     │         │
│    │  代码生成 → 性质验证 → 反馈 → 代码改进 → ...  │         │
│    └─────────────────────────────────────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Tester 智能体的职责**:
1. 定义高层抽象性质（不变式、功能约束）
2. 将性质转换为可执行的属性检查代码
3. 生成多样化的测试输入
4. 策略性选择性质违规案例
5. 提供语义丰富的反馈

#### 4.1.3 实验结果

在 HumanEval、MBPP 和 LiveCodeBench 基准测试上：

| 基准测试 | 相比传统 TDD 的 pass@1 提升 |
|---------|---------------------------|
| HumanEval | 23.1% - 37.3% |
| MBPP | 显著提升 |
| LiveCodeBench | 显著提升 |

**关键发现**:
- 对于直接提示方法难以解决的问题，PGS 效果更显著
- 性质驱动的反馈比传统测试反馈更有效
- 迭代细化过程更加稳健

### 4.2 Meta TestGen-LLM

**论文来源**: "Automated Unit Test Improvement using Large Language Models at Meta" (FSE 2024)

#### 4.2.1 工具定位

TestGen-LLM 是 Meta 开发的工业级工具，用于自动扩展现有的单元测试类，生成额外的测试用例来覆盖之前遗漏的边界情况。

**核心特点**: Assured LLM-based Software Engineering (Assured LLMSE)
- 不是生成代码片段，而是完整的软件改进建议
- 所有建议都有可验证的改进保证
- 过滤机制消除 LLM 幻觉问题

#### 4.2.2 工作流程

```
┌────────────────────────────────────────────────────────────────┐
│                    TestGen-LLM 过滤机制                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   LLM 生成的测试候选                                            │
│         │                                                      │
│         ▼                                                      │
│   ┌─────────────┐                                              │
│   │ Filter 1:   │ 可编译？                                     │
│   │ 构建检查     │────── 否 ──► 丢弃                           │
│   └──────┬──────┘                                              │
│          │是                                                   │
│          ▼                                                     │
│   ┌─────────────┐                                              │
│   │ Filter 2:   │ 测试通过？                                   │
│   │ 执行检查     │────── 否 ──► 丢弃                           │
│   └──────┬──────┘                                              │
│          │是                                                   │
│          ▼                                                     │
│   ┌─────────────┐                                              │
│   │ Filter 3:   │ 非不稳定？(5次执行全通过)                     │
│   │ 稳定性检查   │────── 否 ──► 丢弃                           │
│   └──────┬──────┘                                              │
│          │是                                                   │
│          ▼                                                     │
│   ┌─────────────┐                                              │
│   │ Filter 4:   │ 增加覆盖率？                                  │
│   │ 覆盖率检查   │────── 否 ──► 丢弃                           │
│   └──────┬──────┘                                              │
│          │是                                                   │
│          ▼                                                     │
│   ✅ 提交给工程师审查                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### 4.2.3 部署成果

**Instagram 测试马拉松 (2023年11月)**:
- TestGen-LLM 在 36 名工程师中排名第 6（按生成测试数量）
- 生成了 17 个测试，覆盖 1,460 行代码

**量化结果**:
| 指标 | 数值 |
|------|------|
| 测试类生成成功率 | 75% |
| 测试可靠通过率 | 57% |
| 覆盖率提升比例 | 25% |
| 工程师接受率 | 73% |
| 改进的测试类比例 | 10-11.5% |

**关键经验**:
1. 每个测试用例级别的保证比类级别更重要
2. LLM 生成的测试会模仿现有代码风格
3. Ensemble 方法（多 LLM、多提示）效果更好
4. 温度参数 0 在测试生成中最稳定

### 4.3 LLM 生成 PBT 测试的特性研究

**论文来源**: "Understanding the Characteristics of LLM-Generated Property-Based Tests in Exploring Edge Cases" (2025)

#### 4.3.1 研究设计

在 HumanEval 数据集上，选取 16 个在 HumanEval+ 扩展测试中失败的标准解，使用 Claude-4-sonnet 分别生成 PBT 和 EBT 测试代码。

#### 4.3.2 实验结果

| 检测模式 | 数量 | 占比 |
|---------|------|------|
| 仅 PBT 成功 | 2 | 12.5% |
| 仅 EBT 成功 | 2 | 12.5% |
| 两者都成功 | 9 | 56.25% |
| 两者都失败 | 3 | 18.75% |

**检测率对比**:
- PBT 单独检测率: **68.75%** (11/16)
- EBT 单独检测率: **68.75%** (11/16)
- 组合检测率: **81.25%** (13/16)

#### 4.3.3 互补特性分析

**PBT 优势场景**:
- **性能问题检测**: 大规模输入空间的自动探索
- **隐藏逻辑错误**: 通过性质约束发现非预期行为
- **案例**: HumanEval/55 中，PBT 测试范围是 EBT 的 2.9 倍，成功检测超时问题

**EBT 优势场景**:
- **特定边界条件**: 明确的边界值测试
- **特殊模式**: 特定输入结构的组合
- **案例**: HumanEval/140 中，EBT 明确测试双尾空格的特殊情况

**共同局限**:
- 特殊边界条件的组合
- 极大数值范围的限制

#### 4.3.4 实践建议

1. **混合方法**: 先用 PBT 进行广泛测试，再用 EBT 补充特定边界条件
2. **范围设计**: 明确指定测试范围，特别是性能测试
3. **提示优化**: 包含"从最小值到最大值"、"包含边界情况"等明确指令
4. **系统化验证**: 引入边界值（0、负数、最大值、最小值）自动生成机制

---

## 5. LLM 测试生成技术综述

### 5.1 方法分类

基于 MDPI 发表的综述论文 "A Review of Large Language Models for Automated Test Case Generation"，LLM 测试生成方法可分为四类：

#### 5.1.1 提示设计与工程

通过精心设计的提示词引导 LLM 生成高质量测试：

| 技术 | 描述 | 代表工作 |
|------|------|---------|
| Few-shot Learning | 提供示例测试引导生成 | ChatUniTest |
| Chain-of-Thought | 分步推理生成测试 | CoT-based approaches |
| Role-based Prompting | 定义角色（如测试工程师） | AgentCoder |
| Domain-specific Prompts | 领域特定提示模板 | 行业定制化方案 |

#### 5.1.2 反馈驱动方法

迭代交互机制持续改进测试质量：

```
┌─────────────────────────────────────────────┐
│           反馈驱动测试生成                    │
├─────────────────────────────────────────────┤
│                                             │
│   ┌─────────┐    ┌─────────┐    ┌───────┐  │
│   │ 代码    │───►│ LLM    │───►│ 测试  │  │
│   │ 输入    │    │ 生成    │    │ 输出  │  │
│   └─────────┘    └────┬────┘    └───┬───┘  │
│                       │             │      │
│                       ▲             │      │
│                       │             ▼      │
│                       │      ┌───────────┐ │
│                       └──────│ 执行反馈   │ │
│                              │ (覆盖率、  │ │
│                              │  失败信息) │ │
│                              └───────────┘ │
│                                             │
└─────────────────────────────────────────────┘
```

**代表工作**:
- **CodaMosa**: LLM + 搜索测试结合，突破覆盖率瓶颈
- **ChatTester**: 评估并改进 ChatGPT 测试生成
- **TestART**: 自动生成与修复迭代共进化

#### 5.1.3 模型微调与预训练

针对测试生成任务专门优化模型：

| 方法 | 描述 | 效果 |
|------|------|------|
| CAT-LM | 在代码-测试对齐数据上预训练 | 提升测试生成质量 |
| Assertion Generation | 微调生成断言 | 提高断言准确性 |
| Domain Adaptation | 领域适应微调 | 特定领域效果提升 |

#### 5.1.4 混合方法

将 LLM 与传统测试工具结合：

- **LLM + SBST**: 结合搜索测试优化覆盖率
- **LLM + Fuzzing**: LLM 引导模糊测试
- **LLM + Mutation Testing**: 使用变异测试评估 LLM 生成测试质量
- **LLM + Symbolic Execution**: 符号执行补充 LLM 生成

### 5.2 有效性评估

#### 5.2.1 LLM 优于现有工具的场景

- 语义理解复杂规格
- 生成人类可读的测试代码
- 处理自然语言需求
- 快速生成初始测试框架

#### 5.2.2 LLM 不如现有工具的场景

- 保证代码覆盖率
- 处理大型代码库
- 稳定性和可重现性
- 特定领域知识

#### 5.2.3 混合/依赖上下文的结果

- LLM 效果受项目特定知识影响显著
- 需要适当的上下文信息
- 评估实践存在碎片化问题

### 5.3 当前挑战与局限

| 挑战 | 描述 | 可能的解决方向 |
|------|------|---------------|
| **幻觉问题** | 生成不正确的测试逻辑 | Assured LLMSE 方法 |
| **Oracle 问题** | 难以确定正确输出 | 使用 PBT 定义性质 |
| **覆盖率保证** | 难以保证覆盖目标 | 结合 SBST 方法 |
| **领域知识** | 缺乏特定领域知识 | 微调 + RAG |
| **评估标准** | 缺乏统一评估标准 | 建立标准基准 |

---

## 6. 未来研究方向

### 6.1 技术融合趋势

1. **PBT + LLM 深度整合**
   - LLM 自动从代码和文档推断性质
   - 性质语言的标准化和形式化
   - 基于性质的代码生成验证闭环

2. **多智能体测试系统**
   - 专门化的测试智能体（生成、验证、优化）
   - 人机协作的测试开发流程
   - 持续学习和适应机制

3. **混合测试方法论**
   - PBT + EBT 组合的最佳实践
   - 静态分析 + LLM + PBT 三重保障
   - 自适应测试策略选择

### 6.2 待解决问题

1. **性质推断自动化**
   - 如何从代码/文档自动推断正确的性质？
   - 如何避免推断的性质过于平凡或过于严格？

2. **测试质量保证**
   - 如何确保 LLM 生成的测试本身是正确的？
   - 如何度量测试套件的有效性？

3. **工业级部署**
   - 如何在大型代码库中高效部署？
   - 如何处理遗留系统的测试生成？

4. **并行测试普及**
   - 并行 PBT 开源支持仍然有限
   - 如何降低状态机建模的学习门槛？

### 6.3 产业应用前景

| 应用领域 | 当前状态 | 未来潜力 |
|---------|---------|---------|
| **云服务测试** | 初步应用 | 高（Meta 等已部署） |
| **金融科技** | 探索阶段 | 高（安全性要求高） |
| **嵌入式系统** | 研究阶段 | 中（安全关键系统） |
| **AI 系统测试** | 新兴领域 | 高（模型行为验证） |
| **DevOps 集成** | 工具整合中 | 高（CI/CD 集成） |

---

## 7. 参考文献与资源

### 7.1 核心论文

1. **PBT 原创论文**
   - Claessen, K., & Hughes, J. (2000). QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs. ICFP 2000.

2. **PGS 框架**
   - He, L., et al. (2025). Use Property-Based Testing to Bridge LLM Code Generation and Validation. arXiv:2506.18315.

3. **TestGen-LLM**
   - Alshahwan, N., et al. (2024). Automated Unit Test Improvement using Large Language Models at Meta. FSE 2024.

4. **LLM 生成 PBT 特性研究**
   - Anonymous. (2025). Understanding the Characteristics of LLM-Generated Property-Based Tests in Exploring Edge Cases. arXiv:2510.25297.

5. **LLM 测试生成综述**
   - MDPI. (2025). A Review of Large Language Models for Automated Test Case Generation. Machines, 7(3), 97.

### 7.2 框架官方资源

| 框架 | 官网/仓库 |
|------|----------|
| **Hypothesis** | https://hypothesis.works/ |
| **fast-check** | https://fast-check.dev/ |
| **QuickCheck** | https://hackage.haskell.org/package/QuickCheck |
| **jqwik** | https://jqwik.net/ |
| **PropEr** | https://proper-testing.github.io/ |

### 7.3 相关资源汇总

- **AwesomeLLM4UT**: https://github.com/iSEngLab/AwesomeLLM4UT (LLM 单元测试研究汇总)
- **Awesome-Code-LLM**: https://github.com/codefuse-ai/Awesome-Code-LLM (代码 LLM 资源汇总)
- **PBT 框架对比**: https://github.com/jmid/pbt-frameworks

---

## 附录：名词解释

| 术语 | 英文 | 解释 |
|------|------|------|
| 基于性质测试 | Property-based Testing (PBT) | 定义程序性质而非具体示例的测试方法 |
| 示例驱动测试 | Example-based Testing (EBT) | 使用具体输入-输出对的测试方法 |
| 收缩 | Shrinking | 将失败案例简化为最小形式的过程 |
| 生成器 | Generator/Arbitrary | 生成随机测试数据的组件 |
| 性质 | Property | 程序应满足的不变式或约束条件 |
| 测试预言 | Test Oracle | 判断测试结果正确性的机制 |
| 幂等性 | Idempotence | 多次操作结果相同的性质 |
| 状态机测试 | Stateful/State Machine Testing | 使用状态机模型测试有状态系统的方法 |
| 线性化 | Linearizability | 并发操作正确性验证的理论基础 |

---
