# AWS Lambda 日志转发配置手册

本文档说明如何在 AWS 中配置 Lambda，将 S3、CloudWatch Logs 或 EventBridge 中的日志和事件转发到观测云日志。

## 适用场景

可用于以下 AWS 日志转发场景：

| 日志来源 | 推荐触发方式 |
| --- | --- |
| ALB / ELB 访问日志 | S3 触发器 |
| CloudTrail 日志 | S3 触发器 |
| VPC Flow Logs 投递到 S3 | S3 触发器 |
| WAF 日志投递到 S3 | S3 触发器 |
| CloudFront 日志投递到 S3 | S3 触发器 |
| Lambda / RDS / API Gateway / Step Functions / EKS 等 CloudWatch Logs 日志组 | CloudWatch Logs 订阅筛选器 |
| AWS 服务事件 | EventBridge 规则 |

## 配置前准备

配置前请确认：

1. 已准备 DataKit 或 DataWay 接入地址。
2. Lambda 所在网络可以访问 DataKit 或 DataWay。
3. 已获取工作空间 Token，或已确认 DataKit 地址和端口。
4. 如果日志来源是 S3，目标日志文件已正常写入 S3 bucket。
5. 如果日志来源是 CloudWatch Logs，日志组和 Lambda 建议在同一个 AWS Region。

## 创建 Lambda 函数

1. 打开 AWS Lambda 控制台。
2. 选择创建函数。
3. 选择从头开始创作。
4. 输入函数名称，例如 `guance-log-forwarder`。
5. Runtime 选择 AWS 当前支持的 Python 3 运行时，例如 Python 3.12 或更新版本。
6. Architecture 按账号标准选择 `x86_64` 或 `arm64`。
7. Execution role 可以先选择创建具有基本 Lambda 权限的新角色，后续再补充 S3 或 KMS 权限。
8. 创建函数。

## 上传函数代码

建议使用 ZIP 包上传。如果已经有发布包，直接在 Lambda 控制台上传即可。

如果需要从仓库源码打包，在仓库根目录执行：

```bash
cd AWS/Lambda
zip guance-aws-lambda-forwarder.zip lambda_forward.py setting.py datakit.py dataway.py
```

然后在 Lambda 控制台上传 `AWS/Lambda/guance-aws-lambda-forwarder.zip`。

注意：ZIP 包根目录必须直接包含 `lambda_forward.py`、`setting.py`、`datakit.py`、`dataway.py`，不能放在 `AWS/Lambda/` 子目录中。

上传后确认 Runtime settings：

| 配置项 | 值 |
| --- | --- |
| Handler | `lambda_forward.lambda_handler` |
| Runtime | Python 3 |

建议资源配置：

| 配置项 | 建议值 |
| --- | --- |
| Timeout | 从 60 秒开始，根据单个日志文件大小调整 |
| Memory | 从 256 MB 开始，根据日志量调整 |

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
| `LOG_LEVEL` | `INFO` | Lambda 运行日志级别，可设置为 `DEBUG` 排查问题 |
| `TAGS` | 空 | 附加标签，格式为 `key:value,key2:value2` |

如果同时配置了 `DATAKIT_IP` 和 `DATAWAY_URL`，函数会优先使用 DataKit。

## 配置执行角色权限

Lambda 执行角色需要写入 CloudWatch Logs，用于保存函数运行日志。

基础权限示例：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

如果使用 S3 触发器，还需要允许读取目标日志对象：

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject"
  ],
  "Resource": "arn:aws:s3:::YOUR_BUCKET/YOUR_PREFIX/*"
}
```

如果 S3 对象使用 SSE-KMS 加密，还需要允许解密对应 KMS Key：

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt"
  ],
  "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
}
```

请将示例中的 `YOUR_BUCKET`、`YOUR_PREFIX`、`REGION`、`ACCOUNT_ID`、`KEY_ID` 替换为实际值。

## 配置 S3 触发器

适用于 ALB / ELB、CloudTrail、VPC Flow Logs、WAF、CloudFront 等投递到 S3 的日志。

1. 打开 Lambda 函数页面。
2. 选择添加触发器。
3. 触发源选择 `S3`。
4. Bucket 选择日志所在 bucket。
5. Event type 选择对象创建事件，例如 `s3:ObjectCreated:*`。
6. 根据日志路径配置 Prefix 或 Suffix，避免无关对象触发函数。
7. 确认并保存触发器。

注意事项：

1. 不要让该 Lambda 的输出写回同一个触发前缀，避免递归调用。
2. 大文件会增加 Lambda 运行时间和内存消耗，需要按实际日志量调整 Timeout 和 Memory。
3. 如果日志对象由其他账号写入，请同时检查 bucket policy 和对象所有权配置。

## 配置 CloudWatch Logs 订阅

适用于已经写入 CloudWatch Logs 的日志组。

1. 打开 CloudWatch Logs 控制台。
2. 进入目标日志组。
3. 创建 Subscription filter。
4. Destination 选择当前 Lambda 函数。
5. Filter pattern 按需填写；如需转发全部日志，可以使用空 pattern。
6. 保存订阅配置。

注意事项：

1. 一个日志组可配置的订阅筛选器数量有限，请确认不会影响已有订阅。
2. CloudWatch Logs 需要拥有调用 Lambda 的权限。通过控制台配置时通常会自动添加；通过 IaC 或 CLI 配置时请显式添加 Lambda invoke permission。

## 配置 EventBridge 规则

适用于转发 AWS 服务事件。

1. 打开 EventBridge 控制台。
2. 创建 Rule。
3. 配置 Event pattern。
4. Target 选择当前 Lambda 函数。
5. 保存规则。

通过 IaC 或 CLI 配置时，请确认 EventBridge 拥有调用 Lambda 的权限。

## 验证配置

完成配置后按以下步骤验证：

1. 在 Lambda 控制台确认 handler 为 `lambda_forward.lambda_handler`。
2. 确认已配置 DataKit 或 DataWay 环境变量。
3. 触发一条测试日志或等待真实日志进入。
4. 打开 Lambda 的 CloudWatch Logs，确认函数执行成功。
5. 在观测云日志中查询数据。

## 常见问题

| 问题 | 可能原因 | 处理方式 |
| --- | --- | --- |
| Lambda 报 `Unable to import module` | ZIP 包结构不正确，或 handler 配置错误 | 确认 ZIP 根目录包含 `lambda_forward.py`、`setting.py`、`datakit.py`、`dataway.py`，并确认 handler 为 `lambda_forward.lambda_handler` |
| Lambda 报环境变量错误 | 未配置上报目标 | 配置 `DATAKIT_IP`，或同时配置 `DATAWAY_URL` 和 `WORKSPACE_TOKEN` |
| 读取 S3 报 `AccessDenied` | 执行角色缺少 S3 或 KMS 权限 | 补充 `s3:GetObject`，如使用 KMS 加密则补充 `kms:Decrypt` |
| Lambda 执行成功但平台无数据 | Lambda 无法访问 DataKit / DataWay，或 token / 地址错误 | 检查 VPC、Security Group、NAT、路由、DataWay 地址和 Token |
| S3 有新日志但 Lambda 未触发 | S3 事件通知、Prefix / Suffix 或 Lambda invoke permission 配置不正确 | 检查 S3 触发器配置和 Lambda 权限 |
| CloudWatch Logs 未触发 Lambda | 订阅筛选器或调用权限配置不正确 | 检查 subscription filter 和 Lambda resource-based policy |

## 参考文档

1. AWS Lambda 运行时：https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html
2. S3 触发 Lambda：https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html
3. CloudWatch Logs 订阅筛选器：https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html
4. EventBridge 调用 Lambda：https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-run-lambda-schedule.html
