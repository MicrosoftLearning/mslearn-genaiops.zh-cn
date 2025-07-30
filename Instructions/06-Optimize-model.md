---
lab:
  title: 使用合成数据集优化模型
  description: 了解如何创建合成数据集并使用它们来提高模型的性能和可靠性。
---

## 使用合成数据集优化模型

优化生成式 AI 应用程序涉及利用数据集增强模型的性能和可靠性。 通过使用合成数据，开发人员可以模拟真实数据中可能不存在的各种场景和边缘案例。 此外，评估模型的输出对于获取高质量和可靠的 AI 应用程序至关重要。 可以使用 Azure AI 评估 SDK 高效管理整个优化和评估过程，该 SDK 提供可靠的工具和框架来简化这些任务。

完成本练习大约需要 30 分钟****\*

> \* 此估计时长不包括练习末尾的可选任务。
## 场景

假设你要生成 AI 提供支持的智能指南应用，以增强游客在博物馆的体验。 该应用旨在回答有关历史人物的问题。 要评估来自应用的响应，需要创建全面的合成问答数据集，该数据集涵盖这些人物及其作品的各个方面。

你已选择 GPT-4 模型提供生成式答案。 现在，你需要构建能够生成上下文相关交互的模拟器，以评估 AI 在不同应用场景中的性能。

首先，部署必要的资源来生成此应用程序。

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

1. 预配所有资源后，使用以下命令提取终结点和 AI 服务资源的访问密钥。 请注意，必须将`<rg-env_name>`和`<aoai-xxxxxxxxxx>`替换为资源组和 AI 服务资源的名称。 两者都在部署的输出中打印。

     ```powershell
    Get-AzCognitiveServicesAccount -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property endpoint
     ```

     ```powershell
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. 复制这些值，因为稍后将使用这些值。

## 在 Cloud Shell 中设置开发环境

若要快速试验和迭代，需要在 Cloud Shell 中使用一组 Python 脚本。

1. 在 Cloud Shell 命令行窗格中，输入以下命令，导航到本练习中使用的代码文件所在的文件夹：

     ```powershell
    cd ~/mslearn-genaiops/Files/06/
     ```

1. 输入以下命令以激活虚拟环境，并安装所需的库：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv azure-ai-evaluation azure-ai-projects promptflow wikipedia aiohttp openai==1.77.0
    ```

1. 输入以下命令以打开已提供的配置文件：

    ```powershell
   code .env
    ```

    该文件已在代码编辑器中打开。

1. 在代码文件中，将 your_azure_openai_service_endpoint**** 和 your_azure_openai_service_api_key**** 占位符替换为之前复制的终结点和密钥值。
1. 替换占位符后**，在代码编辑器中，使用 Ctrl+S**** 命令或通过右键单击 >“保存”**** 来保存更改，然后使用 Ctrl+Q**** 命令或通过右键单击 >“退出”**** 来关闭代码编辑器，同时将 cloud shell 命令行保持打开状态。

## 生成综合数据

接下来，你将运行一个脚本，用于生成一个合成数据集，并利用该数据集评估预训练模型的质量。

1. 运行以下命令以编辑所提供的脚本****：

    ```powershell
   code generate_synth_data.py
    ```

1. 在脚本中，找到“#定义回调函数”****。
1. 在此注释下方，粘贴以下代码：

    ```
    async def callback(
        messages: List[Dict],
        stream: bool = False,
        session_state: Any = None,  # noqa: ANN401
        context: Optional[Dict[str, Any]] = None,
    ) -> dict:
        messages_list = messages["messages"]
        # Get the last message
        latest_message = messages_list[-1]
        query = latest_message["content"]
        context = text
        # Call your endpoint or AI application here
        current_dir = os.getcwd()
        prompty_path = os.path.join(current_dir, "application.prompty")
        _flow = load_flow(source=prompty_path)
        response = _flow(query=query, context=context, conversation_history=messages_list)
        # Format the response to follow the OpenAI chat protocol
        formatted_response = {
            "content": response,
            "role": "assistant",
            "context": context,
        }
        messages["messages"].append(formatted_response)
        return {
            "messages": messages["messages"],
            "stream": stream,
            "session_state": session_state,
            "context": context
        }
    ```

    可以通过指定目标回调函数，引入任何应用程序终结点进行模拟。 在本例中，你将使用一个带有 Prompty 文件 `application.prompty` 的 LLM 应用。 上述回调函数通过执行以下任务来处理模拟器生成的每条消息：
    * 检索最新的用户消息。
    * 从 application.prompty 加载提示流。
    * 使用提示流生成响应。
    * 设置响应格式以遵守 OpenAI 聊天协议。
    * 将助手的响应追加到消息列表。

    >**注意**：有关使用 Prompty 的详细信息，请参阅 [Prompty 文档](https://www.prompty.ai/docs)。

1. 接下来，找到“#运行模拟器”****。
1. 在此注释下方，粘贴以下代码：

    ```
    model_config = {
        "azure_endpoint": os.getenv("AZURE_OPENAI_ENDPOINT"),
        "api_key": os.getenv("AZURE_OPENAI_API_KEY"),
        "azure_deployment": os.getenv("AZURE_OPENAI_DEPLOYMENT"),
    }
    
    simulator = Simulator(model_config=model_config)
    
    outputs = asyncio.run(simulator(
        target=callback,
        text=text,
        num_queries=1,  # Minimal number of queries
    ))
    
    output_file = "simulation_output.jsonl"
    with open(output_file, "w") as file:
        for output in outputs:
            file.write(output.to_eval_qr_json_lines())
    ```

   上述代码将初始化模拟器并运行它，以基于之前从维基百科中提取的文本生成合成对话。

1. 接下来，找到“#评估模型”****。
1. 在此注释下方，粘贴以下代码：

    ```
    groundedness_evaluator = GroundednessEvaluator(model_config=model_config)
    eval_output = evaluate(
        data=output_file,
        evaluators={
            "groundedness": groundedness_evaluator
        },
        output_path="groundedness_eval_output.json"
    )
    ```

    现在你已经拥有一个数据集，可以评估你的生成式 AI 应用程序的质量和效果。 在上述代码中，你将使用基础性作为质量指标。

1. 保存所做更改。
1. 在代码编辑器下方的 Cloud Shell 命令行窗格中，输入以下命令以**运行脚本**：

    ```
   python generate_synth_data.py
    ```

    脚本完成后，可以运行 `download simulation_output.jsonl` 和 `download groundedness_eval_output.json` 以下载输出文件，并查看其内容。 如果基础性指标不接近 3.0，可以更改 `application.prompty` 文件中的 LLM 参数（例如 `temperature`、`top_p`、`presence_penalty` 或 `frequency_penalty`），然后重新运行脚本以生成新数据集并进行评估。 此外，还可以更改 `wiki_search_term`，根据不同的上下文获取合成数据集。

## （可选）微调模型

如果还有时间，可以使用生成的数据集在 Azure AI Foundry 中微调模型。 微调操作依赖于云基础结构资源，根据数据中心容量和并发需求，预配所需时间长短可能不尽相同。

1. 打开新的浏览器标签页，导航到 [Azure AI Foundry 门户](https://ai.azure.com)（网址为 `https://ai.azure.com`），并使用 Azure 凭据登录。
1. 在 AI Foundry 主页中，选择在练习开始时创建的项目。
1. 使用左侧菜单导航到“**生成和自定义**”部分下的“**微调**”页。
1. 选择按钮以添加新的微调模型，选择 **gpt-4o** 模型，然后选择“下一步”****。
1. 使用以下配置**微调**该模型：
    - **模型版本**：*选择默认版本*
    - **自定义方法**：监督式
    - **模型后缀**：`ft-travel`
    - **连接的 AI 资源**：*选择创建中心时创建的连接。默认情况下应处于选中状态。*
    - **训练数据**：上传文件

    <details>  
    <summary><b>故障排除提示</b>：权限错误</summary>
    <p>如果收到权限错误，请尝试执行以下操作进行故障排除：</p>
    <ul>
        <li>在 Azure 门户中，选择 AI 服务资源。</li>
        <li>在“资源管理”下的“标识”选项卡中，确认它是系统分配的托管标识。</li>
        <li>导航到关联的存储帐户。 在“IAM”页面上，添加角色分配<em>存储 Blob 数据所有者</em>。</li>
        <li>在“<strong>将访问权限分配给</strong>”下，选择“<strong>托管标识</strong>”、“<strong>+选择成员</strong>”，选择“<strong>所有系统分配的托管标识</strong>”，然后选择 Azure AI 服务资源。</li>
        <li>查看并分配以保存新设置，然后重试上一步。</li>
    </ul>
    </details>

    - **上传文件**：选择在上一步骤中下载的 JSONL 文件。
    - **验证数据**：无
    - **任务参数**：*保留默认设置*
1. 微调将启动，可能需要一些时间才能完成。

    > **备注**：微调和部署可能需要一些时间（30 分钟或更长），因此可能需要定期回查。 截至目前，你可以选择微调模型作业并查看其**日志**选项卡，以查看更多详细进展信息。

## （可选）部署微调的模型

成功完成微调后，即可部署微调后的模型。

1. 选择微调作业链接以打开其详细信息页面。 然后，选择**指标**选项卡，并浏览微调指标。
1. 使用以下配置部署微调后模型：
    - **部署名**：*有效的模型部署名*
    - **部署类型**：标准
    - **每分钟令牌数速率限制（数千个）**：5,000*（或订阅中可用的最大值（如果小于 5,000））
    - **内容筛选器**：默认
1. 等待部署完成，才能对其进行测试，这可能需要一段时间。 检查“**预配状态**”，直到成功（可能需要刷新浏览器才能看到更新的状态）。
1. 部署准备就绪后，导航到微调后模型并选择“**在操场中打开**”。

    现在，你已经部署了经过微调的模型，可以像使用任何基础模型一样，在“聊天”操场中对其进行测试。

## 结束语

在本练习中，你创建了一个模拟用户与聊天补全应用之间对话的合成数据集。 使用此数据集，可以评估应用响应的质量，并对其进行微调，以获得所需的结果。

## 清理

如果已完成对 Azure AI 服务的探索，则应删除在本练习中创建的资源，以避免产生不必要的 Azure 成本。

1. 返回到包含 Azure 门户的浏览器标签页（或在新的浏览器标签页中重新打开 [Azure 门户](https://portal.azure.com?azure-portal=true) ），查看在其中部署了本练习中使用的资源的资源组的内容。
1. 在工具栏中，选择“删除资源组”****。
1. 输入资源组名称，并确认要删除该资源组。
