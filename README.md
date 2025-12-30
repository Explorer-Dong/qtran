# 复现 QTRAN

原项目地址：<https://github.com/QTRANll/QTRAN>，遵循 Apache-2.0 开源协议。

## Prerequisites

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

## Main Process

QTRAN decompose the analysis into two phases: the transfer and mutation phases. It starts with SQL statement pairs from existing MOLTs and extends these pairs to new DBMSs through the two phases.

### Input

The input file for QTRAN is a JSONL file in the dictionary `Input` , where each line contains a piece of test data in JSON format. Each test data is from existing MOLTs and follows the format shown below. This test data is intended for QTRAN to translate the original SQL statements from `sqlite` (a_db) into `clickhouse` (b_db) SQL statement pairs. The corresponding MOLT is NoREC (molt).

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

### Transfer Phase

Execute the following commands to run QTRAN.The demo input file `demo1.jsonl` is in dictionary `Input`.

Navigate to the project directory:

```shell
cd <project_directory>
```

The explanations for the commands are as follows:

| option              | description                                       |
| ------------------- | ------------------------------------------------- |
| `--input_filename`  | Path to the input file (JSONL format).            |
| `--tool`            | Tool name for MOLTs(such as "sqlancer", "pinolo") |
| `--temperature`     | Temperature for LLM(default: 0.3)                 |
| `--model`           | Model to use for LLM(default: gpt-4o-mini)        |
| `--error_iteration` | Enable error iteration(default: True)             |
| `--iteration_num`   | Number of iterations(default: 4)                  |

Default parameter execution,such as:

```bash
python -m src.main --input_filename "Input/demo1.jsonl" --tool "sqlancer" --temperature 0.7 --model="gpt-3.5-turbo" --error_iteration=True --iteration_num 5
```

Custom parameter execution:

```bash
python -m src.main --input_filename "Input/demo1.jsonl" --tool "sqlancer"
```

The intermediate results of the Transfer Phase are stored in the `Output` folder, specifically in `Output/demo1/TransferLLM`. For each test case, information such as `Transfer Result`, `Transfer Cost`, and `Transfer Time` is recorded.

### Mutation Phase

The intermediate results of the Mutation Phase are stored in the `Output` folder, specifically in `Output/demo1/MutationLLM`. For each test case, information such as `Mutation Result`, `Mutation Cost`, and `Mutation Time` as well as `Oracle Check` for MOLT is recorded.

The suspicious logical bugs detected by QTRAN are stored in the `Output` folder, specifically in `Output/demo1/SuspiciousBugs`, including the final SQL statement pairs extended to new DBMSs through the two phases.
