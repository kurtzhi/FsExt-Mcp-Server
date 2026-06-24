# FsExt-MCP-Server (Python)

## 项目概述
**FsExt-MCP-Server (Python)** 是一套基于模型上下文协议（MCP）开发、功能完整、安全可控且高性能的文件操作服务端。为各类AI客户端与MCP集成工具提供标准化接口，完整覆盖文件管理、图像处理、OCR文字识别能力。

本服务重点强化**权限安全管控**与**大文件兼容处理**，解决市面基础MCP文件工具普遍存在的权限管控薄弱、功能单一、大文件读写性能差等痛点。

### 核心特性
- **完整文件与目录管理**：支持文件创建、删除、复制、移动、元数据查询、存在性校验；支持整目录递归复制与移动
- **灵活文件读写能力**：全文读取、分段文本读取、二进制分块读取、文本/二进制覆写与追加写入，完美适配大文件流式处理
- **强大检索与替换工具**：全局目录内容检索、单/多文件带上下文匹配（自定义前后预览行数）、正则匹配、大小写忽略、原地文本替换
- **图像处理工具集**：内置图片缩放（支持锁定比例+边缘填充）、裁剪、旋转，覆盖日常图片编辑需求
- **Tesseract OCR文字识别**：多语言图片文字提取，可自定义Tesseract程序路径与语言数据包路径
- **严格工作空间安全锁定**：提供`--lock-root`根目录锁定参数，所有文件/目录操作强制限定在指定根目录内，禁止跨目录越权访问
- **多传输协议兼容**：支持MCP官方标准传输通道：`stdio`（本地客户端对接）、`sse`（轻量远程长流）、`http`（标准双向远程流传输）

## 快速上手


### 1. uvx 啟動指令 （無需預先安裝）
uvx 會自動拉取已發布到PyPI的安裝包，獨立隔離運行環境，無需手動配置虛擬環境與依賴。
#### 簡寫推薦用法
```bash
# 默認stdio傳輸，無目錄鎖，擁有全部文件訪問權限
uvx fsext-mcp-server

# 安全推薦：限定所有文件操作僅能訪問指定文件夾
uvx fsext-mcp-server --lock-root /你的/工程目錄
```

#### 完整可自定義長指令
```bash
# stdio 本地隔離目錄
uvx fsext-mcp-server --transport stdio --lock-root /你的/工程目錄

# SSE 遠程服務
uvx fsext-mcp-server --transport sse --host 0.0.0.0 --port 8000 --lock-root /你的/工程目錄

# Streamable HTTP 標準雙向遠程服務
uvx fsext-mcp-server --transport http --host 0.0.0.0 --port 8000 --lock-root /你的/工程目錄
```

### 2. 在大模型客戶端/框架中調用
環境無需提前安裝依賴，客戶端建立連接時uvx會自動下載並啟動服務。

#### Claude Desktop / Cursor MCP 配置範例
```json
{
  "mcpServers": {
    "fsext": {
      "command": "uvx",
      "args": [
        "fsext-mcp-server",
        "--lock-root",
        "/你的/工程目錄"
      ],
      "env": {"PYTHONUTF8": "1"}
    }
  }
}
```

#### LangChain / LangGraph 核心偽代碼（僅關鍵片段）
受 `langchain-mcp-adapters` 會話生命週期限制，完整穩定運行代碼需要額外長連接適配，以下為標準調用核心邏輯供參考：
```python
# 核心配置：透過uvx stdio連接FsExt MCP服務
server_config = {
    "fsext": {
        "transport": "stdio",
        "command": "uvx",
        "args": ["fsext-mcp-server", "--lock-root", r"/你的/工程目錄"],
        "env": {"PYTHONUTF8": "1"}
    }
}

# 載入全部文件操作工具
client = MultiServerMCPClient(server_config)
async with client.session("fsext") as session:
    mcp_tools = await load_mcp_tools(session)

# 工具綁定大模型，搭建對話流程
llm = ChatOpenAI(base_url="本地大模型接口地址").bind_tools(mcp_tools)
```

## 傳統 pip 安裝使用方式
### 安裝包
```bash
pip install fsext-mcp-server
```

### 安裝完成後啟動指令
```bash
# 默認stdio本地模式
fsext-mcp-server
# 或者使用短命令
fsext

# 限定操作目錄
fsext --lock-root /你的/工程目錄

# 遠程SSE服務
fsext --transport sse --port 8000
```

---

### 环境前置依赖
- Python ≥ 3.10
- FFmpeg（系统全局安装，媒体相关功能必需）
- Tesseract OCR（可选，用于图片文字识别）

### 依赖一键安装
推荐使用uv快速搭建隔离环境：
```bash
# 克隆代码仓库
git clone https://github.com/kurtzhi/fsext-mcp-server-python
cd fsext-mcp-server-python

# 同步全部项目依赖
uv sync
```

### 核心依赖说明
- **chardet**：自动检测文件编码
- **Pillow**：图像处理核心库，实现缩放、裁剪、旋转功能
- **python-magic**：精准识别文件MIME类型
- **ffmpeg-python**：FFmpeg Python封装，用于媒体文件处理

## 启动使用说明
服务支持三种MCP传输模式，同时可配置工作空间根目录锁定。

### 启动参数对照表
| 参数 | 默认值 | 参数说明 |
| --- | --- | --- |
| `--transport` | stdio | MCP传输类型：`stdio`/`sse`/`http` 官方标准协议 |
| `--host` | 127.0.0.1 | 绑定监听地址（stdio模式下无效） |
| `--port` | 8000 | 绑定监听端口（stdio模式下无效） |
| `--lock-root` | 无 | 锁定所有操作仅能访问指定根目录；不填则无访问限制 |

### 常用启动命令
#### 1. 默认本地标准输入输出模式（适配Claude Desktop / Cursor）
```bash
uv run -m fsext
```

#### 2. 带工作空间锁定的本地安全模式
```bash
uv run -m fsext --lock-root /你的工作目录路径
```

#### 3. 远程SSE长流传输模式
```bash
uv run -m fsext --transport sse --host 0.0.0.0 --port 8000
```
##### 访问接口地址
1. SSE长连接接收通道（客户端订阅服务端推送消息）
   `http://<主机地址>:<端口>/sse`
2. 客户端请求发送通道（调用工具、发送初始化请求）
   `http://<主机地址>:<端口>/messages`

##### MCP Inspector客户端配置
- 传输类型：SSE
- 连接地址填写：`http://127.0.0.1:8000/sse`

#### 4. 官方标准HTTP双向流远程传输
```bash
uv run -m fsext --transport http --host 0.0.0.0 --port 8000
```
##### 统一访问入口
仅单个双向统一接口，无独立`sse`与`messages`路由：
`http://<主机地址>:<端口>/mcp`

##### MCP Inspector客户端配置
- 传输类型：Streamable HTTP
- 连接地址填写：`http://127.0.0.1:8000/mcp`

#### 5. SSE传输与标准HTTP流传输核心差异对比
| 特性 | SSE（HTTP+SSE） | Streamable HTTP标准流 |
| --- | --- | --- |
| 接口架构 | 双接口设计：POST接口接收客户端请求 + GET接口接收服务端推送流 | 单统一接口：同一URL同时支持POST请求与GET长流 |
| 通信方式 | 单向通信：服务端单向推送，客户端无法复用同一条流回传数据 | 可选双向通信：支持普通HTTP响应，同一连接可无缝切换SSE实时推送 |
| 连接稳定性 | 双接口状态管理复杂，极易会话中断、连接丢失 | 自动会话恢复，高并发场景稳定性更强，维护成本更低 |
| 行业标准现状 | 现代分布式项目已逐步淘汰 | 当前远程客户端服务对接官方新标准 |

## MCP完整工具列表
### 统一全局返回规范
所有MCP工具执行结果统一外层包装结构，成功执行、运行报错共用一套格式，所有工具原生业务数据均放置在`res`下的`info`字段中。

#### 结构定义
```json
{
  "res": {
    "success": boolean,
    "info": object
  }
}
```
- `success`：全局执行状态标识
    - `true`：工具正常执行，`info`为各工具定义的业务返回数据
    - `false`：执行失败（工作空间越权、文件不存在、IO异常、参数非法等场景）
- `info`分两种数据格式：
    1. 成功模式（`success: true`）：各工具专属自定义业务对象
    2. 失败模式（`success: false`）：固定错误结构体，包含错误码与可读提示文案
       ```json
       "info": {
         "code": "错误码标识",
         "message": "详细错误描述"
       }
       ```

#### 返回完整示例
##### 1. 执行成功示例（目录遍历工具fs_list_directory）
```json
{
  "res": {
    "success": true,
    "info": {
      "paths": [
        "/tmp/tests/test_util.py",
        "/tmp/tests/__init__.py",
        "/tmp/tests/img/cochem_castle.jpg"
      ]
    }
  }
}
```

##### 2. 执行失败示例（工作空间访问拦截）
```json
{
  "res": {
    "success": false,
    "info": {
      "code": "WORKSPACE_ESCAPE_FORBIDDEN",
      "message": "访问受限：路径 `/tmp/test2` 超出允许的工作目录 `/tmp/tests`"
    }
  }
}
```

全部工具均支持**工作路径访问限制**，严格遵循标准化入参与出参规范。

### 一、基础文件操作工具
#### fs_create_file
**功能说明**：创建文本文件，支持自定义初始内容与文件编码。
**入参**：
- `file_path` (str)：目标文件路径
- `content` (str, 默认空字符串)：文件初始文本内容
- `charset` (str, 默认utf-8)：文件编码格式

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_delete_file
**功能说明**：永久删除单个普通文件，不支持删除文件夹。
**入参**：`file_path` (str)：目标文件路径

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_get_file_info
**功能说明**：获取文件/目录完整元数据（类型、大小、修改时间、权限等）。
**入参**：`file_path` (str)：目标路径

**原生业务返回**：目标条目序列化完整元数字典
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_is_file_exists
**功能说明**：校验工作空间内指定文件或目录是否存在。
**入参**：`file_path` (str)：目标路径

**原生业务返回**：`{"exists": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_copy_file
**功能说明**：复制单个文件，保留原始元数据，支持覆写开关。
**入参**：
- `source_file_path` (str)：源文件路径
- `dest_file_path` (str)：目标文件路径
- `overwrite` (boolean)：是否覆盖已存在的目标文件

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_move_file
**功能说明**：移动单个文件至新路径，支持覆写控制。
**入参**：
- `source_file_path` (str)：源文件路径
- `dest_file_path` (str)：目标文件路径
- `overwrite` (boolean)：是否覆盖冲突文件

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

### 二、目录操作工具
#### fs_list_directory
**功能说明**：递归扫描目录并过滤路径，支持文件类型过滤、仅返回文件模式。
**入参**：
- `source_dir` (str)：待扫描目标目录
- `recursive` (boolean)：是否递归扫描子目录
- `file_only` (boolean)：是否仅返回文件（排除文件夹）
- `file_extension` (str, 默认空)：按后缀筛选文件

**原生业务返回**：`{"paths": list[str]}` 绝对路径列表
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_copy_directory
**功能说明**：递归复制完整目录树，支持覆盖已有目录。
**入参**：
- `source_dir` (str)：源目录路径
- `copy_dest_dir` (str)：目标目录路径
- `overwrite` (boolean)：是否清空并覆盖已有目标目录

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_move_directory
**功能说明**：移动完整目录，目标路径存在则直接失败，避免误覆盖。
**入参**：
- `source_dir` (str)：源目录路径
- `dest_dir` (str)：目标目录路径

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

### 三、文件读写工具
#### fs_read_full_text
**功能说明**：读取文本文件全部内容，并统计总行数。
**入参**：
- `file_path` (str)：目标文件路径
- `charset` (str, 默认utf-8)：文件编码

**原生业务返回**：`{"n_lines": int, "content": str}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_read_text_range
**功能说明**：分段读取文本，支持跳过前置行、限制读取行数，适配超大文本文件。
**入参**：
- `file_path` (str)：目标文件路径
- `lines_to_skip` (int, 默认0)：跳过开头行数
- `max_lines_to_read` (int, 默认0)：限制读取行数（-1代表无限制）
- `line_separator` (str, 默认"\n")：换行分隔符
- `charset` (str, 默认utf-8)：文件编码

**原生业务返回**：`{"n_lines": int, "content": str}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_read_binary_chunk
**功能说明**：二进制文件分块读取，返回Base64编码数据，保障传输安全。
**入参**：
- `file_path` (str)：目标文件路径
- `bytes_to_skip` (int, 默认0)：跳过开头字节
- `max_bytes_to_read` (int, 默认0)：限制读取字节（-1代表无限制）

**原生业务返回**：`{"actual_length": int, "end_of_stream": boolean, "data_base64": str}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_write_text
**功能说明**：写入文本内容，支持覆写与追加两种模式。
**入参**：
- `file_path` (str)：目标文件路径
- `text` (str)：待写入文本内容
- `append` (boolean, 默认false)：追加写入开关

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_write_binary
**功能说明**：分段二进制写入，支持偏移切片与追加模式。
**入参**：
- `file_path` (str)：目标文件路径
- `data` (bytes)：待写入二进制数据
- `offset` (int, 默认0)：数据写入偏移量
- `length` (int, 默认-1)：写入数据长度（-1写入全部缓冲区）
- `append` (boolean, 默认false)：追加写入开关

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

### 四、检索与替换工具
#### fs_search_files_by_content
**功能说明**：扫描目录，检索所有包含目标内容的文件，支持正则、忽略大小写、文件类型过滤。
**入参**：
- `dir_path` (str)：检索根目录
- `recursive` (boolean)：递归检索开关
- `search_term` (str)：检索关键词/正则表达式
- `is_regex` (boolean, 默认false)：是否启用正则匹配
- `ignore_case` (boolean, 默认true)：忽略大小写匹配
- `file_type` (str, 默认空)：文件后缀过滤
- `charset` (str, 默认utf-8)：文件编码

**原生业务返回**：`{"file_paths": list[str]}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_search_in_files_by_content
**功能说明**：多文件内容匹配，返回带自定义前后上下文行的匹配片段，支持限制结果数量。
**入参**：
- `dir_path` (str)：检索根目录
- `recursive` (boolean)：递归检索开关
- `search_term` (str)：检索关键词/正则表达式
- `is_regex` (boolean)：正则启用开关
- `ignore_case` (boolean)：忽略大小写开关
- `limit` (int)：最大匹配结果条数
- `lines_before` (int)：匹配行前置上下文行数
- `lines_after` (int)：匹配行后置上下文行数
- `file_extension` (str, 默认空)：文件后缀过滤
- `charset` (str, 默认utf-8)：文件编码

**原生业务返回**：携带文件路径与行上下文的结构化匹配列表
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_search_in_file_by_content
**功能说明**：单文件精准内容检索，返回携带上下文的匹配内容。
**入参**：
- `file_path` (str)：目标文件路径
- `search_term` (str)：检索关键词/正则表达式
- `is_regex` (boolean)：正则启用开关
- `ignore_case` (boolean)：忽略大小写开关
- `lines_before` (int)：匹配行前置上下文行数
- `lines_after` (int)：匹配行后置上下文行数
- `charset` (str, 默认utf-8)：文件编码

**原生业务返回**：单文件结构化匹配列表
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_file_replace
**功能说明**：单文件原地文本替换，返回修改总行数。
**入参**：
- `file_path` (str)：目标文件路径
- `search_term` (str)：待替换原文内容
- `replacement` (str)：替换新内容
- `line_separator` (str, 默认"\n")：换行分隔符

**原生业务返回**：`{"replaced_line_count": int}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

### 五、图像处理与OCR工具
#### fs_image_resize
**功能说明**：缩放图片至指定尺寸，支持锁定原图比例、边缘填充适配目标尺寸。
**入参**：
- `source_image_path` (str)：源图片路径
- `dest_image_path` (str)：输出图片路径
- `keep_aspect_ratio` (boolean)：锁定原图宽高比
- `pad_to_target` (boolean)：空白填充适配目标尺寸
- `target_width` (int)：目标图片宽度
- `target_height` (int)：目标图片高度

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_image_crop
**功能说明**：从原图裁剪矩形区域并导出新图片文件。
**入参**：
- `source_image_path` (str)：源图片路径
- `dest_image_path` (str)：输出图片路径
- `x` (int)：裁剪起始X坐标
- `y` (int)：裁剪起始Y坐标
- `width` (int)：裁剪区域宽度
- `height` (int)：裁剪区域高度

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_image_rotate
**功能说明**：顺时针旋转图片，自动扩展画布保留完整图像内容。
**入参**：
- `source_image_path` (str)：源图片路径
- `dest_image_path` (str)：输出图片路径
- `degrees` (float)：顺时针旋转角度

**原生业务返回**：`{"success": boolean}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

#### fs_ocr_extract_text
**功能说明**：基于Tesseract引擎提取图片文字，支持多语言识别。
**入参**：
- `image_path` (str)：目标图片路径
- `tesseract_cmd_path` (str)：Tesseract可执行程序路径
- `lang` (str)：识别语言编码
- `tessdata_path` (str, 默认空)：Tesseract语言数据包路径

**原生业务返回**：`{"ocr_text": str}`
> 以上原生业务数据会被外层统一`res`结构包装，参考上方全局返回规范。

## 开源许可证
本项目基于 **Apache License 2.0** 开源，完整协议文本可查看项目根目录下 `LICENSE` 文件。

## 第三方依赖许可说明
项目依赖chardet、Pillow、python-magic、ffmpeg-python等开源库，所有第三方组件均遵循各自开源许可规范。

备注：本项目未内置FFmpeg二进制程序，使用时请自行遵循FFmpeg官方许可条款。

## 仓库与联系方式
GitHub地址：[https://github.com/kurtzhi/fsext-mcp-server-python](https://github.com/kurtzhi/fsext-mcp-server-python)