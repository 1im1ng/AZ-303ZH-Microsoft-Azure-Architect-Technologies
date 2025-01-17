---
lab:
    title: '05: 实现高可用性 Azure IaaS 计算体系结构'
    module: '模块 05：实现负载均衡和网络安全'
---

# 实验室：实现高可用性 Azure IaaS 计算体系结构
# 学生实验室手册

## 实验室场景
  
Adatum 公司有几个本地工作负载在物理服务器和虚拟机的组合上运行。大多数工作负载都需要一定程度的复原能力，包括一系列高可用性 SLA。大多数工作负载都利用 Windows Server 故障转移群集或 Linux Corosync 群集以及 Pacemaker 资源管理器，并在群集节点之间进行同步复制。Adatum 试图确定如何在 Azure 中实现等效功能。特别是，Adatum 企业体系结构团队正在探索 Azure 平台功能，这些功能可满足同一数据中心内以及同一区域中数据中心之间的高可用性要求。

此外，Adatum 企业架构团队意识到仅靠复原能力可能不足以提供其业务运营所期望的可用性级别。一些工作负载具有高度动态的使用模式，当前基于连续监视和自定义脚本解决方案来解决这些问题，并自动预配和取消配置其他群集节点。其他工作负载的使用模式更容易预测，但也需要偶尔进行调整，以解决对磁盘空间、内存或处理资源增加的需求。

为了实现这些目标，体系结构团队希望测试一系列高可用性的 IaaS 计算部署，包括：

-  在 Azure 基本负载均衡器基本后面基于可用性集的 Azure VM 的部署

-  在 Azure 标准负载均衡器后面的 Azure VM 的区域冗余部署

-  在 Azure 应用程序网关后面的 Azure VM 规模集的区域冗余部署

-  Azure VM 规模集的自动水平缩放（自动缩放） 

-  Azure VM 规模集的手动垂直缩放（计算和存储）

可用性集表示 Azure VM 的逻辑分组，用于控制它们在同一 Azure 数据中心内的物理位置。Azure 确保同一可用性集中的 VM 跨多个物理服务器、计算机架、存储单元和网络交换机运行。如果 Azure 中发生硬件或软件故障，那么只有一部分 VM 会受到影响，你的整体解决方案仍然可运行。可用性集对于构建可靠的云解决方案至关重要。借助可用性集，Azure 提供业界最佳的 99.95% VM 运行时间 SLA。

可用性区域在 Azure 区域内具有惟一的物理位置。每个区域由一个或多个配备独立电源、冷却和网络的数据中心组成。区域内可用性区域的物理分离可保护应用程序和数据免受数据中心故障的影响。区域冗余服务在可用性区域之间复制应用程序和数据，以防止单点故障。借助可用性区域，Azure 提供 99.99% 的 VM 运行时间 SLA。

Azure 虚拟机规模集使你能够创建和管理一组相同、负载均衡的 VM。VM 实例数可以自动增加或减少，从而响应需求或定义的计划。规模集为应用程序提供高可用性，以便集中管理、配置和更新大量 VM。借助虚拟机规模集，你可以为计算、大数据和容器工作负荷等领域构建大规模服务。

## 目标
  
完成本实验室后，你将能够：

-  描述驻留在 Azure 基本负载均衡器后面的相同可用性集中的高可用性 Azure VM 的特征

-  描述位于 Azure 标准负载均衡器后面不同可用性区域中的高可用性 Azure VM 的特征

-  描述 Azure VM 规模集自动水平缩放的特征

-  描述 Azure VM 规模集的手动垂直缩放的特征


## 实验室环境
  
Windows Server 管理员凭据

-  用户名： **Admin**

-  密码：**Pa55w.rd**

预计用时：120 分钟


## 实验室文件

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305suba.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.parameters.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.parameters.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.parameters.json

-  \\AZ303\\AllFiles\\Labs\\05\\az30305e-configure_VMSS_with_data_disk.ps1

## 说明

### 练习 1：使用可用性集和 Azure 基本负载均衡器实现和分析高可用性 Azure VM 部署
  
本练习的主要任务如下：

1. 使用 Azure 资源管理器模板将高可用性 Azure VM 部署到 Azure 负载均衡器基础版之后的可用性集中

1. 分析部署到 Azure 基本负载均衡器之后的可用性集中的高可用性 Azure VM

1. 删除练习中部署的 Azure 资源


#### 任务 1：使用 Azure 资源管理器模板将高可用性 Azure VM 部署到 Azure 负载均衡器基础版之后的可用性集中

1. 在实验室计算机上，启动 Web 浏览器，导航至[“Azure 门户”](https://portal.azure.com)，然后通过提供要在本实验中使用的订阅中的所有者角色的用户帐户凭据来登录。

1. 在 Azure 门户中，通过选择搜索文本框右侧的工具栏图标打开 **Cloud Shell** 窗格。

1. 如果提示选择 **“Bash”** 或 **“PowerShell”**，请选择 **“Bash”**。 

    > **注意**： 如果这是你第一次打开 **“Cloud Shell”**，会看到 **“未装载任何存储”** 消息，请选择你在本实验室中使用的订阅，然后选择 **“创建存储”**。 
    
1. 在 Cloud Shell 窗格中运行以下命令，以注册 Microsoft.Insights 资源提供程序，为本实验室后面的练习做准备：

   ```Bash
   az provider register --namespace 'Microsoft.Insights'
   ```

1. 在 Cloud Shell 窗格的工具栏中，选择 **“上传/下载文件”** 图标，在下拉菜单中选择 **“上传”**，然后将文件 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305suba.json** 上传到 Cloud Shell 主目录中。

1. 在 Cloud Shell 窗格中，运行以下命令以分配要在此实验室中使用的 Azure 区域（用订阅中可用于部署 Azure VM 且最接近实验室计算机位置的 Azure 区域的名称替换 `<Azure region>` 占位符）：

   ```Bash
   LOCATION='<Azure region>'
   ```
   
      > **备注**： 若要标识可在其中预配 Azure VM 的 Azure 区域，请参阅[**https://azure.microsoft.com/zh-cn/regions/offers/**](https://azure.microsoft.com/zh-cn/regions/offers/)


      > **备注**： 若要识别 Azure 名称以便在设置 **LOCATION** 变量的值时使用，请运行 `az account list-locations --query "[].{name:name}" -o table`。请使用不含空格的表示法，例如 **eastus** 而不是 **US East**。

1. 在 Cloud Shell 窗格中运行以下命令，以创建网络观察程序的实例，为本实验室后面的练习做准备：

   ```Bash
   az network watcher configure --resource-group NetworkWatcherRG --locations $LOCATION --enabled -o table
   ```

    > **备注**：如果收到指示没有“NetworkWatcherRG”资源组的错误，请从名为 NetworkWatcherRG 的门户创建一个资源组并重新运行该命令。
       
1. 在 Cloud Shell 窗格中，运行以下命令以在指定的 Azure 区域创建资源组。
 
   ```Bash
   az deployment sub create \
   --location $LOCATION \
   --template-file azuredeploy30305suba.json \
   --parameters rgName=az30305a-labRG rgLocation=$LOCATION
   ```
      
1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器模板 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.json**。

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器参数文件 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.parameters.json**。

1. 在 Cloud Shell 窗格中，运行以下命令以将 Azure 负载均衡器基本版及其后端池（由一对托管 Windows Server 2019 数据中心核心的 Azure VM 组成）部署到同一可用性集中（将 `<vm_Size>` 占位符替换为要用于此部署的 Azure VM 的大小，例如`Standard_D2s_v3`）：

   ```Bash
   az deployment group create \
   --resource-group az30305a-labRG \
   --template-file azuredeploy30305rga.json \
   --parameters @azuredeploy30305rga.parameters.json vmSize=<vm_Size>
   ```

    > **注意**： 请等待部署完成后再继续下一个任务。该过程需要约 10 分钟。

1. 在 Azure 门户中，关闭 **Cloud Shell** 窗格。 


#### 任务 2：分析部署到 Azure 基本负载均衡器之后的可用性集中的高可用性 Azure VM

1. 在 Azure 门户中，搜索并选择 **“网络观察程序”**，然后在 **“网络观察程序”** 边栏选项卡中选择 **“拓扑”**。

1. 在 **“网络观察程序 \| “拓扑”** 边栏选项卡上，指定以下设置：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305a-labRG** |
    | 虚拟网络 | **az30305a-vnet** |

1. 查看结果拓扑关系图，注意公用 IP 地址、负载均衡器及其后端池中 Azure VM 网络适配器之间的连接。

1. 在 **“网络观察程序”** 边栏选项卡上，选择 **“有效安全规则”**。

1. 在 **“网络观察程序 \| 有效安全规则”** 边栏选项卡上，指定以下设置：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305a-labRG** |
    | 虚拟机 | **az30305a-vm0** |
    | 网络接口 | **az30305a-nic0** |

1. 查看关联的网络安全组和有效的安全规则，其中包括允许通过 RDP 和 HTTP 进行入站连接的两个自定义规则。  

    > **注意**： 或者，你可以从以下位置查看 **“有效的安全规则”**：
    - **“az30305a-nic0”** 网络接口边栏选项卡。
    - **“az30305a-web-nsg”** 网络安全组边栏选项卡 
    
1. 在 **“网络观察程序”** 边栏选项卡上，选择 **“连接疑难解答”**。

    > **注意**： 目的是验证同一可用性集中两个 Azure VM 的接近度（以网络术语而言）。

1. 在 **“网络观察程序 \| 连接疑难解答”** 边栏选项卡上，指定以下设置并选择 **“检查”**：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305a-labRG** |
    | 源类型 | **虚拟机** |
    | 虚拟机 | **az30305a-vm0** |
    | 目标 | **选择一个虚拟机** |
    | 资源组 | **az30305a-labRG** |
    | 虚拟机 | **az30305a-vm1** |
    | 协议 | **TCP** |
    | 目标端口| **80** |

    > **注意**： 需要等待几分钟才能得到结果，以便在 Azure VM 上安装 **Azure 网络观察程序代理** VM 扩展。

1. 查看结果，并记下 Azure VM 之间的网络连接延迟。

    > **注意**： 由于两个 VM 都在同一可用性集中（在同一 Azure 数据中心内），延迟应为约 1 毫秒。

1. 在 Azure 门户中，导航到 **“az30305a-labRG”** 资源组边栏选项卡，在资源列表中，选择 **“az30305a-avset”** 可用性集条目，在 **“az30305a-avset”** 边栏选项卡上，注意容错域并更新分配给两个 Azure VM 的域值。

1. 在 Azure 门户中，导航回 **“az30305a-labRG”** 资源组边栏选项卡，在资源列表中，选择 **“az30305a-lb”** 负载均衡器条目，在 **“az30305a-lb”** 边栏选项卡上，注意公共 IP 地址条目。

1. 在 Azure 门户中，在 Cloud Shell 窗格中启动 **Bash** 会话。 

1. 在 Cloud Shell 窗格中，运行以下命令以测试流向 Azure 负载均衡器后端池中 Azure VM 的 HTTP 流量的负载均衡（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器前端 IP 地址）：

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注意**： 验证返回的消息是否表明正在以轮询的方式将请求传递到后端 Azure VM

1. 在 **“az30305a-lb”** 边栏选项卡上，选择 **“负载均衡规则”** 条目，在 **“az30305a-lb \| 负载均衡规则”** 边栏选项卡上，选择代表处理 HTTP 流量的负载均衡规则的 **“az303005a-lbruletcp80”** 条目。 

1. 在 **“az303005a-lbruletcp80”** 边栏选项卡的 **“会话持续性”** 下拉列表中，选择 **“客户端 IP”**，然后选择 **“保存”**。

1. 等待更新完成，然后在 Cloud Shell 窗格中重新运行以下内容，以测试流向具有会话持续性 Azure 负载均衡器后端池中 Azure VM 的 HTTP 流量负载均衡（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器前端 IP 地址）：

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注意**： 验证返回的消息是否表明正在将请求传递到相同的后端 Azure VM

1. 在 Azure 门户中，导航回 **“az30305a-lb”** 边栏选项卡，选择 **“入站 NAT 规则”** 条目，并记下两个规则，这些规则允许经由 TCP 端口 33890 和 33891 通过远程桌面分别连接到第一个和第二个后端池 VM。 

1. 在 Cloud Shell 窗格中，运行以下命令以测试通过 NAT 到 Azure 负载均衡器后端池中第一个 Azure VM 的远程桌面连接（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器前端 IP 地址）：

   ```Bash
   curl -v telnet://<lb_IP_address>:33890
   ```

    > **注意**： 验证返回的消息是否表明你已成功连接。 

1. 按 **Ctrl+C** 组合键以返回 Bash shell 提示符，并运行以下命令以测试通过 NAT 到 Azure 负载均衡器后端池中第二个 Azure VM 的远程桌面连接（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器的前端 IP 地址）：

   ```Bash
   curl -v telnet://<lb_IP_address>:33891
   ```

    > **注意**： 验证返回的消息是否表明你已成功连接。 

1. 按 **Ctrl+C** 组合键，返回 Bash shell 提示符。


#### 任务 3：删除练习中部署的 Azure 资源

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```Bash
   az group list --query "[?starts_with(name,'az30305a-')]".name --output tsv
   ```

    > **注意**： 验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```sh
   az group list --query "[?starts_with(name,'az30305a-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 关闭 Cloud Shell 窗格。


### 练习 2：使用可用性区域和 Azure 标准负载均衡器实现和分析高可用性 Azure VM 部署
  
本练习的主要任务如下：

1. 使用 Azure 资源管理器模板将高可用性 Azure VM 部署到 Azure 标准负载平衡器后面的可用性区域中

1. 分析 Azure 负载均衡器后面跨可用性区域部署的高可用性 Azure VM

1. 删除练习中部署的 Azure 资源


#### 任务 1：使用 Azure 资源管理器模板将高可用性 Azure VM 部署到 Azure 标准负载平衡器后面的可用性区域中

1. 如果需要，在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标，打开 **“Cloud Shell”** 窗格。

1. 如果提示选择 **“Bash”** 或 **“PowerShell”**，请选择 **“Bash”**。 

1. 在 Cloud Shell 窗格的工具栏中，选择 **“上传/下载文件”** 图标，在下拉菜单中选择 **“上传”**，然后将文件 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305subb.json** 上传到 Cloud Shell 主目录中。

1. 在 Cloud Shell 窗格中运行以下命令创建资源组（将 `<Azure region>` 占位符替换为订阅中可用且最接近实验室位置的 Azure 区域名称）：

   ```Bash
   LOCATION='<Azure region>'
   ```
   
   ```Bash
   az deployment sub create \
   --location $LOCATION \
   --template-file azuredeploy30305subb.json \
   --parameters rgName=az30305b-labRG rgLocation=$LOCATION
   ```

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器模板 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.json**。

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器参数文件 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.parameters.json**。

1. 在 Cloud Shell 窗格中运行以下命令，部署 Azure 负载均衡器标准版及其后端池，该后端池由跨两个可用性区域托管 Windows Server 2019 Datacenter Core 的一对 Azure VM 组成（将 `<vm_Size>` 占位符替换为要用于此部署的 Azure VM 的大小，例如`Standard_D2s_v3`）：

   ```Bash
   az deployment group create \
   --resource-group az30305b-labRG \
   --template-file azuredeploy30305rgb.json \
   --parameters @azuredeploy30305rgb.parameters.json vmSize=<vm_Size>
   ```

    > **注意**： 请等待部署完成后再继续下一个任务。该过程需要约 10 分钟。

1. 在 Azure 门户中，关闭 Cloud Shell 窗格。 


#### 任务 2：分析 Azure 负载均衡器后面跨可用性区域部署的高可用性 Azure VM

1. 在 Azure 门户中，搜索并选择 **“网络观察程序”**，然后在 **“网络观察程序”** 边栏选项卡中选择 **“拓扑”**。

1. 在 **“网络观察程序 \| “拓扑”** 边栏选项卡上，指定以下设置：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305b-labRG** |
    | 虚拟网络 | **az30305b-vnet** |

1. 查看结果拓扑关系图，注意公用 IP 地址、负载均衡器及其后端池中 Azure VM 网络适配器之间的连接。

    > **注意**： 该图实际上与上一练习中看到的图完全相同，因为尽管 Azure VM 位于不同区域（实际上是 Azure 数据中心），但它们位于同一子网中。
    
1. 在 **“网络观察程序”** 边栏选项卡上，选择 **“有效安全规则”**。

1. 在 **“网络观察程序 \| 有效安全规则”** 边栏选项卡上，指定以下设置：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305b-labRG** |
    | 虚拟机 | **az30305b-vm0** |
    | 网络接口 | **az30305b-nic0** |

1. 查看关联的网络安全组和有效的安全规则，其中包括允许通过 RDP 和 HTTP 进行入站连接的两个自定义规则。 

    > **注意**： 该列表实际上也与你在上一练习中看到的列表完全相同，通过使用与两个 Azure VM 都连接到的子网相关联的网络安全组来实现网络级保护。但是请记住，由于本例使用了 Azure 负载均衡器标准版 SKU（在使用基本版 SKU 时，NSG 是可选的），HTTP 和 RDP 流量需要网络安全组才能到达后端池 Azure VM。  
    
    > **注意**： 或者，你可以从以下位置查看 **“有效的安全规则”**：
    - **“az30305b-nic0”** 网络接口边栏选项卡。
    - **“az30305b-web-nsg”** 网络安全组边栏选项卡 

1. 在 **“网络观察程序”** 边栏选项卡上，选择 **“连接疑难解答”**。

    > **注意**：目的是验证不同区域中（不同 Azure 数据中心内）两个 Azure VM 的接近度（以网络连接而言）。

1. 在 **“网络观察程序 \| 连接疑难解答”** 边栏选项卡上，指定以下设置并选择 **“检查”**：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305b-labRG** |
    | 源类型 | **虚拟机** |
    | 虚拟机 | **az30305b-vm0** |
    | 目标 | **选择一个虚拟机** |
    | 资源组 | **az30305b-labRG** |
    | 虚拟机 | **az30305b-vm1** |
    | 协议 | **TCP** |
    | 目标端口| **80** |

    > **注意**： 需要等待几分钟才能得到结果，以便在 Azure VM 上安装 **Azure 网络观察程序代理** VM 扩展。

1. 查看结果，并记下 Azure VM 之间的网络连接延迟。

    > **注意**： 由于两个 VM 位于不同区域（在不同的 Azure 数据中心内），因此延迟可能会比在上一个练习中观察到的延迟稍高。

1. 在 Azure 门户中，导航到 **“az30305b-labRG”** 资源组边栏选项卡，在资源列表中，选择 **“az30305b-vm0”** 虚拟机条目，在 **“az30305b-vm0”** 边栏选项卡上，记录 **“位置”** 和 **“可用性区域”** 条目。 

1. 在 Azure 门户中，导航到 **“az30305b-labRG”** 资源组边栏选项卡，在资源列表中，选择 **“az30305b-vm1”** 虚拟机条目，在 **“az30305b-vm1”** 边栏选项卡上，记录 **“位置”** 和 **“可用性区域”** 条目。 

    > **注意**： 查看的条目证实了每个 Azure VM 位于不同的可用性区域。

1. 在 Azure 门户中，导航到 **“az30305b-labRG”** 资源组边栏选项卡，然后在资源列表中选择 **“az30305b-lb”** 负载均衡器条目，在 **“az30305b-lb”** 边栏选项卡上，记录公共 IP 地址条目。

1. 在 Azure 门户中的 Cloud Shell 窗格中启动 **Bash** 会话。 

1. 在 Cloud Shell 窗格中，运行以下命令以测试流向 Azure 负载均衡器后端池中 Azure VM 的 HTTP 流量的负载均衡（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器前端 IP 地址）：

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注意**： 验证返回的消息是否表明正在以轮询的方式将请求传递到后端 Azure VM

1. 在 **“az30305b-lb”** 边栏选项卡上，选择 **“负载均衡规则”** 条目，在 **“az30305b-lb \| 负载均衡规则”** 边栏选项卡上，选择代表处理 HTTP 流量的负载均衡规则的 **“az303005b-lbruletcp80”** 条目。 

1. 在 **“az303005b-lbruletcp80”** 边栏选项卡的 **“会话持续性”** 下拉列表中，选择 **“客户端 IP”**，然后选择 **“保存”**。

1. 等待更新完成，然后在 Cloud Shell 窗格中重新运行以下内容，以测试流向具有会话持续性 Azure 负载均衡器后端池中 Azure VM 的 HTTP 流量负载均衡（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器前端 IP 地址）：

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注意**： 验证返回的消息是否表明正在将请求传递到相同的后端 Azure VM

1. 在 Azure 门户中，导航回 **“az30305b-lb”** 边栏选项卡，选择 **“入站 NAT 规则”** 条目，并记下两个规则，这些规则允许经由 TCP 端口 33890 和 33891 通过远程桌面分别连接到第一个和第二个后端池 VM。 

1. 在 Cloud Shell 窗格中，运行以下命令以测试通过 NAT 到 Azure 负载均衡器后端池中第一个 Azure VM 的远程桌面连接（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器前端 IP 地址）：

   ```Bash
   curl -v telnet://<lb_IP_address>:33890
   ```

    > **注意**： 验证返回的消息是否表明你已成功连接。 

1. 按 **Ctrl+C** 组合键以返回 Bash shell 提示符，并运行以下命令以测试通过 NAT 到 Azure 负载均衡器后端池中第二个 Azure VM 的远程桌面连接（将 `<lb_IP_address>` 占位符替换为你之前确定的负载均衡器的前端 IP 地址）：

   ```Bash
   curl -v telnet://<lb_IP_address>:33891
   ```

    > **注意**： 验证返回的消息是否表明你已成功连接。 

1. 请按下 **Ctrl+C** 组合键以返回到 Bash shell 提示符，然后关闭“Cloud Shell”窗格。

1. 在 **“az30305b-lb”** 边栏选项卡上，选择 **“负载均衡规则”** 条目，在 **“az30305b-lb \| 负载均衡规则”** 边栏选项卡上，选择代表处理 HTTP 流量的负载均衡规则的 **“az303005b-lbruletcp80”** 条目。 

1. 在 **“az303005b-lbruletcp80”** 边栏选项卡上的 **“出站源网络地址转换 (SNAT)”** 部分，选择 **“（推荐）使用出站规则为后端池成员提供对 Internet 的访问”**，然后选择 **“保存”**。

1. 导航回 **“az30305b-lb”** 边栏选项卡，选择 **“出站规则”** 条目，然后在 **“az30305b-lb \| “出站规则”** 边栏选项卡中，选择 **“+ 添加”**。

1. 在 **“添加出站规则”** 边栏选项卡中，指定以下设置并选择 **“添加”** （将所有其他设置保留为其默认值）：

    | 设置 | 值 | 
    | --- | --- |
    | 名称 | **az303005b-obrule** | 
    | 前端 IP 地址 | **az30305b-lb** 负载均衡器现有前端 IP 地址的名称 |
    | 后端池 | **az30305b-bepool** |
    | 端口分配 | **手动选择出站端口数** |
    | 选择方式 | **最大后端实例数** |
    | 最大后端实例数 | **3** |

    > **注意**： Azure 负载均衡器标准允许为出站流量指定专属的前端 IP 地址（在分配了多个前端 IP 地址的情况下）。

1. 在 Azure 门户中，导航到 **“az30305b-labRG”** 资源组边栏选项卡，在资源列表中选择 **“az30305b-vm0”** 虚拟机条目，并在 **“az30305b-vm0”** 边栏选项卡的 **“操作”** 边栏选项卡中，选择 **“运行命令”**。

1. 在 **“az30305b-vm0 \| “运行命令”** 边栏选项卡，选择 **“RunPowerShellScript"**。 

1. 在 **“运行命令脚本”** 边栏选项卡的 **“PowerShell 脚本”** 文本框中，键入以下内容并选择 **“运行”**。

   ```powershell
   (Invoke-RestMethod -Uri "http://ipinfo.io").IP
   ```

    > **注意**： 此命令将返回从其中发出 Web 请求的公共 IP 地址。

1. 查看输出结果并验证其与分配给 Azure 负载均衡器标准前端的公共 IP 地址是否匹配，你已将该 IP 地址分配给出站负载均衡规则。


#### 任务 3：删除练习中部署的 Azure 资源

1. 在 Azure 门户中的 Cloud Shell 窗格中启动 **Bash** 会话。 

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```Bash
   az group list --query "[?starts_with(name,'az30305b-')]".name --output tsv
   ```

    > **注意**： 验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```Bash
   az group list --query "[?starts_with(name,'az30305b-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 关闭 Cloud Shell 窗格。


### 练习 3：使用可用性区域和 Azure 应用程序网关实现和分析高可用性 Azure VM 规模集部署。
  
本练习的主要任务如下：

1. 使用 Azure 资源管理器模板将高可用性 Azure VM 规模集部署到 Azure 应用程序网关后面的可用性区域

1. 分析在 Azure 应用程序网关后面的可用性区域中部署的高可用性 Azure VM 规模集

1. 删除练习中部署的 Azure 资源


#### 任务 1：使用 Azure 资源管理器模板将高可用性 Azure VM 规模集部署到 Azure 应用程序网关后面的可用性区域

1. 如果需要，在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标，打开 **“Cloud Shell”** 窗格。

1. 如果提示选择 **“Bash”** 或 **“PowerShell”**，请选择 **“Bash”**。 

1. 在 Cloud Shell 窗格的工具栏中，选择 **“上传/下载文件”** 图标，在下拉菜单中选择 **“上传”**，然后将文件 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305subc.json** 上传到 Cloud Shell 主目录中。

1. 在 Cloud Shell 窗格中运行以下命令创建资源组（将 `<Azure region>` 占位符替换为订阅中可用且最接近实验室位置的 Azure 区域名称）：

   ```Bash
   az deployment sub create --location '<Azure region>' --template-file azuredeploy30305subc.json --parameters rgName=az30305c-labRG rgLocation='<Azure region>'
   ```

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器模板 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.json**。

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器参数文件 **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.parameters.json**。

1. 在 Cloud Shell 窗格中，运行以下命令以部署 Azure 应用程序网关，其后端池由一对 Azure VM 组成，这些 VM 跨不同的可用性区域托管 Windows Server 2019 数据中心核心（将 `<vm_Size>` 占位符替换为要用于此部署的 Azure VM 的大小，例如`Standard_D2s_v3`）：

   ```
   az deployment group create --resource-group az30305c-labRG --template-file azuredeploy30305rgc.json --parameters @azuredeploy30305rgc.parameters.json vmSize= <vm_Size>
   ```

    > **注意**： 请等待部署完成后再继续下一个任务。该过程需要约 10 分钟。

1. 在 Azure 门户中，关闭 Cloud Shell 窗格。 


#### 任务 2：分析在 Azure 应用程序网关后面的可用性区域中部署的高可用性 Azure VM 规模集

1. 在 Azure 门户中，搜索并选择 **“网络观察程序”**，然后在 **“网络观察程序”** 边栏选项卡中选择 **“拓扑”**。

1. 在 **“网络观察程序 \| “拓扑”** 边栏选项卡上，指定以下设置：

    | 设置 | 值 | 
    | --- | --- |
    | 订阅 | 在本实验室中使用的 Azure 订阅的名称 |
    | 资源组 | **az30305c-labRG** |
    | 虚拟网络 | **az30305c-vnet** |

1. 查看生成的拓扑图，记录公共 IP 地址、负载均衡器和 Azure 虚拟机规模集在其后端池中的 Azure VM 实例的网络适配器之间的连接。 

    > **注意**： 此外，部署 Azure 应用程序网关需要一个如关系图中所示的专用子网（尽管未显示网关）。

    > **注意**： 在这种配置中，无法使用网络观察程序查看有效的网络安全规则（这是 Azure VM 与 Azure VM 规模集实例之间的区别之一）。同样，尽管可以使用**连接故障排除**测试来自 Azure 应用程序网关的连接，但是不能依赖使用它来测试来自 Azure VM 规模集实例的网络连接。

1. 在 Azure 门户中，导航到 **“az30305c-labRG”** 资源组边栏选项卡，在资源列表中，选择 **“az30305c-vmss”** 虚拟机规模集条目。 

1. 在 **“az30305c-vmss”** 边栏选项卡中，记录**位置**和**容错域**条目。 

    > **注意**： 与 Azure VM 不同，Azure VM 规模集的各个实例部署到单独的容错域中，包括部署在同一区域中的实例。此外，它们支持 5 个容错域（与 Azure VM 最多可以使用 3 个容错域不同）。 

1. 在 **“az30305c-vmss”** 边栏选项卡中，选择 **“实例”**，在 **“az30305c-vmss \| “实例”** 边栏选项卡中，选择第一个实例，然后通过查看**位置**属性的值确定其可用区域 。 

1. 导航回 **“az30305c-vmss \| “实例”** 边栏选项卡，选择第二个实例，然后通过查看**位置**属性的值确定其可用区域。 

    > **注意**： 验证每个实例是否都位于不同的可用性区域中。

1. 在 Azure 门户中，导航到 **“az30305c-labRG”** 资源组边栏选项卡，然后在资源列表中选择 **“az30305c-appgw”** 负载均衡器条目，并注意 **“az30305c-appgw”** 边栏选项卡中的公共 IP 地址条目。

1. 在 Azure 门户中的 Cloud Shell 窗格中启动 **Bash** 会话。 

1. 在“Cloud Shell”窗格中，运行以下命令以测试到 Azure 应用程序网关后端池中 Azure VM 规模集实例的 HTTP 流量的负载均衡情况（将 `<lb_IP_address> `占位符替换为你之前标识的网关前端的 IP 地址）：

   ```Bash
   for i in {1..4}; do curl <appgw_IPaddress>; done
   ```

    > **注意**： 验证返回的消息是否表明正在以轮询的方式将请求传递到后端 Azure VM

1. 在 **“az30305c-appgw”** 边栏选项卡中，选择 **“HTTP 设置”** 条目，然后在 **“az30305c-appgw \| “HTTP 设置”** 边栏选项卡中，选择代表处理 HTTP 流量的负载均衡规则的 **“appGwBackentHttpSettings”** 条目。 

1. 在 **“appGwBackentHttpSettings”** 边栏选项卡中，请在不进行任何更改的情况下查看现有设置，并注意你可以启用**基于 Cookie 的相关性**。

    > **注意**： 此功能要求客户端支持使用 cookie。

    > **注意**： 不能使用 Azure 应用程序网关来实现 NAT，以用于 Azure VM 规模集实例的 RDP 连接。Azure 应用程序网关仅支持 HTTP / HTTPS 通信。


### 练习 4：使用可用性区域和 Azure 应用程序网关实现 Azure VM 规模集的自动缩放。
  
本练习的主要任务如下：

1. 配置 Azure VM 规模集的自动缩放

1. 测试 Azure VM 规模集的自动缩放

#### 任务 1：配置 Azure VM 规模集的自动缩放

1. 在 Azure 门户中，导航到 **“az30305c-labRG”** 资源组边栏选项卡，在资源列表中，选择 **“az30305c-vmss”** 虚拟机规模集条目，并在 **“az30305c-vmss”** 边栏选项卡上选择 **“缩放”**。 

1. 在 **“az30305c-vmss \| 缩放”** 边栏选项卡上，选择 **“自定义自动缩放”** 选项。

1. 在 **“自定义自动缩放”** 部分，指定以下设置（将其他设置保留为默认值）：

    | 设置 | 值 | 
    | --- | --- |
    | 缩放模式 | **根据指标进行缩放** |
    | 最小实例限制 | **1** |
    | 最大实例限制 | **3** |
    | 默认实例限制 | **1** |

1. 选择 **“+ 添加规则”**。

1. 在 **“缩放规则”** 边栏选项卡上，指定以下设置并选择 **“添加”** （将其他设置保留为默认值）：

    | 设置 | 值 | 
    | --- | --- |
    | 时间聚合 | **最大值** |
    | 指标命名空间 | **虚拟机主机** |
    | 指标名称 | **CPU 百分比** |
    | VMName 运算符 | **=** |
    | 维度值 | **az30305c-vmss_0** |
    | 启用除以实例数后的指标 | **启用** |
    | 运算符 | **大于** |
    | 触发缩放操作的指标阈值 | **1** |
    | 持续时间（分钟） | **1** |
    | 时间粒度统计信息 | **最大值** |
    | 操作 | **计数增加** |
    | 实例计数 | **1** |
    | 冷却（分钟） | **5** |

    > **注意**： 这些值是为实验室目的而严格选择的，以便尽快触发缩放。有关 Azure VM 规模集缩放的指导，请参阅 [Microsoft Docs](https://docs.microsoft.com/zh-cn/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview)。 

1. 回到 **“az30305c-vmss \| 缩放”** 边栏选项卡，选择 **“+ 添加规则”**。

1. 在 **“缩放规则”** 边栏选项卡上，指定以下设置并选择 **“添加”** （将其他设置保留为默认值）：

    | 设置 | 值 | 
    | --- | --- |
    | 时间聚合 | **平均值** |
    | 指标命名空间 | **虚拟机主机** |
    | 指标名称 | **CPU 百分比** |
    | VMName 运算符 | **=** |
    | 维度值 | **已选择 2** |
    | 启用除以实例数后的指标 | **启用** |
    | 运算符 | **小于** |
    | 触发缩放操作的指标阈值 | **1** |
    | 持续时间（分钟） | **1** |
    | 时间粒度统计信息 | **至少有** |
    | 操作 | **计数减少** |
    | 实例计数 | **1** |
    | 冷却（分钟） | **5** |

1. 回到 **“az30305c-vmss | 缩放”** 边栏选项卡，选择 **“保存”**。


#### 任务 2：测试 Azure VM 规模集的自动缩放

1. 在 Azure 门户中的 Cloud Shell 窗格中启动 **Bash** 会话。 

1. 在 Cloud Shell 窗格中，运行以下命令以触发 Azure 应用程序网关后端池中 Azure VM 规模集实例的自动缩放（将 `<lb_IP_address>` 占位符替换为你之前确定的网关前端 IP 地址）：

   ```Bash
   for (( ; ; )); do curl -s <lb_IP_address>?[1-10]; done
   ```
1. 在 Azure 门户中的 **“az30305c-vmss 概览”** 边栏选项卡的“监视”选项卡上，查看 **“CPU（平均）”** 图表，并验证应用程序网关的 CPU 使用率是否增加到足以触发横向扩展。

    > **注意**：你可能需要等待几分钟。

1. 在 **“az30305c-vmss”** 边栏选项卡上，选择 **“实例”** 条目并验证实例数量已增加。

    > **注意**： 你可能需要刷新 **“az30305c-vmss \| 实例”** 边栏选项卡。

    > **注意**： 你可能会看到实例数量增加了 2 个（而不是 1 个）。只要最终运行的实例数为 3，这就是预料之中的。 

1. 在 Azure 门户中，关闭 **“Cloud Shell”** 窗格。 

1. 在 Azure 门户中的 **“az30305c-vmss”** 边栏选项卡上，查看 **“CPU（平均）”** 图表，并验证应用程序网关的 CPU 使用率是否降低到足以触发横向缩减。 

    > **注意**： 你可能需要等待几分钟。

1. 在 **“az30305c-vmss”** 边栏选项卡上，选择 **“实例”** 条目，并验证实例数是否已降低到 2。

    > **注意**： 你可能需要刷新 **“az30305c-vmss \| 实例”** 边栏选项卡。

1. 在 **“az30305c-vmss”** 边栏选项卡上，选择 **“缩放”**。 

1. 在 **“az30305c-vmss \| 缩放”** 边栏选项卡上，选择 **“手动缩放”** 选项，并选择 **“保存”**。

    > **注意**： 这将防止在下一个练习中出现任何不希望看到的自动缩放。 


### 练习 5：实现 Azure VM 规模集的垂直缩放

本练习的主要任务如下：

1. 缩放 Azure 虚拟机规模集实例的计算资源。

1. 缩放 Azure 虚拟机规模集实例的存储资源。


#### 任务 1：缩放 Azure 虚拟机规模集实例的计算资源。

1. 在 Azure 门户中的 **“az30305c-vmss”** 边栏选项卡上，选择 **“大小”**。

1. 在可用大小列表中，选择当前配置以外的任何可用大小，然后选择 **“调整大小”**。

1. 在 **“az30305c-vmss”** 边栏选项卡中，选择 **“实例”** 条目，然后在 **“az30305c-vmss \| 实例”** 边栏选项卡上，观察将现有实例替换为所需大小的新实例的流程。

    > **注意**： 你可能需要刷新 **“az30305c-vmss \| 实例”** 边栏选项卡。

1. 等待实例更新并运行。


#### 任务 2：缩放 Azure 虚拟机规模集实例的存储资源。

1. 在“az30305c-vmss”****边栏选项卡上，选择“磁盘”****，选择“+ 创建并附加新磁盘”****，使用以下设置附加新的托管磁盘（其他设置保留默认值），然后选择“保存”****：

    | 设置 | 值 | 
    | --- | --- |
    | LUN | **0** |
    | 大小 | **32** |
    | 存储帐户类型 | **标准 HDD** |

1. 在 **“az30305c-vmss”** 边栏选项卡中，选择 **“实例”** 条目，然后在 **“az30305c-vmss \| 实例”** 边栏选项卡，观察现有实例的更新过程。

    > **注意**： 上一步中附加的磁盘是原始磁盘。在使用它之前，需要创建一个分区，对其进行格式化并装载。为此，你将通过自定义脚本扩展将 PowerShell 脚本部署到 Azure VM 规模集实例。但是，首先，你需要将其删除。

1. 在 **“az30305c-vmss”** 边栏选项卡中，选择 **“扩展”**，在 **“az30305c-vmss \| 扩展”** 边栏选项卡中，选择 **“customScriptExtension”** 条目，然后在 **“扩展”** 边栏选项卡中，选择 **“卸载”**。

    > **注意**： 等待卸载完成。

1. 在 Azure 门户中，导航到 **“az30305c-labRG”** 资源组边栏选项卡，并在资源列表中选择“存储帐户”资源。 

1. 在“存储帐户”边栏选项卡中，选择 **“容器”**，然后选择 **“+ 容器”**。 

1. 在 **“新建容器”** 边栏选项卡中，指定以下设置（将其他设置保留为默认值）并选择 **“创建”**：

    | 设置 | 值 | 
    | --- | --- |
    | 名称 | **脚本** |
    | 公共访问级别 | **专用（不允许匿名访问**） |
    
1. 返回到显示容器列表的“存储帐户”边栏选项卡，选择 **“脚本”**。

1. 在 **“脚本”** 边栏选项卡中，选择 **“上传”**。

1. 在 **“上传 Blob”** 边栏选项卡中，选择文件夹图标，在 **“打开”** 对话框中，导航到 **“\\\\AZ303\\AllFiles\\Labs\\05”** 文件夹，选择 **“az30305e-configure_VMSS_with_data_disk.ps1”**，选择 **“打开”**，然后返回到 **“上传 Blob”** 边栏选项卡，选择 **“上传”**。 

1. 在 Azure 门户中，导航回到 **“az30305c-vmss”** 虚拟机规模集边栏选项卡。 

1. 在 **“az30305c-vmss”** 边栏选项卡中，选择 **“扩展”**，在 **“az30305c-vmss \| 扩展”** 边栏选项卡中，选择 **“+ 添加”** ，然后选择 **“扩展”** 边栏选项卡中的 **“customScriptExtension”** 条目。

1. 在 **“新建资源”** 边栏选项卡中，选择 **“自定义脚本扩展”**，然后选择 **“创建”**。

1. 从 **“安装扩展”** 边栏选项卡中，选择 **“浏览”**。 

1. 在 **“存储帐户”** 边栏选项卡中，选择你将 **“az30305e-configure_VMSS_with_data_disk.ps1”** 脚本上传其中的存储帐户的名称，在 **“容器”** 边栏选项卡中，选择 **“脚本”**，在 **“脚本”** 边栏选项卡中，选择 **“az30305e-configure_VMSS_with_data_disk.ps1”**，然后选择 **“选择”**。 

1. 返回到 **“安装扩展”** 边栏选项卡，选择 **“确定”**。

1. 在 **“az30305c-vmss”** 边栏选项卡中，选择 **“实例”** 条目，然后在 **“az30305c-vmss  | 实例”** 边栏选项卡中，观察现有实例的更新过程。

    > **注意**： 你可能需要刷新 **“az30305c-vmss \| 实例”** 边栏选项卡。


#### 任务 3：删除练习中部署的 Azure 资源

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```Bash
   az group list --query "[?starts_with(name,'az30305c-')]".name --output tsv
   ```

    > **注意**： 验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```Bash
   az group list --query "[?starts_with(name,'az30305c-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```
   
1. 关闭 Cloud Shell 窗格。
