# 04-上传Jenkins启动job：trigger_jenkins_build

## 实现内容

通过 Jenkins REST API 触发 Job 构建，上传 zip 包并传递 RUN_ID、BASE_URL 参数，返回构建页面 URL。

## 代码来源

`api_app/api_test/api_jenkins_single_runner.py`

---

## 完整函数

```py
def trigger_jenkins_build(zip_bytes, run_id, base_url):
    """POST 到 Jenkins Job，上传 zip 包，返回构建页面 URL"""
    jenkins_base = JENKINS_BASE
    jenkins_job_url = f"{jenkins_base}/job/{JENKINS_JOB}/buildWithParameters"

    # 用 Session 保持会话一致性（Cookie + crumb）
    session = requests.Session()
    session.auth = ('qaadmin', 'qatest2027')
    session.verify = False

    # 获取 CSRF crumb
    crumb_resp = session.get(f"{jenkins_base}/crumbIssuer/api/json", timeout=10)
    crumb_data = crumb_resp.json()
    crumb_field = crumb_data['crumbRequestField']
    crumb_value = crumb_data['crumb']
    session.headers[crumb_field] = crumb_value

    files = {
        'CASE_ZIP': ('test_bundle.zip', zip_bytes, 'application/zip'),
    }
    data = {
        'RUN_ID': str(run_id),
        'BASE_URL': base_url,
        crumb_field: crumb_value,  # 同时作为表单字段
    }

    resp = session.post(jenkins_job_url, files=files, data=data, timeout=30)
    if resp.status_code not in (200, 201):
        raise Exception(f"Jenkins 触发失败: HTTP {resp.status_code}, {resp.text[:200]}")

    return resp.headers.get('Location', '')
```

**关键点**：

- `JENKINS_BASE = "https://172.16.1.240:8443"`，自签证书需 `session.verify = False`。
- `JENKINS_JOB = "Exam_APITest_Platform_Job"`，Job 名称硬编码。
- Jenkins 开启了 CSRF 防护，触发构建前需先 GET `/crumbIssuer/api/json` 获取 crumb。
- crumb 需同时放在 header 和 form data 中（`session.headers[crumb_field]` + `data[crumb_field]`）。
- `files` 字典的 key 必须是 `CASE_ZIP`，与 Jenkins Job 的 File Parameter 名称一致。
- 构建成功返回 201，Location header 指向构建页面 URL，供前端展示和日志下载。
- 非 200/201 响应抛异常，由调用方捕获并更新执行记录状态。

---

## 调用链路

1. 平台侧 `execute_run` 视图调用 `trigger_jenkins_build(zip_bytes, run_id, base_url)`。
2. 函数返回 Jenkins 构建页面 URL。
3. 平台将 URL 写入 `DB_run_result.jenkins_build_url`。
4. 前端结果页展示链接，用户可点击跳转 Jenkins 查看构建进度。
5. Jenkins 执行完毕后 conftest.py 回调平台 `update_run_result`，更新最终状态。
