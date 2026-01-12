# 复现 QTRAN

尝试复现 QTRAN，原项目地址：<https://github.com/QTRANll/QTRAN>，遵循 Apache-2.0 开源协议。

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
