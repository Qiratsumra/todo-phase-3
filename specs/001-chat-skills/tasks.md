# Tasks: Phase III - Chat + Skills

**Input**: Design documents from `/specs/001-chat-skills/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/
**Version**: 2.0.0 | **Date**: 2025-12-15

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1-US8)
- Include exact file paths in descriptions

## Path Conventions

- **Backend**: `backend/` at repository root (Python/FastAPI)
- **Frontend**: `frontend/` at repository root (Next.js/React)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and Gemini API configuration

- [ ] T001 Create backend/agents/ directory structure with __init__.py
- [ ] T002 Create backend/mcp_tools/ directory structure with __init__.py
- [ ] T003 Create backend/db/ directory structure with __init__.py
- [ ] T004 Create backend/routes/ directory structure with __init__.py
- [ ] T005 Create backend/migrations/ directory structure
- [ ] T006 [P] Create backend/agents/agent_config.py with Gemini API configuration using OpenAI SDK syntax and base_url override
- [ ] T007 [P] Add openai>=1.0.0 to backend/requirements.txt
- [ ] T008 [P] Update backend/.env with GEMINI_API_KEY, GEMINI_BASE_URL, GEMINI_MODEL, GEMINI_TEMPERATURE, GEMINI_MAX_TOKENS

**Checkpoint**: Project structure ready, Gemini configuration in place

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Database schema, MCP tools, and base skill class that ALL user stories depend on

**CRITICAL**: No user story work can begin until this phase is complete

### Database Schema

- [ ] T009 Create backend/migrations/003_add_conversations.py with conversations table (id, user_id, created_at, updated_at) and messages table (id, conversation_id, user_id, role, content, tool_calls, skill_used, created_at) including foreign keys and indexes
- [ ] T010 Update backend/models.py to add Conversation and Message SQLAlchemy models with skill_used field and proper relationships
- [ ] T011 Create backend/db/conversations.py with create_conversation(), get_conversation(), store_message(), get_conversation_history() functions

### MCP Tools (All 7)

- [ ] T012 [P] Create backend/mcp_tools/add_task.py that calls POST /api/tasks/ with title, description, priority, due_date, tags parameters
- [ ] T013 [P] Create backend/mcp_tools/list_tasks.py that calls GET /api/tasks/ with optional completed filter
- [ ] T014 [P] Create backend/mcp_tools/complete_task.py that calls POST /api/tasks/{task_id}/complete
- [ ] T015 [P] Create backend/mcp_tools/delete_task.py that calls DELETE /api/tasks/{task_id}
- [ ] T016 [P] Create backend/mcp_tools/update_task.py that calls PATCH /api/tasks/{task_id}
- [ ] T017 [P] Create backend/mcp_tools/search_tasks.py with keyword, completed, priority, date_from, date_to parameters
- [ ] T018 [P] Create backend/mcp_tools/get_task_stats.py that calculates total_tasks, completed_tasks, pending_tasks, completion_rate, tasks_by_day
- [ ] T019 Add GET /api/tasks/stats endpoint to backend/service.py for task statistics

### Base Skill Architecture

- [ ] T020 Create backend/agents/skills/__init__.py package
- [ ] T021 Create backend/agents/skills/base_skill.py with abstract BaseSkill class defining __init__(name, description, tools), get_system_prompt(), can_handle(user_message), process(user_message, conversation_history) methods

### Tool Definitions for Gemini

- [ ] T022 Create backend/mcp_tools/tool_definitions.py with TOOLS list containing all 7 MCP tools in Gemini function calling format

**Checkpoint**: Foundation ready - all MCP tools work, database schema exists, base skill defined

---

## Phase 3: User Story 1 - Send Chat Message and Receive AI Response (Priority: P1)

**Goal**: User can send a message via chat and receive an AI response within 4 seconds

**Independent Test**: Open chat, send "Hello", verify AI responds with greeting

### Implementation for User Story 1

- [ ] T023 [US1] Create backend/agents/skills/task_management.py implementing BaseSkill with system prompt for CRUD operations, can_handle() detecting add/create/delete/complete/update/list/show keywords, process() using Gemini with 5 MCP tools
- [ ] T024 [US1] Create backend/agents/main_agent.py with MainAgent class that initializes TaskManagementSkill, route_intent() method, process_message() orchestrator, returns response with skill_used
- [ ] T025 [US1] Create backend/routes/chat.py with POST /api/{user_id}/chat endpoint that validates request, gets/creates conversation, stores user message, calls main_agent.process_message(), stores assistant response with skill_used, returns ChatResponse
- [ ] T026 [US1] Update backend/main.py to include chat router with prefix /api
- [ ] T027 [P] [US1] Create frontend/types/chat.ts with Message, ChatRequest, ChatResponse TypeScript interfaces
- [ ] T028 [P] [US1] Create frontend/lib/chat-api.ts with sendChatMessage(userId, message, conversationId) function using Better Auth JWT
- [ ] T029 [P] [US1] Create frontend/components/chat/MessageList.tsx displaying messages with user (right/blue) and assistant (left/gray) styling, skill badges, timestamps
- [ ] T030 [P] [US1] Create frontend/components/chat/MessageInput.tsx with textarea, send button, Enter to send, Shift+Enter for newline, disabled state while loading
- [ ] T031 [P] [US1] Create frontend/components/chat/TypingIndicator.tsx with 3 animated dots
- [ ] T032 [US1] Create frontend/app/chat/page.tsx with useState for messages/isLoading/conversationId, handleSend function, MessageList, MessageInput, TypingIndicator, header

**Checkpoint**: User Story 1 complete - can send message and receive AI response via chat UI

---

## Phase 4: User Story 2 - Create Tasks via Natural Language (Priority: P1)

**Goal**: User can create tasks by typing "Add task to buy groceries" and task is created

**Independent Test**: Send "Add task to buy milk", verify task appears in database with correct title

### Implementation for User Story 2

- [ ] T033 [US2] Enhance backend/agents/skills/task_management.py to parse task creation intents and extract title, priority, due_date from natural language
- [ ] T034 [US2] Add friendly response formatting with emojis (checkmark for created, etc.) to TaskManagementSkill

**Checkpoint**: User Story 2 complete - can create tasks via natural language

---

## Phase 5: User Story 3 - View and List Tasks Conversationally (Priority: P1)

**Goal**: User can ask "Show me my tasks" and receive formatted list

**Independent Test**: Create 3 tasks, send "Show all my tasks", verify all 3 are listed in response

### Implementation for User Story 3

- [ ] T035 [US3] Enhance TaskManagementSkill to format task lists clearly with priority, status, due date indicators

**Checkpoint**: User Story 3 complete - can list tasks conversationally

---

## Phase 6: User Story 4 - Complete and Delete Tasks via Chat (Priority: P2)

**Goal**: User can mark tasks done or delete them via chat

**Independent Test**: Create task, send "Mark buy groceries as done", verify task.completed=true

### Implementation for User Story 4

- [ ] T036 [US4] Enhance TaskManagementSkill to handle task identification by name or partial match
- [ ] T037 [US4] Add confirmation messages for complete and delete operations

**Checkpoint**: User Story 4 complete - can complete/delete tasks via chat

---

## Phase 7: User Story 5 - Search Tasks with Complex Queries (Priority: P2)

**Goal**: User can find tasks with "Find tasks containing meeting"

**Independent Test**: Create tasks with various titles, search by keyword, verify correct filtering

### Implementation for User Story 5

- [ ] T038 [US5] Create backend/agents/skills/task_search.py implementing BaseSkill with system prompt for search expertise, can_handle() detecting find/search/where/filter/containing keywords, process() using search_tasks and list_tasks tools
- [ ] T039 [US5] Update backend/agents/main_agent.py to register TaskSearchSkill and route search intents

**Checkpoint**: User Story 5 complete - can search tasks with keywords

---

## Phase 8: User Story 6 - Get Task Analytics and Insights (Priority: P3)

**Goal**: User can ask "What's my completion rate?" and get statistics

**Independent Test**: Create 5 tasks, complete 3, ask for completion rate, verify 60% shown

### Implementation for User Story 6

- [ ] T040 [US6] Create backend/agents/skills/task_analytics.py implementing BaseSkill with system prompt for data analysis, can_handle() detecting how many/completion rate/statistics/productivity keywords, process() using get_task_stats tool
- [ ] T041 [US6] Update backend/agents/main_agent.py to register TaskAnalyticsSkill and route analytics intents

**Checkpoint**: User Story 6 complete - can get task analytics

---

## Phase 9: User Story 7 - Get Smart Task Recommendations (Priority: P3)

**Goal**: User can ask "What should I work on next?" and get suggestions

**Independent Test**: Create tasks with various priorities, ask for recommendation, verify high-priority suggested first

### Implementation for User Story 7

- [ ] T042 [US7] Create backend/agents/skills/task_recommendation.py implementing BaseSkill with system prompt for productivity coaching, can_handle() detecting suggest/recommend/prioritize/what should/next keywords, process() analyzing tasks and providing recommendations
- [ ] T043 [US7] Update backend/agents/main_agent.py to register TaskRecommendationSkill and route recommendation intents

**Checkpoint**: User Story 7 complete - can get task recommendations

---

## Phase 10: User Story 8 - Persistent Conversation History (Priority: P2)

**Goal**: Chat history persists across browser sessions

**Independent Test**: Send messages, close browser, reopen, verify previous messages displayed

### Implementation for User Story 8

- [ ] T044 [US8] Create GET /api/{user_id}/conversations endpoint in backend/routes/chat.py to list user conversations
- [ ] T045 [US8] Create GET /api/{user_id}/conversations/{conversation_id} endpoint to get conversation with messages
- [ ] T046 [US8] Update frontend/app/chat/page.tsx to load existing conversation on mount using useEffect
- [ ] T047 [US8] Add conversation history loading state to chat page

**Checkpoint**: User Story 8 complete - conversation history persists

---

## Phase 11: Polish & Cross-Cutting Concerns

**Purpose**: UI polish, error handling, documentation

### UI Polish

- [ ] T048 [P] Create frontend/components/chat/WelcomeMessage.tsx with greeting and example commands
- [ ] T049 [P] Add smooth scroll to bottom on new message in frontend/app/chat/page.tsx
- [ ] T050 [P] Add auto-focus input after sending message
- [ ] T051 [P] Add empty state handling with WelcomeMessage component
- [ ] T052 Improve mobile responsive styling in chat components

### Error Handling

- [ ] T053 Add Gemini API error handling with retry logic (max 3 retries) in backend/agents/agent_config.py
- [ ] T054 Add user-friendly error messages for common failures in chat endpoint
- [ ] T055 Add rate limit handling (429) with helpful message

### Documentation

- [ ] T056 [P] Update README.md with Phase III features, Gemini setup, skills architecture
- [ ] T057 [P] Create docs/SKILLS_ARCHITECTURE.md documenting how skills work and how to add new ones
- [ ] T058 [P] Create docs/DEPLOYMENT.md with backend and frontend deployment instructions
- [ ] T059 [P] Create docs/DEVELOPMENT.md with local development setup and testing guide

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - start immediately
- **Foundational (Phase 2)**: Depends on Setup - BLOCKS all user stories
- **User Stories (Phase 3-10)**: All depend on Foundational completion
  - US1-US3 (P1): Can proceed immediately after Foundational
  - US4-US5 (P2): Can start after Foundational (parallel with P1 if team allows)
  - US6-US7 (P3): Can start after Foundational
  - US8 (P2): Can start after US1 is complete (needs chat page)
- **Polish (Phase 11)**: Depends on core user stories being complete

### User Story Dependencies

| Story | Depends On | Can Start After |
|-------|------------|-----------------|
| US1 | Foundational | Phase 2 complete |
| US2 | US1 | T024 (main_agent) |
| US3 | US1 | T024 (main_agent) |
| US4 | US1 | T024 (main_agent) |
| US5 | US1 | T024 (main_agent) |
| US6 | US1 | T024 (main_agent) |
| US7 | US1 | T024 (main_agent) |
| US8 | US1 | T032 (chat page) |

### Parallel Opportunities

**Phase 1 (Setup)**:
```
T006, T007, T008 can run in parallel
```

**Phase 2 (Foundational MCP Tools)**:
```
T012, T013, T014, T015, T016, T017, T018 can all run in parallel
```

**Phase 3 (US1 Frontend Components)**:
```
T027, T028, T029, T030, T031 can all run in parallel
```

**Phase 11 (Documentation)**:
```
T056, T057, T058, T059 can all run in parallel
```

---

## Implementation Strategy

### MVP First (User Stories 1-3)

1. Complete Phase 1: Setup (T001-T008)
2. Complete Phase 2: Foundational (T009-T022) - **CRITICAL BLOCKER**
3. Complete Phase 3: User Story 1 (T023-T032) - Core chat working
4. Complete Phase 4: User Story 2 (T033-T034) - Task creation via chat
5. Complete Phase 5: User Story 3 (T035) - Task listing via chat
6. **STOP and VALIDATE**: Test sending messages, creating tasks, listing tasks
7. Deploy/demo MVP

### Incremental Delivery

| Increment | Stories | What User Can Do |
|-----------|---------|------------------|
| MVP | US1-US3 | Chat, create tasks, list tasks |
| +Search | US4-US5 | Complete/delete tasks, search |
| +Analytics | US6-US7 | Get stats, recommendations |
| +History | US8 | Persistent conversations |
| +Polish | - | Better UX, documentation |

---

## Summary

| Category | Task Count |
|----------|------------|
| Setup | 8 |
| Foundational | 14 |
| US1: Chat + Response | 10 |
| US2: Create Tasks | 2 |
| US3: List Tasks | 1 |
| US4: Complete/Delete | 2 |
| US5: Search Tasks | 2 |
| US6: Analytics | 2 |
| US7: Recommendations | 2 |
| US8: Persistence | 4 |
| Polish | 12 |
| **Total** | **59** |

**Priority Breakdown**:
- Critical (P1): 21 tasks (US1-US3 + Foundational)
- High (P2): 8 tasks (US4-US5, US8)
- Medium (P3): 4 tasks (US6-US7)
- Polish: 12 tasks

**Parallel Opportunities**: 27 tasks marked [P]

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [USx] label maps task to specific user story
- Each user story is independently testable after completion
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- MVP = Phase 1 + Phase 2 + US1 + US2 + US3 (minimum viable chat)
