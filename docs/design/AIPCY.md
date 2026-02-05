
# AI Personalized Chat System Design Document v3.1

**Project Code**: AIDAMI Chat Engine  
**Last Updated**: February 2026  
**Target Hardware**: Orin Nano (LLM Inference) + PC (Embedder + Web Services)  
**Primary Languages**: Chinese / Japanese / English (Multilingual memory retrieval and search supported)  
**Document Status**: Complete Version (Includes existing implementation + new full-text summary design)

---

## 1. System Objectives and Non-Functional Requirements

### Core Objectives

- Frontend chat response latency < 3 seconds (determined by external LLM)
- Memory learning completely asynchronous, non-blocking to user experience
- Support multilingual (Chinese/Japanese/English) **refined memories** (preferences, habits, rules, strong emotions, important traits, long-term plans) and **full-text summaries** (complete conversation-level unfiltered summaries) with automatic extraction and semantic search
- Users can proactively query their historical memories, conversation patterns, and **complete conversation-level summaries** using natural language
- Stable operation on consumer-grade PC with 16GB RAM (including background learning + summary generation)

### Non-Functional Requirements Status

| Requirement | Status | Notes |
|-------------|--------|-------|
| Real-time response + background learning separation | ✅ Achieved | Writing completely removed from chat path |
| Multilingual refined memory extraction (qwen2.5:1.5b) | ✅ Achieved | Prompts optimized |
| Vector database (Qdrant) semantic search | ✅ Achieved | bge-m3 + payload filtering |
| Full-text summary vectorization with independent collection | 🔄 Planned (v3.1 core addition) | conversation_summaries collection |
| Frontend semantic search modal with API integration | ✅ Achieved | Extensible to support summary search |
| Background learning worker (scans every 5 minutes) | ✅ Achieved | Configurable MIN_NEW_MESSAGES=1 |
| Memory importance grading (high/medium/low) | ✅ Achieved | Determined by qwen2.5 |
| System resource protection (automatic qwen unloading) | ✅ Achieved | Auto GC after 15 min idle |
| Memory deduplication and learning progress tracking | ✅ Achieved | mem0_learning_progress table |

---

## 2. Mem0 Overall Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          User Input                                     │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Message Manager - Real-time Response Path                  │
└─────────────┬──────────────────────────────────┬────────────────────────┘
              │                                   │
              ▼                                   ▼
┌──────────────────────────┐          ┌─────────────────────────┐
│  External LLM API        │          │   MySQL Message Storage │
│  (Gemini/DeepSeek/Grok)  │          └──────────┬──────────────┘
└──────────────────────────┘                     │
                                                 ▼
                              ┌──────────────────────────────────────┐
                              │  Background Learning Worker          │
                              │  (Scans every 5 minutes)             │
                              └─────┬────────────────────┬───────────┘
                                    │                    │
                 ┌──────────────────┴────────┐          │
                 ▼                            ▼          ▼
    ┌────────────────────────┐   ┌─────────────────────────────────┐
    │ qwen2.5:1.5b           │   │ qwen2.5:1.5b                    │
    │ Refined Memory         │   │ Full-text Summary Generation    │
    │ Extraction             │   │ (No filtering)                  │
    │ (Strict filtering)     │   └────────────┬────────────────────┘
    └──────┬─────────┬───────┘                │
           │         │                         ▼
           ▼         ▼              ┌─────────────────────────┐
    ┌──────────┐  ┌─────────────────────────┐ │ Qdrant:          │
    │ MySQL:   │  │ Qdrant:                 │ │ conversation_    │
    │ user_    │  │ personal_memories_2025  │ │ summaries        │
    │ profile  │  └─────────────────────────┘ └──────────────────┘
    └──────────┘                    ▲                    ▲
                                    │                    │
                              ┌─────┴────────────────────┴──────┐
                              │  Semantic Search Engine          │
                              └─────────────┬────────────────────┘
                                            │
                                            ▼
                              ┌──────────────────────────────────┐
                              │  Frontend Modal                  │
                              │  (Switch refined/summary search) │
                              └──────────────────────────────────┘
```

**Flow Description:**

1. **Frontend Real-time Path**: User input → Message Manager → External LLM API → Response
2. **Background Asynchronous Learning**: MySQL storage → Worker scans → qwen2.5 extracts memories & summaries → Store in Qdrant
3. **Active Query Path**: User search request → Semantic Search Engine → Query both collections → Return to frontend modal

---

## 3. Memory Extraction and Storage Process

### 3.1 Comparison of Two Memory Types

| Type | Purpose | Filtering Level | Prompt Strictness | Storage Location | Typical Quantity (per batch) | Trigger Frequency |
|------|---------|-----------------|-------------------|------------------|------------------------------|-------------------|
| Refined Memory | Long-term preferences, habits, rules, core characteristics | High (only high-value) | Extremely strict | personal_memories_2025 | 0-3 items/batch | Every 1-5 new messages |
| Full-text Summary | Complete conversation context, intermediate facts, discussion flow | No filtering | Completely open | conversation_summaries | 1 item/trigger | Every 30-50 messages or conversation end |

### 3.2 Refined Memory Extraction Prompt (Implemented)

```
You are a minimalist, efficient personalization memory extractor focused on user core characteristics.

Task: Extract from the conversation: **preferences, habits, rules, strong emotions, important traits, long-term plans**.

- Only extract explicitly stated or strongly implied content. Absolutely NO speculation.
- Ignore: objective facts, temporary events, greetings, small talk.
- Priority: strong preferences > habits/rules > emotions > traits > plans

Output strictly as JSON array, each memory:
{
  "content": "One concise sentence with 'user' as subject (supports Chinese, Japanese, English)",
  "category": "preference | habit | rule | emotion | trait | plan",
  "importance": "high | medium | low"
}

If no content worth long-term storage, output empty array [].

Conversation history (oldest to newest):
{conversation_history}
```

### 3.3 Full-text Summary Generation Prompt (v3.1 New)

```
你是一個中立的對話摘要專家。請對以下完整對話內容進行全面、無遺漏的總結。

要求：
- 保留所有重要事實、討論主題、關鍵結論、問題與解答
- 包含當時的背景、情境、具體例子、技術細節
- 不要過濾任何內容，即使是閒聊、重複、情緒表達，只要有資訊價值就保留
- 用自然流暢的中文（或日文，如果對話主要語言是日文）撰寫
- 總結長度控制在 300–600 字之間
- 結構建議：
  1. 開頭：對話主題與背景概述
  2. 中間：按時間或邏輯順序分段描述主要內容
  3. 結尾：列出主要結論、共識、待辦事項或未解問題

對話內容（從舊到新）：
{full_text}
```

### 3.4 Full-text Summary Trigger Timing (v3.1 Recommendations)

| Trigger Condition | Frequency | Implementation Location | Notes |
|-------------------|-----------|-------------------------|-------|
| Every 30-50 new messages accumulated | Higher | Counter in background worker | Suitable for active conversations |
| Conversation state changes to "close" | One-time | conversation_manager.update_state | Suitable for long conversation endings |
| Fixed time period (every 7 days) | Lower | Time check in worker | Prevents missing long unclosed conversations |

**Recommended Starting Point**: Use "every 30-50 messages" trigger - simple and high coverage.

### 3.5 Full-text Summary Qdrant Payload Structure (v3.1 New)

```json
{
  "type": "full_summary",
  "user_id": "YongLi38918908",
  "conversation_id": 12345,
  "summary_text": "本次對話從 Ollama 部署開始，討論了 qwen2.5 在 Orin Nano 上的記憶體佔用……（完整 400 字摘要）",
  "start_message_id": 1001,
  "end_message_id": 1050,
  "message_count": 50,
  "created_at": "2026-02-05T12:00:00Z",
  "period": "2026-02",
  "language": "zh",
  "importance": "medium"
}
```

### 3.6 Background Learning Worker Parameters (Adjustable)

| Parameter | Current Value | Recommended Range | Impact |
|-----------|---------------|-------------------|--------|
| LEARNING_INTERVAL | 300s | 180-600s | Learning frequency |
| MIN_NEW_MESSAGES | 1 | 1-5 | Trigger threshold |
| MAX_MESSAGES_PER_LEARN | 5 | 3-10 | Single batch size |
| MAX_TEXT_LENGTH | 5000 | 3000-8000 | Memory protection |
| INTER_USER_DELAY | 10s | 5-20s | Avoid DB overload |
| PROCESSING_DELAY | 5s | 3-10s | Batch interval |
| FULL_SUMMARY_TRIGGER_COUNT | 30 | 20-50 | Summary trigger threshold (v3.1 new) |

---

## 4. Frontend Semantic Search Usage (v3.1 Update)

**Location**: "Semantic Search" button in top-right of chat interface

### Current Support (Implemented)

- Natural language query (Chinese/Japanese/English)
- Filter conditions: category (preference/habit/rule/...), importance, date range, similarity threshold
- Shortcut: "Get 10 Memories" retrieves recent raw memories (no query input needed)

### v3.1 Recommended Additions (Frontend Modal Extension)

**Search Scope Toggle** (new UI element):
- ☐ Search refined memories only (current)
- ☐ Search full-text summaries only
- ☐ Search both (default)

**Result Display Differentiation**:
- Refined memories: Tag "Preference / Habit / Rule"
- Full-text summaries: Tag "Summary" + conversation_id / time range / message count
- Click result: Jump to corresponding conversation and scroll to related message (requires backend to return conversation_id)

---

## 5. Resource and Performance Impact Assessment (v3.1 Update)

| Item | Added Burden | Query Latency Impact | Memory Impact | Control Method |
|------|--------------|---------------------|---------------|----------------|
| Summary generation | One additional qwen run per trigger | None (background) | +0.5-1GB peak | Trigger frequency control |
| Summary vectorization & storage | 1 vector per summary | None | +Minimal | Independent collection |
| Search latency (two collections) | Query two collections | +20-50ms | None | Add user_id filter + ef_search=64 |
| Total vector growth rate | Original 1-3/batch → +1 summary | Long-term slowdown | Medium | Split by year collection |

**Conclusion**: In personal use scenarios (single user, tens to hundreds of thousands of vectors), the additional burden of full-text summaries is **controllable**, and query latency can be maintained under 100ms.

---

## 6. Next Version (v3.2) Recommended Priorities

| Priority | Item | Estimated Completion | Expected Value |
|----------|------|---------------------|----------------|
| 1 | Implement full-text summary generation and independent collection | 1-2 weeks | ★★★★★ |
| 2 | Frontend modal add "Search Scope" toggle (refined/summary) | 1 week | ★★★★☆ |
| 3 | Search result click-through to original conversation + message positioning | 2-3 weeks | ★★★★★ |
| 4 | Hybrid Search (semantic + keyword must-contain) | 4-6 weeks | ★★★★☆ |
| 5 | Memory citation count statistics and dynamic importance adjustment | 3-4 weeks | ★★★★☆ |

**v3.1 Primary Task**: Complete backend generation and storage of full-text summaries (recommended implementation: trigger every 30-50 messages).

---

## 7. Technical Stack Summary

| Component | Technology | Version/Details |
|-----------|-----------|-----------------|
| LLM Inference | Ollama + qwen2.5:1.5b | On Orin Nano |
| Embedding Model | bge-m3 | Multilingual support |
| Vector Database | Qdrant | Two collections: personal_memories_2025, conversation_summaries |
| Relational Database | MySQL | User profiles, messages, learning progress |
| Backend Framework | Python/FastAPI | Async workers |
| Frontend | React/Vue | Semantic search modal |

---

## 8. Data Flow Diagrams

### 8.1 Message Processing Flow

```
User Message → Message Manager → External LLM → Response to User
                    ↓
              MySQL Storage
                    ↓
        Background Worker (async)
                    ↓
              qwen2.5 Analysis
                ↓           ↓
        Refined Memory   Full Summary
                ↓           ↓
           Qdrant Store  Qdrant Store
```

### 8.2 Search Flow

```
User Search Query → Semantic Search Engine
                           ↓
              ┌────────────┴────────────┐
              ↓                         ↓
   personal_memories_2025    conversation_summaries
              ↓                         ↓
              └────────────┬────────────┘
                           ↓
                    Merge & Rank Results
                           ↓
                    Frontend Display
```

---

## Appendices

### A. Glossary

- **Refined Memory**: High-value, filtered long-term user characteristics
- **Full-text Summary**: Complete, unfiltered conversation-level summary
- **Semantic Search**: Vector similarity-based natural language search
- **Background Worker**: Asynchronous task processor for memory extraction

### B. Version History

- **v3.0**: Initial complete implementation with refined memory extraction
- **v3.1**: Added full-text summary design and independent collection architecture

### C. References

- Qdrant Documentation: https://qdrant.tech/documentation/
- Ollama Model Library: https://ollama.ai/library
- BGE-M3 Paper: https://arxiv.org/abs/2402.03216

---

**Document End**

This design document is comprehensive and includes all existing implementations and planned v3.1 features for the AIDAMI Chat Engine system.