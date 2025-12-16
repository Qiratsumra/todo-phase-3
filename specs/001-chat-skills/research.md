# Research: Phase III - Chat + Skills

**Feature Branch**: `001-chat-skills`
**Created**: 2025-12-15
**Status**: Complete

---

## Research Summary

All technical decisions have been resolved based on user input and codebase analysis. No clarifications needed.

---

## Decision 1: LLM Provider & Integration

**Decision**: Use Google Gemini 2.0 Flash via OpenAI SDK compatibility

**Rationale**:
- Zero cost (2M tokens/day free tier)
- OpenAI SDK syntax familiarity for developers
- Function calling supported for MCP tool execution
- Easy migration path to paid OpenAI if needed later

**Alternatives Considered**:
| Alternative | Rejected Because |
|-------------|------------------|
| OpenAI GPT-4o | Paid API, not free |
| Anthropic Claude | Paid API, not free |
| Local LLM (Ollama) | Inconsistent function calling support |
| Native Gemini SDK | Less familiar syntax, harder to migrate |

**Implementation Details**:
```python
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("GEMINI_API_KEY"),
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)

# Use like standard OpenAI
response = client.chat.completions.create(
    model="gemini-2.0-flash-exp",
    messages=[...],
    tools=[...]  # Function calling
)
```

**Environment Variables**:
- `GEMINI_API_KEY` - API key from Google AI Studio
- `GEMINI_BASE_URL` - `https://generativelanguage.googleapis.com/v1beta/openai/`
- `GEMINI_MODEL` - `gemini-2.0-flash-exp`
- `GEMINI_TEMPERATURE` - `0.7` (default)
- `GEMINI_MAX_TOKENS` - `2048` (default)

---

## Decision 2: Multi-Agent Architecture Pattern

**Decision**: Main Agent + 4 Specialized Skill Agents with keyword-based routing

**Rationale**:
- Separation of concerns (each skill focuses on one domain)
- Expert-level responses per domain via optimized prompts
- Easy to add new skills without modifying core logic
- Fallback to TaskManagementSkill on ambiguity

**Alternatives Considered**:
| Alternative | Rejected Because |
|-------------|------------------|
| Single monolithic agent | Hard to maintain, poor responses for specialized queries |
| LLM-based routing | Additional API call overhead, slower |
| Rule-based decision tree | Brittle, hard to extend |

**Routing Logic**:
```python
SKILL_KEYWORDS = {
    "TaskSearchSkill": ["find", "search", "where", "filter", "show.*with", "containing"],
    "TaskAnalyticsSkill": ["how many", "completion rate", "statistics", "productivity", "trend"],
    "TaskRecommendationSkill": ["suggest", "recommend", "prioritize", "what should", "next"],
    "TaskManagementSkill": ["add", "create", "delete", "remove", "complete", "done", "update", "list", "show"]
}
```

---

## Decision 3: Database Schema for Conversations

**Decision**: Conversations and Messages tables with user isolation

**Rationale**:
- Stateless server design (no in-memory state)
- Conversation context persisted for continuity
- User isolation enforced at query level
- skill_used field enables analytics

**Schema Design**:

### Conversations Table
| Column | Type | Constraints |
|--------|------|-------------|
| id | INTEGER | PK, AUTO_INCREMENT |
| user_id | VARCHAR(255) | NOT NULL, INDEX |
| created_at | TIMESTAMP | DEFAULT NOW() |
| updated_at | TIMESTAMP | DEFAULT NOW(), ON UPDATE |

### Messages Table
| Column | Type | Constraints |
|--------|------|-------------|
| id | INTEGER | PK, AUTO_INCREMENT |
| conversation_id | INTEGER | FK → conversations.id, INDEX |
| user_id | VARCHAR(255) | NOT NULL, INDEX |
| role | VARCHAR(20) | NOT NULL (user/assistant) |
| content | TEXT | NOT NULL |
| tool_calls | JSON | NULLABLE |
| skill_used | VARCHAR(50) | NULLABLE, INDEX |
| created_at | TIMESTAMP | DEFAULT NOW(), INDEX |

**Indexes**:
- `idx_messages_conversation_id` on `conversation_id`
- `idx_messages_user_id` on `user_id`
- `idx_messages_created_at` on `created_at`
- `idx_messages_skill_used` on `skill_used`
- `idx_conversations_user_id` on `user_id`

---

## Decision 4: MCP Tools vs Direct Database Access

**Decision**: MCP Tools call existing REST API endpoints

**Rationale**:
- Reuse existing, tested API logic
- Consistent behavior between UI and chat
- No duplicate business logic
- Easier to maintain

**Alternatives Considered**:
| Alternative | Rejected Because |
|-------------|------------------|
| Direct database access | Duplicates validation logic, harder to maintain |
| New dedicated API | Unnecessary duplication |

**Tool-to-API Mapping**:
| MCP Tool | REST Endpoint | HTTP Method |
|----------|---------------|-------------|
| add_task | /api/tasks/ | POST |
| list_tasks | /api/tasks/ | GET |
| complete_task | /api/tasks/{id}/complete | POST |
| delete_task | /api/tasks/{id} | DELETE |
| update_task | /api/tasks/{id} | PATCH |
| search_tasks | /api/tasks/?keyword=X | GET (new param) |
| get_task_stats | /api/tasks/stats | GET (new endpoint) |

---

## Decision 5: Chat UI Implementation

**Decision**: Custom React components (no ChatKit)

**Rationale**:
- Zero external dependencies/cost
- Full customization control
- Better integration with existing Tailwind UI
- No domain allowlist configuration

**Components**:
1. `MessageList.tsx` - Displays chat history with user/assistant styling
2. `MessageInput.tsx` - Text input with send button
3. `TypingIndicator.tsx` - Animated loading state
4. `ChatInterface.tsx` - Main container orchestrating state

**Styling Approach**:
- User messages: Right-aligned, blue background
- Assistant messages: Left-aligned, gray background
- Skill badges: Colored pills showing which skill responded
- Responsive: Mobile-first with breakpoints

---

## Decision 6: Authentication Integration

**Decision**: Use existing Better Auth JWT tokens

**Rationale**:
- Already implemented in frontend
- Proven authentication flow
- No additional setup required

**Implementation**:
- Frontend: Include JWT in Authorization header
- Backend: Validate JWT, extract user_id
- All database queries scoped to user_id

---

## Decision 7: Error Handling Strategy

**Decision**: Graceful degradation with user-friendly messages

**Error Categories**:
| Error Type | User Message | Technical Action |
|------------|--------------|------------------|
| Gemini API timeout | "I'm thinking... please try again" | Retry up to 3 times |
| Gemini API error | "Something went wrong. Please try again" | Log error, return fallback |
| Task not found | "I couldn't find that task. Try listing your tasks first" | Return helpful suggestion |
| Database error | "Unable to save. Please try again" | Log error, rollback |
| Rate limit | "Too many requests. Please wait a moment" | Return 429 status |

---

## Decision 8: Conversation Context Window

**Decision**: Last 50 messages per conversation

**Rationale**:
- Balances context quality with token limits
- Sufficient for most conversations
- Prevents runaway costs on long conversations
- Can paginate for history viewing

---

## Technical Constraints Confirmed

1. **Gemini Rate Limits**: 60 requests/minute (free tier)
2. **Token Limits**: 2M tokens/day free
3. **Response Time Target**: <4 seconds for single-skill queries
4. **Concurrent Users**: 50+ supported via stateless design
5. **Message Limit**: 50 messages per context window
6. **No MCP SDK**: Custom tool implementation (simpler)

---

## Dependencies to Add

### Backend (Python)
```
openai>=1.0.0  # For Gemini via OpenAI SDK syntax
```

### Frontend (Node.js)
No new dependencies - using existing React, Tailwind, Better Auth

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Gemini function calling differs from OpenAI | Low | Medium | Test early, have fallback parsing |
| Rate limits hit during testing | Medium | Low | Add request queuing |
| Skill routing accuracy <95% | Low | Medium | Comprehensive test suite |
| Mobile chat UI issues | Medium | Medium | Progressive testing |

---

## Open Questions (Resolved)

All questions from the spec have been resolved through research:

1. **Q: How to integrate Gemini with OpenAI syntax?**
   A: Use `base_url` override in OpenAI client

2. **Q: How to route intents to skills?**
   A: Keyword-based matching with fallback

3. **Q: How to persist conversations statelessly?**
   A: Database tables with user isolation

4. **Q: How to implement MCP tools without SDK?**
   A: Custom function definitions for Gemini tool calling

---

**Research Status**: Complete
**Next Step**: Phase 1 - Data Model & Contracts
