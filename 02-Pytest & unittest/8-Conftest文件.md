conftest.py 是 pytest 的局部插件文件（名字固定，按目录生效），主要承载三类内容：
1. hook 函数（pytest 生命周期回调）、
2. 共享 fixture（同目录及子目录自动可用）、
3. 命令行参数/插件注册；

文件名必须精确是 conftest.py，测试函数不能写在这里，hook 放错位置（如 test_*.py）不会被触发。

|**hook**|**触发时机**|**典型用途**|
|---|---|---|
|`pytest_collection_modifyitems`|收集完测试后|自动加 marker、过滤用例|
|`pytest_runtest_setup`|每条用例 setup 前|打印用例名、初始化资源|
|`pytest_runtest_teardown`|每条用例 teardown 后|清理、日志|
|`pytest_runtest_makereport`|生成测试报告时|失败时自动截图|
|`pytest_terminal_summary`|所有用例跑完|回调、通知、统计|
|`pytest_addoption`|解析命令行参数|自定义 `--xxx` 参数|
|`pytest_configure`|pytest 启动时|注册 marker、全局配置|
|`pytest_sessionstart`|session 开始|初始化数据库连接池|
|`pytest_sessionfinish`|session 结束|关闭连接池、生成报告|