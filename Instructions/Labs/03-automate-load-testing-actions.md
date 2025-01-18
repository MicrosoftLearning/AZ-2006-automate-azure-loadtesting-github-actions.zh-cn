---
lab:
  title: '实验室 03：使用 GitHub Actions 自动执行 Azure 负载测试 '
  module: 'Module 3: Implement Azure Load Testing'
---

# 概述

在本实验室中，你将学习如何配置 GitHub Actions 以部署示例 Web 应用，并使用 Azure 负载测试服务启动负载测试。

在此实验中，将执行以下操作：

* 在 Azure 中创建应用服务和负载测试资源。
* 创建并配置服务主体，以支持 GitHub Actions 工作流在你的 Azure 帐户中执行操作。
* 使用 GitHub Actions 工作流将 .NET 8 应用程序部署到 Azure 应用服务。
* 更新 GitHub Actions 工作流，以调用基于 URL 的负载测试。

**预计完成时间：40 分钟**

## 先决条件

* 具有**活动订阅**的 Azure 帐户。 如果你还没有，可在 [https://azure.com/free](https://azure.com/free) 注册免费试用版。
    * 支持 Azure Web 门户的[浏览器](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices)。
    * 在 Azure 订阅中具有参与者或所有者角色的 Microsoft 帐户或 Microsoft Entra 帐户。 有关详细信息，请参阅[使用 Azure 门户列出 Azure 角色分配](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)和[在 Azure Active Directory 中查看和分配管理员角色](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)。
* 一个 GitHub 帐户。 如果还没有可用于此实验室的 GitHub 帐户，请按照[注册新的 GitHub 帐户](https://github.com/join)中的说明创建一个帐户。


## 说明

## 练习 1：将示例应用导入 GitHub 存储库

在本练习中，你会将 [Azure 负载测试示例应用](https://github.com/MicrosoftLearning/azure-load-test-sample-app)存储库导入自己的 GitHub 帐户。

### 任务 1：导入 eShopOnWeb 存储库

1. 在 Web 浏览器中，导航到 [http://github.com](http://github.com) 并使用自己的帐户登录。
1. 启动导入过程 [https://github.com/new/import](https://github.com/new/import)。
1. 在“**将项目导入到 GitHub**”页中输入以下信息。

    | 设置 | 操作 |
    |--|--|
    | **源存储库的 URL** | 输入 `https://github.com/MicrosoftLearning/azure-load-test-sample-app` |
    | **所有者** | 选择 GitHub 别名 |
    | **存储库名称** | 命名存储库 |
    | **隐私** | 选择“**所有者**”后，将显示隐私选项。 选择“公共”。 |

1. 选择“**开始导入**”，等待导入过程完成。
1. 在新存储库页上，选择“**设置**”，然后在左侧导航窗格中选择“**操作 > 常规**”。
1. 在该页的“**操作权限**”部分中，选择“**允许所有操作和可重用工作流**”选项，然后选择“**保存**”。

## 练习 2 - 在 Azure 中创建资源

在本练习中，你将在 Azure 中创建部署应用和运行测试所需的资源。 

### 任务 1：使用 Azure CLI 创建资源。

在此任务中，你将创建以下 Azure 资源：

* 资源组
* 应用服务计划
* 应用服务实例
* 负载测试实例

1. 在浏览器中，导航到 Azure 门户 [https://portal.azure.com](https://portal.azure.com)。
1. 打开“**Cloud Shell**”，然后选择“**Bash**”模式。 **备注：** 如果这是你第一次启动 Cloud Shell，可能需要配置永久性存储。

1. 一次运行一个以下命令，以创建其余步骤中的命令所用的变量。 将 `<mylocation>` 替换为你的首选位置。

    ```
    myLocation=<mylocation>
    myAppName=az2006app$RANDOM
    ```
1. 运行以下命令以创建包含其他资源的资源组。

    ```
    az group create -n az2006-rg -l $myLocation
    ```

1. 运行以下命令，为 **Azure 应用服务**注册资源提供程序：

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. 运行以下命令以创建应用服务计划。 **备注：** 应用服务计划中使用的 B1 计划可能会产生成本。 

    ```
    az appservice plan create -g az2006-rg -n az2006webapp-plan --sku B1
    ```

1. 运行以下命令来为该应用创建应用服务实例。

    ```
    az webapp create -g az2006-rg -p az2006webapp-plan -n $myAppName --runtime "dotnet:8"
    ```

1. 运行以下命令来创建负载测试资源。 如果收到安装**负载**扩展的提示，请选择“是”。

    ```
    az load create -n az2006loadtest -g az2006-rg --location $myLocation
    ```

1. 运行以下命令来检索订阅 ID。 **备注：** 请务必复制并保存命令的输出，此实验室稍后会用到该订阅 ID 值。

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

### 任务 2：创建服务主体并配置授权

在此任务中，你将为应用创建一个服务主体并配置该服务主体，以进行 OpenID Connect 联合身份验证。

1. 在 Azure 门户中，搜索 **Microsoft Entra ID**，然后导航到该服务。

1. 在左侧导航窗格中，选择“**管理**”组中的“**应用注册**”。 

1. 在主面板中选择“**+ 新建注册**”并输入 `GH-Action-webapp` 作为名称，然后选择“**注册**”。

    >**重要提示：** 复制并保存“**应用程序（客户端）ID**”和“**目录（租户）ID**”值，供本实验室稍后使用。


1. 在左侧导航窗格中，选择“管理”组中的“证书和机密”，然后在主窗口中选择“联合凭据”************。 

1. 选择“**添加凭据**”，然后在选择下拉列表中选择“**部署 Azure 资源的 GitHub Actions**”。

1. 在“**连接 GitHub 帐户**”部分中输入以下信息。 **注意：** 这些字段区分大小写。 

    | 字段 | 操作 |
    |--|--|
    | 组织 | 输入用户名或组织名称。 |
    | 存储库 | 输入之前在实验室中创建的存储库的名称。 |
    | 实体类型 | 选择“分支”。 |
    | GitHub 分支名 | 输入 **main**。 |

1. 在“**凭据详细信息**”部分中，为凭据命名，然后选择“**添加**”。

### 任务 3：将角色分配给服务主体

在此任务中，你将必要的角色分配给服务主体以访问资源。

1. 运行以下命令来分配“负载测试参与者”角色，以便 GitHub 工作流可以发送要运行的资源测试。 

    ```
    spAppId=$(az ad sp list --display-name GH-Action-webapp --query "[].{spID:appId}" --output tsv)

    loadTestId=$(az resource show -g az2006-rg -n az2006loadtest --resource-type "Microsoft.LoadTestService/loadtests" --query "id" -o tsv)

    az role assignment create --assignee $spAppId --role "Load Test Contributor"  --scope $loadTestId
    ```

1. 运行以下命令来分配“参与者”角色，以便 GitHub 工作流可以将应用部署到应用服务。 

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)
    
    az role assignment create --assignee $spAppId --role contributor --scope $rgId
    ```

## 练习 3：使用 GitHub Actions 部署和测试 Web 应用

在本练习中，你将存储库配置为运行包含的工作流。

* 工作流位于存储库的 *.github/workflows* 文件夹中。
* 工作流 *deploy.yml* 和 *loadtest.yml* 都配置为手动运行。

在本练习中，你将在浏览器中编辑存储库文件。 选择要编辑的文件后，你可以：
* 选择“**就地编辑**”，并在完成编辑后提交更改。 
* 使用 **github.dev** 打开文件，在浏览器中使用 Visual Studio Code 进行编辑。 如果选择此选项，可以通过选择顶部菜单中的“**返回到存储库**”来返回到默认存储库体验。

    ![编辑选项的屏幕截图。](./media/github-edit-options.png)

### 任务 1：配置机密

在此任务中，将机密添加到存储库，使工作流能够代表你登录到 Azure 并执行操作。

1. 在 Web 浏览器中，导航到“[GitHub](https://github.com)”并选择为此实验室创建的存储库。 
1. 选择存储库顶部的“**设置**”。
1. 在左侧导航窗格中，选择“**机密和变量**”，然后选择“**操作**”。
1. 在“**存储库机密**”部分中，添加以下三个机密。 通过选择“**新建存储库机密**”来添加机密。

    | 名称 | 机密 |
    |--|--|
    | `AZURE_CLIENT_ID` | 输入之前在实验室中保存的**应用程序（客户端）ID**。 |
    | `AZURE_TENANT_ID` | 输入之前在实验室中保存的**目录（租户）ID**。 |
    | `AZURE_SUBSCRIPTION_ID` | 输入之前在实验室中保存的订阅 ID 值。 |

### 任务 2：部署 Web 应用

1. 选择 *.github/workflows* 文件夹中的 *deploy.yml* 文件。

1. 编辑文件，并在 **env:** 部分中更改变量 `AZURE_WEB_APP` 的值。 将 `<your web app name>` 替换为之前在本实验室中创建的 Web 应用的名称。 提交更改。

1. 花些时间查看工作流的内容。

1. 选择存储库顶部导航中的“**操作**”。 

1. 在左侧导航窗格中，选择“**生成并发布**”。

1. 选择“**运行工作流**”下拉列表，然后选择“**运行工作流**”，并保留默认的“**分支: 主**”设置。 工作流可能需要一点时间才能启动。

如果出现问题导致工作流无法成功完成，请选择“**生成并发布**”工作流，然后在下一个屏幕上选择“**生成**”。 它将提供有关工作流的详细信息，并帮助诊断阻止其成功完成的问题。

### 任务 3：运行负载测试

1. 选择 *.github/workflows* 文件夹中的 *loadtest.yml* 文件。

1. 编辑文件，并在 **env:** 部分中更改变量 `AZURE_WEB_APP` 的值。 将 `<your web app name>**` 替换为之前在本实验室中创建的 Web 应用的名称。 提交更改。

1. 花些时间查看工作流的内容。

1. 选择存储库顶部导航中的“**操作**”。 

1. 选择左侧导航窗格中的“**负载测试**”。

1. 选择“**运行工作流**”下拉列表，然后选择“**运行工作流**”，并保留默认的“**分支: 主**”设置。 工作流可能需要一点时间才能启动。

    >**备注：** 完成工作流可能需要 5-10 分钟。 该测试运行时间为两分钟，而负载测试可能需要几分钟时间在 Azure 中排队并启动。 

如果出现问题导致工作流无法成功完成，请选择“**生成并发布**”工作流，然后在下一个屏幕上选择“**生成**”。 它将提供有关工作流的详细信息，并帮助诊断阻止其成功完成的问题。

#### 可选

存储库根目录中的 *config.yaml* 文件可指定负载测试的失败条件。 如果要强制负载测试失败，请执行以下步骤。

1. 编辑存储库根目录中的 *config.yaml* 文件。
1. 将 `- p90(response_time_ms) > 4000` 字段中的值更改为较低的值。 将其更改为 `- p90(response_time_ms) > 50` 最有可能导致测试失败。 这表示在 90% 的时间里，应用将在 50 毫秒内做出响应。 

### 任务 4：查看负载测试结果

从 CI/CD 管道运行负载测试时，可以直接在 CI/CD 输出日志中查看摘要结果。 因为测试结果保存为管道工件，所以还可以下载 CSV 文件进行进一步报告。

![屏幕截图显示工作流日志记录信息。](./media/github-actions-workflow-completed.png)

## 练习 4：清理资源

在本练习中，删除之前在实验室中创建的资源。

1. 导航到 Azure 门户 [https://portal.azure.com](https://portal.azure.com) 并启动 Cloud Shell。 选择 **Bash** shell 会话。

1. 运行以下命令以删除 `az2006-rg` 资源组。 它还将移除应用服务计划和应用服务实例。

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**备注**：该命令以异步方式执行（通过 `--no-wait` 参数设置），因此，尽管之后可立即在同一 Bash 会话中运行另一个 Azure CLI 命令，但实际上要花几分钟才能移除资源组。

## 审阅

在此实验室中，你实现了部署 Azure Web 应用并对其进行负载测试的 GitHub Action 工作流。
