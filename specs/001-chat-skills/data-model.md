# Data Model: Phase III - Chat + Skills

**Feature Branch**: `001-chat-skills`
**Created**: 2025-12-15
**Status**: Complete

---

## Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│     users       │       │  conversations  │       │    messages     │
│  (Better Auth)  │       │                 │       │                 │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id (PK)         │──1:N──│ id (PK)         │──1:N──│ id (PK)         │
│ email           │       │ user_id (FK)    │       │ conversation_id │
│ name            │       │ created_at      │       │ user_id (FK)    │
│ ...             │       │ updated_at      │       │ role            │
└─────────────────┘       └─────────────────┘       │ content         │
                                                     │ tool_calls      │
                                                     │ skill_used      │
                                                     │ created_at      │
                                                     └─────────────────┘

┌─────────────────┐
│     tasks       │
│   (existing)    │
├─────────────────┤
│ id (PK)         │
│ title           │
│ description     │
│ completed       │
│ created_at      │
│ priority        │
│ tags            │
│ due_date        │
│ ...             │
└─────────────────┘
```

---

## Entities

### 1. Conversation

Represents a chat session between a user and the AI assistant.

**Purpose**: Group related messages and maintain context across sessions

**Table Name**: `conversations`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | INTEGER | PK, AUTO_INCREMENT | Unique conversation identifier |
| user_id | VARCHAR(255) | NOT NULL, INDEX | User who owns this conversation |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | When conversation started |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last activity timestamp |

**Indexes**:
- `PRIMARY KEY (id)`
- `INDEX idx_conversations_user_id (user_id)`
- `INDEX idx_conversations_updated_at (updated_at)`

**Relationships**:
- One user has many conversations (1:N)
- One conversation has many messages (1:N)

**SQLAlchemy Model**:
```python
class Conversation(Base):
    __tablename__ = "conversations"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(String(255), nullable=False, index=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    updated_at = Column(DateTime, default=lambda: datetime.now(timezone.utc),
                        onupdate=lambda: datetime.now(timezone.utc))

    messages = relationship("Message", back_populates="conversation")
```

---

### 2. Message

Represents a single message in a conversation (user or assistant).

**Purpose**: Store chat history with skill tracking for analytics

**Table Name**: `messages`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | INTEGER | PK, AUTO_INCREMENT | Unique message identifier |
| conversation_id | INTEGER | FK, NOT NULL, INDEX | Parent conversation |
| user_id | VARCHAR(255) | NOT NULL, INDEX | User who owns this message |
| role | VARCHAR(20) | NOT NULL | "user" or "assistant" |
| content | TEXT | NOT NULL | Message text content |
| tool_calls | JSON | NULLABLE | Tool invocations (if any) |
| skill_used | VARCHAR(50) | NULLABLE, INDEX | Which skill processed this |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | When message was sent |

**Indexes**:
- `PRIMARY KEY (id)`
- `INDEX idx_messages_conversation_id (conversation_id)`
- `INDEX idx_messages_user_id (user_id)`
- `INDEX idx_messages_created_at (created_at)`
- `INDEX idx_messages_skill_used (skill_used)`
- `FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE`

**Relationships**:
- Many messages belong to one conversation (N:1)

**SQLAlchemy Model**:
```python
class Message(Base):
    __tablename__ = "messages"

    id = Column(Integer, primary_key=True, index=True)
    conversation_id = Column(Integer, ForeignKey("conversations.id", ondelete="CASCADE"),
                            nullable=False, index=True)
    user_id = Column(String(255), nullable=False, index=True)
    role = Column(String(20), nullable=False)  # "user" or "assistant"
    content = Column(Text, nullable=False)
    tool_calls = Column(JSON, nullable=True)
    skill_used = Column(String(50), nullable=True, index=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc), index=True)

    conversation = relationship("Conversation", back_populates="messages")
```

**Valid Values for `role`**:
- `"user"` - Message from the human user
- `"assistant"` - Message from the AI assistant

**Valid Values for `skill_used`**:
- `"TaskManagementSkill"` - CRUD operations
- `"TaskSearchSkill"` - Search and filtering
- `"TaskAnalyticsSkill"` - Statistics and insights
- `"TaskRecommendationSkill"` - Suggestions and priorities
- `NULL` - No skill used (simple greeting, etc.)

---

### 3. Task (Existing - No Changes)

Already implemented in Phase I/II. No modifications needed.

**Table Name**: `tasks`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | INTEGER | PK | Task identifier |
| title | VARCHAR | NOT NULL | Task title |
| description | TEXT | NULLABLE | Task description |
| completed | BOOLEAN | DEFAULT FALSE | Completion status |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp |
| priority | INTEGER | DEFAULT 0 | Priority level (0-3) |
| tags | ARRAY[VARCHAR] | NULLABLE | Task tags |
| due_date | DATE | NULLABLE | Due date |
| recurrence_pattern | VARCHAR | NULLABLE | Recurrence rule |
| ... | ... | ... | Other existing fields |

**Note**: Tasks currently have no `user_id` field. This is a known limitation - all tasks are shared. For Phase III, MCP tools will need to be extended to filter by user when multi-user support is added.

---

## State Transitions

### Conversation States

```
┌─────────┐     create_conversation()     ┌─────────┐
│  None   │ ───────────────────────────► │ Active  │
└─────────┘                               └────┬────┘
                                               │
                     add_message()             │ delete_conversation()
                     (updates updated_at)      │
                          │                    │
                          ▼                    ▼
                     ┌─────────┐          ┌─────────┐
                     │ Active  │          │ Deleted │
                     └─────────┘          └─────────┘
```

### Message Creation Flow

```
User sends message
       │
       ▼
┌─────────────────┐
│ Store user msg  │ ─── role="user", skill_used=NULL
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Route to skill  │ ─── Main agent classifies intent
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Skill processes │ ─── May call MCP tools
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│ Store assistant msg │ ─── role="assistant", skill_used="TaskXxxSkill"
└─────────────────────┘
```

---

## Validation Rules

### Conversation
- `user_id` must not be empty
- `user_id` must match authenticated user for all operations

### Message
- `role` must be one of: `"user"`, `"assistant"`
- `content` must not be empty
- `conversation_id` must exist
- `user_id` must match conversation's `user_id`
- `skill_used` must be one of valid skill names or NULL

---

## Query Patterns

### Get or Create Conversation
```sql
-- Check for existing conversation
SELECT * FROM conversations
WHERE user_id = :user_id
ORDER BY updated_at DESC
LIMIT 1;

-- Create new if not exists
INSERT INTO conversations (user_id, created_at, updated_at)
VALUES (:user_id, NOW(), NOW())
RETURNING id;
```

### Get Conversation History (Last 50 Messages)
```sql
SELECT * FROM messages
WHERE conversation_id = :conversation_id
  AND user_id = :user_id
ORDER BY created_at ASC
LIMIT 50;
```

### Store Message
```sql
INSERT INTO messages
  (conversation_id, user_id, role, content, tool_calls, skill_used, created_at)
VALUES
  (:conversation_id, :user_id, :role, :content, :tool_calls, :skill_used, NOW())
RETURNING id;

-- Update conversation timestamp
UPDATE conversations
SET updated_at = NOW()
WHERE id = :conversation_id;
```

### Get Skill Usage Analytics
```sql
SELECT skill_used, COUNT(*) as count
FROM messages
WHERE user_id = :user_id
  AND role = 'assistant'
  AND skill_used IS NOT NULL
GROUP BY skill_used;
```

---

## Migration Plan

### Migration 003: Add Conversations Tables

```python
# backend/migrations/003_add_conversations.py
"""
Add conversations and messages tables for chat functionality.
"""

def upgrade(engine):
    """Create conversations and messages tables."""
    with engine.connect() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS conversations (
                id SERIAL PRIMARY KEY,
                user_id VARCHAR(255) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );

            CREATE INDEX IF NOT EXISTS idx_conversations_user_id
                ON conversations(user_id);
            CREATE INDEX IF NOT EXISTS idx_conversations_updated_at
                ON conversations(updated_at);
        """)

        conn.execute("""
            CREATE TABLE IF NOT EXISTS messages (
                id SERIAL PRIMARY KEY,
                conversation_id INTEGER NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
                user_id VARCHAR(255) NOT NULL,
                role VARCHAR(20) NOT NULL,
                content TEXT NOT NULL,
                tool_calls JSONB,
                skill_used VARCHAR(50),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );

            CREATE INDEX IF NOT EXISTS idx_messages_conversation_id
                ON messages(conversation_id);
            CREATE INDEX IF NOT EXISTS idx_messages_user_id
                ON messages(user_id);
            CREATE INDEX IF NOT EXISTS idx_messages_created_at
                ON messages(created_at);
            CREATE INDEX IF NOT EXISTS idx_messages_skill_used
                ON messages(skill_used);
        """)
        conn.commit()

def downgrade(engine):
    """Remove conversations and messages tables."""
    with engine.connect() as conn:
        conn.execute("DROP TABLE IF EXISTS messages;")
        conn.execute("DROP TABLE IF EXISTS conversations;")
        conn.commit()
```

---

## Data Retention

- **Conversations**: Retained indefinitely (until user deletes)
- **Messages**: Retained with conversation
- **Context Window**: Last 50 messages loaded per request
- **Future**: Consider archival policy for old conversations

---

**Data Model Status**: Complete
**Next Step**: API Contracts
