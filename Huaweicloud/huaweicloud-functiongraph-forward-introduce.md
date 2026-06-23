# 华为云 FunctionGraph 日志转发配置手册

本文档说明如何在华为云中配置 FunctionGraph，将 LTS 或 OBS 中的日志转发到观测云日志。

## 适用场景

| 日志来源 | 推荐触发方式 |
| --- | --- |
| 云日志服务 LTS 日志 | LTS 触发器 |
| OBS 中的日志文件 | OBS 触发器 |

说明：

1. LTS 触发器适合已经写入云日志服务的日志组。
2. OBS 触发器适合对象写入 OBS 后触发函数读取日志文件。

## 配置前准备

配置前请确认：

1. 已准备 DataKit 或 DataWay 接入地址。
2. FunctionGraph 所在网络可以访问 DataKit 或 DataWay。
3. 已获取工作空间 Token，或已确认 DataKit 地址和端口。
4. 如果使用 LTS 触发器，目标日志组和日志流已经存在。
5. 如果使用 OBS 触发器，目标 bucket 已经存在，并且日志对象会写入该 bucket。

## 创建 FunctionGraph 函数

1. 打开华为云 FunctionGraph 控制台。
2. 进入函数列表，选择创建函数。
3. 函数类型选择事件函数。
4. 输入函数名称，例如 `guance-log-forwarder`。
5. 运行时选择当前 FunctionGraph 支持的 Python 3 运行时，例如 Python 3.9、Python 3.10 或 Python 3.12。
6. 委托可以先选择已有委托；如果没有合适委托，可先创建函数，后续按触发器类型补充委托权限。
7. 创建函数。

## 上传函数代码

建议使用 ZIP 包上传。如果已经有发布包，直接在 FunctionGraph 控制台上传即可。

如果需要从仓库源码打包，在仓库根目录执行：

```bash
cd Huaweicloud/Functiongraph
zip guance-functiongraph-forwarder.zip index.py setting.py datakit.py dataway.py
```

然后在 FunctionGraph 控制台上传 `Huaweicloud/Functiongraph/guance-functiongraph-forwarder.zip`。

注意：ZIP 包根目录必须直接包含 `index.py`、`setting.py`、`datakit.py`、`dataway.py`，不能放在 `Huaweicloud/Functiongraph/` 子目录中。

上传后确认函数配置：

| 配置项 | 值 |
| --- | --- |
| 执行入口 / Handler | `index.handler` |
| 运行时 | Python 3 |

建议资源配置：

| 配置项 | 建议值 |
| --- | --- |
| 超时时间 | 从 60 秒开始，根据单个日志文件大小调整 |
| 内存 | 从 256 MB 开始，根据日志量调整 |

如果使用 OBS 触发器，函数会读取 OBS 对象。若运行时报 `No module named 'obs'`，请为函数添加 OBS Python SDK 依赖，或将 OBS SDK 一起打入 ZIP 包。

## 配置环境变量

DataKit 和 DataWay 选择一种配置即可。

### 方式一：上报到 DataKit

| 变量名 | 必填 | 示例 | 说明 |
| --- | --- | --- | --- |
| `DATAKIT_IP` | 是 | `10.0.1.10` | DataKit 地址 |
| `DATAKIT_PORT` | 否 | `9529` | DataKit 写入端口，默认 `9529` |

示例：

```text
DATAKIT_IP=10.0.1.10
DATAKIT_PORT=9529
```

### 方式二：上报到 DataWay

| 变量名 | 必填 | 示例 | 说明 |
| --- | --- | --- | --- |
| `DATAWAY_URL` | 是 | `https://openway.guance.com` | DataWay 地址 |
| `WORKSPACE_TOKEN` | 是 | `tkn_xxxxxxxxxxxxxxxxxxxx` | 工作空间 Token |

示例：

```text
DATAWAY_URL=https://openway.guance.com
WORKSPACE_TOKEN=tkn_xxxxxxxxxxxxxxxxxxxx
```

### 可选变量

| 变量名 | 默认值 | 说明 |
| --- | --- | --- |
| `HTTP_TIMEOUT` | `5` | 上报请求超时时间，单位为秒 |
| `LOG_LEVEL` | `INFO` | 函数运行日志级别，可设置为 `DEBUG` 排查问题 |

如果同时配置了 `DATAKIT_IP` 和 `DATAWAY_URL`，函数会优先使用 DataKit。

## 配置委托权限

FunctionGraph 函数需要通过委托访问其他云服务。请根据触发器类型配置委托权限。

### LTS 触发器

如果只使用 LTS 触发器，函数本身不主动读取 LTS 日志；日志由 LTS 触发器传入函数。创建触发器时，操作用户需要具备 FunctionGraph 和 LTS 触发器配置权限。

建议确认以下权限：

```text
FunctionGraph 函数管理和触发器配置权限
LTS 日志组、日志流读取和触发器配置权限
```

### OBS 触发器

如果使用 OBS 触发器，函数会根据触发事件读取对应 OBS 对象。函数委托需要具备目标 bucket 的对象读取权限。

建议授予最小权限：

```text
obs:object:GetObject
```

权限范围建议限制到实际 bucket 和日志对象前缀。

如果 OBS 对象使用服务端加密，请同时确认函数委托具备读取该对象所需的解密权限。

## 配置 LTS 触发器

适用于已经写入云日志服务 LTS 的日志。

1. 打开 FunctionGraph 函数详情页。
2. 进入触发器配置。
3. 选择创建触发器。
4. 触发器类型选择云日志服务 LTS。
5. 选择目标日志组和日志流。
6. 保存触发器。

注意事项：

1. 建议先选择少量日志流验证，确认数据进入平台后再扩大范围。
2. 如果日志量较大，请根据实际吞吐调整函数内存和超时时间。
3. 若已有其他订阅或触发配置，请确认不会重复转发同一批日志。

## 配置 OBS 触发器

适用于日志文件写入 OBS 后触发函数读取并转发的场景。

1. 打开 FunctionGraph 函数详情页。
2. 进入触发器配置。
3. 选择创建触发器。
4. 触发器类型选择对象存储服务 OBS。
5. 选择日志所在 bucket。
6. 事件类型选择对象创建事件，例如 `ObjectCreated` 或 `Put`。
7. 根据日志路径配置前缀或后缀过滤，避免无关对象触发函数。
8. 保存触发器。

注意事项：

1. 函数和 OBS bucket 建议位于同一区域。
2. 不要让该函数的输出写回同一个触发前缀，避免递归调用。
3. 大文件会增加函数运行时间和内存消耗，需要按实际日志量调整超时时间和内存。
4. 如果 OBS 触发器在当前区域不可用，请以 FunctionGraph 控制台实际支持情况为准。

## 验证配置

完成配置后按以下步骤验证：

1. 在 FunctionGraph 控制台确认执行入口为 `index.handler`。
2. 确认已配置 DataKit 或 DataWay 环境变量。
3. 触发一条测试日志，或等待真实日志进入 LTS / OBS。
4. 查看 FunctionGraph 运行日志，确认函数执行成功。
5. 在观测云日志中查询数据。

建议优先按以下字段检索：

```text
functiongraph_function_name
functiongraph_request_id
```

OBS 触发器还可以按以下字段检索：

```text
bucket_name
object_key
```

## 常见问题

| 问题 | 可能原因 | 处理方式 |
| --- | --- | --- |
| 函数报导入模块失败 | ZIP 包结构不正确，或缺少依赖 | 确认 ZIP 根目录包含 `index.py`、`setting.py`、`datakit.py`、`dataway.py`；OBS 场景若缺少 `obs` 模块，请添加 OBS Python SDK 依赖 |
| 函数报环境变量错误 | 未配置上报目标 | 配置 `DATAKIT_IP`，或同时配置 `DATAWAY_URL` 和 `WORKSPACE_TOKEN` |
| OBS 读取失败 | 委托缺少 OBS 读取权限，或对象加密权限不足 | 检查函数委托、bucket 权限、对象前缀权限和加密权限 |
| 函数执行成功但平台无数据 | 函数无法访问 DataKit / DataWay，或 token / 地址错误 | 检查 VPC、子网、安全组、路由、DataWay 地址和 Token |
| LTS 日志未触发函数 | LTS 触发器配置、日志组/日志流选择或权限不正确 | 检查 LTS 触发器状态和 FunctionGraph 调用权限 |
| OBS 有新对象但函数未触发 | OBS 触发器事件类型、前缀/后缀或区域配置不正确 | 检查 OBS 触发器配置，确认 bucket 与函数区域匹配 |

## 参考文档

1. FunctionGraph 支持的运行时：https://support.huaweicloud.com/usermanual-functiongraph/functiongraph_01_0151.html
2. FunctionGraph Python 事件函数开发：https://support.huaweicloud.com/intl/zh-cn/devg-functiongraph/functiongraph_02_0420.html
3. FunctionGraph OBS 触发器：https://support.huaweicloud.com/usermanual-functiongraph/functiongraph_01_0205.html
4. FunctionGraph 委托权限配置：https://support.huaweicloud.com/usermanual-functiongraph/functiongraph_01_0920.html
5. OBS Python SDK 安装：https://support.huaweicloud.com/sdk-python-devg-obs/obs_22_0400.html
