---
lab:
  title: 协调 RAG 系统
---

## 协调 RAG 系统

检索增强生成 (RAG) 系统将大型语言模型的强大功能与高效的检索机制相结合，以提高生成响应的准确性和相关性。 通过利用 LangChain 实现业务流程并利用 Azure AI Foundry 实现 AI 功能，我们可以创建可靠的管道，从数据集中检索相关信息并生成一致的响应。 在本练习中，你将完成设置环境、预处理数据、创建嵌入和生成索引的步骤，最终使你能够有效实现 RAG 系统。

## 场景

假设你要生成一款提供有关酒店建议的应用。 在应用中，你需要一个代理，该代理不仅可以推荐酒店，还可以回答用户可能提出的问题。

你已选择 GPT-4 模型提供生成式答案。 现在，你需要生成 RAG 系统，该系统将基于其他用户的评论向模型提供基础数据，引导聊天行为提供个性化建议。

首先，部署必要的资源来生成此应用程序。

## 创建 Azure AI 中心和项目

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
        
1. 接下来，输入以下命令以运行初学者模板。 它将使用依赖资源、AI 项目、AI 服务和联机终结点预配 AI 中心。 它还将部署模型 GPT-4 Turbo、GPT-4o 和 GPT-4o mini。

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

## 设置本地开发环境

要快速试验和循环访问，需在 Visual Studio (VS) Code 中将笔记本与 Python 代码配合使用。 让我们准备好 VS Code 以进行本地构思。

1. 打开 VS Code 并**克隆**以下 Git 存储库：[https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. 将克隆存储在本地驱动器上，并在克隆后打开该文件夹。
1. 在 VS Code 资源管理器（左窗格中）的 **Files/04** 文件夹中，打开 **04-RAG.ipynb** 笔记本。
1. 运行笔记本中的所有单元格。

## 清理

如果已完成对 Azure AI 服务的探索，则应删除在本练习中创建的资源，以避免产生不必要的 Azure 成本。

1. 返回到包含 Azure 门户的浏览器标签页（或在新的浏览器标签页中重新打开 [Azure 门户](https://portal.azure.com?azure-portal=true) ），查看在其中部署了本练习中使用的资源的资源组的内容。
1. 在工具栏中，选择“删除资源组”****。
1. 输入资源组名称，并确认要删除该资源组。
