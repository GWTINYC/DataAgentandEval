问：目前我们在数据层打算这样做：阶段1 code to json 的工作采用一个开源项目Understand Anything中的一个SKILL：understand domain，同时自己写一些规范json内部语言风格的SKILL，来生成json文件。然后阶段2 json to 知识卡片的工作，采用自己编写的一个Agent来实现。从整体架构到细节，给出一些建议。

答：
## 总判断

你们这个方案**方向是对的**，但需要加一个关键原则：

> **Understand Anything / understand-domain 只能作为“业务语义提取器”，不能直接作为你们数据层的唯一事实源。**

原因很简单：它擅长把代码映射成 domain / flow / step 这类业务图，官方说明里 `/understand-domain` 就是用 `domain-analyzer` 抽取 business domains、flows、process steps；Understand Anything 本身也采用 Tree-sitter 做确定性结构解析、LLM 做语义总结的混合方案。这个思路适合你们，但生产级内部知识库必须额外加“结构校验、证据绑定、schema 约束、人工抽检、版本回归”。([GitHub][1])

---

# 1. 推荐的整体架构

你们现在是：

```text
阶段1：code -> json
阶段2：json -> 知识卡片
```

建议改成更稳的 5 步：

```text
代码仓库
  ↓
A. 确定性代码抽取层
  ↓
B. Understand-domain 业务语义抽取层
  ↓
C. JSON IR 规范化与质检层
  ↓
D. 知识卡片生成 Agent
  ↓
E. 卡片质检、链接、索引层
```

核心区别在于：**不要直接让 LLM 从代码生成最终 JSON，而是把 JSON 当成“受控中间表示 IR”。**

---

# 2. 阶段1：code to json 的建议

## 2.1 不要只依赖 understand-domain

Understand Anything 的优势是“快速看懂大图”，官方描述里它会构建文件、函数、类、依赖的知识图谱，也支持 domain view，把代码映射为业务流程。([GitHub][2])
但你们的目标不是“可视化理解代码”，而是“长期服务业务问答”。所以它的输出应该是**候选语义**，不是最终真相。

建议拆成两个来源：

```text
来源1：确定性抽取
- 文件
- 类
- 函数
- import
- call
- API 路由
- DB 表访问
- MQ topic
- 配置项
- 枚举状态
- 异常类型

来源2：LLM / understand-domain 抽取
- 业务域
- 业务流程
- 业务动作
- 流程步骤
- 异常场景
- 业务角色
- 业务含义
```

确定性部分可以基于 AST / Tree-sitter / grep / 框架路由解析。Tree-sitter 本身就是增量解析库，可以为源码构建 concrete syntax tree，适合做代码结构抽取。([Tree-sitter][3])

一句话：**结构事实靠程序，业务语义靠 LLM。**

---

## 2.2 JSON 不要设计成“描述文档”，要设计成“业务语义 IR”

推荐你们的 JSON 分成 4 类对象：

```text
1. code_objects：代码对象
2. domain_objects：业务对象
3. flow_objects：业务流程
4. evidence_objects：代码证据
```

一个比较合理的结构如下：

```json
{
  "repo": "xxx-order-service",
  "commit": "a1b2c3d",
  "generated_at": "2026-06-11T10:00:00+08:00",
  "domain": {
    "domain_id": "order_cancel",
    "domain_name": "订单取消",
    "business_summary": "描述这个业务域解决什么业务问题",
    "confidence": 0.86,
    "source": "llm_inferred"
  },
  "flows": [
    {
      "flow_id": "order_cancel_refund_flow",
      "flow_name": "订单取消后的退款处理流程",
      "business_goal": "在订单取消后判断是否需要退款，并完成退款状态闭环",
      "trigger": "用户取消订单或系统自动取消订单",
      "steps": [
        {
          "step_id": "check_cancelable",
          "step_name": "校验订单是否允许取消",
          "business_action": "判断订单当前状态是否还能取消",
          "input_entities": ["订单"],
          "output_entities": ["取消校验结果"],
          "code_refs": [
            {
              "file": "src/main/java/xxx/OrderCancelService.java",
              "symbol": "cancelOrder",
              "line_start": 72,
              "line_end": 105
            }
          ],
          "confidence": 0.91
        }
      ],
      "main_path": ["check_cancelable", "update_order_status", "create_refund", "wait_payment_callback"],
      "exception_paths": [
        {
          "condition": "支付渠道退款失败",
          "steps": ["mark_refund_failed", "enter_retry_job"]
        }
      ]
    }
  ],
  "entities": [
    {
      "entity_id": "refund_order",
      "entity_name": "退款单",
      "entity_type": "business_entity",
      "key_fields": ["refund_id", "order_id", "refund_status"],
      "status_machine": {
        "states": ["INIT", "PROCESSING", "SUCCESS", "FAILED"],
        "transitions": [
          {
            "from": "PROCESSING",
            "to": "SUCCESS",
            "trigger": "收到支付渠道退款成功回调",
            "code_refs": ["PaymentCallbackHandler.handleRefundCallback"]
          }
        ]
      }
    }
  ],
  "relations": [
    {
      "from": "order_cancel_refund_flow",
      "to": "refund_order",
      "type": "creates",
      "evidence": ["RefundService.createRefund"],
      "confidence": 0.88
    }
  ]
}
```

重点字段不是 `description`，而是：

```text
business_goal
trigger
business_action
main_path
exception_paths
state_machine
code_refs
confidence
source
```

这些字段后面才能支撑知识卡片、图谱和问答。

---

## 2.3 必须给 JSON 上 Schema

你们自己写规范 JSON 语言风格的 Skill 是有必要的，但还不够。还需要一个机器可校验的 JSON Schema。

JSON Schema 的价值是保证 JSON 数据的一致性、有效性和互操作性；官方介绍里也明确把它定位为用于大规模 JSON 数据一致性和校验的词汇体系。([JSON Schema][4])

建议强制校验这些东西：

```text
必填字段是否存在
字段类型是否正确
step_id 是否唯一
relation.from / relation.to 是否能找到对象
code_refs 是否绑定真实文件
confidence 是否在 0~1
source 是否只能是 deterministic / llm_inferred / human_verified
```

尤其要防止这种 JSON：

```json
{
  "business_summary": "这个模块主要负责订单相关逻辑"
}
```

这种东西看起来像知识，但没有结构、没有证据、没有可评测性。

---

## 2.4 给 JSON 字段加“事实等级”

建议每条信息都打上来源：

```text
deterministic：程序静态分析确定
llm_inferred：LLM 根据代码推断
human_verified：人工确认
business_doc：来自已有业务文档
runtime_log：来自日志 / 链路追踪
```

例如：

```json
{
  "business_action": "创建退款单",
  "source": "llm_inferred",
  "evidence": ["RefundService.createRefund"],
  "confidence": 0.84
}
```

最终问答时，优先级应该是：

```text
human_verified > deterministic > business_doc > runtime_log > llm_inferred
```

LLM 推断不是不能用，而是不能和确定性事实混在一起。

---

## 2.5 你们自定义 Skill 的重点不是“润色”，而是“约束”

你们说要写一些规范 JSON 内部语言风格的 Skill，这个思路可以，但不要只写成“语言更业务化”。建议 Skill 分成 4 类。

### Skill 1：业务命名规范

把代码名翻译成业务名：

```text
createRefund -> 创建退款单
handleCallback -> 处理支付渠道回调
checkCancelable -> 校验是否允许取消
retryRefund -> 退款失败补偿
```

要求：

```text
函数名不能直接进入 business_action
禁止出现“调用 xxx 方法”
必须用“校验 / 创建 / 更新 / 通知 / 触发 / 补偿 / 拦截 / 回滚”等业务动词
```

---

### Skill 2：流程抽取规范

要求每个 flow 至少有：

```text
触发条件
主流程
异常流程
结束状态
涉及实体
关键代码证据
```

如果缺任意一个，JSON 标记为：

```json
"quality_flags": ["missing_exception_path", "missing_end_state"]
```

---

### Skill 3：证据绑定规范

每个关键业务 claim 必须绑定代码证据。

不要允许：

```json
{
  "claim": "系统会进行幂等处理"
}
```

应该是：

```json
{
  "claim": "系统会基于 refund_id 对支付回调做幂等处理",
  "evidence": [
    {
      "file": "PaymentCallbackHandler.java",
      "symbol": "handleRefundCallback",
      "line_start": 45,
      "line_end": 68
    }
  ]
}
```

---

### Skill 4：不确定性暴露规范

如果模型不确定，不要强行编。

建议输出：

```json
{
  "unknowns": [
    {
      "question": "退款失败后的人工处理入口未在当前代码中找到",
      "possible_reason": "可能在另一个仓库或运营后台系统中",
      "needed_sources": ["ops-admin-service", "refund-manual-doc"]
    }
  ]
}
```

这对后续问答非常重要。业务同学问到证据不足的问题时，助手可以诚实回答：**当前代码仓只能证明到这里，后续人工处理可能在另一个系统。**

---

# 3. 阶段2：json to 知识卡片 Agent 的建议

## 3.1 不要一个 Agent 一步生成完，建议拆成 5 个子步骤

你们的 Agent 应该这样跑：

```text
1. JSON 选择器
   从全量 JSON 中选出一个业务域 / 流程 / 实体相关片段

2. 上下文补全器
   补齐相关实体、状态、异常流程、上下游关系

3. 卡片生成器
   生成业务视角 Markdown

4. 卡片审查器
   检查准确性、完整性、证据绑定、语言风格

5. 链接生成器
   生成 related_cards、aliases、retrieval_keywords
```

不要让一个 prompt 同时做“找信息、理解信息、写卡片、审查卡片、建链接”。这会很不稳定。

---

## 3.2 知识卡片建议分类型

不要所有卡片都长一个样。建议至少分 6 类。

| 卡片类型   | 用途                         |
| ------ | -------------------------- |
| 业务域总览卡 | 解释一个业务域解决什么问题              |
| 业务流程卡  | 解释一个完整流程怎么跑                |
| 状态机卡   | 解释实体有哪些状态、如何流转             |
| 业务规则卡  | 解释关键 if/else 背后的业务约束       |
| 异常补偿卡  | 解释失败、重试、幂等、人工处理            |
| 入口接口卡  | 解释某个 API / MQ / Job 触发什么业务 |

例如“订单取消”不要只生成一张大卡片，应该拆成：

```text
订单取消业务域总览
订单取消主流程
订单取消后的退款流程
订单状态机
退款状态机
取消失败与异常补偿
订单取消接口入口
```

这样后续召回更准。

---

## 3.3 推荐的知识卡片模板

```markdown
# 订单取消后的退款处理流程

## 这张卡片解决什么问题
说明订单取消后，系统如何判断是否需要退款，以及退款状态如何完成闭环。

## 适用场景
- 用户主动取消已支付订单
- 系统自动取消已支付订单
- 商家超时未接单导致取消

## 一句话理解
订单取消不是简单把订单状态改掉；如果订单已经支付，还需要创建退款单、调用支付渠道退款，并等待支付渠道回调确认最终结果。

## 主流程
1. 校验订单是否允许取消
2. 更新订单状态为已取消
3. 判断订单是否已支付
4. 如果已支付，创建退款单
5. 调用支付渠道发起退款
6. 等待支付渠道回调
7. 根据回调结果更新退款状态

## 异常流程
- 如果支付渠道退款失败，退款单会进入失败状态
- 如果存在补偿任务，系统会定时重试
- 如果重复收到回调，需要通过退款单号进行幂等处理

## 关键业务实体
- 订单
- 退款单
- 支付单

## 关键状态
- order_status: PAID / CANCELLED
- refund_status: INIT / PROCESSING / SUCCESS / FAILED

## 代码证据
- OrderCancelService.cancelOrder
- RefundService.createRefund
- PaymentCallbackHandler.handleRefundCallback
- RefundRetryJob.execute

## 关联卡片
- 订单状态机
- 退款状态机
- 支付渠道回调
- 退款失败补偿
```

注意这张卡片是**业务视角**，但它仍然有代码证据。
好的知识卡片不是“脱离代码的业务文档”，而是“有代码证据支撑的业务解释”。

---

## 3.4 Agent 输出卡片时要同时输出 metadata

不要只输出 `.md`，还要输出旁路 metadata，方便后续召回。

例如：

```json
{
  "card_id": "order_cancel_refund_flow",
  "card_type": "flow_card",
  "title": "订单取消后的退款处理流程",
  "domain": "订单",
  "entities": ["订单", "退款单", "支付单"],
  "aliases": ["取消订单退款", "订单取消后钱什么时候退", "退款链路"],
  "related_cards": [
    "order_status_machine",
    "refund_status_machine",
    "payment_callback"
  ],
  "code_refs": [
    "OrderCancelService.cancelOrder",
    "RefundService.createRefund"
  ],
  "retrieval_keywords": [
    "订单取消",
    "退款",
    "支付回调",
    "退款失败",
    "幂等"
  ]
}
```

这个 metadata 对后续生成层非常关键。
业务同学不会问“RefundService.createRefund 是什么”，他会问：

```text
用户取消订单后钱是怎么退的？
为什么订单取消了退款还没到账？
取消订单会不会自动退优惠券？
```

所以 aliases 和 retrieval_keywords 必须用业务说法。

---

# 4. 阶段1 和阶段2 之间要加质量门禁

建议你们在 `json -> card` 之前加一个 `JSON Quality Gate`。

## 4.1 Schema Gate

检查 JSON 是否合规：

```text
字段完整性
类型正确性
ID 唯一性
引用完整性
枚举值合法性
```

---

## 4.2 Evidence Gate

检查关键业务字段有没有证据：

```text
business_action 是否有 code_refs
flow step 是否有 code_refs
state transition 是否有 code_refs
exception path 是否有 code_refs
relation 是否有 evidence
```

没有证据的内容不能直接进入正式卡片，只能进入：

```text
推测区
待确认区
unknowns
```

---

## 4.3 Consistency Gate

检查是否自相矛盾：

```text
同一个状态是否被解释成不同含义
同一个接口是否被归到多个冲突业务域
同一个 flow 是否有多个互相冲突的结束状态
```

---

## 4.4 Business Language Gate

检查语言是否像业务表达：

不合格表达：

```text
调用 RefundService.createRefund 方法，然后调用 PaymentClient.refund 方法。
```

合格表达：

```text
系统会先创建退款单，再向支付渠道发起退款请求；退款是否真正完成，还要等支付渠道回调确认。
```

---

# 5. 对 Understand Anything 的具体使用建议

## 5.1 建议使用方式

你们可以这样用：

```text
/understand
先生成结构图，拿到文件、函数、类、依赖关系

/understand-domain
再生成业务域、流程、步骤

然后将两份结果合并成自己的 Canonical JSON IR
```

Understand Anything 的 README 里说明 `/understand` 会扫描项目并把知识图保存到 `.understand-anything/knowledge-graph.json`，`/understand-domain` 用于抽取 domains、flows、steps；它也支持增量更新，只重新分析变更文件。([GitHub][2])

但不要把 `.understand-anything/knowledge-graph.json` 原样作为你们的核心数据格式。建议做一层转换：

```text
Understand Anything 原始输出
  ↓
adapter
  ↓
你们自己的 domain_ir.json
```

原因是：

```text
开源项目 schema 可能变化
字段命名不一定贴合你们业务
质量标准不一定满足你们问答需求
你们后续要做评测和回归，必须有稳定 IR
```

---

## 5.2 不要魔改 Understand Anything 主流程，优先写 Adapter

更推荐：

```text
保留 Understand Anything 原始输出
自己写 adapter.py / adapter.ts
把它转换成你们内部标准 JSON
```

不要一上来大改开源项目内部 agent，否则后续升级困难。

推荐目录：

```text
data_pipeline/
  raw/
    understand_anything/
      knowledge-graph.json
      domain-graph.json

  ir/
    code_objects.json
    domain_flows.json
    business_entities.json
    evidence_index.json

  cards/
    order/
      order_cancel_flow.md
      refund_status_machine.md

  metadata/
    card_index.json
    relation_index.json
    eval_report.json
```

---

## 5.3 用 understand-domain 产出“候选业务域”，再做二次归并

LLM 很容易把业务域切碎。例如：

```text
订单取消
取消订单
订单关闭
订单终止
```

这几个可能其实是一个业务域。

所以要加一个归并步骤：

```text
domain candidate extraction
  ↓
domain normalization
  ↓
domain merge
  ↓
domain ontology
```

可以维护一个业务词表：

```json
{
  "canonical_terms": {
    "订单取消": ["取消订单", "订单关闭", "订单终止"],
    "退款": ["退钱", "退款处理", "支付退款"],
    "支付回调": ["渠道回调", "支付通知", "异步通知"]
  }
}
```

这个词表要持续从业务同学问题里反哺。

---

# 6. 数据层评测建议

你们的数据层评测不要只看“生成得像不像”，而要看这 6 个指标。

## 6.1 JSON Schema 通过率

```text
schema_pass_rate = 合法 JSON 数 / 总 JSON 数
```

这个指标应该接近 100%。

---

## 6.2 代码对象覆盖率

```text
method_coverage = JSON 中出现的核心方法数 / 仓库核心方法总数
api_coverage = JSON 中出现的 API 数 / 实际 API 数
table_coverage = JSON 中出现的表数 / 实际访问表数
mq_coverage = JSON 中出现的 topic 数 / 实际 topic 数
```

---

## 6.3 业务流程覆盖率

从黄金问题反推：

```text
某个问题需要 5 个流程步骤
JSON 命中了 4 个
flow_step_recall = 4 / 5
```

---

## 6.4 证据绑定率

```text
evidence_rate = 带 code_refs 的业务 claim 数 / 总业务 claim 数
```

这个指标非常关键。低于 80%，后面问答很容易幻觉。

---

## 6.5 关系正确率

抽样人工判断：

```text
A triggers B 是否成立？
A creates B 是否成立？
A updates B 是否成立？
A depends_on B 是否成立？
```

---

## 6.6 稳定性

同一 commit 多跑几次，输出应该基本一致。

```text
same_commit_stability = 两次 JSON 的结构相似度
```

如果同一份代码每次抽出来的业务域都不一样，后续知识库会很难维护。

---

# 7. json to 卡片 Agent 的评测建议

卡片评测要看 5 件事。

| 指标    | 判断方式              |
| ----- | ----------------- |
| 准确性   | 是否被 JSON 和代码证据支持  |
| 完整性   | 是否包含主流程、异常流、边界条件  |
| 业务化程度 | 是否业务同学能看懂         |
| 可召回性  | 标题、别名、关键词是否贴近业务问法 |
| 可链接性  | 是否正确链接到相关卡片       |

RAG 评测里常用 Context Precision、Context Recall、Faithfulness、Response Relevancy 等指标；Ragas 文档里也把这些指标分别用于评估召回上下文排序、是否漏掉相关信息、回答是否忠实于上下文。([Ragas][5])

对应到你们这里：

```text
Context Recall -> 是否召回正确卡片
Faithfulness -> 回答是否被卡片和代码证据支持
Response Relevancy -> 回答是否真正回答业务问题
```

---

# 8. 一个推荐的 Agent 设计

## 8.1 Stage 1 Agent：JSON 生成 Agent

不要叫它“总结 Agent”，建议叫：

```text
Domain IR Builder
```

它内部可以有 4 个节点：

```text
1. Structure Extractor
   确定性抽取代码对象

2. Domain Candidate Extractor
   使用 understand-domain 抽业务域、流程、步骤

3. IR Normalizer
   套你们自己的 schema、业务词表、命名规范

4. IR Reviewer
   检查证据、引用、重复、冲突、不确定性
```

---

## 8.2 Stage 2 Agent：知识卡片 Agent

```text
1. Card Planner
   判断应该生成哪些卡片

2. Context Collector
   收集该卡片需要的 JSON 子图

3. Card Writer
   写 Markdown

4. Card Critic
   检查遗漏、幻觉、语言风格

5. Card Linker
   生成 related_cards、aliases、retrieval_keywords
```

---

# 9. 关键细节建议

## 9.1 每张卡片只讲一个中心问题

不要写成：

```text
订单模块完整介绍
```

应该写成：

```text
订单取消后的退款处理流程
订单状态为什么不能从已发货回到待支付
支付回调为什么需要幂等处理
优惠券在订单取消后如何退回
```

卡片越贴近真实问题，后续召回越准。

---

## 9.2 卡片标题要像业务同学会问的问题

差标题：

```text
RefundService 说明
```

好标题：

```text
订单取消后，退款是怎么发起并确认到账的？
```

差标题：

```text
PaymentCallbackHandler 说明
```

好标题：

```text
支付渠道回调如何影响订单和退款状态？
```

---

## 9.3 卡片必须区分“代码能证明的”和“推测的”

卡片里建议保留一个区域：

```markdown
## 当前代码能确认的结论
...

## 当前代码不能完全确认的部分
...
```

这对企业内部知识库很重要。很多业务链路跨多个仓库，一个仓库看不到全貌，不能强行补全。

---

## 9.4 对跨仓库业务链路要做占位节点

例如订单取消链路可能涉及：

```text
order-service
payment-service
coupon-service
after-sale-service
message-service
```

当前仓库只看到 `order-service`，那 JSON 里不要丢掉外部依赖，而是建立占位节点：

```json
{
  "external_system": "payment-service",
  "relation": "发起退款请求",
  "known": true,
  "internal_detail_available": false
}
```

这样后续接入更多仓库时可以自动补全。

---

# 10. 最推荐你们现在立刻做的 6 件事

## 1. 先定义你们自己的 Canonical JSON Schema

不要让开源项目输出格式决定你们的数据标准。

---

## 2. Understand Anything 输出只作为 raw input 保存

保留：

```text
raw_understand_graph.json
raw_domain_graph.json
```

然后转换成：

```text
domain_ir.json
```

---

## 3. 每个业务 claim 必须有 evidence

没有 evidence 的内容，要么降级为 unknown，要么标记为 inferred。

---

## 4. 做 20 个标杆业务问题

每个问题标注：

```text
必须命中的 flow
必须命中的 step
必须命中的 entity
必须命中的 code_ref
理想回答
禁止错误说法
```

用它们来回归测试整条 pipeline。

---

## 5. 知识卡片先做小而精

先不要全仓库铺开。建议先选：

```text
1 个核心业务域
3 条核心流程
10 张高质量卡片
20 个黄金问题
```

把闭环跑通，再扩大规模。

---

## 6. 把“业务语言风格”做成可检查规则

比如强制：

```text
少说：方法、类、调用、参数、返回值
多说：用户、订单、支付、退款、状态、流程、规则、异常、补偿
```

这不是简单润色，而是**表达视角转换**。

---

# 最终建议

你们当前方案可以落地，但建议调整定位：

```text
Understand Anything / understand-domain
= 候选业务语义抽取器

自定义 JSON Skill
= 业务语义 IR 规范化器

json to card Agent
= 业务知识卡片编译器

质量门禁 + eval
= 生产可用性的核心保障
```

最关键的一点：

> **不要追求一次性生成“看起来很完整”的知识卡片，要追求每条业务结论都能追溯、能评测、能增量更新。**

这样后面生成层才能真正做到：不是照搬代码逻辑，而是用业务语言解释代码背后的业务机制。

[1]: https://github.com/Lum1104/Understand-Anything/releases "Releases · Egonex-AI/Understand-Anything · GitHub"
[2]: https://github.com/Lum1104/Understand-Anything "GitHub - Egonex-AI/Understand-Anything: Graphs that teach > graphs that impress. Turn any code into an interactive knowledge graph you can explore, search, and ask questions about. Works with Claude Code, Codex, Cursor, Copilot, Gemini CLI, and more. · GitHub"
[3]: https://tree-sitter.github.io/?utm_source=chatgpt.com "Tree-sitter: Introduction"
[4]: https://json-schema.org/?utm_source=chatgpt.com "JSON Schema"
[5]: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/?utm_source=chatgpt.com "Context Precision"
