# 06-上传Jenkins启动job：run_custom_jenkins

## 实现内容

Jenkins 执行多接口脚本测试项的入口函数：创建执行记录 → 打包脚本 → 触发 Jenkins 构建 → 返回构建 URL。

## 代码来源

`api_app/api_test/api_jenkins_custom_runner.py`

---

## 完整函数

```py
def run_custom_jenkins(test_item_ids, description, base_url, env=''):
    """Jenkins 执行多接口脚本测试项：打包脚本 → 触发构建 → 由 Jenkins 端回调平台"""
    test_run = DB_run_result.objects.create(
        env=env,
        status="running",
        description=description,
        test_items=','.join(str(i) for i in test_item_ids),
        started_at=timezone.now(),
        trigger_source="jenkins",
        total=len(test_item_ids),
    )

    # 打包脚本
    try:
        zip_bytes = pack_custom_bundle(test_item_ids)
    except Exception as e:
        test_run.status = "failed"
        test_run.finished_at = timezone.now()
        test_run.save()
        return {"code": -1, "message": str(e)}, 400

    # 触发 Jenkins 构建
    try:
        jenkins_url = trigger_jenkins_build(zip_bytes, test_run.id, base_url)
    except Exception as e:
        test_run.status = "failed"
        test_run.finished_at = timezone.now()
        test_run.save()
        return {"code": -1, "message": str(e)}, 500

    test_run.jenkins_build_url = jenkins_url
    test_run.log_file = f"{test_run.id}_app.log"
    test_run.report_file = f"report_{test_run.id}.html"
    test_run.save()

    return {"code": 0, "message": "success", "run_id": test_run.id, "jenkins_build_url": jenkins_url}, 200
```

**关键点**：

- `trigger_source="jenkins"` 区分本次是 Jenkins 触发，与本地执行 `run_custom` 区分。
- 打包失败（脚本文件缺失、路径不存在）标记 failed 并返回 400。
- Jenkins 触发失败（网络不通、认证失败）标记 failed 并返回 500。
- 成功后写入 `jenkins_build_url`、`log_file`、`report_file`，前端结果页根据这些字段展示链接和下载按钮。
- 最终状态由 Jenkins 执行完毕后 conftest.py 回调 `update_run_result` 更新。

---

## 与本地多接口执行的差异

| 对比项 | 本地 run_custom | Jenkins run_custom_jenkins |
| --- | --- | --- |
| 执行方式 | subprocess.Popen 本地 pytest | HTTP POST 触发 Jenkins |
| 文件处理 | 复制脚本到临时目录 | 打包 zip 上传 Jenkins |
| 日志路径 | 写入 data/logs/ | Jenkins 工作区 logs/ |
| 报告格式 | junit xml | HTML (--html) |
| trigger_source | "local" | "jenkins" |
| 回调时机 | pytest 结束 conftest 回调 | Jenkins 构建结束 conftest 回调 |
