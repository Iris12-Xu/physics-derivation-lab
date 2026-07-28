# Physics IR 设计草案

> 状态：概念模型已确定，具体字段、Python 类型和序列化格式可在 MVP 实现中调整。

## 1. 目的

Physics IR 保存符号表达式本身无法表达的物理语义：

- 变量代表什么；
- 使用什么单位和量纲；
- 坐标系与参考方向是什么；
- 模型和近似条件是什么；
- 定律从哪里来、在什么条件下成立；
- 候选解必须满足哪些物理约束。

MVP 至少需要表达五类概念：

```text
Quantity
Law
Constraint
Model
DerivationStep
```

这些名称是当前统一术语，不要求一一对应为五个 Python 类。

## 2. Quantity

物理量建议包含：

- 唯一标识；
- 物理含义；
- 标量、向量、张量或算符类型；
- 单位与量纲；
- 定义域和范围；
- 坐标系或参考方向；
- 已知、未知或目标状态。

示例：

```yaml
id: theta_i
name: incidence_angle
value_type: scalar
dimension: angle
unit: radian
domain: real

range:
  minimum: 0
  maximum: pi / 2

reference:
  measured_from: interface_normal
```

## 3. Law

定律或几何关系需要同时保存可执行表达式和适用条件。

```yaml
id: snell_law
name: Snell's law

equation:
  left: n1 * sin(theta_i)
  relation: equal
  right: n2 * sin(theta_t)

requires:
  - geometric_optics
  - isotropic_medium
  - locally_smooth_interface

source:
  type: textbook
  status: verified
```

### 知识状态

建议区分：

- `verified`：项目审核且已有验证用例；
- `reviewed`：已人工检查，但验证覆盖尚不完整；
- `proposed`：用户或 AI 提交，尚未进入可靠知识；
- `rejected`：已知不适用或错误。

具体枚举名称可调整，但不能混淆信任级别。

## 4. Constraint

约束分为三类。

### 4.1 可计算约束

程序可以直接求值或证明：

```yaml
- n1 > 0
- n2 > 0
- 0 <= theta_i
- theta_i <= pi / 2
- abs(sin_theta_t) <= 1
```

### 4.2 可匹配的模型约束

通过结构化模型匹配：

```yaml
- interface == locally_smooth
- medium_type == isotropic
- propagation_mode == geometric_optics
```

### 4.3 声明性假设

当前任务无法独立证明：

```yaml
- medium_is_isotropic
- absorption_is_negligible
- geometric_optics_is_valid
```

这类条件必须显示为用户接受或模型预设，不能标记为自动验证通过。

## 5. Model

模型定义研究对象和近似范围。

```yaml
id: planar_dielectric_interface

geometry:
  interface: locally_smooth

media:
  incident: medium_1
  transmitted: medium_2

optics:
  approximation: geometric_optics

angle_convention:
  incidence_reference: interface_normal
  transmission_reference: interface_normal
```

同一个公式在不同角度约定或传播方向下可能产生不同解释，因此模型不能只是自由文本备注。

## 6. DerivationStep

每一步应记录：

- 唯一 ID；
- 操作；
- 输入；
- 操作参数；
- 输出；
- 使用的知识条目；
- 验证结果。

```yaml
step_id: solve_theta_t
operation: solve

input:
  - snell_equation

target:
  - theta_t

output:
  - theta_t_candidates

verification:
  method: substitute_into_original
  status: passed
```

## 7. 任务结果

结果不能只返回一个表达式。建议至少包含：

```yaml
status: READY_TO_DERIVE

recommended:
  - solution_1

physically_possible: []

rejected:
  - solution: solution_2
    reason: angle_out_of_range

verification:
  symbolic_substitution: passed
  dimensions: passed
  numerical_cross_check: passed
  geometric_optics_validity: user_confirmed
```

## 8. 表达式安全

MVP 表达式解析器必须：

- 拒绝任意 Python 代码；
- 只接受注册的变量、常量、关系和函数；
- 对函数参数数量和类型进行检查；
- 对未知符号返回明确错误；
- 保留原始文本与解析后表达式，便于审计；
- 区分项目审核知识和单次任务输入。

允许函数的初始白名单可包含：

```text
sin
cos
tan
asin
acos
atan
sqrt
Abs
```

白名单应由实际 MVP 用例驱动，避免在没有测试的情况下扩大。

## 9. MVP 中必须确定的最小集合

实现前需要落定：

- 斯涅耳任务输入模式；
- 折射率、入射角和折射角的 Quantity 表达；
- 斯涅耳定律知识条目；
- 几何光学与界面模型；
- 角度单位规范化策略；
- 候选解和拒绝原因的数据结构；
- 验证报告的最小字段；
- NumPy 后端的输入输出合同。

其余字段在真实用例证明需要后再加入。
