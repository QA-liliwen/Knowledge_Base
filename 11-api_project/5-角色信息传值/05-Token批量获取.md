# Token 批量获取（token_fetcher.py）

## 实现内容

执行准备阶段把"用户名列表"变成"token 字典"：`fetch_tokens` 接收域名/环境段/app 登录域名 + 用户名列表，为每个账号固定获取 oss 和 app 两个 token，返回 `{账号: {'oss': token, 'app': token}}`。登录策略为三级：固定密码优先 → 临时密码回落 → 失败降级空串（不拦截执行）。底层登录函数（dim/oss/app 三系统）实现于 `api_tools/user_info.py`（该文件同时承载"用户信息查询工具"，其工具注册与展示部分另行整理，本篇只覆盖角色链路用到的登录函数）。

## 代码来源

- `api_app/api_test/token_fetcher.py`（全文）
- `api_app/api_tools/user_info.py`（登录函数部分）

---

## 第一部分：域名与固定密码档案

```py
from api_app.api_tools.base import ToolError
from api_app.api_tools.user_info import (
    get_dim_token,
    apply_dim_password,
    oss_dologin,
    app_login,
    FIXED_PASSWORDS,
)

# dim accessToken 统一从测试环境获取
DIM_BASE = 'https://devapi1.lingshi.com/dim/vs2'

DOMAIN_PASSWORD_PROFILE = {
    'devapi1.lingshi.com': 'test',
    'preapi1.lingshi.com': 'pre',
    'api1.lingshi.com': 'prod',
}
```

**关键点**：

- 固定密码表按"环境档案"索引：先用被测域名反查 profile（test/pre/prod），再进 FIXED_PASSWORDS 取该环境下的账号密码表——同一批账号跨环境密码不同也能正确命中。
- dim 网关（临时密码申请的入口）**统一走测试环境**：三个环境的临时密码申请都从 `devapi1` 拿 dim token（实测跨环境通用）。

---

## 第二部分：固定密码表（user_info.py）

```py
# 已知固定密码（按环境维护；未收录的账号直接走临时密码流程）
# 登录策略：先用固定密码，失败再回落临时密码；oss 与移动端分开判断，互不影响
FIXED_PASSWORDS = {
    'test': {
        'adminexam': '888888',
        'adminqa9515': '405824',
        'adminqwqwqw': '111111',
        'admincontent': '111111',
        'adminvswrr': 'Ct77a42z',
    },
    'pre': {},
    'prod': {},
}
```

**关键点**：

- 手工维护的"已知密码"白名单：命中的账号直接登录，省掉一次临时密码申请（dim 调用），加快执行。
- 未收录 = 直接走临时密码，不需要维护全量。
- 密码与账号一一对应（非角色维度）——同一账号被多个角色引用时，登录结果共享。

---

## 第三部分：密码编码 encode_login_param

```py
# 登录参数编码：L + base64(用户名:密码) + S，L/S 为固定标识符
# 实测 'adminvswrr:bclw45' -> 'LYWRtaW52c3dycjpiY2x3NDU=S'，不是真加密，只是 Base64 混淆
def encode_login_param(username, password):
    raw = f'{username}:{password}'.encode('utf-8')
    return 'L' + base64.b64encode(raw).decode('utf-8') + 'S'
```

**关键点**：

- oss 与 app 登录共用同一编码函数（前端客户端同款混淆方式）。
- 只是 Base64 + 前后缀标识，非加密。

---

## 第四部分：dim 登录与临时密码申请

**dim 登录（admin 固定密文）**：

```py
# dim 系统 admin 账号的加密密码（RSA，固定值，三个环境通用）
_DIM_ADMIN_PASSWORD = "mooDo0TDAVq418YpRxi7XMe4dBGfGAxk...（省略）...=="

# 登录 dim 系统拿 accessToken，用于后续接口的 Bearer 鉴权（替代硬编码 token，不再有过期问题）
def get_dim_token(dim_base):
    url = f'{dim_base}/crmsys/login/password'
    resp = requests.post(url, json={"username": "admin", "password": _DIM_ADMIN_PASSWORD}, ...)
    data = resp.json()
    ...
    token = (data.get('data') or {}).get('accessToken', '')
    if not token:
        raise ToolError("dim 登录返回中缺少 accessToken")
    return token
```

**申请临时密码（取"生效中"记录）**：

```py
# 到 dim 系统申请临时密码，返回 recordList 中「生效中」的密码
def apply_dim_password(code, dim_base, token):
    url = f'{dim_base}/crmsys/login/dynamic/apply'
    resp = requests.post(url,
        json={"data": {"code": code}},
        headers={'authorization': f'Bearer {token}', ...})
    data = resp.json()
    ...
    # 密码列表按时间倒序，取第一条生效中的
    record_list = (data.get('data') or {}).get('recordList') or []
    for record in record_list:
        if record.get('pwdStatus') == '生效中':
            password = record.get('password', '')
            if password:
                return password
    raise ToolError("recordList 中没有生效中的密码")
```

**关键点**：

- `code` 就是用户名；dim 系统按账号发放临时密码，返回列表要挑 `pwdStatus == '生效中'` 的那条。
- dim token 是拿临时密码的前置凭证，申请失败抛 ToolError 由上层降级。

---

## 第五部分：oss / app 登录

**oss 登录（返回 userInfo 节点）**：

```py
# 到 oss 系统 doLogin，返回 data.userInfo 节点
def oss_dologin(username, password, oss_base):
    url = f'{oss_base}/schsys/login/doLogin'
    resp = requests.post(url, json={
        "method": "account",
        "userName": username,
        "password": encode_login_param(username, password),
        "server": None,
    }, headers={'apptype': '10', 'devicetype': '5', ...})
    ...
    user_info = (data.get('data') or {}).get('userInfo') or {}
    if not user_info.get('token'):
        raise ToolError("oss 登录返回中缺少 token")
    return user_info
```

**app 登录（PUT，模拟 Android 客户端）**：

```py
# app V2 登录，返回含 token 的完整响应
def app_login(username, password, app_server):
    headers = {
        'user-agent': 'Android/5.6.35.238481 ...',
        'appversion': '5.6.35.238481',
        ...
        'host': app_server,
        'serverid': '0',
        ...
    }
    payload = {
        'LoginUserArgu': {
            'industryType': 'All',
            'key': encode_login_param(username, password),
            'server': app_server,
        }
    }
    resp = requests.put(f'https://{app_server}/center/services/rest/user/V2/Login?serino=110', json=payload, headers=headers)
    data = resp.json()
    ...
    token = data.get('token', '')
    if not token:
        raise ToolError("app 登录返回中缺少 token")
    return data
```

**关键点**：

- 两套登录的地址来源不同：oss 走网关（`https://{domain}/school/{env_seg}`），app 走独立的 app_server 域名（DB_Domain.app_server）。
- 返回结构不同：oss token 在 `data.userInfo.token`、app token 在顶层 `token`。
- 两者都做了结构性校验（无 token 即抛 ToolError）→ 由上层统一降级。

---

## 第六部分：批量入口 fetch_tokens（核心）

```py
# ===== 批量获取 token（入口） =====
# domain:   域名
# env_seg:  环境段
# app_server: app 登录域名
# usernames: 用户名列表
# 返回 {用户名: {'oss': token, 'app': token}}
# 登录失败不拦截，对应系统 token 落空串（用例执行时自然 401，报告可见）
def fetch_tokens(domain, env_seg, app_server, usernames):
    oss_base = f'https://{domain}/school/{env_seg}'
    dim_base = f'https://{domain}/dim/{env_seg}'
    # 当前域名的固定密码表
    fixed_pw_map = FIXED_PASSWORDS.get(DOMAIN_PASSWORD_PROFILE.get(domain, ''), {})
    dim_state = {'token': None}
    temp_pw_map = {}

    # 申请（或复用）某账号的临时密码；失败信息缓存，同账号后续系统不再重复调 dim
    def ensure_temp_pw(username):
        if username not in temp_pw_map:
            try:
                if dim_state['token'] is None:
                    dim_state['token'] = get_dim_token(DIM_BASE)
                password = apply_dim_password(username, dim_base, dim_state['token'])
                temp_pw_map[username] = {'password': password, 'error': None}
            except ToolError as e:
                temp_pw_map[username] = {'password': None, 'error': str(e)}
        return temp_pw_map[username]

    def login_one(username, system):
        # 根据调用的oss|app，将对应函数赋值，以便后续调用。
        login_fn = {'oss': oss_dologin, 'app': app_login}[system]
        login_base = oss_base if system == 'oss' else app_server
        # 第一步：固定密码
        fixed_pw = fixed_pw_map.get(username)
        if fixed_pw:
            try:
                return login_fn(username, fixed_pw, login_base)['token']
            except ToolError:
                pass
        # 第二步：临时密码（申请失败或登录失败均降级空串）
        temp = ensure_temp_pw(username)
        if temp['error'] or not temp['password']:
            print(f"[token_fetcher] 环境[{env_seg}] 账号 {username} 的 {system} token 获取失败：临时密码不可用（{temp['error']}）")
            return ''
        try:
            return login_fn(username, temp['password'], login_base)['token']
        except ToolError as e:
            print(f"[token_fetcher] 环境[{env_seg}] 账号 {username} 的 {system} token 获取失败：{e}")
            return ''

    # 每个账号固定拿 oss + app 两个 token
    tokens = {}
    for username in sorted(set(usernames)):
        tokens[username] = {system: login_one(username, system) for system in ('oss', 'app')}
    return tokens
```

**关键点**：

- **输入扁平化**：调用方只传"用户名列表"，不需要知道每个账号要哪些系统；输出固定双系统 `{账号: {'oss': ..., 'app': ...}}`。
- **三级登录策略**（每个账号×每个系统独立判断）：
  1. 固定密码命中 → 直接登录成功；
  2. 固定密码失败/未收录 → 临时密码（dim 申请）；
  3. 临时密码申请失败或登录仍失败 → 返回**空串**，只 print 提示，不抛出。
- **懒申请 + 缓存**：`ensure_temp_pw` 每个账号只申请一次（`temp_pw_map` 缓存含失败信息）；dim token 全局只登一次（`dim_state` 缓存）。oss/app 分开判断——oss 用固定密码成功不影响 app 也走一遍策略。
- **降级空串不拦截**：失败账号 token 为 `''`，预组装时该端头就没 token，用例执行自然 401，失败信息在报告与前置日志里可见——与"系统不需要过度严谨"的整体取舍一致。
- `tokens` 里键是**账号**（不是角色）：多角色共用账号时结果天然一份（配合 04 的 usernames 去重）。

---

## 整体流程

1. `fetch_tokens(domain, env_seg, app_server, usernames)` 被 dispatch_run 调用。
2. 按域名反查环境档案 → 取对应固定密码表；拼出 oss_base / dim_base。
3. 遍历去重后的用户名：对 oss、app 各执行一遍"固定密码 → 临时密码 → 空串"策略。
4. 返回 `{账号: {oss, app}}` 交给 `build_headers_map` 回填进请求头。
