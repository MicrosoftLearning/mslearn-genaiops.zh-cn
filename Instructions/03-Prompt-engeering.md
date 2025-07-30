---
lab:
  title: 使用 Prompty 探索提示工程
  description: 了解如何使用 Prompty 快速测试并优化与语言模型配合使用的不同提示，确保提示的设计和编排能够实现最佳效果。
---

## 使用 Prompty 探索提示工程

本练习大约需要 **45** 分钟。

> **注意**：本练习假设你对 Azure AI Foundry 有一定的了解，因此某些说明故意不那么详细，以鼓励更积极的探索和实践学习。

## 简介

在构思过程中，你需要使用语言模型快速测试和改进不同的提示。 可以采用多种方法实施提示工程，可通过 Azure AI Foundry 门户中的操场，或使用 Prompty 执行更多代码优先方法。

在本练习中，你将使用通过 Azure AI Foundry 部署的模型，在 Azure Cloud Shell 中探索如何使用 Prompty 进行提示工程。

## 设置环境

要完成本练习中的任务，你需要：

- 一个 Azure AI Foundry 中心、
- 一个 Azure AI Foundry 项目、
- 一个已部署的模型（如 GPT-4o）。

### 创建 Azure AI 中心和项目

> **备注**：如果已有 Azure AI 中心和项目，则可以跳过此过程并使用现有项目。

可以通过 Azure AI Foundry 门户手动创建 Azure AI 中心和项目，以及部署练习中使用的模型。 但是，还可以通过将模板应用程序与 [Azure Developer CLI (azd)](https://aka.ms/azd) 结合使用来自动执行此过程。

1. 在 Web 浏览器中打开 [Azure 门户](https://portal.azure.com)，网址为：`https://portal.azure.com`，然后使用 Azure 凭据登录。

1. 使用页面顶部搜索栏右侧的 **[\>_]** 按钮在 Azure 门户中创建新的 Cloud Shell，选择 ***PowerShell*** 环境。 Cloud Shell 在 Azure 门户底部的窗格中提供命令行接口。 有关如何使用 Azure Cloud Shell 的详细信息，请参阅 [Azure Cloud Shell 文档](https://docs.microsoft.com/azure/cloud-shell/overview)。

    > **备注**：如果以前创建了使用 *Bash* 环境的 Cloud Shell，请将其切换到 ***PowerShell***。

1. 在 PowerShell 窗格中，输入以下命令以克隆本练习的存储库：

     ```powershell
    rm -r mslearn-genaiops -f
    git clone https://github.com/MicrosoftLearning/mslearn-genaiops
     ```

1. 克隆存储库后，输入以下命令以初始化初学者模板。 
   
     ```powershell
    cd ./mslearn-genaiops/Starter
    azd init
     ```

1. 出现提示后，为新环境命名，因其将用作为所有预配资源提供唯一名称的依据。
        
1. 接下来，输入以下命令以运行初学者模板。 它将使用依赖资源、AI 项目、AI 服务和联机终结点预配 AI 中心。

     ```powershell
    azd up
     ```

1. 出现提示时，请选择要使用的订阅，然后选择以下任一资源预配位置：
   - 美国东部
   - 美国东部 2
   - 美国中北部
   - 美国中南部
   - 瑞典中部
   - 美国西部
   - 美国西部 3
    
1. 等待脚本完成 - 此过程通常需要大约 10 分钟；但在某些情况下可能需要更长的时间。

    > **备注**：Azure OpenAI 资源在租户级别受区域配额限制。 上面列出的区域包括本练习中所用模型类型的默认配额。 随机选择区域可降低单个区域达到其配额限制的风险。 如果达到配额限制，则可能需要在其他区域中创建另一个资源组。 详细了解 [每个区域的模型可用性](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models?tabs=standard%2Cstandard-chat-completions#global-standard-model-availability)

    <details>
      <summary><b>故障排除提示</b>：给定区域中没有可用的配额</summary>
        <p>如果由于所选区域中没有可用配额而收到任何模型的部署错误，请尝试运行以下命令：</p>
        <ul>
          <pre><code>azd env set AZURE_ENV_NAME new_env_name
   azd env set AZURE_RESOURCE_GROUP new_rg_name
   azd env set AZURE_LOCATION new_location
   azd up</code></pre>
        将<code>new_env_name</code>、<code>new_rg_name</code>和<code>new_location</code>替换为新值。 新位置必须是练习开始时列出的任一区域，例如<code>eastus2</code>、<code>northcentralus</code>等。
        </ul>
    </details>

1. 预配所有资源后，使用以下命令提取终结点和 AI 服务资源的访问密钥。 请注意，必须将`<rg-env_name>`和`<aoai-xxxxxxxxxx>`替换为资源组和 AI 服务资源的名称。 两者都在部署的输出中打印。

     ```powershell
    Get-AzCognitiveServicesAccount -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property endpoint
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. 复制这些值，因为稍后将使用这些值。
   
### 设置本地开发环境

要快速试验和循环访问，请在 Visual Studio (VS) Code 中使用 Prompty。 让我们准备好 VS Code 以进行本地构思。

1. 打开 VS Code 并**克隆**以下 Git 存储库：[https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. 将克隆存储在本地驱动器上，并在克隆后打开该文件夹。
1. 在 VS Code 中的“扩展”窗格中，搜索并安装 **Prompty** 扩展。
1. 在 VS Code 资源管理器（左窗格）中，右键单击“**Files/03**”文件夹。
1. 从下拉菜单中选择“**新建 Prompty**”。
1. 打开名为 **basic.prompty** 的新建文件。
1. 选择右上角的“**播放**”按钮（或按 F5）运行 Prompty 文件。
1. 提示登录时，选择“**允许**”。
1. 选择 Azure 帐户并登录。
1. 返回到 VS Code，其中“**输出**”窗格将打开并显示错误消息。 错误消息应告知未指定或找不到已部署的模型。

要修复此错误，需要配置模型以供 Prompty 使用。

## 更新提示元数据

要执行 Prompty 文件，需要指定用于生成响应的语言模型。 元数据在 Prompty 文件的 *frontmatter* 中定义。 让我们使用模型配置和其他信息更新元数据。

1. 打开 Visual Studio Code 终端窗格。
1. 复制 **basic.prompty** 文件（在同一文件夹中），并将副本重命名为`chat-1.prompty`。
1. 打开 **chat-1.prompty** 并更新以下字段以更改一些基本信息：

    - **名称：**

        ```yaml
        name: Python Tutor Prompt
        ```

    - **说明：**

        ```yaml
        description: A teaching assistant for students wanting to learn how to write and edit Python code.
        ```

    - **已部署的模型**：

        ```yaml
        azure_deployment: ${env:AZURE_OPENAI_CHAT_DEPLOYMENT}
        ```

1. 接下来，在 **azure_deployment** 参数下为 API 密钥添加以下占位符。

    - **终结点密钥**：

        ```yaml
        api_key: ${env:AZURE_OPENAI_API_KEY}
        ```

1. 保存更新后的 Prompty 文件。

Prompty 文件现在具有所有必要的参数，但某些参数使用占位符来获取所需的值。 占位符存储在同一文件夹的 **.env** 文件中。

## 更新模型配置

要指定 Prompty 使用的模型，需要在 .env 文件中提供模型的信息。

1. 在 **Files/03** 文件夹中打开 **.env** 文件。
1. 使用此前从 Azure 门户中模型部署的输出中复制的值更新每个占位符：

    ```yaml
    - AZURE_OPENAI_CHAT_DEPLOYMENT="gpt-4"
    - AZURE_OPENAI_ENDPOINT="<Your endpoint target URI>"
    - AZURE_OPENAI_API_KEY="<Your endpoint key>"
    ```

1. 保存 .env 文件。
1. 再次运行 **chat-1.prompty** 文件。

现在，应收到 AI 生成的响应，尽管它只使用示例输入而与应用场景无关。 让我们更新模板，使其成为 AI 助教。

## 编辑示例部分

示例部分指定对 Prompty 的输入，并提供要使用的默认值（如果未提供任何输入）。

1. 编辑以下参数的字段：

    - **firstName**：选择任何其他名称。
    - **context**：移除整个部分。
    - **question**：将提供的文本替换为：

    ```yaml
    What is the difference between 'for' loops and 'while' loops?
    ```

    现在，**示例**部分应该如下所示：
    
    ```yaml
    sample:
    firstName: Daniel
    question: What is the difference between 'for' loops and 'while' loops?
    ```

    1. 运行更新的 Prompty 文件并查看输出。

