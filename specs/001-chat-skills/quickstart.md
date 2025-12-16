# Quickstart Guide: Phase III - Chat + Skills

**Feature Branch**: `001-chat-skills`
**Created**: 2025-12-15

---

## Prerequisites

- Python 3.11+
- Node.js 18+
- PostgreSQL database (Neon free tier)
- Gemini API key (free from Google AI Studio)

---

## 1. Get Gemini API Key (Free)

1. Go to [Google AI Studio](https://aistudio.google.com/)
2. Sign in with Google account
3. Click "Get API Key" → "Create API Key"
4. Copy the key (starts with `AIza...`)

---

## 2. Backend Setup

```bash
# Navigate to backend
cd backend

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Add new dependency for Gemini
pip install openai>=1.0.0

# Create/update .env file
cat >> .env << EOF
GEMINI_API_KEY=your_api_key_here
GEMINI_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/
GEMINI_MODEL=gemini-2.0-flash-exp
GEMINI_TEMPERATURE=0.7
GEMINI_MAX_TOKENS=2048
EOF

# Run database migration
python -c "from database import create_tables; create_tables()"

# Start backend
uvicorn main:app --reload --port 8000
```

---

## 3. Frontend Setup

```bash
# Navigate to frontend
cd frontend

# Install dependencies
npm install

# Update .env.local with backend URL
echo "NEXT_PUBLIC_API_URL=http://localhost:8000" >> .env.local

# Start frontend
npm run dev
```

---

## 4. Test the Chat

1. Open http://localhost:3000
2. Sign in with Better Auth
3. Navigate to /chat
4. Try these commands:

```
"Add a task to buy groceries"
→ TaskManagementSkill creates task

"Show me my tasks"
→ TaskManagementSkill lists tasks

"Find tasks with 'groceries'"
→ TaskSearchSkill searches

"What's my completion rate?"
→ TaskAnalyticsSkill shows stats

"What should I work on next?"
→ TaskRecommendationSkill suggests
```

---

## 5. Verify Skill Routing

Each response shows which skill was used:

```
🛒 TaskManagementSkill - CRUD operations
🔍 TaskSearchSkill - Search and filtering
📊 TaskAnalyticsSkill - Statistics
💡 TaskRecommendationSkill - Suggestions
```

---

## 6. API Testing

```bash
# Test chat endpoint
curl -X POST http://localhost:8000/api/test-user/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{"message": "Add a task to test the API"}'

# Expected response:
{
  "conversation_id": 1,
  "response": "I've created a new task 'test the API' for you!",
  "skill_used": "TaskManagementSkill",
  "created_at": "2025-12-15T10:30:00Z"
}
```

---

## Common Issues

### Gemini API Error
```
Error: API key not valid
```
**Fix**: Verify GEMINI_API_KEY in .env

### Rate Limit
```
Error: Rate limit exceeded
```
**Fix**: Free tier = 60 requests/min. Wait and retry.

### Database Connection
```
Error: Connection refused
```
**Fix**: Check DATABASE_URL in .env

---

## Project Structure

```
backend/
├── agents/
│   ├── main_agent.py           # Intent router
│   ├── agent_config.py         # Gemini config
│   └── skills/
│       ├── base_skill.py       # Abstract base
│       ├── task_management.py  # CRUD skill
│       ├── task_search.py      # Search skill
│       ├── task_analytics.py   # Analytics skill
│       └── task_recommendation.py
├── mcp_tools/
│   ├── add_task.py
│   ├── list_tasks.py
│   ├── complete_task.py
│   ├── delete_task.py
│   ├── update_task.py
│   ├── search_tasks.py
│   └── get_task_stats.py
├── db/
│   └── conversations.py        # DB helpers
└── routes/
    └── chat.py                 # Chat endpoint

frontend/
├── app/
│   └── chat/
│       └── page.tsx            # Chat page
├── components/
│   ├── MessageList.tsx
│   ├── MessageInput.tsx
│   └── TypingIndicator.tsx
└── lib/
    └── chat-api.ts             # API client
```

---

## Environment Variables Reference

### Backend (.env)
```bash
# Database
SQLALCHEMY_DATABASE_URL=postgresql://...
DB_HOST=localhost
DB_PORT=5433
DB_PASSWORD=your_password

# Gemini (NEW)
GEMINI_API_KEY=AIza...
GEMINI_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/
GEMINI_MODEL=gemini-2.0-flash-exp
GEMINI_TEMPERATURE=0.7
GEMINI_MAX_TOKENS=2048

# CORS
BACKEND_CORS_ORIGINS=["http://localhost:3000"]
```

### Frontend (.env.local)
```bash
NEXT_PUBLIC_API_URL=http://localhost:8000
BETTER_AUTH_SECRET=your_secret
```

---

## Next Steps

1. **Run tests**: `pytest backend/tests/ -v`
2. **Deploy backend**: Railway or Kubernetes
3. **Deploy frontend**: Vercel
4. **Monitor**: Check skill routing accuracy

---

**Quickstart Status**: Ready for implementation
