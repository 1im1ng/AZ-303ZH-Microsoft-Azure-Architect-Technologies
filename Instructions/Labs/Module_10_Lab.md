---
lab:
    title: '10: 管理 Azure 基于角色的访问控制'
    module: '模块 10：实现和管理 Azure 治理'
---

# 实验室：管理 Azure 基于角色的访问控制
# 学生实验室手册

## 实验室场景

随着 Azure Active Directory (Azure AD) 成为标识管理环境不可或缺的一部分，Adatum 企业体系结构团队还必须确定最佳的授权方法。在对 Azure 资源进行访问控制的上下文中，这种方法必须使用 Azure 基于角色的访问控制 (RBAC)。Azure RBAC 是基于 Azure 资源管理器生成的授权系统，可提供精细的 Azure 资源访问管理。

Azure RBAC 的关键概念是角色分配。角色分配包括三个要素：安全主体、角色定义和作用域。安全主体是表示请求访问 Azure 资源的用户、组、服务主体或托管标识的对象。角色定义是角色分配授权的操作（例如读取、写入或删除）的集合。角色可以是一般性的，也可以是特定于资源的。Azure 包括四个内置的一般角色（所有者、参与者、读取者和用户访问管理员）和大量特定于资源的内置角色（例如虚拟机参与者，其中包括创建和管理 Azure 虚拟机的权限）。还可以定义自定义角色。作用域是访问权限适用的资源集。作用域可以设置在多个级别：管理组、订阅、资源组或资源。作用域采用父子关系结构。

Adatum 企业体系结构团队希望使用自定义的基于角色的访问控制角色来测试 Azure 管理的委派。为了开始评估，团队打算创建一个自定义角色，用来提供对 Azure 虚拟机的受限访问。 
  

## 目标
  
完成本实验室后，你将能够：

-  定义自定义 RBAC 角色 

-  分配自定义 RBAC 角色


## 实验室环境
  
Windows Server 管理员凭据

-  用户名： **Student**

-  密码：**Pa55w.rd1234**

预计用时：60 分钟


## 实验室文件

-  \\\\AZ303\\AllFiles\\Labs\\10\\azuredeploy30310suba.json

-  \\\\AZ303\\AllFiles\\Labs\\10\\azuredeploy30310rga.json

-  \\\\AZ303\\AllFiles\\Labs\\10\\azuredeploy30310rga.parameters.json

-  \\\\AZ303\\AllFiles\\Labs\\10\\roledefinition30310.json


## 说明

### 练习 0：准备实验室环境

本次练习的主要任务如下：

1. 使用 Azure 资源管理器模板部署 Azure VM

1. 创建 Azure Active Directory 用户


#### 任务 1：使用 Azure 资源管理器模板部署 Azure VM

1. 在实验室计算机上，启动 Web 浏览器，导航至 [“Azure 门户”](https://portal.azure.com)，然后通过提供要在本实验中使用的订阅中的所有者角色的用户帐户凭据来登录。

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 提示选择 **“Bash”** 或 **“PowerShell”** 时，选择 **“PowerShell”**。 

    >**注意**：如果这是你第一次打开 **“Cloud Shell”**，会看到 **“未装载任何存储”** 消息，请选择你在本实验室中使用的订阅，然后选择 **“创建存储”**。 

1. 在“Cloud Shell”窗格的工具栏中，选择 **“上传/下载文件”** 图标，在下拉菜单中选择 **“上传”**，然后将文件 **\\\\AZ303\\AllFiles\Labs\\10\\azuredeploy30310suba.json** 上传到 Cloud Shell 主目录中。

1. 在 Cloud Shell 窗格中，通过运行以下命令创建资源组（用订阅中可用于部署 Azure VM 且最接近实验室计算机位置的 Azure 区域的名称替换 `<Azure region>` 占位符）：

   ```powershell
   $location = '<Azure region>'
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30310subaDeployment `
     -TemplateFile $HOME/azuredeploy30310suba.json `
     -rgLocation $location `
     -rgName 'az30310a-labRG'
   ```

      > **备注**：若要标识可在其中预配 Azure VM 的 Azure 区域，请参阅[**https://azure.microsoft.com/zh-cn/regions/offers/**](https://azure.microsoft.com/zh-cn/regions/offers/)

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器模板 **\\\\AZ303\\AllFiles\Labs\\10\\azuredeploy30310rga.json**。

1. 在“Cloud Shell”窗格中，上传 Azure 资源管理器参数文件 **\\\\AZ303\\AllFilesLabs\\10\\azuredeploy30310rga.parameters.json**。

1. 在“Cloud Shell”窗格中运行以下命令，以部署将在本实验室中使用的运行 Windows Server 2019 的 Azure VM：

   ```powershell
   New-AzResourceGroupDeployment `
     -Name az30310rgaDeployment `
     -ResourceGroupName 'az30310a-labRG' `
     -TemplateFile $HOME/azuredeploy30310rga.json `
     -TemplateParameterFile $HOME/azuredeploy30310rga.parameters.json `
     -AsJob
   ```

    > **注意**：不要等待部署完成，而是继续执行下一个任务。部署时间应少于 5 分钟。


#### 任务 2：创建 Azure Active Directory 用户

1. 在 Azure 门户“Cloud Shell”窗格中的 PowerShell 会话中运行以下命令，对与你的 Azure 订阅关联的 Azure AD 租户进行身份验证：

   ```powershell
   Connect-AzureAD
   ```
   
1. 在 Cloud Shell 窗格中，运行以下命令以标识 Azure AD DNS 域名：

   ```powershell
   $domainName = ((Get-AzureAdTenantDetail).VerifiedDomains)[0].Name
   ```

1. 在 Cloud Shell 窗格中，运行下列命令以创建新的 Azure AD 用户：

   ```powershell
   $passwordProfile = New-Object -TypeName Microsoft.Open.AzureAD.Model.PasswordProfile
   $passwordProfile.Password = 'Pa55w.rd1234'
   $passwordProfile.ForceChangePasswordNextLogin = $false
   New-AzureADUser -AccountEnabled $true -DisplayName 'az30310aaduser1' -PasswordProfile $passwordProfile -MailNickName 'az30310aaduser1' -UserPrincipalName "az30310aaduser1@$domainName"
   ```

1. 在 Cloud Shell 窗格中，运行下列命令以标识新创建的 Azure AD 用户的用户主体名称：

   ```powershell
   (Get-AzureADUser -Filter "MailNickName eq 'az30310aaduser1'").UserPrincipalName
   ```

      > **备注**：记录新创建的 Azure AD 用户的用户主体名称。稍后将在本实验室用到它。

1. 关闭“Cloud Shell”窗格。


### 练习 1：定义自定义 RBAC 角色
  
本练习的主要任务如下：

1. 标识通过 RBAC 授权的操作

1. 在 Azure AD 租户中创建自定义 RBAC 角色


#### 任务 1：确定通过 RBAC 委派的操作

1. 在 Azure 门户中导航到 **“az30310a-labRG”** 边栏选项卡。

1. 在 **“az30310a-LabRG”** 边栏选项卡中选择 **“访问控制 (IAM)”**。

1. 在 **“az30310a-labRG - 访问控制 (IAM)”** 边栏选项卡中，选择 **“角色”**。

1. 在 **“角色”** 边栏选项卡中选择 **“所有者”**。

1. 在 **“所有者”** 边栏选项卡中选择 **“权限”**。

1. 在 **“权限(预览)”** 边栏选项卡中选择 **“Microsoft 计算”**。

1. 在 **“Microsoft 计算”** 边栏选项卡中选择 **“虚拟机”**。

1. 在 **“虚拟机”** 边栏选项卡中，查看可通过 RBAC 委派的管理操作列表。请注意，它们包括**取消分配虚拟机**和**启动虚拟机**操作。


#### 任务 2： 在 Azure AD 租户中创建自定义 RBAC 角色

1. 在实验室计算机中打开文件 **“\\\\AZ303\\AllFiles\\Labs\\10\\roledefinition30310.json”** 并审查其内容：

   ```json
   {
      "Name": "Virtual Machine Operator (Custom)",
      "Id": null,
      "IsCustom": true,
      "Description": "Allows to start/restart Azure VMs",
      "Actions": [
          "Microsoft.Compute/*/read",
          "Microsoft.Compute/virtualMachines/restart/action",
          "Microsoft.Compute/virtualMachines/start/action"
      ],
      "NotActions": [
      ],
      "AssignableScopes": [
          "/subscriptions/SUBSCRIPTION_ID"
      ]
   }
   ```

1. 在实验室计算机显示 Azure 门户的浏览器窗口中，启动 **“Cloud Shell”** 内的 **“PowerShell”** 会话。 

1. 在“Cloud Shell”窗格中，将 Azure 资源管理器模板 **“\\\\AZ303\\AllFiles\\Labs\\10\\roledefinition30310.json”** 上传至主目录。

1. 在“Cloud Shell”窗格中运行下列命令，使用 Azure 订阅的 ID 值替换 `SUBSCRIPTION_ID` 占位符：

   ```powershell
   $subscription_id = (Get-AzContext).Subscription.id
   (Get-Content -Path $HOME/roledefinition30310.json) -Replace 'SUBSCRIPTION_ID', "$subscription_id" | Set-Content -Path $HOME/roledefinition30310.json
   ```

1. 在“Cloud Shell”窗格中运行下列命令，以验证已使用 Azure 订阅的 ID 值替换 `SUBSCRIPTION_ID` 占位符：

   ```powershell
   Get-Content -Path $HOME/roledefinition30310.json
   ```

1. 在 Cloud Shell 窗格中，运行下列命令以创建自定义角色定义：

   ```powershell
   New-AzRoleDefinition -InputFile $HOME/roledefinition30310.json
   ```

1. 在 Cloud Shell 窗格中，运行下列命令以验证是否已成功创建角色：

   ```powershell
   Get-AzRoleDefinition -Name 'Virtual Machine Operator (Custom)'
   ```

1. 关闭“Cloud Shell”窗格。


### 练习 2：分配并测试自定义 RBAC 角色
  
本练习的主要任务如下：

1. 创建 RBAC 角色分配

1. 测试 RBAC 角色分配


#### 任务 1：创建 RBAC 角色分配
 
1. 在 Azure 门户中导航到 **“az30310a-labRG”** 边栏选项卡。

1. 在 **“az30310a-LabRG”** 边栏选项卡中选择 **“访问控制 (IAM)”**。

1. 在 **“az30310a-labRG - 访问控制 (IAM)”** 边栏选项卡上，选择 **“+ 添加”**，然后选择 **“添加角色分配”** 选项。

1. 在 **“添加角色分配”** 边栏选项卡上，指定以下设置（将其他设置保留为现有值）并选择 **“保存”**：

    | 设置 | 数值 | 
    | --- | --- |
    | 角色 | **虚拟机操作员（自定义）** |
    | 分配对以下内容的访问权限 | **Azure AD 用户、组或服务主体** |
    | 选择 | **az30310aaduser1** |


#### 任务 2：测试 RBAC 角色分配

1. 在实验室计算机中启动新的私有 Web 浏览器会话，导航到 [“Azure 门户”](https://portal.azure.com)，然后使用 **“az30310aaduser1”** 用户帐户和 **“Pa55w.rd1234”** 密码登录。

    > **注意**：确保使用之前在本实验中前记录的用户主体名为 **“az30310aaduser1”** 的用户帐户。

1. 在 Azure 门户中，导航到 **“资源组”** 边栏选项卡。请注意，你无法看到任何资源组。 

1. 在 Azure 门户中，导航到**所有资源**边栏选项卡。请注意，只能看到 **“az30310a-vm0”** 及其托管磁盘。

1. 在 Azure 门户中，请导航到 **az30310a-vm0** 边栏选项卡。尝试停止虚拟机。查看通知区域中的错误消息，并注意到此操作失败了，因为当前用户无权执行此操作。

1. 重新启动虚拟机并验证操作是否已成功完成。

1. 关闭私有 Web 浏览器会话。


#### 任务 3：删除实验室中部署的 Azure 资源

1. 在实验室计算机上显示 Azure 门户的现有浏览器窗口中，在“Cloud Shell”窗格中启动 PowerShell 会话。

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```powershell
   Get-AzResourceGroup -Name 'az30310*'
   ```

    > **注意**：验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```powershell
   Get-AzResourceGroup -Name 'az30310*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. 关闭“Cloud Shell”窗格。

1. 在 Azure 门户中，导航到与你的 Azure 订阅关联的 Azure Active Directory 租户的 **“用户”** 边栏选项卡。

1. 在用户帐户列表中，依次选择代表 **“az30310aaduser1”** 用户帐户的条目、工具栏中的省略号图标、 **“删除用户”**，然后在提示确认时选择 **“是”**。  

1. 在 Azure 门户中，导航到显示 Azure 订阅属性的边栏选项卡，选择 **“访问控制 (IAM)”** 条目，然后选择 **“角色”**。

1. 在角色列表中，选择 **“虚拟机操作员(自定义)”** 条目，然后选择 **“删除”**，在提示确认时选择 **“是”**。
