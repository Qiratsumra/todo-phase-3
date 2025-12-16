# Implementation Plan: Phase III - Chat + Skills

**Branch**: `001-chat-skills` | **Date**: 2025-12-15 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-chat-skills/spec.md`

---

## Summary

Transform the existing REST API-based todo application into an AI-powered conversational interface using Google Gemini 2.0 Flash (free tier) with a multi-agent skills architecture. The system routes user intents to specialized skill agents (TaskManagement, TaskSearch, TaskAnalytics, TaskRecommendation) via MCP tools that call the existing REST API.

---

## Technical Context

**Language/Version**: Python 3.11+ (backend), TypeScript/Node.js 18+ (frontend)
**Primary Dependencies**: FastAPI, SQLAlchemy, OpenAI SDK (Gemini), Next.js 16, React 19, Better Auth
**Storage**: PostgreSQL (Neon free tier) - existing + new conversations/messages tables
**Testing**: pytest (backend), manual E2E testing (frontend)
**Target Platform**: Web application (desktop + mobile responsive)
**Project Type**: Web application (frontend + backend monorepo)
**Performance Goals**: <4s response time (single-skill), <6s (multi-skill), 50+ concurrent users
**Constraints**: Zero LLM cost (Gemini free tier), 60 req/min rate limit, 50 message context window
**Scale/Scope**: Single user conversations, 4 skill agents, 7 MCP tools, custom chat UI

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| Minimal complexity | PASS | 2 projects (frontend/backend), no unnecessary abstractions |
| Test coverage | PASS | pytest for backend, E2E manual testing planned |
| Code reuse | PASS | MCP tools reuse existing REST API |
| Security | PASS | JWT auth, user isolation in DB queries |
| Stateless design | PASS | No in-memory state, DB persistence |

**Post-Design Re-Check**: All gates remain PASS. No complexity violations.

---

## Project Structure

### Documentation (this feature)

```text
specs/001-chat-skills/
├── spec.md              # Feature specification
├── plan.md              # This file
├── research.md          # Phase 0 - Technical decisions
├── data-model.md        # Phase 1 - Database schema
├── quickstart.md        # Phase 1 - Setup guide
├── contracts/
│   ├── chat-api.yaml    # OpenAPI specification
│   └── mcp-tools.md     # MCP tool contracts
└── tasks.md             # Phase 2 - Implementation tasks (via /sp.tasks)
```

### Source Code (repository root)

```text
backend/
├── agents/
│   ├── __init__.py
│   ├── main_agent.py           # Intent router (Main Agent)
│   ├── agent_config.py         # Gemini API configuration
│   └── skills/
│       ├── __init__.py
│       ├── base_skill.py       # Abstract base class
│       ├── task_management.py  # TaskManagementSkill
│       ├── task_search.py      # TaskSearchSkill
│       ├── task_analytics.py   # TaskAnalyticsSkill
│       └── task_recommendation.py  # TaskRecommendationSkill
├── mcp_tools/
│   ├── __init__.py
│   ├── add_task.py
│   ├── list_tasks.py
│   ├── complete_task.py
│   ├── delete_task.py
│   ├── update_task.py
│   ├── search_tasks.py         # NEW
│   └── get_task_stats.py       # NEW
├── db/
│   ├── __init__.py
│   └── conversations.py        # Conversation CRUD helpers
├── routes/
│   └── chat.py                 # Chat API endpoint
├── migrations/
│   └── 003_add_conversations.py  # New tables
├── tests/
│   ├── test_mcp_tools.py
│   ├── test_conversations_db.py
│   ├── test_task_management_skill.py
│   ├── test_task_search_skill.py
│   ├── test_task_analytics_skill.py
│   ├── test_task_recommendation_skill.py
│   ├── test_main_agent.py
│   └── test_chat_endpoint.py
├── models.py                   # UPDATED with Conversation, Message
├── service.py                  # UPDATED with /tasks/stats endpoint
├── database.py                 # Existing
└── main.py                     # UPDATED to include chat router

frontend/
├── app/
│   └── chat/
│       └── page.tsx            # Chat page
├── components/
│   ├── chat/
│   │   ├── ChatInterface.tsx   # Main chat container
│   │   ├── MessageList.tsx     # Message display
│   │   ├── MessageInput.tsx    # User input
│   │   ├── TypingIndicator.tsx # Loading state
│   │   └── WelcomeMessage.tsx  # Initial greeting
│   └── ui/                     # Existing UI components
├── lib/
│   └── chat-api.ts             # Chat API client
├── types/
│   └── chat.ts                 # Chat TypeScript types
└── ...                         # Existing files
```

**Structure Decision**: Web application with frontend/backend separation (Option 2). Extends existing monorepo structure with new `/agents`, `/mcp_tools`, `/db`, and `/routes` directories in backend.

---

## Implementation Phases

### Phase 1: Foundation & Infrastructure

**Goal**: Database schema, Gemini setup, MCP tools

**Deliverables**:
1. `backend/migrations/003_add_conversations.py` - Conversations and Messages tables
2. `backend/models.py` - Updated with Conversation, Message models
3. `backend/agents/agent_config.py` - Gemini API configuration
4. `backend/mcp_tools/*.py` - 7 MCP tool implementations
5. `backend/db/conversations.py` - Conversation CRUD helpers
6. `backend/service.py` - Extended with `/tasks/stats` endpoint

**Acceptance Criteria**:
- [ ] Migration runs without errors
- [ ] Gemini API connection works
- [ ] All 7 MCP tools pass tests
- [ ] Conversation CRUD operations work

---

### Phase 2: Multi-Agent Skills Architecture

**Goal**: Main agent + 4 specialized skill agents

**Deliverables**:
1. `backend/agents/skills/base_skill.py` - Abstract base class
2. `backend/agents/skills/task_management.py` - TaskManagementSkill
3. `backend/agents/skills/task_search.py` - TaskSearchSkill
4. `backend/agents/skills/task_analytics.py` - TaskAnalyticsSkill
5. `backend/agents/skills/task_recommendation.py` - TaskRecommendationSkill
6. `backend/agents/main_agent.py` - Intent router

**Acceptance Criteria**:
- [ ] All skills inherit from BaseSkill
- [ ] Each skill has optimized system prompt
- [ ] Main agent routes to correct skill >95%
- [ ] Fallback to TaskManagementSkill works
- [ ] All skill tests pass

---

### Phase 3: Chat Endpoint & UI

**Goal**: Backend endpoint and custom frontend chat UI

**Deliverables**:
1. `backend/routes/chat.py` - Chat API endpoint
2. `frontend/app/chat/page.tsx` - Chat page
3. `frontend/components/chat/*.tsx` - Chat UI components
4. `frontend/lib/chat-api.ts` - API client
5. `frontend/types/chat.ts` - TypeScript types

**Acceptance Criteria**:
- [ ] Chat endpoint accepts POST requests
- [ ] JWT validation works
- [ ] Messages persist to database
- [ ] skill_used tracked correctly
- [ ] Chat UI displays messages
- [ ] Typing indicator shows
- [ ] Mobile responsive

---

### Phase 4: Testing & Polish

**Goal**: End-to-end testing, bug fixes, documentation

**Deliverables**:
1. Complete test suite (40+ tests)
2. E2E test report
3. Bug fixes
4. Updated documentation

**Acceptance Criteria**:
- [ ] All tests pass
- [ ] No critical bugs
- [ ] Zero API costs verified
- [ ] Documentation complete

---

## Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| LLM Provider | Gemini 2.0 Flash via OpenAI SDK | Zero cost, function calling support |
| Agent Architecture | Main Agent + 4 Skills | Separation of concerns, expert responses |
| Intent Routing | Keyword-based | Simple, fast, no extra API calls |
| Tool Implementation | REST API calls | Reuse existing logic |
| Chat UI | Custom React components | Zero dependencies, full control |
| Context Window | 50 messages | Balance context quality vs tokens |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Gemini rate limits (60/min) | Request queuing, graceful error messages |
| Skill routing errors | Fallback to TaskManagementSkill |
| Custom UI bugs | Progressive testing, simple-first approach |
| Large conversations | 50 message limit, pagination for history |

---

## Success Metrics

- **Response time**: <4s (single-skill), <6s (multi-skill)
- **Routing accuracy**: >95%
- **API cost**: $0/month
- **Test coverage**: >85%
- **User satisfaction**: Natural conversational experience

---

## Generated Artifacts

| Artifact | Path | Status |
|----------|------|--------|
| Research | `specs/001-chat-skills/research.md` | Complete |
| Data Model | `specs/001-chat-skills/data-model.md` | Complete |
| API Contract | `specs/001-chat-skills/contracts/chat-api.yaml` | Complete |
| MCP Tools | `specs/001-chat-skills/contracts/mcp-tools.md` | Complete |
| Quickstart | `specs/001-chat-skills/quickstart.md` | Complete |

---

## Next Steps

Run `/sp.tasks` to generate the detailed task breakdown for implementation.

---

**Plan Status**: Complete
**Ready for**: `/sp.tasks`
