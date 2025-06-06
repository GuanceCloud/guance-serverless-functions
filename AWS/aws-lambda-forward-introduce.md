# 采集器「AWS-Lambda」配置手册

通过 AWS 中的 Lambda 对 AWS 中的 S3 数据进行抓取并上报到观测云日志中。

## 例如日志源为 ALB 时创建访问日志打到 S3

1. 创建 S3 存储桶

2. 将策略附加到 S3 存储桶

3. 配置访问日志

4. 确认存储桶权限

> 具体步骤可看「[为 Application Load Balancer 启用访问日志]( https://docs.amazonaws.cn/elasticloadbalancing/latest/application/enable-access-logging.html)」

## 配置 Lambda

### 一、使用控制台创建 Lambda 函数

1. 打开 Lamba 控制台的函数页面。

2. 选择创建函数

3. 选择从头开始创作

4. 输入函数名称

5. 设置 `运行时` 选项为 `Python 3.10`

6. 在 Execution Role（执行角色）中，选择 Create a new role with basic Lambda permissions（创建具有基本 Lambda 权限的新角色，具体权限列表请见附录，如已存在可使用最小权限角色可直接使用）。Lambda 创建执行角色，该角色授予函数上载日志到 Amazon CloudWatchlogs 的权限。在您调用函数时，Lambda 函数担任执行角色，并使用该执行角色为 Amazon 软件开发工具包创建凭证和从事件源读取数据

7. 点击创建函数

8. 在 GitHub 中拉取同步代码至下方代码源中将 lambda-forward.py 内容复制到 lambda-function.py 中

9. 在将 lambda-function.py 相同目录下新建 setting.py、datakit.py、dataway.py 文件，并将 GitHub 中相应文件代码复制进去

10. 添加环境变量

    1. DATAKIT_IP：datakit 部署的 ip 地址，上报数据源为 datakit，必选
    2. DATAKIT_PORT：datakit 服务端口，上报数据源为 datakit，非必选，默认：`9529`
    3. GUANCE_NODE：观测云节点，上报数据源为 dataway，必选，选值范围：
       - `default`: 中国区 1（杭州）
       - `aws`: 中国区 2（宁夏）
       - `cn4`: 中国区 4（广州）
       - `cn6`: 中国区 6（香港）
       - `us1`: 美洲区 1（俄勒冈）
       - `eu1`: 欧洲区 1（法兰克福）
       - `ap1`: 亚太区 1（新加坡）
       - `za1`: 非洲区 1（南非）
       - `id1`: 印尼区 1（雅加达）
    4. GUANCE_TOKEN：观测云工作空间 `Token`，上报数据源为 dataway

    **注意：上报数据源 datakit 与 dataway 必选一个，选择 datakit 请配置`DATAKIT_IP`， 选择 dataway 请配置`GUANCE_NODE`、`GUANCE_TOKEN`**

11. 如果 datakit 端口不是默认的 `9529` 可添加环境变量 DATAKIT_PORT 填写为正确的端口地址（`此变量非必填`）

12. 点击 `Depoly` 发布

### 二、配置 Lambda 触发器

1. 点击 `添加触发器`

2. 设置`选择一个源`为`S3` 

3. 选择需要监听的`bucket`

4. 选择要触发 Lambda 函数的事件 Event types

5. 同意 我承认不推荐对输入和输出使用相同的 S3 bucket，并且这种配置可能导致递归调用、增加 Lambda 使用和增加成本

6. 点击添加

## X. 附录

### 操作所需最小权限

logs: CreateLogGroup
logs: CreateLogStream
logs: PutLogEvents
lambda: *
AmazonS3ReadOnlyAccess