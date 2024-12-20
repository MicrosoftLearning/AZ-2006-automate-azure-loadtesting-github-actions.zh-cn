---
lab:
  title: 实验室 02：使用适用于 Azure 的 GitHub Actions 将 Web 应用发布到 Azure 应用服务
  module: 'Module 2: Implement GitHub Actions for Azure'
---

# 概述

在此实验室中，你将学习如何实现将 Web 应用部署到 Azure 应用服务的 GitHub Actions 工作流。

完成本实验室后，你将能够：

* 为 CI/CD 实现 GitHub Action 工作流。
* 解释 GitHub Action 工作流的基本特征。

**预计完成时间：40 分钟**

## 先决条件

* 具有**活动订阅**的 Azure 帐户。 如果你还没有，可在 [https://azure.com/free](https://azure.com/free) 注册免费试用版。
    * 支持 Azure Web 门户的[浏览器](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices)。
    * 在 Azure 订阅中具有参与者或所有者角色的 Microsoft 帐户或 Microsoft Entra 帐户。 有关详细信息，请参阅[使用 Azure 门户列出 Azure 角色分配](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)和[在 Azure Active Directory 中查看和分配管理员角色](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)。
* 一个 GitHub 帐户。 如果还没有可用于此实验室的 GitHub 帐户，请按照[注册新的 GitHub 帐户](https://github.com/join)中的说明创建一个帐户。

## 说明

## 练习 1：将 eShopOnWeb 导入 GitHub 存储库

在本练习中，你会将 [eShopOnWeb](https://github.com/MicrosoftLearning/eShopOnWeb) 存储库导入到自己的 GitHub帐户。 存储库按以下方式组织：

| Folder | 目录 |
| -- | -- |
| **.ado** | Azure DevOps YAML 管道 |
| **.devcontainer** | 进行配置，以使用容器（在 VS Code 本地或在 GitHub Codespaces 中）开发。 |
| **infra** | 某些实验室方案中使用的 Bicep 和 ARM 基础结构即代码模板 |
| **.github** | YAML GitHub 工作流定义 |
| **src** | 用于实验室方案的 .NET 8 网站 |

### 任务 1：导入 eShopOnWeb 存储库

1. 在 Web 浏览器中，导航到 [http://github.com](http://github.com) 并使用自己的帐户登录。
1. 启动导入过程 [https://github.com/new/import](https://github.com/new/import)。
1. 在“**将项目导入到 GitHub**”页中输入以下信息。

    | 设置 | 操作 |
    |--|--|
    | **源存储库的 URL** | 输入 `https://github.com/MicrosoftLearning/eShopOnWeb` |
    | **所有者** | 选择 GitHub 别名 |
    | **存储库名称** | 输入 **eShopOnWeb** |
    | **隐私** | 选择“**所有者**”后，将显示隐私选项。 选择“公共”。 |

1. 选择“**开始导入**”，等待导入过程完成。
1. 在存储库页上，选择“**设置**”，然后在左侧导航窗格中选择“**操作 > 常规**”。
1. 在该页的“**操作权限**”部分中，选择“**允许所有操作和可重用工作流**”选项，然后选择“**保存**”。

> **备注：** eShopOnWeb 是一个大型存储库，可能需要 5-10 分钟才能完成导入。

## 练习 2：创建 Azure 资源并配置 GitHub 

在本练习中，你将创建一个 Azure 服务主体，以授权 GitHub 从 GitHub Actions 访问 Azure 订阅。 你还将查看并修改 GitHub 工作流，该工作流用于生成、测试网站并将网站部署到 Azure。

### 任务 1：创建 Azure 服务主体并将其另存为 GitHub 机密

在此任务中，你将创建资源组和 Azure 服务主体。 GitHub 使用该服务主体部署所需的 eShopOnWeb 应用。

1. 在浏览器中，导航到 Azure 门户 [https://portal.azure.com](https://portal.azure.com)。
1. 打开“**Cloud Shell**”，然后选择“**Bash**”模式。 **备注：** 如果这是你第一次启动 Cloud Shell，则需要配置永久性存储。
1. 使用以下 `az group create` 命令创建资源组。 将 `<location>` 替换为自己附近的区域。

    ```
    az group create -n az2006-rg -l <location>
    ```

1. 运行以下命令，为稍后将在实验室中部署的 **Azure 应用服务**注册资源提供程序。

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. 运行以下命令，为部署到 Azure 应用服务的 Web 应用生成随机名称。 复制并保存命令输出的名称，以供稍后在该实验室中使用。

    ```
    myAppName=az2006app$RANDOM
    echo $myAppName
    ```

1. 运行以下命令来检索订阅 ID。 请务必复制并保存命令的输出，此订阅 ID 值稍后将在本实验室中使用。

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

1. 使用以下命令创建服务主体。 第一个命令将资源组的 ID 存储在变量中。

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)

    az ad sp create-for-rbac --name GH-Action-eshoponweb --role contributor --scopes $rgId
    ```

    >**重要提示：** 该命令将输出一个 JSON 对象，其中包含用于以 Microsoft Entra 标识（服务主体）的名称对 Azure 进行身份验证的标识符。 复制 JSON 对象，以便在以下步骤中使用。 

1. 在浏览器窗口中，导航到 **eShopOnWeb** GitHub 存储库。
1. 在存储库页上，选择“**设置**”，然后在左侧导航窗格中选择“**机密和变量 > 操作**”。
1. 选择“**新建存储库机密**”，并输入以信息：
    * **名称**：`AZURE_CREDENTIALS`
    * **机密**：输入创建服务主体时生成的 JSON 对象。
1. 选择“添加机密”。

### 任务 2：修改和执行 GitHub 工作流

在此任务中，你将修改提供的 *eshoponweb-cicd.yml* GitHub 工作流并执行该工作流，以在自己的订阅中部署解决方案。

1. 在浏览器窗口中，返回到 eShopOnWeb GitHub 存储库。
1. 选择“**<> 代码**”，然后在主分支中选择 **eShopOnWeb/.github/workflows** 文件夹中的 **eshoponweb-cicd.yml**。 此工作流定义了 eShopOnWeb 应用的 CI/CD 流程。

    ![显示文件在文件夹结构中位置的屏幕截图。](./media/eshop-cid-workflow.png)
1. 选择“**编辑此文件**”。
1. 将该文件的 `env:` 部分中的字段更改为以下值。

    | 字段 | 操作 |
    |--|--|
    | TEMPLATE-FILE： | `az2006-rg` |
    | LOCATION： | `eastus`（或者，创建资源组时选择的区域。） |
    | TEMPLATE-FILE： | 无更改 |
    | SUBSCRIPTION-ID： | 你的订阅 ID。 |
    | WEBAPP-NAME： | 之前在实验室中创建的随机生成的 Web 应用名称。 |

1. 仔细阅读工作流，提供的注释可帮助你理解工作流中的步骤。
1. 通过删除 `#`，取消注释文件顶部的 **on** 部分。 每次推送到主分支时工作流都会触发，同时也提供了手动触发 (`workflow_dispatch`)。
1. 选择页面右上角的“**提交更改...**”。
1. 将显示一个弹出窗口。 接受默认值（直接提交到主分支）并选择“**提交更改**”。 工作流将自动执行。

### 任务 3：查看 GitHub 工作流执行

在此任务中，你将查看 GitHub 工作流执行并查看正在运行的应用程序。

1.选择“**操作**”，在执行之前会看到工作流设置。

1. 在页面的“**所有工作流**”部分中，选择“**eShopOnWeb 生成和测试**”作业。 

1. 工作流由两个操作组成：**buildandtest** 和 **deploy**。 可以选择任一操作并查看其进度，或等待作业完成。

1. 导航到 Azure 门户 [https://portal.azure.com](https://portal.azure.com)，然后导航到之前创建的 **az2006-rg** 资源组。 请注意，GitHub Actions 使用 bicep 模板在资源组中创建了 Azure 应用服务计划和应用服务。 

1. 选择应用服务资源（之前生成的唯一应用名称），然后选择页面顶部附近的“**浏览**”，查看已部署的 Web 应用。

## 练习 3：清理资源

在本练习中，删除之前在实验室中创建的资源。

1. 导航到 Azure 门户 [https://portal.azure.com](https://portal.azure.com) 并启动 Cloud Shell。 选择 **Bash** shell 会话。

1. 运行以下命令以删除 `az2006-rg` 资源组。 它还将移除应用服务计划和应用服务实例。

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**备注**：该命令以异步方式执行（通过 `--no-wait` 参数设置），因此，尽管之后可立即在同一 Bash 会话中运行另一个 Azure CLI 命令，但实际上要花几分钟才能移除资源组。

## 审阅

在此实验室中，你实现了部署 Azure Web 应用的 GitHub Action 工作流。
