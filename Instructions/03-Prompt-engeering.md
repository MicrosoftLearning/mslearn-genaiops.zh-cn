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

> **注意**：如果你已有 Azure AI 项目，可以跳过这一过程，并使用现有项目。

可以通过 Azure AI Foundry 门户手动创建 Azure AI 项目，以及部署练习中使用的模型。 但是，还可以通过将模板应用程序与 [Azure Developer CLI (azd)](https://aka.ms/azd) 结合使用来自动执行此过程。

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

### 在 Cloud Shell 中设置虚拟环境

若要快速试验和迭代，需要在 Cloud Shell 中使用一组 Python 脚本。

1. 在 Cloud Shell 命令行窗格中，输入以下命令，导航到本练习中使用的代码文件所在的文件夹：

     ```powershell
    cd ~/mslearn-genaiops/Files/03/
     ```

1. 输入以下命令以激活虚拟环境，并安装所需的库：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv openai tiktoken azure-ai-projects prompty[azure]
    ```

1. 输入以下命令以打开已提供的配置文件：

    ```powershell
   code .env
    ```

    该文件已在代码编辑器中打开。

1. 在代码文件中，将 ENDPOINTNAME**** 和 APIKEY**** 占位符替换为之前复制的终结点和密钥值。
1. 替换占位符后**，在代码编辑器中，使用 Ctrl+S**** 命令或通过右键单击 >“保存”**** 来保存更改，然后使用 Ctrl+Q**** 命令或通过右键单击 >“退出”**** 来关闭代码编辑器，同时将 cloud shell 命令行保持打开状态。

## 优化系统提示

在生成式 AI 的大规模部署中，尽可能缩短系统提示的长度，同时保持其功能性，是一项关键任务。 更短的提示可以加快响应速度，因为 AI 模型处理的词元更少，此外，还能降低计算资源的消耗。

1. 请输入以下命令，打开已提供的应用程序文件：

    ```powershell
   code optimize-prompt.py
    ```

    查看代码，注意脚本执行的 `start.prompty` 模板文件已经有预定义的系统提示。

1. 运行 `code start.prompty` 以查看系统提示。 请思考如何在保持意图清晰有效的前提下缩短提示。 例如：

   ```python
   original_prompt = "You are a helpful assistant. Your job is to answer questions and provide information to users in a concise and accurate manner."
   optimized_prompt = "You are a helpful assistant. Answer questions concisely and accurately."
   ```

   删除冗余字词，聚焦核心指令。 将优化后的提示保存到文件中。

### 测试和验证优化

测试提示更改非常重要，可以确保在不降低质量的情况下减少词元使用量。

1. 运行 `code token-count.py` 以打开并查看练习中提供的词元计数器应用。 如果你使用的优化提示不同于上述示例中提供的提示，也可以在此应用中使用它。

1. 使用 `python token-count.py` 运行脚本，并观察词元计数的差异。 确保优化后的提示仍可以生成高质量的回复。

## 分析用户交互

了解用户与应用的交互方式，有助于识别可提高词元使用率的模式。

1. 查看用户提示的示例数据集：

    - “总结《战争与和平》的情节。”******
    - “关于猫有哪些有趣的事实？”****
    - “为一家利用 AI 优化供应链的初创企业撰写一份详细的商业计划。”****
    - “将 "Hello, how are you?" 翻译成法语。”****
    - “向 10 岁儿童解释量子纠缠。”****
    - “为我提供 10 个有关科幻短篇小说的创意。”****

    对于每个示例提示，判断它更可能导致 AI 生成简短、篇幅适中还是较长/复杂的回复。************

1. 查看分类。 你注意到哪些模式？ 如下所示：

    - **** 抽象程度（例如，创造性与事实性）是否会影响回复长度？
    - **** 开放式提示是否倾向于生成更长的内容？
    - **** 指令复杂度（例如，“用 10 岁孩子能理解的方式解释”）会如何影响生成的回复？

1. 输入以下命令以运行优化-提示**** 应用程序：

    ```
   python optimize-prompt.py
    ```

1. 使用上面提供的一些示例来验证分析。
1. 现在，使用以下长格式提示并查看其输出：

    ```
   Write a comprehensive overview of the history of artificial intelligence, including key milestones, major contributors, and the evolution of machine learning techniques from the 1950s to today.
    ```

1. 按以下要求重写此提示：

    - 限制范围
    - 明确要求简洁
    - 使用格式设置或结构来引导回复

1. 对比重写前后的回复，验证是否获得了更简洁的答案。

> 注意****：可以使用 `token-count.py` 来比较两次回复的词元使用量。
<br>
<details>
<summary><b>重写后的提示示例：</b></summary><br>
<p>“用项目符号列出 AI 历史上的 5 个关键里程碑。”</p>
</details>

## [**** 可选] 在实际应用场景中应用优化

1. 假设你正在构建一个客户支持聊天机器人，它需要提供快速、准确的答复。
1. 将你优化后的系统提示和模板集成到聊天机器人的代码中（可以使用 `optimize-prompt.py` 作为起点**）。
1. 使用各种用户查询测试聊天机器人，确保其响应高效且准确。

## 结束语

在生成式 AI 应用程序中，提示优化是降低成本、提高性能的关键技能。 通过缩短提示、使用模板以及分析用户交互，可以打造更高效、更具可扩展性的解决方案。

## 清理

如果已完成对 Azure AI 服务的探索，则应删除在本练习中创建的资源，以避免产生不必要的 Azure 成本。

1. 返回到包含 Azure 门户的浏览器标签页（或在新的浏览器标签页中重新打开 [Azure 门户](https://portal.azure.com?azure-portal=true) ），查看在其中部署了本练习中使用的资源的资源组的内容。
1. 在工具栏中，选择“删除资源组”****。
1. 输入资源组名称，并确认要删除该资源组。
