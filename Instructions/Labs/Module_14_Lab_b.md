---
lab:
    title: '14B：配置基于消息的集成架构'
    module: '模块 14：实现应用程序基础结构'
---

# 实验室：配置基于消息的集成架构

# 学生实验室手册

## 实验室场景

Adatum 公司有多个 Web 应用程序，可以定期处理上传到本地文件服务器的文件。文件大小各不相同，但最高可达 100 MB。Adatum 正在考虑将应用程序迁移到 Azure App Service 或基于 Azure Functions 的应用程序，并使用 Azure Storage 来托管上传的文件。你计划测试两种场景：

- 使用 Azure Functions 自动处理上传到 Azure Storage 容器的新 Blob。

  - 使用事件网格生成 Azure Storage 队列消息，该消息将引用上传到 Azure Storage 容器的新 Blob。

这些场景旨在解决基于消息传递的体系结构在发送、接收和操作大型消息时遇到的常见挑战。建议不要直接将大型消息发送到消息队列，因为这类消息需要使用更多资源，导致消耗更多带宽，并对处理速度产生负面影响，因为消息传递平台通常经过微调以处理大量小型消息。此外，消息传递平台通常会限制可以处理的最大消息大小。

一种可能的解决方案是将消息有效负载存储到外部服务（如 Azure Blob Store、Azure SQL 或 Azure Cosmos DB）中，获取对存储的有效负载的引用，然后仅将该引用发送到消息总线。这种架构模式称为“索物标签” (claim check)。如果需要，对处理该特定消息感兴趣的客户可以使用已获得的引用来检索有效负载。

在 Azure 上，这种模式可以通过多种方式和不同的技术实现，但它通常依赖于事件来自动生成声明检查并将其推送到消息总线以供客户端使用，或直接触发有效负载处理。此类实现中包含的常见组件之一是事件网格，它是一个事件路由服务，负责在可配置的时间段（最多 24 小时）内传递事件。之后，事件被丢弃或成为死信。如果需要事件内容的归档或事件流的可重放性，则可以通过设置事件中心的事件网格订阅或 Azure Storage 中的队列来促进满足此要求，其中消息可以保留更长时间并且支持消息的归档。

在本实验中，你将使用 Azure Storage Blob 服务来存储要处理的文件。客户端只需要将要共享的文件放入指定的 Azure Blob 容器中。在第一个练习中，文件将由 Azure Function 直接使用，充分利用其无服务器性质。你还将利用 Application Insights 提供检测，便于监视和分析文件处理。在第二个练习中，你将使用事件网格自动生成索物标签消息并将其发送到 Azure Storage 队列。这允许客户端应用程序轮询队列，获取消息，然后使用存储的参考数据直接从 Azure Blob 存储下载有效负载。

值得注意的是，第一个练习中的 Azure Function 依赖于 Blob 存储触发器。在处理以下要求时，你应该选择事件网格触发器而不是 Blob 存储触发器：

- 仅限 blob 的存储帐户：blob 存储帐户支持 blob 输入和输出绑定，但不支持 blob 触发器。Blob 存储触发器需要通用存储帐户。

  - 高规模：高规模可以松散地定义为容器中包含超过 100,000 个 blob，或者存储帐户中每秒有超过 100 个 blob 更新。

  - 减少延迟：如果你的函数应用程序处于消耗计划中，而且函数应用程序空闲，则处理新 blob 的过程中可能会有 10 分钟的延迟。要避免此延迟，你可以使用事件网格触发器或切换到启用了“始终开启”设置的应用服务计划。

  - 处理 blob 删除事件：blob 存储触发器不支持 blob 删除事件。

## 目标
  
完成本实验室后，你将能够：

- 配置并验证 Azure Function 应用程序 Blob 存储触发器

- 配置并验证基于 Azure 事件网格订阅的队列消息传递

## 实验室环境
  
预计用时：60 分钟

## 实验室文件

无

## 说明

### 练习 1：配置并验证 Azure Function 应用程序 Blob 存储触发器
  
本练习的主要任务如下：

1. 配置 Azure Function 应用程序 Blob 存储触发器

1. 验证 Azure Function 应用程序 Blob 存储触发器

#### 任务 1：配置 Azure Function 应用程序 Blob 存储触发器

1. 在实验室计算机上，启动 Web 浏览器，导航至[“Azure 门户”](https://portal.azure.com)，然后通过提供要在本实验中使用的订阅中的所有者角色的用户帐户凭据来登录。

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 如果提示选择 **“Bash”** 或 **“PowerShell”**，请选择 **“Bash”**。

    >**注意**： 如果这是你第一次打开 **“Cloud Shell”**，会看到 **“未装载任何存储”** 消息，请选择你在本实验室中使用的订阅，然后选择 **“创建存储”**。

1. 在 Cloud Shell 窗格中，运行以下命令以生成伪随机字符串，该字符串将用作你将在本练习中配置的资源名称的前缀：

   ```sh
   export PREFIX=$(echo `openssl rand -base64 5 | cut -c1-7 | tr '[:upper:]' '[:lower:]' | tr -cd '[[:alnum:]]._-'`)
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以指定要在本实验室中为其预配资源的 Azure 区域（将 `<Azure region>` 占位符替换为订阅中可用的且最接近实验室计算机位置的 Azure 区域名称）：

   ```sh
   export LOCATION='<Azure region>'
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建一个资源组，该资源组将托管你将在本实验中配置的所有资源：

   ```sh
   export RESOURCE_GROUP_NAME='az30314b-labRG'

   az group create --name "${RESOURCE_GROUP_NAME}" --location "$LOCATION"
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以创建 Azure 存储帐户，该帐户将托管由 Azure Function 处理包含 Blob 的容器：

   ```sh
   export STORAGE_ACCOUNT_NAME="az30314b${PREFIX}"

   export CONTAINER_NAME="workitems"

   export STORAGE_ACCOUNT=$(az storage account create --name "${STORAGE_ACCOUNT_NAME}" --kind "StorageV2" --location "${LOCATION}" --resource-group "${RESOURCE_GROUP_NAME}" --sku "Standard_LRS")
   ```

    > **注意**： Azure Function 也将使用相同的存储帐户来满足自身的处理要求。在实际场景中，你可能需要考虑为此目的创建单独的存储帐户。

1. 在 Cloud Shell 窗格中，运行以下命令以创建存储 Azure Storage 帐户的连接字符串属性值的变量：

   ```sh
   export STORAGE_CONNECTION_STRING=$(az storage account show-connection-string --name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" -o tsv)
   ```

1. 在“Cloud Shell”窗格中运行以下命令以创建容器，该容器将托管由 Azure Function 处理的 Blob：

   ```sh
   az storage container create --name "${CONTAINER_NAME}" --account-name "${STORAGE_ACCOUNT_NAME}" --connection-string "${STORAGE_CONNECTION_STRING}"
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建 Application Insights 资源，该资源将监视 Azure Function 处理 blob 并将其密钥存储在一个变量中：

   ```sh
   export APPLICATION_INSIGHTS_NAME="az30314bi${PREFIX}"

   az resource create --name "${APPLICATION_INSIGHTS_NAME}" --location "${LOCATION}" --properties '{"Application_Type": "other", "ApplicationId": "function", "Flow_Type": "Redfield"}' --resource-group "${RESOURCE_GROUP_NAME}" --resource-type "Microsoft.Insights/components"

   export APPINSIGHTS_KEY=$(az resource show --name "${APPLICATION_INSIGHTS_NAME}" --query "properties.InstrumentationKey" --resource-group "${RESOURCE_GROUP_NAME}" --resource-type "Microsoft.Insights/components" -o tsv)
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建 Azure Function，以处理与 Azure Storageblob 的创建相对应的事件：

   ```sh
   export FUNCTION_NAME="az30314f${PREFIX}"

   az functionapp create --name "${FUNCTION_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --app-insights "$APPLICATION_INSIGHTS_NAME" --app-insights-key "$APPINSIGHTS_KEY" --storage-account "${STORAGE_ACCOUNT_NAME}" --consumption-plan-location "${LOCATION}" --runtime "dotnet" --functions-version 2
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以配置新创建的函数的应用程序设置，将其链接到 Application Insights 和 Azure Storage 帐户：

   ```sh
   az functionapp config appsettings set --name "${FUNCTION_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --settings "APPINSIGHTS_INSTRUMENTATIONKEY=$APPINSIGHTS_KEY" FUNCTIONS_EXTENSION_VERSION=~2

   az functionapp config appsettings set --name "${FUNCTION_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --settings "STORAGE_CONNECTION_STRING=$STORAGE_CONNECTION_STRING" FUNCTIONS_EXTENSION_VERSION=~2
   ```

1. 切换到 Azure 门户并导航到你在此任务中先前创建的 Azure Function 应用程序的刀片。

1. 在“Azure Function 应用”边栏选项卡上，选择 **“函数”**，然后选择 **“+ 添加”**。

1. 在 **“添加函数”** 边栏选项卡中选择 **“Azure Blob 存储触发器”** 模板。

1. 在 **“添加函数”** 边栏选项卡中指定以下内容，并选择 **“添加”** 以在 Azure 函数中新建一个函数：

    | 设置 | 值 |
    | --- | --- |
    | 名称 | **BlobTrigger** |
    | 路径 | **workitems/{name}** |
    | 存储帐户连接 | **STORAGE_CONNECTION_STRING** |

1. 在 Azure 函数应用 **“BlobTrigger”** 函数边栏选项卡中选择 **“代码 + 测试”**，并查看 run.csx 文件中的内容。

   ```csharp
   public static void Run(Stream myBlob, string name, ILogger log)
   {
       log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");
   }
   ```

    > **注意**： 默认情况下，该函数配置为仅记录与创建新 blob 相对应的事件。为了执行 blob 处理任务，你可以修改此文件的内容。

#### 任务 2：验证 Azure Function 应用程序 Blob 存储触发器

1. 如有必要，请在 Cloud Shell 中重新启动 Bash 会话。

1. 在 Cloud Shell 窗格中，运行以下命令以重新填充你在上一个任务中使用的变量：

   ```sh
   export RESOURCE_GROUP_NAME='az30314b-labRG'

   export STORAGE_ACCOUNT_NAME="$(az storage account list --resource-group "${RESOURCE_GROUP_NAME}" --query "[0].name" --output tsv)"

   export CONTAINER_NAME="workitems"
   ```

1. 在 Cloud Shell 窗格中，运行以下命令将测试 blob 上传到你在先前此任务中创建的 Azure Storage 帐户：

   ```sh
   export STORAGE_ACCESS_KEY="$(az storage account keys list --account-name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query "[0].value" --output tsv)"

   export WORKITEM='workitem1.txt'

   touch "${WORKITEM}"

   az storage blob upload --file "${WORKITEM}" --container-name "${CONTAINER_NAME}" --name "${WORKITEM}" --auth-mode key --account-key "${STORAGE_ACCESS_KEY}" --account-name "${STORAGE_ACCOUNT_NAME}"
   ```

1. 在 Azure 门户中，导航回显示你在上一个任务中创建的 Azure Function 应用程序的刀片。

1. 在 Function 应用中，单击 **“函数”**。

1. 在函数列表中，单击函数名称。

1. 在“Azure Function 应用”边栏选项卡上，选择 **“监视”** 条目。

1. 注意表示 blob 上传的单个事件条目。选择该条目以查看 **“调用跟踪”** 详细信息。

    > **注意**： 由于本练习中的 Azure Function 应用程序在消耗计划中运行，因此上传 blob 和触发函数的过程之间可能会有几分钟的延迟。通过使用 App Service（而非消耗）计划实现 Function 应用程序，可以最大限度地减少延迟。

1. 在“调用跟踪”****下，单击“在 Application Insights 中运行查询”****。

1. 在 Application Insights 门户中，查看自动生成的 Kusto 查询及其结果。

### 练习 2：配置并验证基于 Azure 事件网格订阅的队列消息传递
  
本练习的主要任务如下：

1. 配置基于 Azure 事件网格订阅的队列消息传递

1. 验证基于 Azure 事件网格订阅的队列消息传递

1. 删除实验室中部署的 Azure 资源

#### 任务 1：配置基于 Azure 事件网格订阅的队列消息传递

1. 如有必要，请在 Cloud Shell 中重新启动 Bash 会话。

1. 在“Cloud Shell”窗格中运行以下命令，以在订阅中注册 eventgrid 资源提供程序：

   ```sh
   az provider register --namespace microsoft.eventgrid
   ```
  
1. 在 Cloud Shell 窗格中，运行以下命令以生成伪随机字符串，该字符串将用作你将在本练习中配置的资源名称的前缀：

   ```sh
   export PREFIX=$(echo `openssl rand -base64 5 | cut -c1-7 | tr '[:upper:]''[:lower:]' | tr -cd '[[:alnum:]]._-'`)
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以标识托管目标资源组及其现有资源的 Azure 区域：

   ```sh
   export RESOURCE_GROUP_NAME_EXISTING='az30314b-labRG'

   export LOCATION=$(az group list --query "[?name == '${RESOURCE_GROUP_NAME_EXISTING}'].location" --output tsv)

   export RESOURCE_GROUP_NAME='az30314c-labRG'

   az group create --name "${RESOURCE_GROUP_NAME}" --location $LOCATION
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以创建 Azure 存储帐户，该帐户将托管在此任务中由配置的事件网格订阅使用的容器：

   ```sh
   export STORAGE_ACCOUNT_NAME="az30314cst${PREFIX}"

   export CONTAINER_NAME="workitems"

   export STORAGE_ACCOUNT=$(az storage account create --name "${STORAGE_ACCOUNT_NAME}" --kind "StorageV2" --location "${LOCATION}" --resource-group "${RESOURCE_GROUP_NAME}" --sku "Standard_LRS")
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建存储 Azure Storage 帐户的连接字符串属性值的变量：

   ```sh
   export STORAGE_CONNECTION_STRING=$(az storage account show-connection-string --name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" -o tsv)
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以创建将由事件网格订阅使用的容器：

   ```sh
   az storage container create --name "${CONTAINER_NAME}" --account-name "${STORAGE_ACCOUNT_NAME}" --connection-string "${STORAGE_CONNECTION_STRING}"
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建存储 Azure Storage 帐户的资源 Id 属性值的变量：

   ```sh
   export STORAGE_ACCOUNT_ID=$(az storage account show --name "${STORAGE_ACCOUNT_NAME}" --query "id" --resource-group "${RESOURCE_GROUP_NAME}" -o tsv)
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建存储帐户队列，该队列将存储你将在此任务中配置的事件网格订阅生成的消息：

   ```sh
   export QUEUE_NAME="az30314cq${PREFIX}"

   az storage queue create --name "${QUEUE_NAME}" --account-name "${STORAGE_ACCOUNT_NAME}" --connection-string "${STORAGE_CONNECTION_STRING}"
   ```

1. 在 Cloud Shell 窗格中，运行以下命令以创建事件网格订阅，以便在 Azure Storage 队列中生成消息，以响应上传到 Azure Storage 帐户中的指定容器的 blob：

   ```sh
   export QUEUE_SUBSCRIPTION_NAME="az30314cqsub${PREFIX}"

   az eventgrid event-subscription create --name "${QUEUE_SUBSCRIPTION_NAME}" --included-event-types 'Microsoft.Storage.BlobCreated' --endpoint "${STORAGE_ACCOUNT_ID}/queueservices/default/queues/${QUEUE_NAME}" --endpoint-type "storagequeue" --source-resource-id "${STORAGE_ACCOUNT_ID}"
   ```

#### 任务 2：验证基于 Azure 事件网格订阅的队列消息传递

1. 在 Cloud Shell 窗格中，运行以下命令将测试 blob 上传到你在先前此任务中创建的 Azure Storage 帐户：

   ```sh
   export AZURE_STORAGE_ACCESS_KEY="$(az storage account keys list --account-name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query "[0].value" --output tsv)"

   export WORKITEM='workitem2.txt'

   touch "${WORKITEM}"

   az storage blob upload --file "${WORKITEM}" --container-name "${CONTAINER_NAME}" --name "${WORKITEM}" --auth-mode key --account-key "${AZURE_STORAGE_ACCESS_KEY}" --account-name "${STORAGE_ACCOUNT_NAME}"
   ```

1. 在 Azure 门户中，导航到显示你在本练习的上一个任务中创建的 Azure Storage 帐户的刀片。

1. 在“Azure 存储帐户”边栏选项卡中选择“队列”****，以显示其队列列表。

1. 选择代表你在本练习的上一个任务中所创建队列的条目。

1. 请注意，队列包含一条消息。选择其条目以显示“消息属性”****边栏选项卡。

1. 在**消息正文**中，注意 **url** 属性的值，它代表你在上一个任务中上传的 Azure 存储 blob 的 URL。

#### 任务 3：删除实验室中部署的 Azure 资源

1. 在 Cloud Shell 窗格中，运行以下命令以列出你在本练习中创建的资源组：

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv
   ```

    > 备注****：验证输出结果是否仅包含在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 关闭 Cloud Shell 窗格。
