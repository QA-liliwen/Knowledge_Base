# 03-单接口打包zip文件：pack_test_bundle

## 实现内容

将单接口测试所需的文件（用例、conftest、run_request、日志配置、依赖、pytest.ini）打包为 zip，供 Jenkins 工作区解压后直接执行。

## 代码来源

`api_app/api_test/api_jenkins_single_runner.py`

---

## 完整函数

```py
def pack_test_bundle(test_run, cases_txt_path):
    cases_filename = os.path.basename(cases_txt_path)
    # 读取测试txt文件
    with open(cases_txt_path, "r", encoding="utf-8") as f:
        cases_content = f.read()

    # 读取 conftest 文件
    with open(os.path.join(BASE_DIR, "conftest.py"), "r", encoding="utf-8") as f:
        conftest_content = f.read()

    # 读取 run_request.py 文件
    with open(os.path.join(BASE_DIR, "run_request.py"), "r", encoding="utf-8") as f:
        run_request_content = f.read()

    # 打包 zip
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, 'w', zipfile.ZIP_DEFLATED) as zf:
        zf.writestr("conftest.py", conftest_content)
        zf.writestr("run_request.py", run_request_content)
        zf.writestr("log_config.py", build_jenkins_log_config())
        zf.writestr("requirements.txt", "requests\npytest\npytest-html\n")
        zf.writestr(cases_filename, cases_content)
        # 用例文件与收集规则固化到 pytest.ini，Jenkins 端统一执行 pytest .
        # run_request.py 不符合 pytest 默认的 test_*.py 命名，需显式加入 python_files
        zf.writestr(
            "pytest.ini",
            f"[pytest]\naddopts = --cases-file={cases_filename}\npython_files = run_request.py test_*.py\n"
        )

    return buf.getvalue()
```

**关键点**：

- `cases_txt_path` 是本地临时生成的 JSON 用例文件，文件名带 run_id（如 `cases_123.json`），打包时保留原文件名。
- `conftest.py` 和 `run_request.py` 从 `api_test/` 目录读取，是单接口执行的核心文件。
- `log_config.py` 调用 `build_jenkins_log_config()` 获取改造后的内容，而非直接复制原文件。
- `requirements.txt` 硬编码写入，Jenkins 端每次 `pip install -r requirements.txt` 安装依赖。
- `pytest.ini` 通过 `addopts` 指定用例文件路径，`python_files` 声明 `run_request.py` 作为测试文件（默认 `test_*.py` 匹配不到）。
- `io.BytesIO()` 在内存中创建 zip，不写磁盘临时文件，返回 bytes 直接传给 `trigger_jenkins_build`。

---

## zip 包内文件清单

| 文件名 | 来源 | 作用 |
| --- | --- | --- |
| conftest.py | api_test/conftest.py | pytest 钩子，执行完回调平台 |
| run_request.py | api_test/run_request.py | pytest 入口，发送请求、执行断言 |
| log_config.py | build_jenkins_log_config() | 改造后的日志配置 |
| requirements.txt | 硬编码 | 依赖列表 |
| cases_{run_id}.json | cases_txt_path | 测试用例数据 |
| pytest.ini | 动态生成 | pytest 收集规则 |
