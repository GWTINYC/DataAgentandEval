问：给我一版数据层的更新文档。要求逐步进行，一开始肯定拼凑的比较粗，优先跑通流程，然后再一步步优化，保证质量。整体计划要考虑实际并且可行

答：
# 企业内部知识库数据层建设更新方案

## 1. 背景与目标

当前项目目标是面向企业内部代码仓库，构建业务视角的智能问答助手，帮助业务同学理解业务代码、业务流程和系统逻辑。

数据层是整个系统的基础，负责将原始代码仓库转化为业务视角的结构化知识。当前数据层计划分为两个阶段：

1. **阶段一：Code to JSON**
   基于 Understand Anything 中的 `understand-domain` 能力，结合自定义 Skill，将原始代码仓库抽取为结构化 JSON。

2. **阶段二：JSON to 知识卡片**
   基于自研 Agent，将 JSON 中的代码结构、业务流程、业务实体、调用关系和代码证据，进一步转化为业务视角的 Markdown 知识卡片。

本方案的核心原则是：

> 前期优先跑通流程，不追求一步到位；中期补齐结构规范和质量门禁；后期再逐步提高知识卡片的准确性、完整性和可维护性。

---

## 2. 数据层整体定位

数据层不是简单地“总结代码”，而是将代码仓库编译成业务知识。

数据层最终需要产出两类核心资产：

1. **结构化 JSON IR**

   * 用来承载代码对象、业务域、业务流程、业务实体、状态流转、异常流程和代码证据。
   * 作为后续知识卡片、图谱构建和问答召回的中间表示。

2. **业务知识卡片**

   * 用业务同学能理解的语言解释代码背后的业务机制。
   * 每张卡片聚焦一个业务问题、业务流程、业务规则或业务实体。
   * 关键结论需要尽量绑定代码证据。

整体链路如下：

```text
原始代码仓库
  ↓
Understand Anything / understand-domain
  ↓
自定义 Skill 规范化
  ↓
Raw JSON
  ↓
Adapter 转换
  ↓
Canonical JSON IR
  ↓
JSON 质量检查
  ↓
知识卡片生成 Agent
  ↓
Markdown 知识卡片
  ↓
卡片质量检查与索引
```

---

## 3. 建设原则

### 3.1 先跑通，再变好

第一阶段不要一开始追求完美抽取。更现实的目标是：

```text
能扫描代码
能生成 JSON
能生成知识卡片
能被后续召回
能回答一小批典型问题
```

先把端到端链路打通，再逐步提高每一层质量。

---

### 3.2 结构事实和业务推断分开

代码中的确定性事实和 LLM 推断出来的业务含义不能混在一起。

建议区分：

```text
deterministic：通过代码结构、AST、调用关系、配置等确定得到
llm_inferred：由 LLM 根据代码上下文推断得到
human_verified：经过人工确认
unknown：当前代码仓无法确认
```

这样可以避免后续问答时把“模型猜测”当成“代码事实”。

---

### 3.3 JSON 是中间表示，不是最终文档

JSON 不应该只是自然语言摘要，而应该是稳定、可校验、可转换的中间表示。

好的 JSON 应该能支持：

```text
生成知识卡片
构建知识图谱
做问题召回
做质量评测
做增量更新
做错误归因
```

---

### 3.4 知识卡片要业务化，但不能脱离代码证据

知识卡片的表达要面向业务同学，少说类名、函数名、参数名，多说业务动作、业务状态、业务规则和异常场景。

但同时，每个关键业务结论都应该尽量能追溯到代码证据。

---

## 4. 分阶段建设计划

## 阶段 0：准备阶段

### 目标

明确数据层第一版建设范围，避免一开始全仓库铺开导致质量不可控。

### 工作内容

1. 选择一个核心业务域作为试点。
2. 选择 1～3 个核心代码仓库。
3. 收集 10～20 个业务同学常问的问题。
4. 人工梳理 5～10 张理想知识卡片作为标杆。
5. 定义第一版最小 JSON 字段。

### 第一版试点范围建议

优先选择：

```text
业务价值高
链路相对清晰
代码规模适中
业务同学经常提问
跨模块关系不要过于复杂
```

不建议一开始选择：

```text
历史包袱太重的模块
强依赖多个外部系统的模块
业务概念非常混乱的模块
代码质量很差且缺少命名规范的模块
```

### 交付物

```text
试点业务域清单
试点代码仓库清单
典型问题集
标杆知识卡片
最小 JSON Schema 草案
```

---

## 阶段 1：MVP 跑通阶段

### 目标

在不追求高质量的前提下，打通完整链路：

```text
code → raw json → canonical json → markdown card
```

这一阶段的核心目标不是“生成得很好”，而是“链路能跑通，问题能暴露出来”。

---

### 1.1 Code to Raw JSON

使用 Understand Anything 的 `understand-domain` 能力，先从代码中抽取候选业务域、业务流程和步骤。

同时保留 Understand Anything 的原始输出，不直接覆盖或改写。

推荐目录结构：

```text
data_layer/
  raw/
    understand_anything/
      knowledge_graph.json
      domain_graph.json

  ir/
    canonical_domain_ir.json

  cards/
    markdown/

  reports/
    quality_report.json
```

---

### 1.2 自定义 Skill 的第一版目标

第一版自定义 Skill 不要写得太复杂，先控制三件事：

#### 1. 统一业务动作表达

将代码动作转成业务动作：

```text
createRefund → 创建退款单
checkCancelable → 校验是否允许取消
handleCallback → 处理支付渠道回调
retryRefund → 退款失败补偿
```

避免 JSON 里充满“调用某某方法”“执行某某类”这种代码视角描述。

---

#### 2. 强制保留代码证据

每个核心业务步骤尽量保留：

```text
file
class
method
line_range
repo
commit
```

前期如果行号拿不到，至少要有：

```text
file
class
method
```

---

#### 3. 标记不确定内容

如果模型无法确认，不要强行补全。

例如：

```json
{
  "unknowns": [
    {
      "question": "退款失败后的人工处理入口未在当前仓库中找到",
      "possible_reason": "可能位于运营后台或其他服务"
    }
  ]
}
```

---

### 1.3 最小 JSON IR 结构

第一版 JSON 不要设计得过重，建议先保留这些字段：

```json
{
  "repo": "repo_name",
  "commit": "commit_hash",
  "domain": {
    "domain_id": "order_cancel",
    "domain_name": "订单取消",
    "summary": "该业务域负责处理订单取消及其后续影响"
  },
  "flows": [
    {
      "flow_id": "order_cancel_flow",
      "flow_name": "订单取消流程",
      "trigger": "用户主动取消订单或系统自动取消订单",
      "steps": [
        {
          "step_id": "check_cancelable",
          "step_name": "校验订单是否允许取消",
          "business_action": "判断订单当前状态是否还能取消",
          "code_refs": [
            {
              "file": "OrderCancelService.java",
              "symbol": "cancelOrder"
            }
          ],
          "source": "llm_inferred",
          "confidence": 0.8
        }
      ]
    }
  ],
  "entities": [
    {
      "entity_id": "order",
      "entity_name": "订单",
      "entity_type": "business_entity"
    }
  ],
  "unknowns": []
}
```

第一版 JSON 只需要支撑知识卡片生成，不要过早追求完整图谱能力。

---

### 1.4 JSON to 知识卡片 Agent

第一版 Agent 可以先做成单流程：

```text
读取一个业务域 JSON
  ↓
识别主要 flow
  ↓
生成一张流程卡片
  ↓
输出 Markdown
```

第一版知识卡片模板：

```markdown
# 业务流程名称

## 这张卡片解决什么问题
说明这张卡片解释哪个业务问题。

## 一句话理解
用一句业务语言解释该流程的核心逻辑。

## 触发场景
说明什么时候会进入这个流程。

## 主流程
1. 第一步
2. 第二步
3. 第三步

## 异常情况
说明失败、重试、幂等、补偿等情况。如果当前代码无法确认，要明确说明。

## 涉及的业务实体
- 实体 A
- 实体 B

## 关键代码证据
- file / class / method

## 当前不能确认的部分
说明当前仓库无法证明或需要人工确认的内容。
```

---

### 阶段 1 验收标准

阶段 1 不追求高分，验收标准建议设得现实一些：

```text
能够完成 1 个业务域的端到端生成
能够生成 5～10 张知识卡片
每张卡片至少包含主流程和代码证据
能够支撑 10 个典型问题中的一部分回答
能暴露出主要质量问题
```

阶段 1 的核心产出是“可运行链路”和“问题清单”。

---

## 阶段 2：结构规范阶段

### 目标

在 MVP 基础上，将 JSON 从“能用”提升为“可维护、可校验、可回归”。

---

### 2.1 建立 Canonical JSON Schema

不要直接依赖 Understand Anything 的原始输出格式，而是定义自己的标准 JSON IR。

建议增加这些字段：

```text
card_candidates：适合生成哪些知识卡片
relations：实体、流程、步骤之间的关系
state_transitions：状态流转
exception_paths：异常流程
quality_flags：质量问题标记
source_type：信息来源
confidence：置信度
```

---

### 2.2 建立 Adapter 层

将 Understand Anything 的原始输出转换成内部标准格式。

```text
Understand Anything raw output
  ↓
adapter
  ↓
canonical_domain_ir.json
```

Adapter 的职责：

```text
字段映射
命名统一
业务域归并
重复 flow 合并
代码证据补全
source 和 confidence 标记
```

不要直接把开源项目的输出作为生产数据格式，避免后续受其 schema 变化影响。

---

### 2.3 增加 JSON 质量检查

第一版质量检查可以先做规则，不需要复杂模型。

检查项：

```text
JSON 是否合法
必填字段是否存在
flow_id / step_id 是否唯一
relation 引用对象是否存在
code_refs 是否为空
confidence 是否在 0～1
source_type 是否合法
是否存在明显空泛描述
```

空泛描述示例：

```text
“该模块负责处理业务逻辑”
“该方法用于处理数据”
“该流程完成相关操作”
```

这些描述要打低分或要求重写。

---

### 阶段 2 验收标准

```text
Canonical JSON Schema 初版完成
Understand Anything 输出可以稳定转换为内部 IR
JSON 质量检查脚本可运行
主要字段通过率达到 80% 以上
每个 flow 至少有 trigger、steps、code_refs
```

---

## 阶段 3：知识卡片质量提升阶段

### 目标

将知识卡片从“代码摘要”提升为“业务视角解释”。

---

### 3.1 知识卡片分类

不要所有内容都生成同一种卡片。建议逐步支持 5 类卡片：

```text
业务域总览卡
业务流程卡
状态机卡
业务规则卡
异常补偿卡
```

前期优先做：

```text
业务流程卡
业务规则卡
异常补偿卡
```

因为这些最贴近业务同学的问题。

---

### 3.2 Agent 拆分

第一版 Agent 可以简单，但从阶段 3 开始建议拆成多个节点：

```text
Card Planner
  判断应该生成哪些卡片

Context Collector
  从 JSON 中收集该卡片需要的 flow、entity、code_ref、relation

Card Writer
  生成 Markdown 卡片

Card Reviewer
  检查准确性、完整性、业务语言、证据绑定

Card Linker
  生成 aliases、related_cards、retrieval_keywords
```

---

### 3.3 卡片质量标准

每张卡片至少检查 6 个维度：

```text
准确性：是否被 JSON 和代码证据支持
完整性：是否包含主流程、异常流、边界条件
业务化：是否用业务语言解释，而不是代码语言
证据性：关键结论是否有代码证据
可召回性：标题、别名、关键词是否贴近业务问法
可链接性：是否能关联到相关卡片
```

---

### 3.4 卡片 metadata

每张 Markdown 卡片旁边建议生成一个 metadata JSON：

```json
{
  "card_id": "order_cancel_refund_flow",
  "title": "订单取消后，退款是如何发起并确认的？",
  "card_type": "flow_card",
  "domain": "订单取消",
  "entities": ["订单", "退款单", "支付单"],
  "aliases": [
    "取消订单后怎么退款",
    "订单取消后钱什么时候退",
    "退款链路"
  ],
  "retrieval_keywords": [
    "订单取消",
    "退款",
    "支付回调",
    "退款失败",
    "补偿"
  ],
  "related_cards": [
    "refund_status_machine",
    "payment_callback",
    "refund_retry"
  ],
  "code_refs": [
    "OrderCancelService.cancelOrder",
    "RefundService.createRefund"
  ]
}
```

metadata 的价值很大，后续召回层主要依赖它提升召回效果。

---

### 阶段 3 验收标准

```text
每张卡片有固定模板
每张卡片有 metadata
每张卡片至少有 3 个业务关键词
核心卡片具备 related_cards
人工抽检通过率达到 70% 以上
10～20 个典型问题能召回相关卡片
```

---

## 阶段 4：评测与回归阶段

### 目标

建立数据层自己的评测体系，避免每次改动后只能凭感觉判断质量。

---

### 4.1 建立小规模黄金集

每个典型问题不要只标注理想答案，还要标注：

```text
必须命中的业务域
必须命中的 flow
必须命中的 step
必须命中的 entity
必须命中的 code_ref
必须生成或召回的 card
禁止出现的错误说法
```

示例：

```json
{
  "question": "订单取消后，退款为什么不是马上到账？",
  "expected_domain": "订单取消",
  "expected_flows": ["order_cancel_refund_flow"],
  "expected_steps": [
    "create_refund",
    "request_payment_refund",
    "wait_payment_callback"
  ],
  "expected_entities": ["订单", "退款单", "支付单"],
  "expected_cards": [
    "order_cancel_refund_flow",
    "payment_callback",
    "refund_status_machine"
  ],
  "must_have_points": [
    "订单取消后如果已支付，需要进入退款流程",
    "退款需要调用支付渠道",
    "最终结果依赖支付渠道回调"
  ],
  "forbidden_claims": [
    "不能说订单取消后一定立即到账"
  ]
}
```

---

### 4.2 数据层评测指标

建议先做 6 个指标：

```text
JSON Schema 通过率
业务流程覆盖率
代码证据绑定率
卡片生成成功率
卡片人工抽检通过率
典型问题卡片召回命中率
```

其中最关键的是：

```text
代码证据绑定率
典型问题卡片召回命中率
```

因为前者决定知识是否可信，后者决定后续问答能不能用上这些知识。

---

### 4.3 错误归因

每次发现卡片质量差，要归因到具体环节：

```text
Understand Anything 抽取漏了
自定义 Skill 表达不规范
Adapter 字段转换丢信息
JSON Schema 设计不合理
Card Agent 没用上关键字段
Card Reviewer 没发现问题
```

不要看到最终卡片差就只改写作 prompt。很多问题其实出在 JSON 信息不完整或字段设计不合理。

---

### 阶段 4 验收标准

```text
黄金问题集不少于 20 条
每次数据层改动后可以自动跑评测
评测报告能指出问题卡片和问题字段
核心问题的卡片召回命中率达到 80% 以上
```

---

## 阶段 5：增量更新与规模化阶段

### 目标

让数据层可以随着代码仓库变化持续更新，而不是每次全量重建。

---

### 5.1 增量更新思路

```text
代码 diff
  ↓
识别变更文件、类、函数、接口、配置
  ↓
定位受影响 domain / flow / card
  ↓
局部更新 JSON
  ↓
局部重写知识卡片
  ↓
更新索引和关系
```

---

### 5.2 变更影响分析

当代码变更时，需要判断：

```text
影响了哪些业务流程
影响了哪些业务规则
影响了哪些状态流转
影响了哪些知识卡片
影响了哪些典型问题
```

如果某次变更影响了核心流程卡片，则需要进入人工抽检。

---

### 5.3 卡片版本管理

每张卡片建议记录：

```text
card_id
version
source_repo
source_commit
generated_at
last_verified_at
changed_code_refs
quality_score
```

这样后续可以回答：

```text
这张卡片基于哪个代码版本？
最近代码改动是否影响了这张卡片？
这张卡片是否需要重新审核？
```

---

### 阶段 5 验收标准

```text
支持基于 commit diff 的局部更新
支持定位受影响卡片
支持卡片版本管理
核心卡片可以追溯到代码版本
```

---

## 5. 近期优先级建议

按照实际可行性，建议近期优先级如下：

### P0：必须先做

```text
选定一个试点业务域
跑通 code → json → card
定义最小 JSON 字段
保留 code_refs
生成 5～10 张知识卡片
用 10～20 个典型问题验证
```

### P1：第二步做

```text
建立 Canonical JSON Schema
写 Adapter 层
加 JSON 基础校验
规范业务动作表达
生成 card metadata
```

### P2：第三步做

```text
卡片分类
Card Reviewer
related_cards 自动链接
aliases 和 retrieval_keywords
黄金集评测
```

### P3：后续做

```text
增量更新
影响面分析
跨仓库业务链路补全
图谱化编译
人工审核工作台
```

---

## 6. 质量门禁设计

前期不建议质量门禁过严，否则链路跑不通。建议分阶段收紧。

### MVP 阶段

质量门禁只做提示，不强制阻断。

```text
JSON 不合法：阻断
缺少 code_refs：警告
描述空泛：警告
缺少异常流程：警告
缺少 related_cards：允许
```

### 规范阶段

开始对关键字段做阻断。

```text
JSON 不合法：阻断
flow 没有 steps：阻断
step 没有 business_action：阻断
核心 card 没有 code_refs：阻断
```

### 质量阶段

对核心业务域做更严格要求。

```text
核心流程缺异常路径：阻断或人工审核
核心业务 claim 无证据：阻断或降级为 unknown
核心卡片没有 aliases：阻断
核心卡片没有 related_cards：警告
```

---

## 7. 主要风险与应对

### 风险 1：LLM 生成的业务理解不准确

应对：

```text
区分 deterministic 和 llm_inferred
关键结论绑定代码证据
核心卡片人工抽检
不确定内容进入 unknowns
```

---

### 风险 2：JSON 太粗，知识卡片质量上不去

应对：

```text
逐步扩展 JSON Schema
增加 flow、step、entity、state、exception_path 字段
从坏卡片反推 JSON 缺失字段
```

---

### 风险 3：一开始设计过重，导致迟迟跑不通

应对：

```text
第一版只做一个业务域
第一版只生成流程卡片
第一版质量门禁只做基础检查
先生成问题清单，再逐步优化
```

---

### 风险 4：卡片语言仍然偏代码视角

应对：

```text
自定义业务语言 Skill
禁止直接复述函数调用
强制生成“一句话理解”
强制描述触发场景、业务动作、异常影响
```

---

### 风险 5：后续代码更新导致知识过期

应对：

```text
卡片绑定 source_commit
基于 diff 做受影响卡片识别
核心卡片定期重新生成和抽检
```

---

## 8. 推荐的第一轮落地节奏

### 第 1 轮：跑通

目标：

```text
完成 1 个业务域
生成 raw json
转换 canonical json
生成 5 张卡片
手工检查问题
```

重点不是质量，而是发现链路问题。

---

### 第 2 轮：规范

目标：

```text
补 JSON Schema
补 Adapter
统一字段
统一业务动作表达
给卡片加 metadata
```

重点是让数据格式稳定。

---

### 第 3 轮：提质

目标：

```text
引入 Card Reviewer
增加 aliases 和 related_cards
补异常流程
补状态流转
建立 20 条黄金问题集
```

重点是让卡片真正能支撑问答。

---

### 第 4 轮：回归

目标：

```text
每次改动自动跑评测
输出质量报告
定位问题环节
决定是改 Skill、改 Schema、改 Adapter，还是改 Agent
```

重点是形成可持续迭代闭环。

---

## 9. 最终目标形态

数据层最终应形成如下能力：

```text
能从代码仓库中抽取业务域、业务流程、业务实体和代码证据
能将原始抽取结果转换为稳定的 Canonical JSON IR
能基于 JSON IR 生成业务视角知识卡片
能通过质量检查发现空泛、缺证据、缺流程的问题
能通过黄金问题集评估数据层对问答的支撑能力
能在代码变更后局部更新相关 JSON 和知识卡片
```

最终数据层不只是问答系统的“资料来源”，而是整个智能问答助手的知识地基。

---

## 10. 一句话总结

当前阶段最重要的不是一次性做出完美知识库，而是：

```text
先用 Understand Anything 和自定义 Skill 跑通 code → json，
再用自研 Agent 跑通 json → card，
然后围绕 schema、证据、评测和回归逐步提质。
```

只有先跑通流程，才能看到真实问题；只有把问题归因到具体环节，后续优化才不会变成盲目调 prompt。
