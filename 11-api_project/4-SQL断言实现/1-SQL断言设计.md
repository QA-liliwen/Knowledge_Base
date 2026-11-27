## 一、断言语法（无包裹，JSON 零转义）

SQL 断言写在每条 case 自己的断言列里（Excel / 测试JSON），与现有断言同方式存储，按 `;` 拆分后按书写顺序执行。

| 前缀     | 语法                   | 说明           |
| ------ | -------------------- | ------------ |
| `set:` | `set:变量名=反参JSON路径`   | 从响应提取变量入变量池  |
| `sql:` | `sql:SQL语句=入参路径=预期值` | 落库校验（预期值可省略） |

**SQL 与断言的分界（关键实现点）**：取第一个「比较操作符紧跟 `/`」的位置（如 `=/data/answer`）。SQL 内部的等号后面跟的是变量/数字/字段名（`id=${qid}`、`score>=60`、`name='x'`），不会误判；因此 SQL 无需引号/括号包裹，在测试JSON里写起来零转义（双引号包裹方案已废弃：JSON 字符串里的双引号要转义，冲突）。

### 完整示例

入参 `{"name":"qname","data":{"stem":"tigan","answer":"c"}}`，反参 `{"question_id":123}`：

```
set:qid=/question_id; sql:select answer from t where id=${qid}=/data/answer='c'
```

- `set:qid=/question_id` → 从反参提取 qid=123，供 WHERE 用
- `/data/answer` → 从入参取值的路径（路径最后一段同时决定查库结果取哪列）
- `'c'` → 手输预期值

### 两种校验形式

| 形式 | 写法 | 校验逻辑 |
| --- | --- | --- |
| 三方校验（带预期值） | `sql:...where id=${qid}=/data/answer='c'` | 入参值=预期值 且 查库值=预期值 |
| 两方校验（省略预期值） | `sql:...where id=${qid}=/questions/0/solution` | 查库值 == 入参值（适合复杂 JSON 字段整体比对） |

### JSON 归一化比较（关键实现点）

数据库常以字符串存 JSON（如 `{"ids":["D"]}`），入参是结构化 dict/list：
- 双方先直接 `==`；任一侧能解析成 JSON 的一律解析后深度比较
- 空格/键序差异不影响结果（`{ "ids" : [ "D" ] }` == `{"ids":["D"]}`）
- 查库列取值：字段名 = 入参路径最后一段；字段不存在且仅一列时取唯一列，多列报错（SQL 里用 as 别名对齐或只查该列）

**其他关键点**：
- 变量用 `${name}` 引用（set: 提供），未定义直接报错
- 多条断言用 `;` 分隔，因此 **SQL 内禁止 `;`**
- `set:` 必须写在引用它的 `sql:` 之前；每条用例独立变量池
- 入参非 JSON（GET 字符串）时，入参路径断言明确报错

## 二、环境机制（切域名 = 切 SQL 环境）

无独立 SQL 环境概念：执行时 RunList 选中的**域名**（`DB_Domain.domain`，如 devapi1.lingshi.com）就是 SQL 环境。

| 层 | 存哪 | 切环境时 |
| --- | --- | --- |
| 连接（host/port/账号） | `DB_SqlEnv`（env_name = 域名，一条） | 变化 |
| 库名 | `DB_TestItem.sql_database`（每个测试项自己配） | 不变 |
| 表名 | SQL 语句内 | 不变 |

**关键点**：
- 关联键是 `DB_Domain.domain`（devapi1 / preapi1 / api1.lingshi.com）
- 当前域名在 `DB_SqlEnv` 无记录（如生产 api1.lingshi.com 不配）→ 忽略本次执行所有 `sql:` 断言（日志可见），不报错、不连库
- 测试项库名为空 → 同样忽略该测试项的所有 `sql:` 断言
- 连接和库由域名 + 测试项注入，SQL 里只写表名——切环境 SQL 断言零改动
- 维护走 Django admin（10_SQL连接表）

### DB_SqlEnv 模型（只管连接，不管库）

```py
class DB_SqlEnv(models.Model):
    env_name = models.CharField('环境名称', max_length=50, unique=True)  # 与 DB_Domain.domain 对应，如 devapi1.lingshi.com
    host = models.CharField('主机', max_length=200)
    port = models.IntegerField('端口', default=3306)
    user = models.CharField('账号', max_length=100)
    password = models.CharField('密码', max_length=200)
    is_del = models.BooleanField('是否删除', default=False)
```

### DB_TestItem 新增字段

```py
sql_database = models.CharField('SQL库名', max_length=200, blank=True, default='')  # 空 = 不启用 SQL 断言
```

## 三、执行链路（已实现）

```
Excel 断言列 / 测试JSON
  → cases[].assertions 列表（含 set: / sql: 段）
  → dispatch_run（api_executor.py）：
      按 DB_Domain.domain 查 DB_SqlEnv
        → 有记录：组装四要素 sql_conn
        → 无记录：sql_conn = None（忽略所有 sql: 断言）
      build_test_item_cases：每条 case 带所属测试项的 sql_database
      本地：run_single_local(test_run, cases_txt_path, sql_conn)
            → pytest_env 注入 DB_HOST / DB_PORT / DB_USER / DB_PASSWORD
      Jenkins：trigger_jenkins_build(zip_bytes, run_id, base_url, sql_conn)
            → 构建表单加同名 4 参数（Job 需预先声明，一次性配置）
  → run_request.py 断言循环，按书写顺序：
      set:  → assert_set：反参提取变量入池
      sql:  → assert_sql：
               run_sql_assertion（sql_assert.py）：
                 跳过判定（库名空 / 四要素缺）→ 变量替换 → 安全校验
                 → pymysql + DictCursor 执行 → 包装 {"rows": [...], "row_count": N}
               入参取值 + 查库取值：
                 预期值省略 → 两方校验（smart_equal 归一化）
                 预期值存在 → 三方校验（= 比较带 JSON 归一化）
  → 任一步失败：该用例 failed，错误信息标注阶段
```

**关键点**：
- `sql_assert.py` 只读 `os.environ` 拿四要素，本地 / Jenkins 同一条代码路径
- Jenkins 打包 zip 必须带上 `sql_assert.py`（pack_test_bundle 已处理），requirements 含 pymysql
- 每条 `sql:` 独立建连，执行完关闭（connect 10s / read 30s / write 30s 超时）
- 老用例（断言里没有 sql:/set:）行为完全不变

## 四、失败信息分级

| 阶段 | 处理 |
| --- | --- |
| set: 路径不存在 | 报错 `变量提取失败: qid <- /question_id（路径不存在）` |
| set: 格式错误 | 报错 `变量提取格式错误（正确格式 set:变量名=JSON路径）` |
| sql: 变量未定义 | 报错 `SQL 变量未定义: qid（请检查 set: 是否写在 sql: 之前）` |
| sql: 格式错误 | 报错 `SQL断言格式错误（未找到断言边界，正确格式: sql:SELECT ...=/路径=预期值）` |
| SQL 安全拦截 | 报错 `SQL安全校验失败（只允许 SELECT / WITH 开头 / SQL内禁止分号）` |
| SQL 执行错误 | 报错 `SQL执行失败: ...`（pymysql 异常包装） |
| 入参值 ≠ 预期 | 报错 `SQL断言失败(入参值): /data/answer=d = 期望=c` |
| 查库值 ≠ 预期 | 报错 `SQL断言失败(查库值): answer=a = 期望=c` |
| 两方校验不一致 | 报错 `SQL断言失败: 查库值 solution=... != 入参值 /questions/0/solution=...` |
| 环境未配连接 / 库名为空 | **不报错**，跳过该 `sql:` 断言（执行日志可见） |

## 五、安全规则

| 规则 | 原因 |
| --- | --- |
| 只允许 `SELECT` / `WITH` 开头 | 只读保护 |
| SQL 内禁止 `;` | 断言分隔符，顺带防多语句注入 |
| connect(10s) / read(30s) / write(30s) 超时 | 防慢查询拖垮执行 |
| 生产域名不配 `DB_SqlEnv` | 物理隔离，平台永远连不上生产库 |

## 六、实现清单（全部完成）

| # | 模块 | 内容 |
| --- | --- | --- |
| 1 | `api_app/models.py` | `DB_SqlEnv`（连接四要素）；`DB_TestItem.sql_database`；迁移 0023 已执行 |
| 2 | `api_app/admin.py` | `DB_SqlEnv` 注册 admin |
| 3 | `api_app/views_test_item.py` | `update_test_item` 保存 / `get_test_item_detail` 回填 `sql_database` |
| 4 | 前端 `TestItemList.vue` | 编辑弹窗「SQL库名」输入框（单接口用例类型显示） |
| 5 | `api_test/api_executor.py` | case 带 `sql_database`；按 `DB_Domain.domain` 查 `DB_SqlEnv` 组装 `sql_conn` |
| 6 | `api_test/api_local_single_runner.py` | `pytest_env` 注入四要素 |
| 7 | `api_test/api_jenkins_single_runner.py` | 构建参数加 4 变量；打包含 `sql_assert.py`；requirements 加 pymysql |
| 8 | `api_test/sql_assert.py`（新） | 无包裹解析、边界识别、安全校验、变量替换、`smart_equal`、执行包装、降级跳过 |
| 9 | `api_test/run_request.py` | `assert_set` / `assert_sql` 处理器、per-case 变量池 |

## 七、边界

- 本地与 Jenkins 均支持，四要素经环境变量注入，同一条代码路径
- 当前域名无连接配置或测试项库名为空 → 忽略 `sql:` 断言（日志可见），属正常设计而非异常
- Jenkins job 需预先声明 4 个参数：`DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD`
- 深层路径取值（如 `/questions/0/solution/choiceSolution/ids`）若库列存的是整段 JSON，应把路径写到整体字段比对，或 SQL 里用 `JSON_EXTRACT` 取片段
