# pytest 执行入口

pytest 通过 `--cases-file` 参数接收用例文件，动态参数化生成多个 `test_api` 调用，每个用例执行 `run_main` 发请求并校验结果。

---

## 流程

```
pytest 启动
  → pytest_addoption 注册 --cases-file 参数
  → pytest_generate_tests 读取 cases 文件
  → metafunc.parametrize 生成 N 个 test_api
  → test_api(case) 调用 run_main(case)
  → run_main 发请求 → 状态码断言 → 【断言处理见下一节】
```

---

## 注册自定义参数

来源：`api_app/api_test/run_request.py`

```py
import pytest


def pytest_addoption(parser):
    parser.addoption(
        "--cases-file",
        action="store",
        default=None,
        help="测试用例文件路径"
    )
```

### 特殊处理说明

| 代码 | 特殊处理原因 |
|------|-------------|
| `pytest_addoption` | pytest 钩子，用于注册自定义命令行参数 |
| `action="store"` | 标准存值方式 |
| `default=None` | 未传参数时默认 None，方便后续判断 |

---

## 动态生成测试用例

来源：`api_app/api_test/run_request.py`

```py
import json
import os


def pytest_generate_tests(metafunc):
    cases_file = metafunc.config.getoption("--cases-file") or os.environ.get("CASES_FILE")
    if not cases_file:
        return

    with open(cases_file, "r", encoding="utf-8") as f:
        cases_data = json.load(f)

    metafunc.parametrize(
        "case",
        cases_data,
        ids=[c['CaseID'] for c in cases_data]
    )


def test_api(case):
    run_main(case)
```

### 特殊处理说明

| 代码 | 特殊处理原因 |
|------|-------------|
| `metafunc.config.getoption("--cases-file")` | 优先从命令行参数取 cases 文件路径 |
| `or os.environ.get("CASES_FILE")` | Jenkins 等场景可通过环境变量传路径 |
| `if not cases_file: return` | 没传路径时不生成用例，避免 pytest 报错 |
| `encoding="utf-8"` | 确保中文用例正常读取 |
| `metafunc.parametrize` | 把 cases 列表拆成多个独立测试 |
| `ids=[c['CaseID']]` | 用 CaseID 命名每个测试，报告和日志可读 |
| `test_api(case)` | 统一的测试入口函数 |
| `run_main(case)` | 实际执行请求和断言的入口 |

---

## 用例执行主函数（断言逻辑省略）

来源：`api_app/api_test/run_request.py`

```py
import requests
from log_config import get_logger

log = get_logger(__name__)


def run_main(case):
    if case.get('is_active') != 'Y':
        log.debug(f"用例 {case['CaseID']} 未启用")
        pytest.skip(f"用例 {case['CaseID']} 未启用")

    resp = run_request(
        url=case['url'],
        method=case['method'],
        headers=case['headers'],
        body=case.get('body', '')
    )

    log.info(f"状态码断言: 实际={resp.status_code}, 期望={case['expected_status_code']}")
    assert resp.status_code == case['expected_status_code'], \
        f"状态码: 实际={resp.status_code}, 期望={case['expected_status_code']}"

    log.info(f"数据断言: {case['assertions']}")
    log.info(f"响应体: {resp.text[:1000]}")
    resp_json = resp.json()

    # 【此处省略断言解析与执行逻辑，见下一节】


# 执行层
import requests


def run_request(url, method, headers, body=None):
    log.info(f"准备发送请求: {method.upper()} {url}")
    log.debug(f"请求头: {headers}")
    log.debug(f"请求体: {body}")

    if method == 'GET':
        if body:
            url = url + '?' + str(body)
        return requests.get(url, headers=headers)
    elif method == 'POST':
        return requests.post(url, headers=headers, json=body)
    else:
        return requests.request(method, url, headers=headers, json=body)
```

### 特殊处理说明

| 代码 | 特殊处理原因 |
|------|-------------|
| `case.get('is_active') != 'Y'` | Excel 中 Y 表示执行，其他值跳过 |
| `pytest.skip(...)` | 未启用用例标记为 skipped，不计入失败 |
| `run_request(...)` | 封装 HTTP 请求，统一处理 GET/POST/其他方法 |
| `resp.status_code == case['expected_status_code']` | 先断言状态码，失败直接抛异常 |
| `resp.text[:1000]` | 日志只打印响应前 1000 字符，避免日志过大 |
| `resp.json()` | 后续断言需要解析 JSON 响应体 |
| `method.upper()` | 统一大写后判断请求方法 |
| GET 带 body 时转 query string | `url + '?' + str(body)` |
| POST 用 `json=body` | 自动设置 Content-Type: application/json |
