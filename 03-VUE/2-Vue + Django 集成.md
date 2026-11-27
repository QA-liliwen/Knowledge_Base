## 一、跨域中间件（前后端分离联调）

前后端分离开发时，Vue（如 localhost:8080）和 Django（如 172.16.2.60:8000）端口不同，浏览器会拦截跨域请求，需要 Django 加跨域中间件放行。

1. 在 settings.py 的 MIDDLEWARE 列表最上面加一行：
```python
MIDDLEWARE = [
    'api_project.cors.CorsMiddleware',
    # ...其余原有中间件
]
```

2. 新建文件 api_project/cors.py：
```python
class CorsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        response["Access-Control-Allow-Origin"] = "*"
        response["Access-Control-Allow-Headers"] = "*"
        response["Access-Control-Allow-Methods"] = "*"
        if request.method == "OPTIONS":   # 预检请求直接放行
            response.status_code = 200
        return response
```


## 二、Vue 打包到 Django 的链条

### 2.1 打包命令（唯一的更新动作）

```powershell
cd d:\code\api_project\vue_project
npm run build
```

- 产物自动输出到 `vue_project\dist\`
- 浏览器 Ctrl+F5 强刷即可看到新版，**Django 不需要重启**（静态文件直接读磁盘）

### 2.2 三个关键配置

**① vue.config.js —— 声明打包产物的引用前缀**
```javascript
const { defineConfig } = require('@vue/cli-service')
module.exports = defineConfig({
  transpileDependencies: true,
  publicPath: '/static/'   // index.html 里引用 js/css 都以 /static/ 开头
})
```

**② settings.py —— 模板目录指向 dist**
```python
TEMPLATES = [
  {
    'BACKEND': 'django.template.backends.django.DjangoTemplates',
    'DIRS': [os.path.join(BASE_DIR, 'vue_project', 'dist')],  # 找 index.html
    ...
  }
]
```

**③ settings.py —— 静态文件目录也指向 dist**
```python
STATIC_URL = '/static/'
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'vue_project', 'dist'),  # /static/js/... 从这里找
]
```

### 2.3 请求链路全景

```
浏览器
  │
  ├─ GET http://172.16.2.60:8000/              → Django urls.py 兜底路由 → dist/index.html
  │                                              （re_path r'^.*$' 渲染 Vue 入口页）
  │
  ├─ GET /static/js/app.xxx.js                 → STATICFILES_DIRS → dist/js/app.xxx.js
  │
  └─ POST /run_tool/ 等 API                    → Django 视图函数 → JsonResponse
                                                 （Vue 里用 axios 调这些接口）
```

要点：
- **页面**走 Django 模板（其实只是把 Vue 的 index.html 吐出去）
- **js/css** 走 `/static/` 静态文件服务，实际读的是同一个 dist 目录
- **数据**走 Django 的 API 接口（urls.py 里 path 注册的那些）
- Vue 前端路由（URL 带 #）在浏览器本地切换，不经过后端

### 2.4 开发 vs 生产

| 阶段 | 前端跑法 | 后端跑法 | 跨域 |
| --- | --- | --- | --- |
| 开发联调 | `npm run serve`（localhost:8080，热更新） | `python manage.py runserver 0.0.0.0:8000` | 需要 CorsMiddleware |
| 生产部署 | `npm run build` 出 dist | Django 直接服务 dist | 同源（8000），无跨域问题 |
