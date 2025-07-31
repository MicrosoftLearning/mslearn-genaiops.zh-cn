---
lab:
  title: 从模型目录中选择语言模型
  description: 了解如何比较和选择适合生成式 AI 项目的模型。
---

## 从模型目录中选择语言模型

定义用例后，可以使用模型目录来探索 AI 模型是否已解决你的问题。 可以使用模型目录选择要部署的模型，然后可以进行比较以探索最适合你需求的模型。

在本练习中，你将通过 Azure AI Foundry 门户中的模型目录比较两种语言模型。

该练习大约需要 **30** 分钟。

## 场景

假设你要生成一款应用，以帮助学生了解如何在 Python 中编写代码。 在应用中，你需要一个自动化导师，可以帮助学生编写和评估代码。 在一个练习中，学生需要根据以下示例图像编写必要的 Python 代码来绘制饼图：

![饼图显示数学 (34.9%)、物理 (28.6%)、化学 (20.6%) 和英语 (15.9%) 部分考试取得的分数](./images/demo.png)

需要选择接受图像作为输入的语言模型，并能够生成准确的代码。 满足这些条件的可用模型是 GPT-4 Turbo、GPT-4o 和 GPT-4o mini。

首先，在 Azure AI Foundry 门户中部署必要的资源来使用这些模型。

## 创建 Azure AI 中心和项目

可以通过 Azure AI Foundry 门户手动创建 Azure AI 中心和项目，以及部署练习中使用的模型。 但是，还可以通过将模板应用程序与 [Azure Developer CLI (azd)](https://aka.ms/azd) 结合使用来自动执行此过程。

1. 在 Web 浏览器中打开 [Azure 门户](https://portal.azure.com)，网址为：`https://portal.azure.com`，然后使用 Azure 凭据登录。

1. 使用页面顶部搜索栏右侧的 **[\>_]** 按钮在 Azure 门户中创建新的 Cloud Shell，选择 ***PowerShell*** 环境。 Cloud Shell 在 Azure 门户底部的窗格中提供命令行接口。 有关如何使用 Azure Cloud Shell 的详细信息，请参阅 [Azure Cloud Shell 文档](https://docs.microsoft.com/azure/cloud-shell/overview)。

    > **备注**：如果以前创建了使用 *Bash* 环境的 Cloud Shell，请将其切换到 ***PowerShell***。

1. 在 Cloud Shell 工具栏的“**设置**”菜单中，选择“**转到经典版本**”。

    **<font color="red">在继续操作之前，请确保已切换到 Cloud Shell 的经典版本。</font>**

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

## 比较模型

你知道，有三个模型可以接受图像作为输入，其推理基础结构由 Azure 完全托管。 现在，你需要比较这些模型，以确定哪个模型最适合我们的用例。

1. 在新的浏览器标签页中，打开 [Azure AI Foundry 门户](https://ai.azure.com)（网址为 `https://ai.azure.com`），并使用 Azure 凭据登录。
1. 如果系统提示，请选择之前创建的 AI 项目。
1. 使用左侧菜单导航到“**模型目录**”页。
1. 选择“**比较模型**”（在“搜索”窗格中查找筛选器旁边的按钮）。
1. 移除所选模型。
1. 逐个添加要比较的三个模型：**GPT-4**、**GPT-4o** 和 **GPT-4o-mini**。 对于 **GPT-4**，请确保所选版本为 **turbo-2024-04-09**，因其是唯一接受图像作为输入的版本。
1. 将 X 轴更改为“**准确度**”。
1. 确保将 Y 轴设置为“**成本**”。

查看绘图并尝试回答以下问题：

- *哪个模型更准确？*
- *使用哪种模型更便宜？*

基准指标准确性是根据公开可用的泛型数据集计算的。 从绘图中，我们已经可以筛选出其中一个模型，因其每个令牌的成本最高，但准确性却不是最高。 做出决定之前，让我们针对用例探索其余两个模型的输出质量。

## 在 Cloud Shell 中设置开发环境

若要快速试验和迭代，需要在 Cloud Shell 中使用一组 Python 脚本。

1. 返回到 Azure 门户选项卡，导航到之前通过部署脚本创建的资源组，然后选择“Azure AI Foundry”**** 资源。
1. 在资源的“概述”页中，选择“单击此处以查看终结点”，并复制 AI Foundry API 终结点。********
1. 将终结点保存在记事本中。 你将会使用它连接到客户端应用程序中的项目。
1. 返回“Azure 门户”选项卡，如果你之前关闭了 Cloud Shell，请重新打开它，并运行以下命令，导航到本练习中所使用的代码文件所在的文件夹：

     ```powershell
    cd ~/mslearn-genaiops/Files/02/
     ```

1. 在 Cloud Shell 命令行窗格中，输入以下命令以安装需要的库：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv azure-identity azure-ai-projects openai matplotlib
    ```

1. 输入以下命令以打开已提供的配置文件：

    ```powershell
   code .env
    ```

    该文件已在代码编辑器中打开。

1. 在代码文件中，将 your_project_endpoint**** 占位符替换为之前复制的项目终结点。 请注意，本练习中使用的第一个和第二个模型分别是 gpt-4o**** 和 gpt-4o-mini****。
1. 替换占位符后**，在代码编辑器中，使用 Ctrl+S**** 命令或通过右键单击 >“保存”**** 来保存更改，然后使用 Ctrl+Q**** 命令或通过右键单击 >“退出”**** 来关闭代码编辑器，同时将 cloud shell 命令行保持打开状态。

## 将提示发送到已部署的模型

现在，你将运行多个脚本，向已部署的模型发送不同的提示。 这些交互将生成数据，你可以稍后在 Azure Monitor 中查看这些数据。

1. 运行以下命令以**查看第一个提供的脚本**：

    ```powershell
   code model1.py
    ```

脚本会将本练习中使用的图像编码为数据 URL。 此 URL 将用于将图像直接嵌入聊天补全请求中，并与第一个文本提示一同发送。 接下来，脚本将输出模型响应，并将其添加到聊天历史记录中，然后提交第二个提示。 二个提示的提交和存储是为了让稍后观察到的指标更具参考价值，但你也可以取消注释代码中的可选部分，以同时输出第二个响应。

1. 在 Cloud Shell 命令行窗格中，输入以下命令以登录到 Azure。

    ```
   az login
    ```

    **<font color="red">必须登录到 Azure - 即使 Cloud Shell 会话已经过身份验证。</font>**

    > **备注**：在大多数情况下，仅使用 *az login* 就足够了。 但是，如果在多个租户中有订阅，则可能需要使用 *--tenant* 参数指定租户。 有关详细信息，请参阅[使用 Azure CLI 以交互方式登录到 Azure](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)。
    
1. 出现提示时，请按照说明在新选项卡中打开登录页，并输入提供的验证码和 Azure 凭据。 然后在命令行中完成登录过程，并在出现提示时选择包含 Azure AI Foundry 中心的订阅。
1. 登录后，输入以下命令来运行应用程序：

    ```powershell
   python model1.py
    ```

    该模型将生成一个响应，并使用 Application Insights 捕获，以便进一步分析。 我们使用第二个模型来探索其中的差异。

1. 在代码编辑器下面的 Cloud Shell 命令行窗格中，输入以下命令以运行第二个脚本：****

    ```powershell
   python model2.py
    ```

    现在得到了这两个模型的输出，两者有什么区别？

    > **注意**：你还可以选择测试作为答案提供的脚本，方法是：复制代码块、运行命令 `code your_filename.py`、将代码粘贴到编辑器中、保存文件，然后运行命令 `python your_filename.py`。 如果脚本成功运行，你应该会得到一张已保存的图像，可以通过 `download imgs/gpt-4o.jpg` 或 `download imgs/gpt-4o-mini.jpg` 下载。

## 比较各模型的标记使用情况

最后，你将运行第三个脚本，用于绘制每个模型随时间处理的标记数量图表。 此数据是从 Azure Monitor 获取的。

1. 在运行最后一个脚本之前，需要先从 Azure 门户复制 Azure AI Foundry 资源的资源 ID。 转到 Azure AI Foundry 资源的“概述”页，然后选择“JSON 视图”。**** 复制资源 ID 并替换代码文件中的 `your_resource_id` 占位符：

    ```powershell
   code plot.py
    ```

1. 保存所做更改。

1. 在代码编辑器下面的 Cloud Shell 命令行窗格中，输入以下命令以运行第三个脚本：****

    ```powershell
   python plot.py
    ```

1. 脚本完成后，输入以下命令以下载指标图：

    ```powershell
   download imgs/plot.png
    ```

## 结束语

查看指标图并回忆之前观察到的准确性与成本图的基准值，能否得出最适合你的用例的模型？ 输出准确性的差异是否大于生成的标记（以及成本）的差异？

## 清理

如果已完成对 Azure AI 服务的探索，则应删除在本练习中创建的资源，以避免产生不必要的 Azure 成本。

1. 返回到包含 Azure 门户的浏览器标签页（或在新的浏览器标签页中重新打开 [Azure 门户](https://portal.azure.com?azure-portal=true) ），查看在其中部署了本练习中使用的资源的资源组的内容。
1. 在工具栏中，选择“删除资源组”****。
1. 输入资源组名称，并确认要删除该资源组。
