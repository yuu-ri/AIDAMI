```
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
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "chmod +x sandbox_service/app/tools/*.sh",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "make_tools_executable",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sandbox_service/app/tools/arxiv_daily.sh",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "collect_arxiv_papers",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sandbox_service/app/tools/fetch_github_trending_ai.sh",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "collect_github_trending",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sandbox_service/app/tools/fetch_huggingface_trending.sh",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "collect_huggingface_models",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sandbox_service/app/tools/fetch_hackernews_ai.sh",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "collect_hackernews_stories",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "date +%Y-%m-%d | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "get_current_date_for_filename",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "cat <<'EOF' | python3 -m json.tool\n{\n  \"arxiv\": {{collect_arxiv_papers.output.stdout}},\n  \"github\": {{collect_github_trending.output.stdout}},\n  \"huggingface\": {{collect_huggingface_models.output.stdout}},\n  \"hackernews\": {{collect_hackernews_stories.output.stdout}}\n}\nEOF",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "format_json_content",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{format_json_content.output.stdout}}",
            "filepath": "data/daily/{{get_current_date_for_filename.output.stdout}}.json"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_aggregated_data_to_file",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sudo cp -r data /host/",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "copy_to_host",
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
      "action": "execute_command",
      "inputs": {
        "command": "find /host/data/daily -name '*.json' -mtime -7 -type f | sort",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "list_weekly_files",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "jq -s '{\n  arxiv:      (map(.arxiv      // []) | add // []),\n  github:     (map(.github     // []) | add // []),\n  huggingface:(map(.huggingface// []) | add // []),\n  hackernews: (map(.hackernews // []) | add // [])\n}' /host/data/daily/*.json",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "aggregate_weekly_data",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "You are a top-tier AI research analyst with deep expertise in machine learning, NLP, computer vision, and emerging AI technologies.\n\nAnalyze the following week's AI data and provide:\n\n1. **Top 3 Overarching Trends**: Identify the most significant macro-trends (e.g., 'Emergence of Small Language Models for Edge AI', 'Multimodal AI Breakthroughs', 'Open Source vs Closed Models Debate'). For each trend, explain WHY it matters and what it signals about the future.\n\n2. **Top 5 Most Important Items**: Rank the 5 most significant individual items (papers, repositories, models, or news) with a Significance Score (1-10). Consider factors like:\n   - Technical innovation and novelty\n   - Potential real-world impact\n   - Community interest and adoption\n   - Paradigm-shifting implications\n\n3. **Sharp Analysis**: For each trend and top item, provide concise but insightful commentary. Avoid generic statements.\n\n**Weekly Data:**\n```json\n{{aggregate_weekly_data.output.stdout}}\n```\n\nOutput your analysis in clear, well-structured Markdown format with proper headings and bullet points."
      },
      "step_id": "analyze_trends_and_rank",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_structured_data",
      "inputs": {
        "prompt": "Based on the following AI trend analysis, create a practical, actionable learning plan for an AI developer/researcher for the upcoming week.\n\nFor each major trend or important development:\n- Suggest ONE concrete action item (e.g., 'Read paper X and implement key algorithm', 'Explore repo Y and run examples', 'Study model Z architecture')\n- Provide specific resource links when available\n- Prioritize items by learning value and time investment\n\n**Trend Analysis:**\n{{analyze_trends_and_rank.output.response}}\n\nRespond with ONLY a valid JSON object in this exact format:\n```json\n{\n  \"priority_topics\": [\n    {\n      \"topic\": \"Topic name\",\n      \"reason\": \"Why this topic is important to learn now\",\n      \"difficulty\": \"beginner|intermediate|advanced\"\n    }\n  ],\n  \"action_items\": [\n    {\n      \"task\": \"Specific action to take\",\n      \"resource_link\": \"URL to paper/repo/model\",\n      \"estimated_time\": \"Time estimate (e.g., '2 hours', '1 day')\",\n      \"priority\": \"high|medium|low\"\n    }\n  ]\n}\n```"
      },
      "step_id": "generate_learning_plan",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "date +'%Y-W%V' | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "get_week_identifier",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "Create a polished, professional weekly report combining the trend analysis and learning plan below. Use this structure:\n\n# \ud83d\ude80 AI Frontier Weekly Report\n## Week {{get_week_identifier.output.stdout}}\n\n### \ud83d\udcca Executive Summary\n[2-3 sentence overview of the week's most important developments]\n\n### \ud83d\udd25 Top Trends\n[Insert trend analysis here with proper formatting]\n\n### \u2b50 Spotlight: Top 5 Developments\n[Insert ranked items with scores and analysis]\n\n### \ud83d\udcda Learning Plan for Next Week\n#### Priority Topics\n[Format the priority_topics from JSON as a clear list]\n\n#### Action Items\n[Format the action_items from JSON as a checklist with links]\n\n### \ud83d\udcc8 Statistics\n- Total Papers Tracked: [count from data]\n- Total GitHub Repos: [count from data]\n- Total Models: [count from data]\n- Total HN Stories: [count from data]\n\n---\n*Generated on {{get_week_identifier.output.stdout}} | Powered by AI Workflow Platform*\n\n**Trend Analysis:**\n{{analyze_trends_and_rank.output.response}}\n\n**Learning Plan (JSON):**\n```json\n{{generate_learning_plan.output.response}}\n```\n\n**Raw Data Summary:**\n```json\n{{aggregate_weekly_data.output.stdout}}\n```"
      },
      "step_id": "generate_weekly_report",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "echo '{{generate_learning_plan.output.response}}' | python3 -c 'import sys, json; data = json.load(sys.stdin); print(json.dumps(data, indent=2, ensure_ascii=False))'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "serialize_learning_plan",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{generate_weekly_report.output.response}}",
            "filepath": "data/weekly/report_{{get_week_identifier.output.stdout}}.md"
          },
          {
            "content": "{{serialize_learning_plan.output.stdout}}",
            "filepath": "data/weekly/learning_plan_{{get_week_identifier.output.stdout}}.json"
          },
          {
            "content": "{{aggregate_weekly_data.output.stdout}}",
            "filepath": "data/weekly/aggregated_data_{{get_week_identifier.output.stdout}}.json"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_weekly_outputs",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sudo cp -r data /host/",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "copy_to_host",
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
  "name": "AI Topic Deep Dive Research",
  "steps": [
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "echo '{{task}}' | python3 -c \"import sys, re, json; task=sys.stdin.read().strip(); match=re.search(r'[\\'\\\"](.+?)[\\'\\\"]', task); topic=match.group(1) if match else task.replace('Deep dive on ', '').replace('deep dive on ', ''); print(json.dumps({'topic': topic, 'slug': re.sub(r'[^a-z0-9]+', '_', topic.lower()).strip('_')}, ensure_ascii=False), end='')\" | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "validate_and_extract_topic",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "mkdir -p data/study_guides data/deep_dive",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "create_directories",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "TOPIC=$(echo '{{validate_and_extract_topic.output.stdout}}' | python3 -c 'import sys, json; print(json.loads(sys.stdin.read())[\"topic\"], end=\"\")'); bash sandbox_service/app/tools/search_arxiv_topic.sh \"$TOPIC\" 10",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "search_arxiv_papers",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "TOPIC=$(echo '{{validate_and_extract_topic.output.stdout}}' | python3 -c 'import sys, json; print(json.loads(sys.stdin.read())[\"topic\"], end=\"\")'); bash sandbox_service/app/tools/search_github_repos.sh \"$TOPIC\" 5",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "search_github_repos",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "You are an expert AI researcher. Analyze the following research papers on the topic from the research data below.\n\n**Topic Extraction Output:**\n{{validate_and_extract_topic.output.stdout}}\n\n**Papers Data:**\n```json\n{{search_arxiv_papers.output.stdout}}\n```\n\nFirst, extract the topic name from the Topic Extraction Output JSON, then provide:\n1. **Core Concepts Summary** (3-5 key ideas that define this topic)\n2. **Top 3 Must-Read Papers** (with title, why it's important, and key takeaway)\n3. **Technical Challenges** (what are the hard problems in this area)\n4. **Current State-of-the-Art** (what approaches are winning)\n\nFormat in clear Markdown with sections."
      },
      "step_id": "analyze_papers_with_ai",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "You are a senior AI engineer. The topic extraction output is:\n\n{{validate_and_extract_topic.output.stdout}}\n\nAnalyze these GitHub repositories:\n\n```json\n{{search_github_repos.output.stdout}}\n```\n\nFirst extract the topic from the JSON above, then provide:\n1. **Ecosystem Overview** (what kinds of tools/libraries exist)\n2. **Top 2 Recommended Repos** (which ones to explore first and why)\n3. **Implementation Patterns** (common approaches you see in the codebases)\n\nBe concise and practical."
      },
      "step_id": "analyze_github_ecosystem",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_structured_data",
      "inputs": {
        "prompt": "Based on the research above, create a structured learning roadmap.\n\n**Topic Info:**\n{{validate_and_extract_topic.output.stdout}}\n\n**Paper Analysis:**\n{{analyze_papers_with_ai.output.response}}\n\n**GitHub Ecosystem:**\n{{analyze_github_ecosystem.output.response}}\n\nFirst extract the topic name from the Topic Info JSON, then respond with ONLY valid JSON in this format:\n```json\n{\n  \"topic\": \"extracted topic name\",\n  \"prerequisites\": [\"concept1\", \"concept2\"],\n  \"learning_path\": [\n    {\n      \"phase\": \"1. Foundations\",\n      \"goals\": [\"goal1\", \"goal2\"],\n      \"resources\": [{\"type\": \"paper|repo|tutorial\", \"title\": \"...\", \"url\": \"...\"}],\n      \"estimated_time\": \"X hours/days\"\n    }\n  ],\n  \"hands_on_projects\": [\n    {\"title\": \"project name\", \"description\": \"what to build\", \"difficulty\": \"beginner|intermediate|advanced\"}\n  ]\n}\n```"
      },
      "step_id": "generate_learning_roadmap",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "You are a Python expert. Based on the following research, write a complete, runnable Python code example that demonstrates the CORE concept.\n\n**Topic Info:**\n{{validate_and_extract_topic.output.stdout}}\n\nRequirements:\n- Include all necessary imports\n- Use only common libraries (transformers, torch, numpy, etc.)\n- Add detailed comments explaining each step\n- Keep it under 100 lines but fully functional\n- Include a simple usage example at the end\n\n**Research Context:**\n{{analyze_papers_with_ai.output.response}}\n\n**Available Libraries (from GitHub):**\n{{analyze_github_ecosystem.output.response}}\n\nFirst extract the topic from the Topic Info JSON, then provide ONLY the Python code, no explanations before or after."
      },
      "step_id": "generate_starter_code",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "echo '{{validate_and_extract_topic.output.stdout}}' | python3 -c 'import sys, json; data=json.loads(sys.stdin.read()); print(data[\"topic\"], end=\"\")' | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "extract_topic_name",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "echo '{{validate_and_extract_topic.output.stdout}}' | python3 -c 'import sys, json; data=json.loads(sys.stdin.read()); print(data[\"slug\"], end=\"\")' | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "extract_slug",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "Compile everything into a polished study guide.\n\n**Topic:** {{extract_topic_name.output.stdout}}\n\nUse this structure:\n\n```markdown\n# 📚 Deep Dive Study Guide: {{extract_topic_name.output.stdout}}\n\n> Generated on $(date +%Y-%m-%d) | Research Depth: Comprehensive\n\n## 🎯 What You'll Learn\n[2-3 sentence overview of what this topic is and why it matters]\n\n## 🧠 Core Concepts\n[Insert paper analysis core concepts section]\n\n## 📖 Essential Reading\n[Insert top papers from analysis]\n\n## 💻 GitHub Ecosystem\n[Insert GitHub analysis]\n\n## 🗺️ Learning Roadmap\n[Format the JSON roadmap as readable sections with checkboxes]\n\n## 🔬 Hands-On: Starter Code\n[Insert the generated Python code in a code block]\n\n### How to Run\n```bash\npip install [required libraries]\npython starter_code.py\n```\n\n## 🚀 Next Steps\n- [ ] Read the top 3 papers\n- [ ] Clone and explore recommended repos\n- [ ] Modify the starter code for your use case\n- [ ] Build one of the hands-on projects\n\n## 📚 Additional Resources\n- Official documentation links\n- Community forums\n- Related topics to explore next\n\n---\n*This guide was generated by AI Workflow Platform's Deep Dive system*\n```\n\n**Data to integrate:**\n\nPaper Analysis:\n{{analyze_papers_with_ai.output.response}}\n\nGitHub Analysis:\n{{analyze_github_ecosystem.output.response}}\n\nLearning Roadmap JSON:\n```json\n{{generate_learning_roadmap.output.response}}\n```\n\nStarter Code:\n```python\n{{generate_starter_code.output.response}}\n```\n\nProvide the complete, formatted study guide."
      },
      "step_id": "create_comprehensive_study_guide",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{create_comprehensive_study_guide.output.response}}",
            "filepath": "data/study_guides/{{extract_slug.output.stdout}}.md"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_study_guide",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{generate_starter_code.output.response}}",
            "filepath": "data/study_guides/{{extract_slug.output.stdout}}_code.py"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_starter_code",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "Convert the following data to properly formatted JSON string. If it's already JSON, return it as-is. If it's a Python dict/object representation, convert it to valid JSON.\n\nData:\n{{generate_learning_roadmap.output.response}}\n\nReturn ONLY the JSON string, no explanations."
      },
      "step_id": "format_roadmap_json",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{format_roadmap_json.output.response}}",
            "filepath": "data/deep_dive/{{extract_slug.output.stdout}}_roadmap.json"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_roadmap_json",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{search_arxiv_papers.output.stdout}}",
            "filepath": "data/deep_dive/{{extract_slug.output.stdout}}_papers.json"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_papers_json",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "write_files",
      "inputs": {
        "files": [
          {
            "content": "{{search_github_repos.output.stdout}}",
            "filepath": "data/deep_dive/{{extract_slug.output.stdout}}_repos.json"
          }
        ],
        "environment_id": "{{environment_id}}"
      },
      "step_id": "save_repos_json",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "sudo cp -r data /host/",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "copy_to_host",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "echo '{{search_arxiv_papers.output.stdout}}' | python3 -c 'import sys, json; data=json.loads(sys.stdin.read()); print(len(data.get(\"papers\", [])), end=\"\")' | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "count_papers",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "execute_command",
      "inputs": {
        "command": "echo '{{search_github_repos.output.stdout}}' | python3 -c 'import sys, json; data=json.loads(sys.stdin.read()); print(len(data) if isinstance(data, list) else 0, end=\"\")' | tr -d '\\n'",
        "environment_id": "{{environment_id}}"
      },
      "step_id": "count_repos",
      "return_as_final_answer": false
    },
    {
      "type": "action",
      "action": "generate_text_response",
      "inputs": {
        "prompt": "Create a concise executive summary of the deep dive research completed.\n\n**Topic:** {{extract_topic_name.output.stdout}}\n\n**Outputs Generated:**\n- Study Guide: data/study_guides/{{extract_slug.output.stdout}}.md\n- Starter Code: data/study_guides/{{extract_slug.output.stdout}}_code.py\n- Learning Roadmap: data/deep_dive/{{extract_slug.output.stdout}}_roadmap.json\n\n**Research Stats:**\n- Papers Analyzed: {{count_papers.output.stdout}}\n- Repos Reviewed: {{count_repos.output.stdout}}\n\nProvide:\n1. 🎯 **Key Insight** (one sentence: what's the most important thing to know)\n2. ⚡ **Quick Start** (what to do first to begin learning)\n3. 📊 **Complexity Rating** (beginner/intermediate/advanced and why)\n4. 🔗 **All Generated Files** (list with descriptions)\n\nKeep it under 200 words, actionable and encouraging."
      },
      "step_id": "generate_summary_report",
      "return_as_final_answer": true
    }
  ],
  "description": "Comprehensive research workflow for any AI topic: search papers, analyze repos, generate study guide with code examples"
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
```