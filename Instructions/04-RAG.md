---
lab:
  title: 协调 RAG 系统
  description: 了解如何在应用中实现检索增强生成 (RAG) 系统，以提高生成的响应的准确性和相关性。
---

## 协调 RAG 系统

检索增强生成 (RAG) 系统将大型语言模型的强大功能与高效的检索机制相结合，以提高生成响应的准确性和相关性。 通过利用 LangChain 实现业务流程并利用 Azure AI Foundry 实现 AI 功能，我们可以创建可靠的管道，从数据集中检索相关信息并生成一致的响应。 在本练习中，你将完成设置环境、预处理数据、创建嵌入和生成索引的步骤，最终使你能够有效实现 RAG 系统。

该练习大约需要 **30** 分钟。

## 场景

假设你要构建一个应用，用于提供有关伦敦的酒店的建议。 在应用中，你需要一个代理，该代理不仅可以推荐酒店，还可以回答用户可能提出的问题。

你已选择 GPT-4 模型提供生成式答案。 现在，你需要生成 RAG 系统，该系统将基于其他用户的评论向模型提供基础数据，引导聊天行为提供个性化建议。

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
    cd ~/mslearn-genaiops/Files/04/
     ```

1. 输入以下命令以激活虚拟环境，并安装所需的库：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv langchain-text-splitters langchain-community langchain-openai
    ```

1. 输入以下命令以打开已提供的配置文件：

    ```powershell
   code .env
    ```

    该文件已在代码编辑器中打开。

1. 在代码文件中，将 your_azure_openai_service_endpoint**** 和 your_azure_openai_service_api_key**** 占位符替换为之前复制的终结点和密钥值。
1. 替换占位符后**，在代码编辑器中，使用 Ctrl+S**** 命令或通过右键单击 >“保存”**** 来保存更改，然后使用 Ctrl+Q**** 命令或通过右键单击 >“退出”**** 来关闭代码编辑器，同时将 cloud shell 命令行保持打开状态。

## 实现 RAG

现在，你将运行一个脚本，该脚本会引入并预处理数据、创建嵌入，并生成向量存储和索引，从而有效支持实现一个 RAG 系统。

1. 运行以下命令以编辑所提供的脚本****：

    ```powershell
   code RAG.py
    ```

1. 在脚本中，查找“# 初始化将从 LangChain 的集成套件中使用的组件”****。 在此注释下方，粘贴以下代码：

    ```python
   # Initialize the components that will be used from LangChain's suite of integrations
   llm = AzureChatOpenAI(azure_deployment=llm_name)
   embeddings = AzureOpenAIEmbeddings(azure_deployment=embeddings_name)
   vector_store = InMemoryVectorStore(embeddings)
    ```

1. 查看脚本，你会注意到它使用了一个包含酒店评论的 .csv 文件作为基础数据。 可以通过在命令行窗格中运行命令 `download app_hotel_reviews.csv` 并打开此文件来查看此文件的内容。
1. 接下来，查找“# 将文档拆分为区块以用于嵌入和矢量存储”****。 在此注释下方，粘贴以下代码：

    ```python
   # Split the documents into chunks for embedding and vector storage
   text_splitter = RecursiveCharacterTextSplitter(
       chunk_size=200,
       chunk_overlap=20,
       add_start_index=True,
   )
   all_splits = text_splitter.split_documents(docs)
    
   print(f"Split documents into {len(all_splits)} sub-documents.")
    ```

    上面的代码将一组大型文档拆分为较小的区块。 这一点很重要，因为许多嵌入模型（如用于语义搜索或矢量数据库的模型）都有令牌限制，并且在较短的文本中表现更好。

1. 接下来，查找“# 嵌入每个文本区块的内容并将这些嵌入内容插入到矢量存储中”****。 在此注释下方，粘贴以下代码：

    ```python
   # Embed the contents of each text chunk and insert these embeddings into a vector store
   document_ids = vector_store.add_documents(documents=all_splits)
    ```

1. 接下来，查找“# 根据用户输入从矢量存储中检索相关文档”****。 在此注释下方，粘贴以下代码，注意正确的缩进：

    ```python
   # Retrieve relevant documents from the vector store based on user input
   retrieved_docs = vector_store.similarity_search(question, k=10)
   docs_content = "\n\n".join(doc.page_content for doc in retrieved_docs)
    ```

    上面的代码在矢量存储中搜索与用户输入的问题最相似的文档。 使用用于文档的同一嵌入模型将问题转换为矢量。 然后，系统将此矢量与所有存储的矢量进行比较，并检索最相似的矢量。

1. 保存所做更改。
1. 在命令行中输入以下命令以**运行脚本**：

    ```powershell
   python RAG.py
    ```

1. 应用程序运行后，你可以开始提问，例如 `Where can I stay in London?`，然后继续提出更具体的问题进行深入查询。

## 结束语

在本练习中，你构建了一个典型的 RAG 系统及其主要组件。 通过使用你自己的文档来辅助模型的响应，你为 LLM 提供了在制定响应内容时使用的基础数据。 对于企业解决方案，这意味着可以将生成式 AI 的输出限定在企业内容范围内。

## 清理

如果已完成对 Azure AI 服务的探索，则应删除在本练习中创建的资源，以避免产生不必要的 Azure 成本。

1. 返回到包含 Azure 门户的浏览器标签页（或在新的浏览器标签页中重新打开 [Azure 门户](https://portal.azure.com?azure-portal=true) ），查看在其中部署了本练习中使用的资源的资源组的内容。
1. 在工具栏中，选择“删除资源组”****。
1. 输入资源组名称，并确认要删除该资源组。
