# Excel 上传解析

单接口测试项支持通过 Excel 批量导入用例，前端上传 `.xlsx` 文件后，Django 接收、解析并返回结构化 JSON 数据回填到表单。

---

## 流程

```
前端选择文件
  → POST /upload_case/ (multipart/form-data)
  → Django 保存为临时 xlsx
  → openpyxl 读取并解析
  → 返回 cases 列表
  → 前端回填到 cases_json 编辑框
```

---

## Excel 模板格式

| CaseID | 用例名称 | 是否执行 | 请求体 | 预期状态码 | 断言 | 描述 |
|--------|---------|---------|--------|-----------|------|------|
| CASE_001 | 有效参数_全参 | Y | `{ "classifyAlias":"question_set", ... }` | 200 | /code=0;/message=success | |
| CASE_002 | 缺少必传参数 | Y | `{ "classifyAlias":"test111" }` | 200 | /code=-1003 | |

断言列多条用 `;` 分隔。

---

## 前端：上传控件

```html
<div v-if="edit_form.type === 1" class="form-row">
    <label class="form-label-fixed">上传用例：</label>
    <div style="flex: 1; display: flex; gap: 8px; align-items: center">
        <input type="file" class="form-control" accept=".xlsx" ref="fileInput" @change="upload_file" style="flex: 1">
    </div>
</div>
```

```js
upload_file() {
    const fileInput = this.$refs.fileInput;
    if (!fileInput || !fileInput.files || fileInput.files.length === 0) {
        return;
    }
    const formData = new FormData();
    formData.append('fileUpload', fileInput.files[0]);
    axios.post('http://172.16.2.60:8000/upload_case/', formData, {
        headers: {'Content-Type': 'multipart/form-data'}
    }).then(res => {
        if (res.data.cases) {
            this.edit_form.cases_json = JSON.stringify(res.data.cases, null, 2);
            this.edit_form.uploaded_filename = res.data.filename || '';
        } else {
            alert(res.data.msg || '解析失败');
        }
    }).catch(err => {
        alert('上传失败: ' + (err.response?.data?.msg || err.message));
    })
}
```

### 特殊处理说明

| 代码 | 特殊处理原因 |
|------|-------------|
| `accept=".xlsx"` | 限制文件选择框只显示 xlsx 文件 |
| `ref="fileInput"` | 通过 ref 获取原生 input 节点，读取 files |
| `FormData` | 必须用这种格式传文件，不能传 JSON |
| `multipart/form-data` | 告诉后端这是文件上传请求 |
| `fileInput.files[0]` | 原生文件对象，用于 append 到 FormData |
| `res.data.cases` | 后端成功时返回解析后的用例数组 |
| `JSON.stringify(..., null, 2)` | 格式化 JSON，便于编辑框里阅读 |

---

## 后端：接收并保存

```py
import os
import json
import openpyxl
from django.http import JsonResponse

TEMP_DIR = "temp_uploads"


def upload_case(request):
    os.makedirs(TEMP_DIR, exist_ok=True)
    file = request.FILES.get("fileUpload")
    if not file or not file.name.endswith(".xlsx"):
        return JsonResponse({"msg": "请上传 xlsx 文件"}, status=400)

    upload_xlsx_name = file.name
    file_path = os.path.join(TEMP_DIR, upload_xlsx_name)
    with open(file_path, "wb") as f:
        for chunk in file.chunks():
            f.write(chunk)

    test_item_data = read_single_case(file_path)
    if not test_item_data:
        return JsonResponse({"msg": "用例解析失败，请检查文件格式"}, status=400)

    return JsonResponse({"msg": "解析成功", "cases": test_item_data, "filename": upload_xlsx_name})
```

### 特殊处理说明

| 代码 | 特殊处理原因 |
|------|-------------|
| `request.FILES.get("fileUpload")` | 前端 FormData 的 key 必须和后端一致 |
| `file.name.endswith(".xlsx")` | 服务端再校验一次文件类型 |
| `os.makedirs(TEMP_DIR, exist_ok=True)` | 临时目录不存在时自动创建 |
| `file.chunks()` | Django 大文件分块读取，避免内存爆炸 |
| `"wb"` 模式 | 二进制写入，避免中文编码问题 |
| `read_single_case(file_path)` | 解析失败返回空列表，统一做非空判断 |
| `JsonResponse({"msg": ...}, status=400)` | 失败时返回明确错误信息和 400 状态码 |

---

## 后端：解析 Excel

```py
xlsx_header_cn = ["CaseID", "用例名称", "是否执行", "请求体", "预期状态码", "断言", "描述"]
xlsx_header_en = ["case_id", "case_name", "is_active", "body", "expected_status_code", "assertions", "description"]


def read_single_case(file_path: str):
    wb = openpyxl.load_workbook(file_path, read_only=True)
    try:
        rows = list(wb.active.iter_rows(values_only=True))
        if not rows:
            return []

        headers = [str(h).strip() for h in rows[0]]
        if headers != xlsx_header_cn:
            return []

        data = []
        for row_idx, row in enumerate(rows[1:], start=2):
            row_dict = {}
            for idx, header in enumerate(xlsx_header_en):
                value = row[idx] if idx < len(row) else ""
                if value is None:
                    value = ""
                elif isinstance(value, str):
                    value = value.strip()
                    if value.startswith("{") and value.endswith("}"):
                        try:
                            value = json.loads(value)
                        except json.JSONDecodeError:
                            pass
                row_dict[header] = value

            if not row_dict.get("expected_status_code") or not row_dict.get("assertions"):
                print(f"错误：第 {row_idx} 行 预期状态码/断言 为空")
                return []

            assertions = row_dict["assertions"]
            if isinstance(assertions, str):
                row_dict["assertions"] = [a.strip() for a in assertions.split(";") if a.strip()]
            else:
                row_dict["assertions"] = [str(assertions)]

            data.append(row_dict)

        print(f"解析成功，共 {len(data)} 条用例")
        return data
    finally:
        wb.close()
```

### 特殊处理说明

| 代码 | 特殊处理原因 |
|------|-------------|
| `openpyxl.load_workbook(..., read_only=True)` | 只读模式，大文件不全部加载到内存 |
| `iter_rows(values_only=True)` | 直接返回单元格值，不返回 Cell 对象 |
| `wb.active` | 默认取第一个工作表 |
| `rows[0]` | 第一行作为表头 |
| `str(h).strip()` | 表头可能是数字或带空格，统一转字符串去空格 |
| `headers != xlsx_header_cn` | 表头必须完全一致（包括顺序），否则直接拒绝解析 |
| `enumerate(rows[1:], start=2)` | 跳过表头，行号从第 2 行开始，便于报错定位 |
| `value is None` | Excel 空单元格读出来是 None，转成空字符串 |
| `value.startswith("{") and value.endswith("}")` | 只有长得像 JSON 的字符串才尝试解析 |
| `try/except json.JSONDecodeError` | JSON 解析失败时不抛异常，保留原始字符串 |
| `expected_status_code` 或 `assertions` 为空 | 必填校验，为空整批失败 |
| `assertions.split(";")` | 断言按分号拆成多条 |
| `finally: wb.close()` | 确保文件句柄释放，防止 Windows 下文件被占用 |
