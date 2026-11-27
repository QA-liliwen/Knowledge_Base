# sql_assert.py
## 执行顺序总览

```
run_request.py: assert_sql(expr, sql_database, variables)
 └─① run_sql_assertion(expr, sql_database, variables)  ← 模块唯一入口
     ├─② parse_sql_assertion：拆 (SQL语句, 断言表达式) ← 用 SQL_BOUNDARY 正则
     ├─③ 跳过判定一：sql_database 为空 → return None（日志 INFO）
     ├─④ 跳过判定二：get_conn_config() 四要素缺失 → return None（日志 INFO）
     │       └─ get_conn_config：os.environ 读 DB_HOST/DB_PORT/DB_USER/DB_PASSWORD
     ├─⑤ replace_vars：${qid} → 变量池的值（未定义报错）
     └─⑥ execute_sql：真正连库
         ├─ get_conn_config（再取一次四要素）
         ├─ check_sql_safety（SELECT/WITH 白名单 + 禁分号）
         ├─ pymysql.connect（DictCursor / utf8mb4 / 超时三件套）
         ├─ cursor.execute + fetchall
         ├─ finally: conn.close
         └─ 包装 {"rows": [...], "row_count": N}
     返回 (assert_expr, result) 回引擎

 ── 引擎侧后续调用（函数也定义在本模块，按引擎调用顺序）──
 ⑦ parse_sql_assert_expr：拆 (路径, 操作符, 预期值/None) ← 正则与操作符表已内联进函数
 ⑧ extract_db_value：入参分支的查库取值（列名=路径末段，单列 fallback）
 ⑨ smart_equal：JSON 归一化深度比较（两方校验 / 三方 = 比较）
```

## 一、模块加载（import 时就完成的事）

```py
import os
import re
import json
import pymysql
from log_config import get_logger

log = get_logger(__name__)

# SQL 断言语法: sql:SELECT ...=/rows/0/status=1 （无需引号/括号包裹，JSON 零转义）
# 分界规则：断言部分必然以「比较操作符紧跟 /」开头（如 =/rows/0/status、>=/row_count）
SQL_BOUNDARY = re.compile(r'(==|!=|>=|<=|=|>|<)\s*/')  # 「操作符紧跟/」= 分界点（正常SQL里几乎不出现）
```

**解释**：import sql_assert 的瞬间（pytest 收集 run_request.py 时），SQL_BOUNDARY 正则就编译好了，后续所有断言共用，不重复编译。

**关键点**：
- 模块级常量只剩 `SQL_BOUNDARY`（服务第②步拆 SQL 与断言）；第⑦步拆断言表达式用的正则和操作符表已**内联进 parse_sql_assert_expr 函数**（断言量少，每次调用重新编译的开销可接受）；禁分号校验直接用字面量 `";"`，不再设常量
- 函数物理顺序按调用链排：parse_sql_assertion → replace_vars → get_conn_config → check_sql_safety → execute_sql → run_sql_assertion → 比较期三个函数

## 二、run_sql_assertion：唯一入口

```py
def run_sql_assertion(expr, sql_database, variables):  
	# 拆出 SQL 语句和断言表达式
    sql, assert_expr = parse_sql_assertion(expr)  
  
    # 测试项未配库名 -> 跳过  
    if not sql_database:  
        log.info(f"SQL断言跳过（测试项未配置SQL库名）: {sql[:100]}")  
        return None  
  
    # 环境未配连接四要素 -> 跳过  
    if not get_conn_config():  
        log.info(f"SQL断言跳过（当前环境未配置SQL连接）: {sql[:100]}")  
        return None  
        
    # 替换 ${var} 为变量池的值（来自 set: 提取）
    sql = replace_vars(sql, variables)  
    try:
        # 连库执行查询
        result = execute_sql(sql, sql_database)  
    except AssertionError:  
        raise  
    except Exception as e:  
        log.error(f"SQL执行失败: {e}")  
        raise AssertionError(f"SQL执行失败: {e}")  
    return assert_expr, result
```

**解释**：引擎只认识这一个函数（其余函数都是被它或引擎的后续步骤调用）。三态返回：`None`=跳过、`(assert_expr, result)`=成功、抛 AssertionError=失败。

**关键点**：入参没有 body——执行器不碰入参，「拿到查询结果」和「拿入参做比较」职责分离，比较全在引擎侧（run_request.py 掌握 body 和 OPS）。

## 三、 parse_sql_assertion：拆 SQL 与断言表达式

```py
# 解析 sql: 断言段: 剥离前缀后的内容 -> (SQL语句, 断言表达式)
def parse_sql_assertion(expr):
    text = expr.strip()
    # 找第一个「操作符紧跟/」的位置 = SQL 与断言的分界
    match = SQL_BOUNDARY.search(text)
    if not match:
        raise AssertionError(f"SQL断言格式错误（未找到断言边界，正确格式: sql:SELECT ...=/路径=预期值）: {expr}")
    # 分割 SQL 和断言表达式
    sql = text[:match.start()].strip()
    assert_expr = text[match.start():].strip()
    if not sql:
        raise AssertionError(f"SQL断言格式错误（SQL语句为空）: {expr}")
    return sql, assert_expr
```

**解释**（`select answer from t where id=${qid}=/data/answer='c'` 走一遍）：
- `SQL_BOUNDARY.search` 找第一个「操作符紧跟/」的位置——本例是 `=/data` 的 `=`
- 切出 sql=`select answer from t where id=${qid}`，assert_expr=`=/data/answer='c'`（注意保留了前导操作符，供第⑦步识别）

**为什么分界可靠**：SQL 内部的操作符后面跟的是变量/数字/字符串（`id=${qid}`、`score>=60`、`name='x'`），只有断言操作符后面必然跟 `/`（路径开头）。实测 `>=`、`<=`、`!=`、`<>`、`COUNT(*)` 均不误判。

**为什么不用引号包裹 SQL**：断言存 JSON 数组里元素本身用双引号包裹，SQL 里再写双引号必须转义（`\"`），Excel 里写起来痛苦。无包裹方案 JSON 零转义。

**关键点**：分界找不到或 SQL 为空都立即报格式错误——语法问题任何环境都该暴露（区别于后面的配置跳过）。

## 四、两个跳过判定 + get_conn_config

```py
    # 测试项未配库名 -> 跳过
    if not sql_database:
        log.info(f"SQL断言跳过（测试项未配置SQL库名）: {sql[:100]}")
        return None

    # 环境未配连接四要素 -> 跳过
    if not get_conn_config():
        log.info(f"SQL断言跳过（当前环境未配置SQL连接）: {sql[:100]}")
        return None
```

被调用的 get_conn_config：

```py
# 从环境变量读取连接四要素
def get_conn_config():
    host = os.environ.get("DB_HOST")
    port = os.environ.get("DB_PORT")
    user = os.environ.get("DB_USER")
    password = os.environ.get("DB_PASSWORD")
    if not all([host, port, user, password]):
        return None
    return {"host": host, "port": int(port), "user": user, "password": password}
```

**解释**：
- 跳过一：测试项编辑弹窗没配「SQL库名」→ 这个测试项不启用 SQL 断言
- 跳过二：当前域名在 DB_SqlEnv 无记录（如生产 api1.lingshi.com）→ executor 没注入四要素 → os.environ 读不到
- 两者都是**正常降级**：日志 INFO 记录（执行日志可查），用例继续跑其余断言，不算失败

**关键点**：
- 跳过判定放在解析**之后**：先验语法（写错任何环境都该报），再判配置（环境问题才跳过）
- 四要素只从 os.environ 读——本地（pytest_env 注入）和 Jenkins（构建参数转环境变量）是同一条代码路径，本地测试过的 Jenkins 直接可用
- `all([...])` 缺任何一个都返回 None，int(port) 在 all 之后才做（避免对 None 转 int 报错）

## 五、replace_vars：变量替换

```py
# ${var} 变量替换，未定义直接报错
def replace_vars(sql, variables):
    def repl(match):
        # 括号分组捕获到的变量名，如 ${qid} 里的 qid
        name = match.group(1)
        if name not in variables:
            raise AssertionError(f"SQL 变量未定义: {name}（请检查 set: 是否写在 sql: 之前）")
        return str(variables[name])

    # 匹配 ${变量名}，每匹配到一个 ${xxx} 就回调_repl函数一次
    return re.sub(r"\$\{(\w+)\}", repl, sql)
```

**解释**：
- `re.sub(r"\$\{(\w+)\}", repl, sql)`传参了：“正则，函数名，sql字符串”。
  如果匹配到了${变量名}，每匹配到一个 ${xxx} 就回调_repl函数一次。
- `name = match.group(1)`正则中，`group(0)` 是**完整匹配**，`group(1)` 是**第 1 个括号**捕获的内容。
```py
正则: \$\{(\w+)\} 
文本: ${qid} 

group(0) → "${qid}" ← 整个匹配（带 ${} 外壳） 
group(1) → "qid" ← 只有括号里的变量名
```
- 未被定义的变量名则返回错误。
-  repl执行完成后，继续调用re.sub，如果有匹配的${xx}，会继续调用。

**关键点**：未定义直接报错而不是留原样——留原样会生成 `id=${qid}` 的 SQL，数据库报的语法错误信息难懂；明确提示「检查 set: 是否写在之前」直指根因（断言按书写顺序执行）。

## 六、execute_sql：连库执行

```py
# 执行 SQL 并包装结果 {"rows": [...], "row_count": N}
def execute_sql(sql, database):
    # 数据库连接配置
    conn_config = get_conn_config()
    if not conn_config:
        return None
        
    # 安全校验
    check_sql_safety(sql)
    
    # 建数据库连接：库名=测试项配置的 sql_database；utf8mb4 防中文乱码；
    # 超时控制防慢 SQL 卡死用例；DictCursor 让每行结果是 {列名: 值} 字典
    conn = pymysql.connect(
        database=database,
        charset="utf8mb4",
        connect_timeout=10,   # 建连接超时（秒）
        read_timeout=30,      # 读结果超时（秒）
        write_timeout=30,     # 写超时（秒）
        cursorclass=pymysql.cursors.DictCursor,
        **conn_config,        # 展开 host/port/user/password 四要素
    )
    try:
        with conn.cursor() as cursor:  # with 结束自动关闭游标
            cursor.execute(sql)        # 执行查询
            rows = cursor.fetchall()   # 取回全部结果行
    finally:
        conn.close()  # 无论成功失败都关连接，防止连接泄漏
        
    log.debug(f"SQL执行成功: {sql[:100]}... 共 {len(rows)} 行")
    # 统一包装成固定形态，供断言按 /rows/... 或 /row_count 取值
    return {"rows": rows, "row_count": len(rows)}
```

内部先调用的安全校验：

```py
# 安全校验
def check_sql_safety(sql):
    # 去前导空白并转大写（SELECT / select 都能识别）
    sql_head = sql.lstrip().upper()
    if not (sql_head.startswith("SELECT") or sql_head.startswith("WITH")):
        raise AssertionError(f"SQL安全校验失败（只允许 SELECT / WITH 开头）: {sql[:100]}")
    if ";" in sql:
        raise AssertionError(f"SQL安全校验失败（SQL内禁止分号）: {sql[:100]}")
```

**解释**（按调用顺序）：
1. 再次取四要素（execute_sql 也可独立调用，所以自己也要判）
2. **安全白名单**：lstrip().upper() 后必须 SELECT/WITH 开头（WITH 覆盖 CTE），挡掉一切写操作；**禁分号**——断言分隔符就是 `;`，SQL 内出现会破坏拆分，顺带防多语句注入
3. **连接参数**：DictCursor 返回 `[{"列名": 值}]`（第⑨步按列名取值靠它）；utf8mb4 中文不乱码；connect/read/write 超时 10/30/30s 防慢查询拖垮整轮执行
4. **执行与关闭**：cursor 上下文管理 + finally close——每条 sql: 断言独立建连，无连接池（测试场景频率低），但保证不泄漏

**关键点**：pymysql 原生异常（连接超时、库不存在、密码错）在入口函数被统一包装成 AssertionError 带「SQL执行失败」前缀，pytest 报告里可读。

至此模块内主链路结束，返回 `(assert_expr, result)` 回引擎侧。以下三个函数由引擎在比较阶段调用（is_query_result_path 已随形式2一起删除，详见 5-入参的断言操作.md）。
