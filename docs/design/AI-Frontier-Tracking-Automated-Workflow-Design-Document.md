
## **设计文档：AI驱动的“AI前沿追踪”自动化工作流系统**

### 1. 概述与愿景

**目标**: 构建一个全自动、自我进化的系统，利用你现有的AI工作流平台，持续追踪、分析、总结全球AI前沿动态，并生成高价值、个性化的情报和可执行任务。

**最终产物**: 系统每日自动生成一份《AI前沿日报》，每周生成一份《AI趋势分析与学习周报》，并能按需对任何技术主题进行深度研究。这套系统将成为你的“AI研究助理”，将你从繁杂的信息收集中解放出来。

**核心理念**: **用AI追踪AI (Using AI to track AI)**。这不仅是一个应用，更是对你平台能力的一次终极检验和驱动。

### 2. 核心设计原则

1.  **分层工作流架构 (Layered Architecture)**：将复杂的持续追踪任务分解为三个不同频次、不同深度的独立工作流，使系统结构清晰、易于管理和扩展。
    *   **L1 - 每日信息采集 (Daily Collector)**: 高频、自动化、宽泛地收集原始数据。**目标是“不错过”**。
    *   **L2 - 每周趋势分析 (Weekly Analyzer)**: 低频、深度AI分析，从一周的数据中提炼趋势、洞察和学习计划。**目标是“看懂”**。
    *   **L3 - 按需深度研究 (On-Demand Deep Dive)**: 手动触发，针对特定主题进行深入研究、资料整理和代码生成。**目标是“学会”**。

2.  **原子化与可复用的工具集 (Atomic & Reusable Tools)**：不依赖于庞大复杂的`curl`命令，而是为每个信息源在`sandbox`中创建专用的、健壮的Shell脚本工具。这使得Plan更简洁、更稳定、更易维护。

3.  **Plan即代码 (Plan as Code)**：将三个核心工作流的逻辑固化为三个独立的`Plan JSON`模板。通过平台的`plan_data_override`功能来调用这些预设好的“金牌工作流”，实现高效复用。

4.  **数据驱动的闭环 (Data-Driven Loop)**：L1工作流的产出是L2工作流的输入。L2的产出（学习计划）可以触发L3工作流。这形成了一个从数据采集到深度学习的完整闭环。

### 3. 系统架构与实现

#### **Phase 1: 基础设施与工具集**

此阶段的目标是为`sandbox`配备一套强大的、专用的信息采集工具。这些工具是所有上层工作流的基石。

**建议的工具列表 (Shell脚本, 位于`sandbox`的工具目录中):**

1.  **`fetch_arxiv_daily.sh`**:
    *   **功能**: 获取Arxiv `cs.AI`, `cs.LG`, `cs.CL`分类下过去24小时内发布的最新论文。
    *   **输出**: 返回一个JSON数组，包含`title`, `summary`, `pdf_url`, `published_date`等字段。脚本内部处理XML解析和日期过滤。

2.  **`fetch_github_trending_ai.sh`**:
    *   **功能**: 获取GitHub Trending日榜，并筛选出仓库名或描述中包含AI相关关键词（如ai, llm, agent, diffusion）的项目。
    *   **输出**: 返回一个JSON数组，包含`repo_name`, `description`, `stars_today`, `language`等字段。

3.  **`fetch_huggingface_trending.sh`**:
    *   **功能**: 调用Hugging Face API，获取趋势模型（Trending Models）列表。
    *   **输出**: 返回一个JSON数组，包含`model_id`, `author`, `downloads`, `tags`等字段。

4.  **`fetch_hackernews_ai.sh`**:
    *   **功能**: 调用Hacker News API或通过爬虫，获取首页包含AI/ML等关键词的热门帖子。
    *   **输出**: 返回一个JSON数组，包含`title`, `url`, `points`, `comments_count`。

*(可根据需要添加更多工具，如`fetch_tech_blog.sh`等)*

#### **Phase 2: 核心工作流Plan模板**

以下是三个核心工作流的`Plan JSON`模板，可保存为文件，通过`plan_data_override`调用。

---

**1. L1 - 每日信息采集工作流 (`daily_collector_plan.json`)**

*   **目标**: 并行抓取所有信源的最新数据，聚合后存为带日期的原始JSON文件。
*   **触发方式**: 定时任务 (Cron Job / Temporal Schedule)，每日凌晨执行。

```json
{
  "steps": [
    {
      "type": "action", "step_id": "collect_arxiv", "action": "fetch_arxiv_daily", "inputs": {}
    },
    {
      "type": "action", "step_id": "collect_github", "action": "fetch_github_trending_ai", "inputs": {}
    },
    {
      "type": "action", "step_id": "collect_hf", "action": "fetch_huggingface_trending", "inputs": {}
    },
    {
      "type": "action", "step_id": "collect_hn", "action": "fetch_hackernews_ai", "inputs": {}
    },
    {
      "type": "action",
      "step_id": "aggregate_daily_data",
      "action": "generate_structured_data",
      "inputs": {
        "prompt": "You are a data aggregation AI. Combine the following data sources into a single, clean JSON object. Add a 'source' field to each item. Do not summarize or alter the content. \n\nArxiv:\n{{json.dumps(collect_arxiv.output.response)}}\n\nGitHub:\n{{json.dumps(collect_github.output.response)}}\n\nHuggingFace:\n{{json.dumps(collect_hf.output.response)}}\n\nHackerNews:\n{{json.dumps(collect_hn.output.response)}}\n\nRespond with ONLY a valid JSON object of the format `{\"arxiv\": [...], \"github\": [...], \"huggingface\": [...], \"hackernews\": [...]}`."
      }
    },
    {
      "type": "action",
      "step_id": "save_daily_raw_data",
      "action": "write_files",
      "inputs": {
        "environment_id": "{{environment_id}}",
        "files": [{
          "filepath": "/data/raw/{{current_date('YYYY-MM-DD')}}.json",
          "content": "{{aggregate_daily_data.output.response}}"
        }]
      },
      "return_as_final_answer": true
    }
  ]
}
```

---

**2. L2 - 每周趋势分析工作流 (`weekly_analyzer_plan.json`)**

*   **目标**: 读取过去7天的原始数据，进行深度AI分析，生成周报和学习计划。
*   **触发方式**: 定时任务，每周一早上执行。

```json
{
  "steps": [
    {
      "type": "action",
      "step_id": "read_weekly_data",
      "action": "execute_command",
      "inputs": {
        "environment_id": "{{environment_id}}",
        "command": "find /data/raw -name '*.json' -mtime -7 | xargs cat | jq -s 'add'"
      }
    },
    {
      "type": "action",
      "step_id": "analyze_trends_and_rank",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "You are a top-tier AI research analyst. Analyze the past week's AI data below. Identify:\n1. The 3 most significant overarching trends (e.g., 'Emergence of Small Language Models for Edge AI').\n2. The 5 most important individual items (papers, repos, models) and assign a 'Significance Score' from 1-10.\n3. A brief, sharp analysis for each trend and top item.\n\nData:\n{{read_weekly_data.output.stdout}}\n\nOutput in clear, structured Markdown."
      }
    },
    {
      "type": "action",
      "step_id": "generate_learning_plan",
      "action": "generate_structured_data",
      "inputs": {
        "prompt": "Based on the following trend analysis, create a practical, actionable learning plan for an AI developer for the upcoming week. For each trend, suggest one concrete action item (e.g., 'Read paper X', 'Run Colab notebook for repo Y').\n\nTrend Analysis:\n{{analyze_trends_and_rank.output.response}}\n\nRespond with ONLY a valid JSON object of the format `{\"priority_topics\": [{\"topic\": \"\", \"reason\": \"\"}], \"action_items\": [{\"task\": \"\", \"resource_link\": \"\"}]}`."
      }
    },
    {
      "type": "action",
      "step_id": "generate_weekly_report",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "Combine the trend analysis and learning plan into a single, polished weekly report in Markdown format. The title should be 'AI Frontier Weekly Report - Week {{current_date('W')}}, {{current_date('YYYY')}}'.\n\nAnalysis Content:\n{{analyze_trends_and_rank.output.response}}\n\nLearning Plan JSON:\n{{json.dumps(generate_learning_plan.output.response)}}"
      }
    },
    {
      "type": "action",
      "step_id": "save_and_notify_report",
      "action": "write_files",
      "inputs": {
        "environment_id": "{{environment_id}}",
        "files": [{
            "filepath": "/reports/weekly_report_{{current_date('YYYY')}}_W{{current_date('W')}}.md",
            "content": "{{generate_weekly_report.output.response}}"
        }]
      },
      "return_as_final_answer": true
    }
  ]
}
```

---

**3. L3 - 按需深度研究工作流 (`deep_dive_plan.json`)**

*   **目标**: 接收一个主题作为输入，围绕该主题收集资料、阅读论文、生成学习笔记，甚至编写示例代码。
*   **触发方式**: 手动API调用，传入`task`参数，如 `"task": "Deep dive on 'Retrieval-Augmented Generation'"`。

```json
{
  "steps": [
    {
      "type": "action",
      "step_id": "search_papers_for_topic",
      "action": "execute_command",
      "inputs": {
        "environment_id": "{{environment_id}}",
        "command": "YOUR_ARXIV_SEARCH_SCRIPT '{{task_topic}}'" 
      }
    },
    {
      "type": "action",
      "step_id": "summarize_key_papers",
      "action": "process_chunks_with_ai",
      "inputs": {
        "chunks": "{{search_papers_for_topic.output.response.papers}}",
        "prompt_template": "Read the summary of the paper titled '{{chunk.title}}'. Explain its core contribution, methodology, and key results in 3 bullet points. Paper Summary: {{chunk.summary}}",
        "output_format": "text"
      }
    },
    {
      "type": "action",
      "step_id": "generate_starter_code",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "Based on the key concepts from the following paper summaries, generate a minimal, runnable Python code example demonstrating the core idea of '{{task_topic}}'. Include necessary library imports and comments.\n\nSummaries:\n{{json.dumps(summarize_key_papers.output.processed_chunks)}}"
      }
    },
    {
      "type": "action",
      "step_id": "create_study_guide",
      "action": "write_files",
      "inputs": {
        "environment_id": "{{environment_id}}",
        "files": [{
          "filepath": "/study_guides/{{task_topic_slug}}.md",
          "content": "# Study Guide: {{task_topic}}\n\n## Key Papers & Summaries\n{{...}}\n\n## Core Concepts\n{{...}}\n\n## Starter Code Example\n```python\n{{generate_starter_code.output.response}}\n```"
        }]
      },
      "return_as_final_answer": true
    }
  ]
}

```

### 4. 实施路线图 (Roadmap)

1.  **第一周: 奠定基础 (Foundation)**
    *   [ ] 在`sandbox`中开发并测试至少3个核心采集工具 (`fetch_arxiv_daily.sh`, `fetch_github_trending_ai.sh`, `fetch_huggingface_trending.sh`)。
    *   [ ] 手动执行`daily_collector_plan.json`，验证数据能否被正确采集和保存。
    *   [ ] 设计数据存储目录结构 (`/data/raw`, `/reports`, `/study_guides`)。

2.  **第二周: 实现自动化与分析 (Automation & Analysis)**
    *   [ ] 配置定时任务，实现`daily_collector_plan`的每日自动执行。
    *   [ ] 完善`weekly_analyzer_plan.json`，并手动执行，调试AI分析的Prompt，直到产出满意的周报。
    *   [ ] 将`weekly_analyzer_plan`也加入定时任务。

3.  **第三周: 释放深度学习能力 (Deep Dive & Integration)**
    *   [ ] 完善`deep_dive_plan.json`，使其能够根据用户输入的主题，完成研究和笔记生成。
    *   [ ] （可选）集成通知功能，例如在周报生成后，通过`execute_command`调用`curl`将报告内容推送到Webhook（企业微信、Slack、Discord等）。

4.  **第四周及以后: 迭代与进化 (Evolution)**
    *   [ ] **反馈循环**: 增加一个机制，让你能对周报中的推荐内容进行评分，并将评分反馈给下一周的分析模型，以实现个性化推荐。
    *   **知识图谱**: 运行一个额外的AI步骤，从每周的实体（论文、项目、技术）中提取关系，并构建一个简单的知识图谱。
    *   **自动实验**: 探索让L3工作流不仅生成代码，还能在`sandbox`环境中自动执行代码，并将结果附加到学习笔记中。

### 5. 预期成果

通过实施此方案，你的AI工作流平台将转变为一个强大的、自主的AI研究系统，为你带来：

*   ✅ **每日10分钟**，即可掌握AI领域的关键动态，告别信息焦虑。
*   ✅ **每周1小时**，获得一份包含深刻洞察和定制化学习路径的深度报告。
*   ✅ **按需启动**，即可在数分钟内获得任何一个AI新主题的完整学习入门包。
*   ✅ **真正实现**“让AI为你工作”，将你的时间和精力聚焦于最高价值的创造性活动上。