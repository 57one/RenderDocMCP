# RenderDoc MCP 服务器

作为 RenderDoc UI 扩展运行的 MCP 服务器。支持 AI 助手访问 RenderDoc 的捕获数据，辅助 DirectX 11/12 图形调试。

## 架构

**混合进程隔离方式**:

```
Claude/AI Client (stdio)
        │
        ▼
MCP Server Process (标准Python + FastMCP 2.0)
        │ 基于文件的IPC (%TEMP%/renderdoc_mcp/)
        ▼
RenderDoc Process (Extension + File Polling)
```

## 项目结构

```
RenderDocMCP/
├── mcp_server/                        # MCP服务器
│   ├── server.py                      # FastMCP入口点
│   ├── config.py                      # 配置
│   └── bridge/
│       └── client.py                  # 基于文件的IPC客户端
│
├── renderdoc_extension/               # RenderDoc扩展
│   ├── __init__.py                    # register()/unregister()
│   ├── extension.json                 # 清单文件
│   ├── socket_server.py               # 基于文件的IPC服务器
│   ├── request_handler.py             # 请求处理
│   └── renderdoc_facade.py            # RenderDoc API封装
│
└── scripts/
    └── install_extension.py           # 扩展安装脚本
```

## MCP 工具列表

| 工具名称 | 说明 |
|---------|------|
| `list_captures` | 获取指定目录下的 .rdc 文件列表 |
| `open_capture` | 打开捕获文件（已有捕获会自动关闭） |
| `get_capture_status` | 确认捕获加载状态 |
| `get_draw_calls` | 获取 Draw Call 列表（层级结构，支持过滤） |
| `get_frame_summary` | 获取帧整体统计信息（Draw Call 数量、Marker 列表等） |
| `find_draws` | 统一反向查找接口：通过 Shader 名称 / 纹理名称 / 资源 ID 查找 Draw Call（`by` 参数指定模式） |
| `get_draw_call_details` | 获取特定 Draw Call 的详细信息 |
| `get_action_timings` | 获取 Action 的 GPU 执行时间 |
| `get_shader_info` | 获取 Shader 源码/常量缓冲区 |
| `get_buffer_contents` | 获取 Buffer 数据（可指定偏移/长度） |
| `get_texture_info` | 获取纹理元数据 |
| `get_texture_data` | 获取纹理像素数据（支持 mip/slice/3D切片） |
| `get_pipeline_state` | 获取完整管线状态 |

### get_draw_calls 过滤选项

```python
get_draw_calls(
    include_children=True,      # 包含子 Action
    marker_filter="Camera.Render",  # 仅获取该 Marker 下的内容
    exclude_markers=["GUI.Repaint", "UIR.DrawChain"],  # 排除指定 Marker
    event_id_min=7372,          # event_id 范围起始
    event_id_max=7600,          # event_id 范围结束
    only_actions=True,          # 排除 Marker（仅 Draw Call）
    flags_filter=["Drawcall", "Dispatch"],  # 仅指定 Flag 类型
)
```

### 捕获管理工具

```python
# 枚举目录内的捕获文件
list_captures(directory="D:\\captures")
# → {"count": 3, "captures": [{"filename": "game.rdc", "path": "...", "size_bytes": 12345, "modified_time": "..."}, ...]}

# 打开捕获文件（已有捕获会自动关闭）
open_capture(capture_path="D:\\captures\\game.rdc")
# → {"success": true, "filename": "game.rdc", "api": "D3D11"}
```

### 反向查找工具（统一接口）

```python
# 通过 Shader 名称搜索（部分匹配）
find_draws(by="shader", shader_name="Toon", stage="pixel")

# 通过纹理名称搜索（部分匹配）
find_draws(by="texture", texture_name="CharacterSkin")

# 通过资源 ID 搜索（精确匹配）
find_draws(by="resource", resource_id="ResourceId::12345")
```

### GPU 时序获取

```python
# 获取所有 Action 的时序信息
get_action_timings()
# → {"available": true, "unit": "CounterUnit.Seconds", "timings": [...], "total_duration_ms": 12.5, "count": 150}

# 仅获取特定 event ID 的时序
get_action_timings(event_ids=[100, 200, 300])

# 通过 Marker 过滤
get_action_timings(marker_filter="Camera.Render", exclude_markers=["GUI.Repaint"])
```

**注意**：GPU 时序计数器可能因硬件/驱动不同而无法使用。
若返回 `available: false`，则该捕获无法获取时序信息。

## 通信协议

基于文件的 IPC：
- IPC 目录：`%TEMP%/renderdoc_mcp/`
- `request.json`：请求文件（MCP 服务器 → RenderDoc）
- `response.json`：响应文件（RenderDoc → MCP 服务器）
- `lock`：写入中的锁文件
- 轮询间隔：100ms（RenderDoc 侧）

## 开发说明

- 由于 RenderDoc 内置 Python 不含 socket/QtNetwork 模块，因此采用基于文件的 IPC
- RenderDoc 扩展仅使用 Python 3.6 标准库
- 访问 ReplayController 需通过 `BlockInvoke` 进行

## 参考链接

- [FastMCP](https://github.com/jlowin/fastmcp)
- [RenderDoc Python API](https://renderdoc.org/docs/python_api/index.html)
- [RenderDoc Extension Registration](https://renderdoc.org/docs/how/how_python_extension.html)
