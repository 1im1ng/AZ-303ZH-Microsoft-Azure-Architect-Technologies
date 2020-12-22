---
lab:
    title: '14A：借助暂存槽位实现 Azure 应用服务 Web 应用'
    module: '模块 14：实现应用程序基础结构'
---

# 实验室：借助暂存槽位实现 Azure 应用服务 Web 应用
# 学生实验室手册

## 实验室场景

Adatum Corporation 有许多 Web 应用的更新频率相对较高。尽管 Adatum 尚未完全接受 DevOps 原则，但却依赖 Git 进行版本控制，并且正在探索简化应用更新的选项。随着 Adatum 将一些工作负载转换到 Azure，Adatum 企业体系结构团队决定评估 Azure 应用服务及其部署槽位的使用，从而实现此目标。 

部署槽位是具有自身主机名的动态应用。应用内容与配置元素可以在两个部署槽（包括生产槽）之间交换。将应用部署到非生产槽具有以下优势：

- 可以在将应用交换到生产槽位之前验证暂存部署槽位中的应用更改。

- 首先将应用部署到一个槽，然后将其交换到生产，这确保该槽的所有实例都已准备好，然后交换到生产。这消除了应用部署期间的停机时间。流量重定向是无缝的，且不会因为交换操作而丢弃任何请求。通过将不需要预交换验证的情况配置为自动交换，可以自动完成整个工作流。

- 交换后，带有先前暂存的应用的槽位具有以前的生产应用。如果需要撤销交换到生产槽位的更改，只需立即进行另一次交换以返回到上一个已知的良好状态即可。

部署槽位可促进两种常见的部署模式：蓝/绿和 A/B 测试。蓝绿部署涉及将更新部署到与实时应用程序分离的生产环境中。在验证部署之后，流量路由将切换到更新后的版本。A/B 测试涉及将一些流量逐步路由到暂存站点，以测试新版本的应用。

Adatum 体系结构团队希望将 Azure 应用服务 Web 应用与部署槽位一起使用，以测试以下两种部署模式：

-  蓝/绿部署 

-  A/B 测试 


## 目标
  
完成本实验室后，你将能够：

-  通过使用 Azure 应用服务 Web 应用的部署槽位实现蓝/绿部署模式

-  通过使用 Azure 应用服务 Web 应用的部署槽位执行 A/B 测试


## 实验室环境
  
预计用时：60 分钟


## 实验室文件

无

## 说明

### 练习 1：实现 Azure 应用服务 Web 应用

1. 部署 Azure 应用服务 Web 应用

1. 创建应用服务 Web 应用部署槽位

#### 任务 1：部署 Azure 应用服务 Web 应用

1. 在实验室计算机上，启动 Web 浏览器，导航至 [“Azure 门户”](https://portal.azure.com)，然后通过提供要在本实验中使用的订阅中的所有者角色的用户帐户凭据来登录。

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 提示你选择 **“Bash”**  还是 **“PowerShell”** 时，请选择 **“Bash”**。 

    >**注意**：如果这是你第一次打开 **“Cloud Shell”**，会看到 **“未装载任何存储”** 消息，请选择你在本实验室中使用的订阅，然后选择 **“创建存储”**。 

1. 在“Cloud Shell”窗格中运行以下命令，以新建名为 **az30314a1** 的目录，并将其设置为当前目录：

   ```sh
   mkdir az30314a1
   cd ~/az30314a1/
   ```

1. 在“Cloud Shell”窗格中运行以下命令，将示例应用存储库克隆到 **az30314a1** 目录：

   ```sh
   REPO=https://github.com/Azure-Samples/html-docs-hello-world.git
   git clone $REPO
   cd html-docs-hello-world
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以配置部署用户：

   ```sh
   USERNAME=az30314user$RANDOM
   PASSWORD=az30314pass$RANDOM
   az webapp deployment user set --user-name $USERNAME --password $PASSWORD 
   echo $USERNAME
   echo $PASSWORD
   ```
1. 验证部署用户是否已成功创建。如果收到指示冲突的错误消息，请重复上一步。

    >**注意**：确保记录用户名和相应密码的值。

1. 在“Cloud Shell”窗格中运行以下命令，以创建托管应用服务 Web 应用的资源组（将 `<location>` 占位符替换为订阅中可用的且最接近实验室计算机位置的 Azure 区域名称）：

   ```sh
   LOCATION='<location>'
   RGNAME='az30314a-labRG'
   az group create --location $LOCATION --resource-group $RGNAME
   ```

1. 在“Cloud Shell”窗格中，运行以下命令以新建应用服务计划：

   ```sh
   SPNAME=az30314asp$LOCATION$RANDOM
   az appservice plan create --name $SPNAME --resource-group $RGNAME --location $LOCATION --sku S1
   ```

1. 在“Cloud Shell”窗格中运行下列命令，以新建启用了 Git 的应用服务 Web 应用：

   ```sh
   WEBAPPNAME=az30314$RANDOM$RANDOM
   az webapp create --name $WEBAPPNAME --resource-group $RGNAME --plan $SPNAME --deployment-local-git
   ```

    >**注意**： 等待部署完成。 

1. 在“Cloud Shell”窗格中运行下列命令，以检索新建应用服务 Web 应用的发布 URL：

   ```sh
   URL=$(az webapp deployment list-publishing-credentials --name $WEBAPPNAME --resource-group $RGNAME --query scmUri --output tsv)
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以设置代表启用了 Git 的 Azure 应用服务 Web应用的 git 远程别名：

   ```sh
   git remote add azure $URL
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以使用 git push azure master 推送到 Azure 远程：

   ```sh
   git push azure master
   ```

    >**注意**： 等待部署完成。 

1. 在“Cloud Shell”窗格中运行以下命令，以确定新部署的应用服务 Web 应用的 FQDN。 

   ```sh
   az webapp show --name $WEBAPPNAME --resource-group $RGNAME --query defaultHostName --output tsv
   ```

1. 关闭“Cloud Shell”窗格。


#### 任务 2：创建应用服务 Web 应用部署槽位

1. 在 Azure 门户中搜索并选择 **“应用服务”**，然后在 **“应用服务”** 边栏选项卡中选择新建的应用服务 Web 应用。

1. 在 Azure 门户中，导航到显示新部署的应用服务 Web 应用的“边栏选项卡”，选择 **“URL”** 链接，并验证它是否显示 **Azure 应用服务 - 示例静态 HTML 站点**。让“浏览器”选项卡处于打开状态。

1. 在“应用服务 Web 应用”边栏选项卡的 **“部署”** 部分，选择 **“部署槽位”**，然后选择 **“+ 添加槽位”**。

1. 在 **“添加槽位”** 边栏选项卡中指定以下设置，选择 **“添加”**，然后选择 **“关闭”**。

    | 设置 | 数值 | 
    | --- | --- |
    | 名称 | **暂存** |
    | 克隆设置来源 | Web 应用的名称 |


### 练习 2：管理应用服务 Web 应用部署槽位
  
本次练习的主要任务如下：

1. 将 Web 内容部署到应用服务 Web 应用暂存槽位

1. 交换应用服务 Web 应用过渡槽位

1. 配置 A/B 测试

1. 删除实验室中部署的 Azure 资源


#### 任务 1：将 Web 内容部署到应用服务 Web 应用暂存槽位

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 在“Cloud Shell”窗格中运行以下命令，以确保当前设置 **az30314a1/html-docs-hello-world** 作为当前目录：

   ```sh
   cd ~/az30314a1/html-docs-hello-world
   ```

1. 在“Cloud Shell”窗格中，运行以下命令以启动内置编辑器：

   ```sh
   code index.html
   ```
1. 在“Cloud Shell”窗格中的代码编辑器中，替换以下行：

   ```html
   <h1>Azure App Service - Sample Static HTML Site</h1>
   ```

   使用以下行：

   ```html
   <h1>Azure App Service - Sample Static HTML Site v1.0.1</h1>
   ```

1. 保存更改并关闭编辑器窗口。 

1. 在“Cloud Shell”窗格中运行下列命令，以指定必需的全局 git 配置设置：

   ```sh
   git config --global user.email "user@az30314.com"
   git config --global user.name "user az30314"
   ```

1. 在“Cloud Shell”窗格中运行以下命令，将本地应用的更改提交到主分支：

   ```sh
   git add index.html
   git commit -m 'v1.0.1'
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以检索应用服务 Web 应用新建的暂存槽位的发布 URL：

   ```sh
   RGNAME='az30314a-labRG'
   WEBAPPNAME=$(az webapp list --resource-group $RGNAME --query "[?starts_with(name,'az30314')]".name --output tsv)
   SLOTNAME='staging'
   URLSTAGING=$(az webapp deployment list-publishing-credentials --name $WEBAPPNAME --slot $SLOTNAME --resource-group $RGNAME --query scmUri --output tsv)
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以设置 git 远程别名，该别名表示启用了 Git 的 Azure 应用服务 Web 应用的暂存槽位：

   ```sh
   git remote add azure-staging $URLSTAGING
   ```

1. 在“Cloud Shell”窗格中运行以下命令，以使用 git push azure master 推送到 Azure 远程：

   ```sh
   git push azure-staging master
   ```

    >**注意**： 等待部署完成。 

1. 关闭“Cloud Shell”窗格。

1. 在 Azure 门户中，导航到显示应用服务 Web 应用部署槽位的边栏选项卡，然后选择“暂存槽位”。

1. 在显示暂存槽位概述的边栏选项卡上，选择 **“URL”** 链接。


#### 任务 2：交换应用服务 Web 应用过渡槽位

1. 在 Azure 门户中，导航回到显示应用服务 Web 应用的边栏选项卡，并选择 **“部署槽位”**。

1. 在“部署槽位”边栏选项卡上，选择 **“交换”**。

1. 在 **“交换”** 边栏选项卡中选择 **“交换”**，然后选择 **“关闭“**。

1. 切换到显示应用服务 Web 应用的“浏览器”选项卡，然后刷新“浏览器窗口”。验证它是否显示对暂存槽位所部署的更改。

1. 切换到显示应用服务 Web 应用暂存槽位的“浏览器”选项卡，然后刷新“浏览器窗口”。验证它是否显示原始部署中包含的原始网页。 


#### 任务 3：配置 A/B 测试

1. 在 Azure 门户中，导航回到显示应用服务 Web 应用部署槽位的“边栏选项卡”。

1. 在 Azure 门户中显示“应用服务 Web 应用部署槽位”的边栏选项卡上，在显示暂存槽位的行中，将 **“流量 %”** 列的值设置为“50”。这将自动将代表生产槽位的行中的**流量 ％** 值设置为 50。

1. 在显示“应用服务 Web 应用部署槽位”的边栏选项卡上，选择 **“保存”**。 

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 在“Cloud Shell”窗格中运行以下命令，以验证设置代表目标 Web 应用及其通讯组名称的变量：

   ```sh
   RGNAME='az30314a-labRG'
   WEBAPPNAME=$(az webapp list --resource-group $RGNAME --query "[?starts_with(name,'az30314')]".name --output tsv)
   ```

1. 在“Cloud Shell”窗格中数次运行以下命令，以确定两个槽位之间的流量分配情况。

   ```sh
   curl -H 'Cache-Control: no-cache' https://$WEBAPPNAME.azurewebsites.net --stderr - | grep '<h1>Azure App Service - Sample Static HTML Site'
   ```

    >**注意**：流量分配不完全是确定性的，但是应该可以看到来自每个目标站点的多个响应。

#### 任务 4：删除实验室中部署的 Azure 资源

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv
   ```

    > **注意**：验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 在“Cloud Shell”窗格中，运行下列命令以删除 **“az30314a1”** 目录：

   ```sh
   rm -r -f ~/az30314a1
   ```
   
1. 关闭“Cloud Shell”窗格。
