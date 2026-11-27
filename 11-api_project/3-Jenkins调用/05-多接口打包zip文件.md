# 05-多接口打包zip文件：pack_custom_bundle

## 实现内容

将多接口测试项的脚本文件（多个 test_flow_*.py）与公共文件（conftest、日志配置、依赖）打包为 zip，供 Jenkins 工作区解压后执行。

## 代码来源

`api_app/api_test/api_jenkins_custom_runner.py`

---

## 完整函数

```py
def pack_custom_bundle(test_item_ids):
    # 读取 conftest 文件（跑完后回调平台）
    with open(os.path.join(BASE_DIR, "conftest.py"), "r", encoding="utf-8") as f:
        conftest_content = f.read()

    # 逐个读取测试项对应的脚本内容
    scripts = {}
    for item_id in test_item_ids:
        test_item = DB_TestItem.objects.get(id=item_id)
        if not test_item.script_filename:
            raise ValueError(f"测试项 {item_id} 未上传脚本文件")
        src_path = os.path.join(SCRIPTS_DIR, test_item.script_filename)
        if not os.path.exists(src_path):
            raise FileNotFoundError(f"脚本文件不存在: {test_item.script_filename}")
        with open(src_path, "r", encoding="utf-8") as f:
            scripts[f"test_flow_{item_id}.py"] = f.read()

    # 打包 zip
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, 'w', zipfile.ZIP_DEFLATED) as zf:
        zf.writestr("conftest.py", conftest_content)
        zf.writestr("log_config.py", build_jenkins_log_config())
        zf.writestr("requirements.txt", "requests\npytest\npytest-html\n")
        for name, content in scripts.items():
            zf.writestr(name, content)

    return buf.getvalue()
```

**关键点**：

- 与单接口打包相比，不需要 `run_request.py` 和 `pytest.ini`，因为多接口脚本是标准 `test_*.py` 命名，pytest 自动扫描。
- 每个测试项脚本从 `data/scripts/{script_filename}` 读取，重命名为 `test_flow_{item_id}.py` 写入 zip。
- 校验：脚本文件名不为空、源文件存在，任一失败抛异常，由调用方捕获。
- `scripts` 字典收集所有脚本内容后统一写入 zip，避免多次打开 zip 对象。

---

## 与单接口打包的差异

| 对比项 | 单接口 pack_test_bundle | 多接口 pack_custom_bundle |
| --- | --- | --- |
| 测试文件 | run_request.py + cases.json | 多个 test_flow_*.py |
| pytest.ini | 需要（指定用例文件） | 不需要（pytest 自动扫描） |
| 脚本来源 | api_test 目录内 | data/scripts 目录 |
| 参数 | test_run, cases_txt_path | test_item_ids |
