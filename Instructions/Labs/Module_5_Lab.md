---
lab:
    title: '5: 实现和配置 Azure 存储文件和 Blob 服务'
    module: '模块 5：实现存储帐户'
---

# 实验室：实现和配置 Azure 存储文件和 Blob 服务
# 学生实验室手册

## 实验室场景
 
Adatum Corporation 在其本地存储中存储大量非结构化和半结构化数据。其维护的复杂性和成本越来越高。为了满足数据保留需求，有些数据要保留很长时间。Adatum 企业体系结构团队正在寻找廉价的替代方案，以支持分层存储，同时允许安全访问，从而最大程度地降低数据外泄的可能性。虽然团队知道 Azure 存储实际上提供了无限的容量，但他们关心的是帐户密钥的使用，该密钥授予对相应存储帐户全部内容的无限访问权限。尽管密钥可以有序轮换，但实施轮换操作需要适当规划。此外，访问密钥专门构成了一种授权机制，这限制了正确审核其使用的能力。

为了解决这些缺点，体系结构团队决定探索共享访问签名的使用。共享访问签名 (SAS) 提供对存储帐户中资源的安全委派访问权限，同时将意外数据泄露的可能性降到最低。SAS 提供了对数据访问的精细控制，包括限制访问对单个存储对象（例如 blob）的能力，将此类访问限制为自定义时间窗口以及将网络访问筛选到指定 IP 地址范围。此外，体系结构团队希望评估 Azure 存储与 Azure Active Directory 之间的集成级别，希望解决其审核需求。体系结构团队还决定确定 Azure 文件存储的适用性，以替代某些本地文件共享。

为了实现这些目标，Adatum Corporation 将测试 Azure 存储资源的一系列身份验证和授权机制，包括：

-  在帐户、容器和对象级别使用共享访问签名

-  配置 Blob 的访问级别 

-  实现基于 Azure Active Directory 的授权

-  使用存储帐户访问密钥


## 目标
  
完成本实验室后，你将能够：

-  利用共享访问签名实现对 Azure 存储 Blob 的授权

-  利用 Azure Active Directory 实现对 Azure 存储 Blob 的授权

-  利用访问密钥实现 Azure 存储文件共享的授权


## 实验室环境
  
Windows Server 管理员凭据

-  用户名： **Student**

-  密码：**Pa55w.rd1234**

预计用时：90 分钟


## 实验室文件

-  \\\\AZ303\\AllFiles\\Labs\\02\\azuredeploy30302suba.json

-  \\\\AZ303\\AllFiles\\Labs\\02\\azuredeploy30302rga.json

-  \\\\AZ303\\AllFiles\\Labs\\02\\azuredeploy30302rga.parameters.json


### 练习 0：准备实验室环境

本次练习的主要任务如下：

 1. 使用 Azure 资源管理器模板部署 Azure VM


#### 任务 1：使用 Azure 资源管理器模板部署 Azure VM

1. 在实验室计算机上，启动 Web 浏览器，导航至 [“Azure 门户”](https://portal.azure.com)，然后通过提供要在本实验中使用的订阅中的所有者角色的用户帐户凭据来登录。

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 提示选择 **“Bash”** 或 **“PowerShell”** 时，选择 **“PowerShell”**。 

    >**注意**：如果这是你第一次打开 **“Cloud Shell”**，会看到 **“未装载任何存储”** 消息，请选择你在本实验室中使用的订阅，然后选择 **“创建存储”**。 

1. 在“Cloud Shell”窗格的工具栏中，选择 **“上传/下载文件”** 图标，在下拉菜单中选择 **“上传”**，然后将文件 **\\\\AZ303\\AllFiles\Labs\\02\\azuredeploy30302suba.json** 上传到 Cloud Shell 主目录中。

1. 在 Cloud Shell 窗格中，通过运行以下命令创建资源组（用订阅中可用于部署 Azure VM 且最接近实验室计算机位置的 Azure 区域的名称替换 `<Azure region>` 占位符）：

   ```powershell
   $location = '<Azure region>'
   ```
   
   ```powershell
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30302subaDeployment `
     -TemplateFile $HOME/azuredeploy30302suba.json `
     -rgLocation $location `
     -rgName 'az30302a-labRG'
   ```

      > **备注**：若要标识可在其中预配 Azure VM 的 Azure 区域，请参阅 [**https://azure.microsoft.com/zh-cn/regions/offers/**](https://azure.microsoft.com/zh-cn/regions/offers/)

1. 在 Cloud Shell 窗格中，上传 Azure 资源管理器模板 **\\\\AZ303\\AllFiles\Labs\\02\\azuredeploy30302rga.json**。

1. 在“Cloud Shell”窗格中，上传 Azure 资源管理器参数文件 **\\\\AZ303\\AllFilesLabs\\02\\azuredeploy30302rga.parameters.json**。

1. 在“Cloud Shell”窗格中运行以下命令，以部署将在本实验室中使用的运行 Windows Server 2019 的 Azure VM：

   ```powershell
   New-AzResourceGroupDeployment `
     -Name az30302rgaDeployment `
     -ResourceGroupName 'az30302a-labRG' `
     -TemplateFile $HOME/azuredeploy30302rga.json `
     -TemplateParameterFile $HOME/azuredeploy30302rga.parameters.json `
     -AsJob
   ```

    > **注意**：不要等待部署完成，而是继续执行下一个练习。部署时间应少于 5 分钟。

1. 在 Azure 门户中，关闭 **“Cloud Shell”** 窗格。 


### 练习 1：使用共享访问签名配置 Azure 存储帐户授权。
  
本次练习的主要任务如下：

1. 创建 Azure 存储账户

1. 安装存储资源管理器

1. 生成帐户级别的共享访问签名

1. 使用 Azure 存储资源管理器创建 Blob 容器

1. 使用 AzCopy 将文件上传到 Blob 容器

1. 使用 Blob 级共享访问签名访问 Blob


#### 任务 1：创建 Azure 存储账户

1. 在 Azure 门户中，搜索并选择 **“存储帐户”**，然后在 **“存储帐户”** 边栏选项卡上，选择 **“+ 添加”**。

1. 在 **“创建存储帐户”** 边栏选项卡的 **“基本设置”** 选项卡上，指定以下设置（将其他设置保留为默认值）：

    | 设置 | 数值 | 
    | --- | --- |
    | 订阅 | 在本实验室使用的 Azure 订阅的名称 |
    | 资源组 | 新资源组名称  **az30302a-labRG** |
    | 存储帐户名称:  | 由字母和数字组成、长度介于 3 到 24 之间的任何全局唯一名称 |
    | 地点 | 可以在其中创建 Azure 存储帐户的 Azure 区域的名称  |
    | 性能 | **标准** |
    | 帐户类型： | **StorageV2（常规用途 v2）** |
    | 复制 | **本地冗余存储 (LRS)** |

1. 选择 **“下一步: 网络 >”**，在 **“创建存储帐户”** 边栏选项卡的 **“网络”** “选项卡上，查看可用选项，接受默认选项 **“公共终结点(所有网络)”**，然后选择 **“下一步: 数据保护 >”**。

1. 在 **“创建存储帐户”** 边栏选项卡的 **“数据保护”** 选项卡上，查看可用选项，接受默认设置，选择 **“下一步：“高级”>**。

1. 在 **“创建存储帐户”** 边栏选项卡的 **“高级”** 选项卡上，查看可用选项，接受默认设置，选择 **“查看 + 创建”**，等待验证流程完成，然后选择 **“创建”**。

    >**注意**：等待创建完存储帐户。该操作需约 2 分钟。


#### 任务 2：安装存储资源管理器

   > **注意**：在继续之前，请确保在本实验室开始时启动的 Azure VM 部署已完成。 

1. 在 Azure 门户中，搜索并选择 **“虚拟机”**，在 **“虚拟机”** 边栏选项卡上的虚拟机列表中，选择 **“az30302a-vm0”**。

1. 在 **“az30302a-vm0”** 边栏选项卡上，选择 **“连接”**，在下拉菜单中选择 **“RDP”**，然后选择 **“下载 RDP 文件”**。

1. 出现提示时，请使用以下凭据登录：

    | 设置 | 数值 | 
    | --- | --- |
    | 用户名 | **Student** |
    | 密码 | **Pa55w.rd1234** |

1. 在 **“az30302a-vm0”** 的远程桌面会话中，在“服务器管理器”窗口中，选择 **“本地服务器”**，再选择 **“IE 增强的安全配置”** 标签旁的 **“启用”** 链接，然后在 **“IE 增强的安全配置”** 对话框中选择两者的 **“关闭”** 选项。

1. 在 **“az30302a-vm0”** 的远程桌面会话中，启动 Internet Explorer 并导航到 [“Azure 存储资源管理器”](https://azure.microsoft.com/zh-cn/features/storage-explorer/)的下载页面。

1. 在 **“az30302a-vm0”** 的远程桌面会话中，使用默认设置下载并安装 Azure 存储资源管理器。 


#### 任务 3：生成帐户级别的共享访问签名

1. 在 **“az30302a-vm0”** 的远程桌面会话中，启动 Internet Explorer，导航到 [“Azure 门户”](https://portal.azure.com)，然后通过提供要在本实验室中使用的订阅中具有所有者角色的用户帐户凭据来登录。

1. 导航到新创建的存储帐户的边栏选项卡，选择 **“访问密钥”** 并查看目标边栏选项卡的设置。

    >**注意**：每个存储帐户都有两个密钥，可以独立地重新生成这些密钥。知道存储帐户名和两个密钥中的任何一个，就可对整个存储帐户进行完全访问。 

1. 在“存储帐户”边栏选项卡中选择 **“共享访问签名”** 并查看目标边栏选项卡的设置。

1. 在生成的边栏选项卡上指定以下设置（将其他设置保留为默认值）：

    | 设置 | 数值 | 
    | --- | --- |
    | 允许的服务 | **blob** |
    | 允许的资源类型 | **服务** 和**容器** |
    | 允许的权限 | **读取**、**列表** 和**创建** |
    | Blob 版本控制权限 | 禁用 |
    | 开始时间 | 当前时区内当前时间前 24 小时 | 
    | 结束时间 | 当前时区内当前时间后 24 小时 |
    | 允许的协议 | **仅限 HTTPS** |
    | 签名密钥 | **key1** |

1. 选择 **“生成 SAS 和连接字符串”**。

1. 将 **“Blob 服务 SAS URL”** 的值复制到剪贴板。


#### 任务 4： 使用 Azure 存储资源管理器创建 Blob 容器

1. 在与 **“az30302a-vm0”** 的远程桌面会话中，启动 Azure 存储资源管理器。 

1. 在 Azure 存储资源管理器窗口的 **“连接到 Azure 存储”** 窗口中，选择 **“使用共享访问签名 (SAS) URI”**，然后选择 **“下一步”**。

1. 在 **“使用 SAS URI 附加”** 窗口中的 **“显示名称”** 文本框中，键入 **“az30302a-blobs”**， 在 **“URI”** 文本框中，将复制的值粘贴到剪贴板，然后选择 **“下一步”**。 

    >**注意**：这应自动填充 **“Blob 终结点”** 文本框的值。

1. 在 **“连接摘要”** 窗口中，选择 **“连接”**。 

1. 在 Azure 存储资源管理器窗口的 **“资源管理器”** 窗格中，导航到 **“az30302a-blobs”** 条目，将其展开并注意你只有访问 **“Blob 容器”** 终结点的权限。 

1. 右键选择 **“az30302a-blobs”** 条目，在右键菜单中选择 **“创建 Blob 容器”**，然后使用空白文本框将容器名称设置为 **“container1”**。

1. 选择 **“container1”**，在 **“container1”** 窗格中，选择 **“上传”**，然后在下拉列表中选择 **“上传文件”**。

1. 在 **“上传文件”** 窗口中，选择 **“选定的文件”** 标签旁边的省略号按钮，在 **“选择要上传的文件”** 窗口中，选择 **C:\Windows\system.ini**，然后选择 **“打开”**。

1. 回到 **“上传文件”** 窗口，选择 **“上传”**，并记录在 **“活动”** 列表中显示的错误消息。 

    >**注意**：这是预料之中的，因为共享访问签名不提供对象级权限。 

1. 使“Azure 存储资源管理器”窗口保持打开状态。


#### 任务 5： 使用 AzCopy 将文件上传到 Blob 容器

1. 在与 **“az30302a-vm0”** 的远程桌面会话中，在浏览器窗口中的 **“共享访问签名”** 边栏选项卡上，指定以下设置（将其他设置保留为默认值）：

    | 设置 | 数值 | 
    | --- | --- |
    | 允许的服务 | **Blob** |
    | 允许的资源类型 | **Object** |
    | 允许的权限 | **读取**， **创建** |
    | Blob 版本控制权限 | 禁用 |
    | 开始时间 | 当前时区内当前时间前 24 小时 | 
    | 结束时间 | 当前时区内当前时间后 24 小时 |
    | 允许的协议 | **仅限 HTTPS** |
    | 签名密钥 | **key1** |

1. 选择 **“生成 SAS 和连接字符串”**。

1. 将 **“SAS 令牌”** 值复制到剪贴板中。

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 提示选择 **“Bash”** 或 **“PowerShell”** 时，选择 **“PowerShell”**。 

1. 在 Cloud Shell 窗格中运行以下命令，创建一个文件并向其中添加一行文本：

   ```powershell
   New-Item -Path './az30302ablob.html'

   Set-Content './az30302ablob.html' '<h3>Hello from az30302ablob via SAS</h3>'
   ```

1. 在 Cloud Shell 窗格中运行以下命令，将新创建的文件作为 Blob 上传到之前在本练习中所创建 Azure 存储帐户的 container1 中（将 `<sas_token>` 占位符替换为你之前在此任务中复制到剪贴板的共享访问签名值）：

   ```powershell
   $storageAccountName = (Get-AzStorageAccount -ResourceGroupName 'az30302a-labRG')[0].StorageAccountName

   azcopy cp './az30302ablob.html' "https://$storageAccountName.blob.core.windows.net/container1/az30302ablob.html<sas_token>"
   ```

1. 查看 azcopy 生成的输出结果，并验证作业是否成功完成。

1. 关闭“Cloud Shell”窗格。

1. 在与 **“az30302a-vm0”** 的远程桌面会话中，在浏览器窗口的存储帐户边栏选项卡的 **“Blob 服务”** 部分，选择 **“容器”**。

1. 在容器列表中选择 **“container1”**。

1. 在 **“container1”** 边栏选项卡中，验证 **az30302ablob.html** 是否出现在 blob 列表中。


#### 任务 6： 使用 Blob 级共享访问签名访问 Blob

1. 在与 **az30302a-vm0** 的远程桌面会话中，在浏览器窗口中的 **“container1”** 边栏选项卡中选择 **“更改访问级别”**，验证是否将其设置为 **“私人（无匿名访问）”**，然后选择 **“取消”**。

    >**注意**：要允许匿名访问，可以将公共访问级别设置为 **“Blob（仅限 Blob 的匿名读取访问）”** 或 **“容器（容器和 Blob 的匿名读取访问）”**。

1. 在 **“container1”** 边栏选项卡中选择 **“az30302ablob.html”**。

1. 在 **“az30302ablob.html”** 边栏选项卡中选择 **“生成 SAS”**，查看可用选项而不进行修改，然后选择 **“生成 SAS 令牌和 URL”**。

1. 将 **“Blob SAS URL”** 值复制到剪贴板中。

1. 在浏览器窗口中打开一个新选项卡，并导航到在上一步中复制到剪贴板的 URL 处。

1. 验证消息 **“Hello from az30302ablob via SAS”** 是否出现在浏览器窗口中。


### 练习 2： 使用 Azure Active Directory 配置 Azure 存储 Blob 服务授权
  
本次练习的主要任务如下：

1. 创建 Azure AD 用户

1. 为 Azure 存储 Blob 服务启用 Azure Active Directory 授权

1. 使用 AzCopy 将文件上传到 Blob 容器


#### 任务 1：创建 Azure AD 用户

1. 在与 **“az30302a-vm0”** 的远程桌面会话中，在浏览器窗口的 **“Cloud Shell”** 窗格内打开 **“PowerShell”** 会话。

1. 在 Cloud Shell 窗格中运行以下命令，向你的 Azure 广告租户进行明确身份验证：

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
   New-AzureADUser -AccountEnabled $true -DisplayName 'az30302auser1' -PasswordProfile $passwordProfile -MailNickName 'az30302auser1' -UserPrincipalName "az30302auser1@$domainName"
   ```

1. 在 Cloud Shell 窗格中，运行下列命令以标识新创建的 Azure AD 用户的用户主体名称：

   ```powershell
   (Get-AzureADUser -Filter "MailNickName eq 'az30302auser1'").UserPrincipalName
   ```

1. 注意用户主体名称。在本次练习中，你后续将需要该域名。 

1. 关闭“Cloud Shell”窗格。


#### 任务 2：为 Azure 存储 Blob 服务启用 Azure Active Directory 授权

1. 在与 **“az30302a-vm0”** 的远程桌面会话中，在显示 Azure 门户的浏览器窗口中，导航回 **“container1”** 边栏选项卡。

1. 在 **“container1”** 边栏选项卡上，选择 **“切换到 Azure AD 用户帐户”**。

1. 请注意，错误消息表明你不再具有列出 blob 容器数据的权限。这在预料之中。

    >**注意**：尽管在订阅中拥有 **“所有者”** 角色，你还需要分配内置角色或自定义角色，以提供对存储帐户 Blob 内容的访问权限，例如 **“存储 Blob 数据所有者”**、 **“存储 Blob 数据参与者”** 或 **“存储 Blob 数据读取者”**。

1. 在 Azure 门户中，导航回托管 **“container1”** 的存储帐户的边栏选项卡，选择 **“访问控制 (IAM) ”**，选择 **“+ 添加”**，然后在下拉列表中选择 **“添加角色分配”**。 

    >**注意**：写下存储帐户的名称。需要在下一个任务中使用它。

1. 在 **“添加角色分配”** 边栏选项卡上，在 **“角色”** 下拉列表中，选择 **“存储 Blob 数据所有者”**，请确保 **“分配访问权限至”** 下拉列表条目设置为 **“Azure AD 用户、组或服务主体”**，从显示在 **“选择”** 文本框下面的列表中选择你的用户帐户和你在上一个任务中创建的用户帐户，然后选择 **“保存”**。

1. 导航回 **“container1”** 边栏选项卡，并验证是否可以看到容器内容。


#### 任务 3：使用 AzCopy 将文件上传到 Blob 容器

1. 在 **az30302a-vm0** 远程桌面会话中，在浏览器窗口，导航至 [“开始使用 AzCopy”](https://docs.microsoft.com/zh-cn/azure/storage/common/storage-use-azcopy-v10)。

1. 下载 azcopy.zip 文件并将 azcopy.exe 解压缩到 **C:\\Labfiles** 文件夹（如果需要，请创建文件夹）。

1. 在 **az30302a-vm0** 远程桌面会话中，启动 Windows PowerShell。 

1. 在 Windows PowerShell 提示符下，运行以下命令下载 **azcopy.zip** 存档，提取其内容，然后切换到包含 **azcopy.exe** 的位置：

   ```powershell
   $url = 'https://aka.ms/downloadazcopy-v10-windows'
   $zipFile = '.\azcopy.zip'

   Invoke-WebRequest -Uri $Url -OutFile $zipFile

   Expand-Archive -Path $zipFile -DestinationPath '.\'

   Set-Location -Path 'azcopy*'
   ```

1. 在 Windows PowerShell 提示符后运行以下命令，使用在本练习的第一个任务中创建的 Azure AD 用户帐户对 AzCopy 进行身份验证。 

   ```powershell
   .\azcopy.exe login
   ```

    >**注意**： 不能将 Microsoft 帐户用于此目的，这是必须首先创建 Azure AD 用户帐户的原因。

1. 按照上一步中通过运行命令生成的消息中提供的说明对 **az30302auser1** 用户帐户进行身份验证。当提示输入凭据时，请提供在本练习任务 1 中所记录帐户的用户主体名及其密码 **Pa55w.rd1234**。

1. 成功验证身份后，在 Windows PowerShell 提示符后运行以下命令，创建要上传到 **container1** 的文件：

   ```powershell
   New-Item -Path './az30302bblob.html'

   Set-Content './az30302bblob.html' '<h3>Hello from az30302bblob via Azure AD</h3>'
   ```

1. 在 Windows PowerShell 提示符后运行以下命令，将新建的文件作为 Blob 上传到在上一个练习中所创建 Azure 存储帐户的 **container1**（将 `<storage_account_name>` 占位符替换为你在上一个任务中记录的存储帐户的值）：

   ```powershell
   .\azcopy cp './az30302bblob.html' 'https://<storage_account_name>.blob.core.windows.net/container1/az30302bblob.html'
   ```

1. 查看 azcopy 生成的输出结果，并验证作业是否成功完成。

1. 在 Windows PowerShell 提示符后运行以下命令，验证你对那些 由 AzCopy 实用程序提供的安全性上下文之外的已上传 blob 没有访问权限（将 `<storage_account_name>` 占位符替换为你在上一个任务中记录的存储帐户的值）：

   ```powershell
   Invoke-WebRequest -Uri 'https://<storage_account_name>.blob.core.windows.net/container1/az30302bblob.html'
   ```

1. 在 **az30302a-vm0** 远程桌面会话中，在浏览器窗口，导航回 **“container1”**。

1. 在 **“container1”** 边栏选项卡中，验证 **az30302bblob.html** 是否出现在 blob 列表中。

1. 在 **“container1”** 边栏选项卡中，选择 **“更改访问级别”**，将“公共访问级别”设置为 **“Blob（仅针对 Blob 的匿名读取访问权限）”**，然后选择 **“确定”**。 

1. 切换回 Windows PowerShell 提示符，然后重新运行以下命令，以验证你现在是否可以匿名访问上传的 Blob（将 `<storage_account_name>` 占位符替换位你在上一个任务中记录的存储帐户的值）：

   ```powershell
   Invoke-WebRequest -Uri 'https://<storage_account_name>.blob.core.windows.net/container1/az30302bblob.html'
   ```


### 练习 3：实现 Azure 文件存储。
  
本次练习的主要任务如下：

1. 创建 Azure 存储文件共享

1. 将驱动器从 Windows 映射到 Azure 存储文件共享

1. 删除实验室中部署的 Azure 资源


#### 任务 1：创建 Azure 存储文件共享

1. 在 **az30302a-vm0** 远程桌面会话中，在显示 Azure 门户的浏览器窗口，导航回到你在本实验室的第一个练习中创建的“存储帐户边栏选项卡”，然后在 **“文件服务”** 部分，选择 **“文件共享”**。

1. 选择 **“+ 文件共享”** 并使用以下设置创建文件共享：

    | 设置 | 数值 |
    | --- | --- |
    | 名称 | **az30302a-share** |
    | 配额 | **1024** |


#### 任务 2：将驱动器从 Windows 映射到 Azure 存储文件共享

1. 选择新创建的文件共享，然后选择 **“连接”**。

1. 在 **“连接”** 边栏选项卡上，请确保 **“Windows”** 选项卡处于选中状态，然后选择 **“复制到剪贴板”**。

    >**注意**： Azure 存储文件共享映射分别使用存储帐户名以及两个存储帐户密钥之一作为用户名和密码的等效项，以便获得对目标共享的访问权限。

1. 在 **“az30302a-vm0”** 的远程桌面会话中，在 PowerShell 提示符后粘贴并执行你复制的脚本。

1. 验证脚本是否成功完成。 

1. 启动文件资源管理器，导航到 **Z:** 驱动器并验证映射是否成功。 

1. 在文件资源管理器中创建名为 **“Folder1”** 的文件夹，并在该文件夹内创建一个名为 **“File1.txt”** 的文本文件。

1. 切换回显示 Azure 门户的浏览器窗口，在 **“az30302a-share”** 边栏选项卡中选择 **“刷新”**，并验证 **“Folder1”** 是否出现在文件夹列表中。 

1. 选择 **“Folder1”** 并验证 **“File1.txt”** 是否出现在文件列表中。


#### 任务 3：删除实验室中部署的 Azure 资源

1. 在 **“az30302a-vm0”** 的远程桌面会话中 ，在显示 Azure 门户的浏览器窗口中，在“Cloud Shell”窗格中启动 PowerShell 会话。

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```powershell
   Get-AzResourceGroup -Name 'az30302*'
   ```

    > **注意**：验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```powershell
   Get-AzResourceGroup -Name 'az30302*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. 关闭“Cloud Shell”窗格。

1. 在 Azure 门户中，导航到与你的 Azure 订阅关联的 Azure Active Directory 租户的 **“用户”** 边栏选项卡。

1. 在用户帐户列表中，选择代表 **“az30302auser1”** 用户帐户的条目，选择工具栏中的省略号图标，选择 **“删除用户”**，并在提示确认时选择 **“是”**。  
