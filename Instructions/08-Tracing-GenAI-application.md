---
lab:
  title: 使用跟踪分析和调试生成式 AI 应用
  description: 了解如何通过跟踪从用户输入到模型响应和后处理的工作流来调试生成式 AI 应用程序。
---

# 使用跟踪分析和调试生成式 AI 应用

此练习大约需要 **30** 分钟。

> **注意**：本练习假设你对 Azure AI Foundry 有一定的了解，因此某些说明故意不那么详细，以鼓励更积极的探索和实践学习。

## 简介

在本练习中，你将运行一个多步骤生成式 AI 助手，该助手会推荐徒步旅行并建议户外装备。 你将使用 Azure AI 推理 SDK 的跟踪功能来分析应用程序如何执行和识别模型及其周围的逻辑做出的关键决策点。

你将与已部署的模型交互以模拟真实用户旅程，跟踪应用程序的每个阶段，从用户输入到模型响应到后期处理，以及查看 Azure AI Foundry 中的跟踪数据。 这将帮助你了解跟踪如何增强可观测性、简化调试并支持生成式 AI 应用程序的性能优化。

## 设置环境

要完成本练习中的任务，你需要：

- 一个 Azure AI Foundry 中心、
- 一个 Azure AI Foundry 项目、
- 一个已部署模型（如 GPT-4o）以及
- 一个已连接的 Application Insights 资源。

### 在 Azure AI Foundry 项目中部署模型

为快速设置 Azure AI Foundry 项目，下面提供了使用 Azure AI Foundry 门户 UI 的简单说明。

1. 在 Web 浏览器中打开 [Azure AI Foundry 门户](https://ai.azure.com)，网址为：`https://ai.azure.com`，然后使用 Azure 凭据登录。
1. 在主页的“**浏览模型和功能**”部分中，搜索 `gpt-4o` 模型；我们将在项目中使用它。
1. 在搜索结果中，选择 **gpt-4o** 模型以查看其详细信息，然后在模型的页面顶部，选择“**使用此模型**”。
1. 当提示创建项目时，输入项目的有效名称并展开“**高级选项**”。
1. 选择“**自定义**”，并为项目指定以下设置：
    - **Azure AI Foundry 资源**：*Azure AI Foundry 资源的有效名称*
    - **订阅**：Azure 订阅
    - **资源组**：*创建或选择资源组*
    - **区域**：*选择任何**支持 AI 服务的位置***\*

    > \* 某些 Azure AI 资源受区域模型配额约束。 如果稍后在练习中达到配额限制，你可能需要在不同的区域中创建另一个资源。

1. 选择“**创建**”并等待项目（包括所选的 gpt-4 模型部署）创建。
1. 在左侧的导航窗格中，选择“概述”以查看项目的主页。****
1. 在“终结点和密钥”区域，确保已选择 Azure AI Foundry 库，并查看“Azure AI Foundry 项目终结点”************。
1. 将终结点保存在记事本中。**** 你将使用此终结点连接到客户端应用程序中的项目。

### 连接 Application Insights

将 Application Insights 连接到 Azure AI Foundry 中的项目，开始收集数据以进行分析。

1. 使用左侧的菜单，然后选择“**跟踪**”页。
1. **创建新的** Application Insights 资源以连接到应用。
1. 输入 Application Insights 资源名称，然后选择“创建”****。

Application Insights 现已连接到项目，数据将开始收集以供分析。

## 使用 Cloud Shell 运行生成式 AI 应用

你将从 Azure Cloud Shell 连接到 Azure AI Foundry 项目，并通过编程方式与已部署的模型交互，作为生成式 AI 应用程序的一部分。

### 与已部署的模型交互

首先，检索与已部署的模型交互所需的身份验证信息。 然后，你将访问 Azure Cloud Shell 并更新生成式 AI 应用的代码。

1. 打开新的浏览器选项卡（使 Azure AI Foundry 门户在现有选项卡中保持打开状态）。
1. 在新选项卡中，浏览到 [Azure 门户](https://portal.azure.com)，网址为：`https://portal.azure.com`；如果出现提示，请使用 Azure 凭据登录。
1. 使用页面顶部搜索栏右侧的 **[\>_]** 按钮在 Azure 门户中新建 Cloud Shell，选择订阅中不含存储的 ***PowerShell*** 环境。
1. 在 Cloud Shell 工具栏的“**设置**”菜单中，选择“**转到经典版本**”。

    **<font color="red">在继续操作之前，请确保已切换到 Cloud Shell 的经典版本。</font>**

1. 在 Cloud Shell 窗格中，输入并运行以下命令：

    ```
    rm -r mslearn-genaiops -f
    git clone https://github.com/microsoftlearning/mslearn-genaiops mslearn-genaiops
    ```

    此命令会克隆包含本练习代码文件的 GitHub 存储库。

1. 克隆存储库后，导航到包含应用程序代码文件的文件夹：  

    ```
   cd mslearn-genaiops/Files/08
    ```

1. 在 Cloud Shell 命令行窗格中，输入以下命令以安装需要的库：

    ```
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv openai azure-identity azure-ai-projects opentelemetry-instrumentation-openai-v2 azure-monitor-opentelemetry
    ```

1. 输入以下命令以打开已提供的配置文件：

    ```
   code .env
    ```

    该文件已在代码编辑器中打开。

1. 在代码文件：

    1. 在代码文件中，将 **your_project_endpoint** 占位符替换为项目的终结点（从 Azure AI Foundry 门户中的项目“**概述**”页复制）。
    1. 将 **your_model_deployment** 占位符替换为你为 GPT-4o 模型部署分配的名称（默认方式`gpt-4o`）。

1. 替换占位符后，在代码编辑器中，使用 CTRL+S 命令或右键单击>“保存”以保存更改，然后使用 CTRL+Q 命令或右键单击>“退出”以关闭代码编辑器，同时让 cloud shell 命令行保持打开状态。**********************

### 更新生成式 AI 应用的代码

设置环境并配置 .env 文件后，可以准备 AI 助手脚本以供执行。 在连接到 AI 项目并启用 Application Insights 旁边，需要：

- 与部署的模型交互。
- 定义函数以指定提示。
- 定义调用所有函数的主流。

将这三个部分添加到起始脚本。

1. 运行以下命令以**打开已提供的脚本**：

    ```
   code start-prompt.py
    ```

    你将看到多个关键行已留空或标记为空 #注释。 任务是通过将下面的正确行复制并粘贴到适当的位置来完成脚本。

1. 在脚本中，找到 **# Function to call the model and handle tracing**。
1. 在此注释下方，粘贴以下代码：

    ```
   # Function to call the model and handle tracing
   def call_model(system_prompt, user_prompt, span_name):
       with tracer.start_as_current_span(span_name) as span:
           span.set_attribute("session.id", SESSION_ID)
           span.set_attribute("prompt.user", user_prompt)
           start_time = time.time()
    
           response = chat_client.chat.completions.create(
               model=model_deployment,
               messages=[
                   { 
                       "role": "system", 
                       "content": system_prompt 
                   },
                   { 
                       "role": "user", 
                       "content": user_prompt
                   }
               ]
           )
    
           duration = time.time() - start_time
           output = response.choices[0].message.content
           span.set_attribute("response.time", duration)
           span.set_attribute("response.tokens", len(output.split()))
           return output
    ```

1. 在脚本中，找到 **# Function to recommend a hike based on user preferences**。
1. 在此注释下方，粘贴以下代码：

    ```
   # Function to recommend a hike based on user preferences 
   def recommend_hike(preferences):
        with tracer.start_as_current_span("recommend_hike") as span:
            prompt = f"""
            Recommend a named hiking trail based on the following user preferences.
            Provide only the name of the trail and a one-sentence summary.
            Preferences: {preferences}
            """
            response = call_model(
                "You are an expert hiking trail recommender.",
                prompt,
                "recommend_model_call"
            )
            span.set_attribute("hike_recommendation", response.strip())
            return response.strip()
    ```

1. 在脚本中，找到 **#---- Main Flow ----**。
1. 在此注释下方，粘贴以下代码：

    ```
   if __name__ == "__main__":
       with tracer.start_as_current_span("trail_guide_session") as session_span:
           session_span.set_attribute("session.id", SESSION_ID)
           print("\n--- Trail Guide AI Assistant ---")
           preferences = input("Tell me what kind of hike you're looking for (location, difficulty, scenery):\n> ")

           hike = recommend_hike(preferences)
           print(f"\n✅ Recommended Hike: {hike}")

           # Run profile function


           # Run match product function


           print(f"\n🔍 Trace ID available in Application Insights for session: {SESSION_ID}")
    ```

1. **保存在脚本中所做的更改**。
1. 在 Cloud Shell 命令行窗格中，输入以下命令以登录到 Azure。

    ```
   az login
    ```

    **<font color="red">必须登录到 Azure - 即使 Cloud Shell 会话已经过身份验证。</font>**

    > **备注**：在大多数情况下，仅使用 *az login* 就足够了。 但是，如果在多个租户中有订阅，则可能需要使用 *--tenant* 参数指定租户。 有关详细信息，请参阅[使用 Azure CLI 以交互方式登录到 Azure](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)。
    
1. 出现提示时，请按照说明在新选项卡中打开登录页，并输入提供的验证码和 Azure 凭据。 然后在命令行中完成登录过程，并在出现提示时选择包含 Azure AI Foundry 中心的订阅。
1. 登录后，输入以下命令来运行应用程序：

    ```
   python start-prompt.py
    ```

1. 提供要查找的徒步旅行类型的一些说明，例如：

    ```
   A one-day hike in the mountains
    ```

    该模型将生成一个响应，并使用 Application Insights 捕获。 可以在 **Azure AI Foundry 门户**中可视化跟踪。

> **注意**：监控数据可能需要几分钟才能显示在 Azure Monitor 中。

## 在 Azure AI Foundry 门户中查看跟踪数据

运行脚本后，你捕获了 AI 应用程序的执行跟踪。 现在，你将使用 Azure AI Foundry 中的 Application Insights 进行探索。

> **备注：** 稍后将再次运行代码，并在 Azure AI Foundry 门户中再次查看跟踪。 让我们先了解一下在何处查找用于可视化的跟踪。

### 导航到 Azure AI Foundry 门户

1. **使 Cloud Shell 保持打开状态！** 你将返回此代码，以更新代码并再次运行它。
1. 在浏览器中导航至已打开 **Azure AI Foundry 门户** 的标签页。
1. 使用左侧的菜单，选择“**跟踪**”。
1. *如果*未显示任何数据，**请刷新**视图。
1. 选择跟踪 **train_guide_session**，以打开显示更多详细信息的新窗口。

### 查看跟踪

此视图显示“跟踪指南 AI 助手”的完整会话的跟踪。

- **顶级范围**：trail_guide_session 这是父范围。 它表示助手从头到尾的整个执行。

- **嵌套子范围**：每个缩进行表示嵌套操作。 你将找到：

    - **recommend_hike** 捕获逻辑以决定徒步旅行。
    - **recommend_model_call** 是由 call_model() inside recommend_hike 创建的范围。
    - **chat gpt-4o** 由 Azure AI 推理 SDK 自动检测，以显示实际的 LLM 交互。

1. 可以单击任意范围以查看：

    1. 持续时间。
    1. 其属性，如用户提示、使用的令牌、响应时间。
    1. 使用 **span.set_attribute(...)** 附加的任何错误或自定义数据。

## 向代码添加更多函数

1. 导航到浏览器中的选项卡，并打开 Azure 门户****。
1. 运行以下命令**重新打开脚本：**

    ```
   code start-prompt.py
    ```

1. 在脚本中，找到 **# Function to generate a trip profile for the recommended hike**。
1. 在此注释下方，粘贴以下代码：

    ```
   def generate_trip_profile(hike_name):
       with tracer.start_as_current_span("trip_profile_generation") as span:
           prompt = f"""
           Hike: {hike_name}
           Respond ONLY with a valid JSON object and nothing else.
           Do not include any intro text, commentary, or markdown formatting.
           Format: {{ "trailType": ..., "typicalWeather": ..., "recommendedGear": [ ... ] }}
           """
           response = call_model(
               "You are an AI assistant that returns structured hiking trip data in JSON format.",
               prompt,
               "trip_profile_model_call"
           )
           print("🔍 Raw model response:", response)
           try:
               profile = json.loads(response)
               span.set_attribute("profile.success", True)
               return profile
           except json.JSONDecodeError as e:
               print("❌ JSON decode error:", e)
               span.set_attribute("profile.success", False)
               return {}
    ```

1. 在脚本中，找到 **# Function to match recommended gear with products in the catalog**。
1. 在此注释下方，粘贴以下代码：

    ```
   def match_products(recommended_gear):
       with tracer.start_as_current_span("product_matching") as span:
           matched = []
           for gear_item in recommended_gear:
               for product in mock_product_catalog:
                   if any(word in product.lower() for word in gear_item.lower().split()):
                       matched.append(product)
                       break
           span.set_attribute("matched.count", len(matched))
           return matched
    ```

1. 在脚本中，找到 **# Run profile function**。
1. 在下面并与注释**对齐**，粘贴以下代码：

    ```
           profile = generate_trip_profile(hike)
           if not profile:
               print("Failed to generate trip profile. Please check Application Insights for trace.")
               exit(1)

           print(f"\n📋 Trip Profile for {hike}:")
           print(json.dumps(profile, indent=2))
    ```

1. 在脚本中，找到 **# Run match product function**。
1. 在下面并与注释**对齐**，粘贴以下代码：

    ```
           matched = match_products(profile.get("recommendedGear", []))
           print("\n🛒 Recommended Products from Lakeshore Retail:")
           print("\n".join(matched))
    ```

1. **保存在脚本中所做的更改**。
1. 在代码编辑器下方的 Cloud Shell 命令行窗格中，输入以下命令以**运行脚本**：

    ```
   python start-prompt.py
    ```

1. 提供要查找的徒步旅行类型的一些说明，例如：

    ```
   I want to go for a multi-day adventure along the beach
    ```

<br>
<details>
<summary><b>解决方案脚本</b>：如果代码不起作用。</summary><br>
<p>若检查 generate_trip_profile 函数的 LLM 调用日志，会发现助手返回的响应中包含反引号与 json 关键字，用于将输出格式化为代码块。

虽然这有助于显示，但它会导致代码中的问题，因为输出不再是有效的 JSON。 这会导致在进一步处理过程中出现分析错误。

此错误可能是由 LLM 指示遵守其输出的特定格式引起的。 将指令包含在在用户提示中似乎比将其置于系统提示中更有效。</p>
</details>


> **注意**：监控数据可能需要几分钟才能显示在 Azure Monitor 中。

### 在 Azure AI Foundry 门户中查看新跟踪

1. 导航回到 Azure AI Foundry 门户。
1. 应显示同名 **trail_guide_session** 的新跟踪。 如有必要，请刷新视图。
1. 选择新跟踪以打开更详细的视图。
1. 查看新的嵌套子范围 **trip_profile_generation** 和 **product_matching**。
1. 选择 **product_matching** 并查看显示的元数据。

    在 product_matching 函数中，包含了 **span.set_attribute("matched.count", len(matched))**。 通过使用键值对 **matched.count** 和匹配变量的长度来设置属性，可以将此信息添加到 **product_matching** 跟踪。 可以在元数据中的**属性**下找到此键值对。

## （可选）跟踪错误

如果有额外的时间，可以查看在出错时如何使用跟踪。 向你提供一个可能会引发错误的脚本。 运行并查看跟踪。

这是一个挑战性练习，因此说明刻意简化了。

1. 在 Cloud Shell 中，打开 **error-prompt.py** 脚本。 此脚本与 **start-prompt.py** 脚本位于同一目录中。 查看内容。
1. 运行 **error-prompt.py** 脚本。 出现提示时，在命令行中提供答案。
1. *希望*输出消息包含 **Failed to generate trip profile. Please check Application Insights for trace.**。
1. 导航到 **trip_profile_generation** 的跟踪，并检查错误的原因。

<br>
<details>
<summary><b>获取答案</b>：为什么会遇到错误...</summary><br>
<p>若检查 generate_trip_profile 函数的 LLM 调用日志，会发现助手返回的响应中包含反引号与 json 关键字，用于将输出格式化为代码块。

虽然这有助于显示，但它会导致代码中的问题，因为输出不再是有效的 JSON。 这会导致在进一步处理过程中出现分析错误。

此错误可能是由 LLM 指示遵守其输出的特定格式引起的。 将指令包含在在用户提示中似乎比将其置于系统提示中更有效。</p>
</details>

## 其他实验室的位置

你可以在 [Azure AI Foundry 学习门户](https://ai.azure.com)中探索更多实验室和练习，或参阅课程的**实验室部分**以了解其他可用活动。
