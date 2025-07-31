---
lab:
  title: 监视生成式 AI 应用程序
  description: 了解如何监视与已部署模型的交互，并深入了解如何使用生成式 AI 应用程序优化其使用情况。
---

# 监视生成式 AI 应用程序

此练习大约需要 **30** 分钟。

> **注意**：本练习假设你对 Azure AI Foundry 有一定的了解，因此某些说明故意不那么详细，以鼓励更积极的探索和实践学习。

## 简介

在本练习中，你将为聊天完成应用启用监控，并在 Azure Monitor 中查看其性能。 与已部署的模型交互以生成数据，借助 Insights for Generate AI 应用程序仪表板查看生成的数据，并设置警报以帮助优化模型的部署。

## 设置环境

要完成本练习中的任务，你需要：

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

将 Application Insights 连接到你在 Azure AI Foundry 中的项目，开始收集数据进行监视。

1. 使用左侧的菜单，然后选择“**跟踪**”页。
1. **创建新的** Application Insights 资源以连接到应用。
1. 输入 Application Insights 资源名称，然后选择“创建”****。

Application Insights 现已连接到项目，数据将开始收集以供分析。

## 与已部署的模型交互

你将使用 Azure Cloud Shell 设置与 Azure AI Foundry 项目的连接，以编程方式与已部署的模型交互。 这样，你就可以向模型发送提示，并生成用于监控的数据。

### 通过 Cloud Shell 连接模型

首先，检索与模型交互所需的身份验证信息。 然后，你将访问 Azure Cloud Shell 并更新配置，将提供的提示发送到你已部署的模型。

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

    > **提示**：将命令粘贴到 Cloudshell 中时，输出可能会占用大量屏幕缓冲区。 可以通过输入 `cls` 命令来清除屏幕，以便更轻松地专注于每项任务。

1. 克隆存储库后，导航到包含应用程序代码文件的文件夹：  

    ```
   cd mslearn-genaiops/Files/07
    ```

1. 在 Cloud Shell 命令行窗格中，输入以下命令以安装需要的库：

    ```
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv openai azure-identity azure-ai-projects azure-ai-inference azure-monitor-opentelemetry
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

### 将提示发送到已部署的模型

现在，你将运行多个脚本，向已部署的模型发送不同的提示。 这些交互将生成数据，你可以稍后在 Azure Monitor 中查看这些数据。

1. 运行以下命令以**查看第一个提供的脚本**：

    ```
   code start-prompt.py
    ```

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

    该模型将生成一个响应，并使用 Application Insights 捕获，以便进一步分析。 让我们改变提示，以探索其效果。

1. 打开并审阅脚本，该脚本中的提示指示模型仅使用一个句子和一个列表作答：********

    ```
   code short-prompt.py
    ```

1. 在命令行中输入以下命令以**运行脚本**：

    ```
   python short-prompt.py
    ```

1. 下一个脚本具有类似的目标，但输出的说明包含在**系统消息**中，而不是用户消息：

    ```
   code system-prompt.py
    ```

1. 在命令行中输入以下命令以**运行脚本**：

    ```
   python system-prompt.py
    ```

1. 最后，让我们尝试运行一个包含**过多令牌**的提示，以触发错误：

    ```
   code error-prompt.py
    ```

1. 在命令行中输入以下命令以**运行脚本**。 请注意，你 **很可能会遇到错误！**

    ```
   python error-prompt.py
    ```

现在，你已经与模型交互，可以查看 Azure Monitor 中的数据。

> **注意**：监控数据可能需要几分钟才能显示在 Azure Monitor 中。

## 在 Azure Monitor 中查看监视数据

要查看从模型交互收集的数据，你将访问链接到 Azure Monitor 中工作簿的仪表板。

### 从 Azure AI Foundry 门户导航到 Azure Monitor

1. 在浏览器中导航至已打开 **Azure AI Foundry 门户** 的标签页。
1. 使用左侧的菜单，选择“**跟踪**”。
1. 选择顶部的链接，显示 **“查看有关生成式 AI 应用程序的见解”仪表板**。 该链接将在新选项卡中打开 Azure Monitor。
1. 审阅**概述**，提供与已部署模型交互的汇总数据。

## 解读 Azure Monitor 中的监视指标

现在是时候深入分析数据，并开始解读它所传达的信息了。

### 审阅令牌使用情况

首先查看**令牌使用情况**部分，并审阅以下指标：

- **提示令牌**：所有模型调用中，你发送的提示所使用的令牌总数。

> 这相当于向模型*提问时产生的成本*。

- **完成令牌**：模型输出中返回的令牌数量，实质上是响应的长度。

> 生成的完成令牌通常占据主要的令牌使用量和成本，尤其是在生成较长或内容详细的回答时。

- **令牌合计数量**：提示令牌与完成令牌的合计数量。

> 这是衡量计费与性能的重要指标，直接影响延迟与成本。

- **调用总数**：单独的推理请求数，即模型被调用的次数。

> 有助于分析吞吐量，并了解每次调用的平均成本。

### 比较各个提示

向下滚动查找 **Gen AI Spans**，它以表格形式显示，每个提示都作为一行新数据呈现。 审阅并比较以下列的内容：

- **状态**：模型调用是成功还是失败。

> 使用此方法识别有问题的提示或配置错误。 最后一个提示可能失败，因为提示过长。

- **持续时间**：显示模型响应所花费的时间（以毫秒为单位）。

> 比较各行，以探索哪些提示模式会导致处理时间更长。

- **输入**：显示发送到模型的用户消息。

> 使用此列评估哪些提示构造高效或存在问题。

- **系统**：显示提示中使用的系统消息（如有）。

> 比较条目，以评估使用或更改系统消息的影响。

- **输出**：包含模型的响应。

> 使用它来评估详细程度、相关性和一致性。 特别是在令牌数和持续时间方面。

## （可选）创建警报

如果有额外时间，尝试设置警报，以在模型延迟超过特定阈值时通知你。 这是一个挑战性练习，因此说明刻意简化了。

- 在 Azure Monitor 中为 Azure AI Foundry 项目和模型创建**新预警规则**。
- 选择**请求持续时间 (ms)** 等指标，并定义阈值（例如超过 4000 毫秒）。
- 创建新的**操作组**，以定义通知方式。

设置警报有助于提前监控，做好上线前的准备。 配置的警报，取决于项目的优先级，以及团队决定如何衡量和缓解风险。

## 其他实验室的位置

你可以在 [Azure AI Foundry 学习门户](https://ai.azure.com)中探索更多实验室和练习，或参阅课程的**实验室部分**以了解其他可用活动。
