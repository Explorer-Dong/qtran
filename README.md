# 复现 QTRAN

## Introduction

需求：需要给很多数据库生成 SQL 语句，来检测其中可能存在的逻辑漏洞。

目前最先进的方法：基于少量给定的 SQL，进行大量变异得到数量充足的 SQL 作为测试集。

> 引自 QTRAN：
>
> ![基于变异的数据库测试样例生成方法是目前的 SOTA 方法](https://cdn.dwj601.cn/images/20260106231921709.png)

目前方法的问题：变异 SQL 的逻辑需要手搓很多代码。

> 引自 QTRAN：各种变异工具为了兼容不同的数据库所新增的代码行数：
>
> ![各种变异工具为了兼容不同的数据库所新增的代码行数](https://cdn.dwj601.cn/images/20260106232132282.png)

树立靶子：市面上 [几百种数据库](https://db-engines.com/en/ranking)，每一种都写这么多代码来兼容显然不现实，所以很有必要设计一种人工量低的方法来构造逻辑漏洞检测用的测试集。

QTRAN 的思路：直接用 LLM 生成逻辑漏洞检测用的 SQL 就好了。

> 引自 QTRAN：用 LLM 生成 SQL 的一个有效方案是，让 LLM 将其他数据库的 SQL 翻译为目标数据库的 SQL：
>
> ![用 LLM 生成 SQL 的一个有效方案是，让 LLM 将其他数据库的 SQL 翻译为目标数据库的 SQL](https://cdn.dwj601.cn/images/20260106234627028.png)
>
> **注意**：这里引用的论文其中的 SQL 转换并没有跨数据库，所以解释的很牵强。

特化靶子：用 LLM 翻译 SQL。

明确靶子：

1. LLM 对很多数据库的语言特性不熟悉，导致直接翻译会出现很多语法和语义错误；

    > 引自 QTRAN：
    >
    > ![LLM 对很多数据库的语言特性不熟悉，导致直接翻译会出现很多语法和语义错误](https://cdn.dwj601.cn/images/20260106235101323.png)

    解决方案：用 RAG 来扩充 LLM 的知识。

2. 直接翻译变异后的 SQL 或者直接变异原始 SQL 对是不准确的，会丢失很多关键信息（**注意**：这个理由我感觉提出的很牵强，这难道不是语义错误？）

    > 引自 QTRAN：
    >
    > ![image-20260107000818202](https://cdn.dwj601.cn/images/20260107000820348.png)
    >
    > 分析：
    >
    > - QTRAN-POM：让 LLM 变异查询。得到的有效变异对（语法正确、语义正确、变异逻辑正确）比例较低；
    > - QTRAN-DTM：让 LLM 翻译原始变异对。得到的有效变异对比例更低；
    > - QTRAN：用传统方法得到的变异对（源数据库）微调 LLM，然后再让 LLM 变异（目标数据库）种子 SQL。

    解决方法：指令微调 LLM，提升 LLM 变异的能力。**注意**：这里 QTRAN 得到的变异对几乎全部正确了，那我怎么打？

## Limitation & Innovation

论文提到的局限性主要是：

- 转换阶段的正确率不高；
- LLM 响应费时间。

其中 LLM 响应费时间基本是废话，调 API 都这样，就不说了。可以从「提升转换正确率」的角度来思考创新点，也就是转换阶段。这似乎是 RAG 工程方面的事情，想办法 **多凑点 Prompt 或者上点 RAG 工程** 应该就能刷转换阶段的 SOTA 了。

另外微调阶段虽然 QTRAN 几乎全对，但是其指标是基于正确转换的 SQL 进行的，但按道理转换和变异是串行的，不能抛开转换只看变异，所以变异阶段的准确率应当考虑转换阶段的失败率，那么就可以把 QTRAN 变异阶段的指标大幅降低了。那么如何刷变异阶段的 SOTA 呢？我的想法是：直接 **拿开源的小点的 LLM 搞参数微调**，多爬点数据应该能调起来，就是不知道能不能打过指令微调。

## 环境配置

### 配置 Python 🤨

原论文使用的是 Python 3.11，并且使用 pip 管理，考虑转移到 uv。

### 配置数据库 🤨

原论文使用了 8 个数据库，并且使用 docker compose 管理，在 [database_connector_args.json](src/Tools/DatabaseConnect/database_connector_args.json) 文件中配置。

具体参考 [Config Databases in Docker](./docs/config-db-in-docker.md) 中的内容。

### 配置 LLM API Key 🤨

原论文直接 `SET OPENAI_API_KEY=${your_api_key}`，可以考虑使用 `.env` 文件并修改源码。

### 指令微调变异阶段的 LLM 🤨

原论文在变异阶段指令微调了 GPT-4o mini，微调数据集基于 4 种 MOLT 方法 (NoREC, TLP, PINOLO, DQE) 构建。

具体参考 [Fine-tune Mutation-LLM](./docs/fine-tune-mutation-llm.md) 中的内容。

注意，原论文需要设置微调后的 LLM ID：

```bash
SET NOREC_MUTATION_LLM_ID = ${your_norec_mutation_llm_id}
SET TLP_MUTATION_LLM_ID = ${your_tlp_mutation_llm_id}
SET PINOLO_MUTATION_LLM_ID = ${your_pinolo_mutation_llm_id}
SET DQE_MUTATION_LLM_ID = ${your_dqe_mutation_llm_id}
```

## 运行方法

### 输入数据格式

QTRAN 的输入数据是已经使用 MOLT 转换后的 SQL 语句。

下面的示例表示从 a_db 转换到 b_db，MOLT 方法是 NoREC：

```json
{
  "index": 0,
  "a_db": "sqlite",
  "b_db": "clickhouse",
  "molt": "norec",
  "sqls": [
    "CREATE TABLE t0(c0 INT UNIQUE);",
    "INSERT INTO t0(c0) VALUES (1);",
    "SELECT COUNT(*) FROM t0 WHERE '1' IN (t0.c0); -- unexpected: fetches row"
  ]
}
```

### 运行命令

```bash
python -m src.main \
  --input_filename "Input/demo1.jsonl" \
  --tool "sqlancer" \
  --temperature 0.7 \
  --model="gpt-3.5-turbo" \
  --error_iteration=True \
  --iteration_num 5
```

转换和变异的结果、开销均记录在 `Output` 文件夹中。

## 引用参考

原项目地址：<https://github.com/QTRANll/QTRAN>，遵循 Apache-2.0 开源协议。
