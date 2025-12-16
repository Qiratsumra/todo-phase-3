# MCP Tools Contract: Phase III - Chat + Skills

**Feature Branch**: `001-chat-skills`
**Created**: 2025-12-15
**Status**: Complete

---

## Overview

MCP (Model Context Protocol) tools are functions that skills can call to interact with the todo system. Each tool is defined with a name, description, and parameters schema for Gemini function calling.

---

## Tool Registry

| Tool Name | Skill(s) | Description |
|-----------|----------|-------------|
| add_task | TaskManagementSkill | Create a new task |
| list_tasks | TaskManagementSkill, TaskSearchSkill | List tasks with optional filters |
| complete_task | TaskManagementSkill | Mark a task as completed |
| delete_task | TaskManagementSkill | Delete a task |
| update_task | TaskManagementSkill | Update task attributes |
| search_tasks | TaskSearchSkill | Advanced search with keywords/dates |
| get_task_stats | TaskAnalyticsSkill | Get task statistics |

---

## Tool Definitions

### 1. add_task

**Purpose**: Create a new task for the user

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "title": {
      "type": "string",
      "description": "The title of the task (required)"
    },
    "description": {
      "type": "string",
      "description": "Optional description of the task"
    },
    "priority": {
      "type": "integer",
      "description": "Priority level: 0=none, 1=low, 2=medium, 3=high",
      "enum": [0, 1, 2, 3],
      "default": 0
    },
    "due_date": {
      "type": "string",
      "format": "date",
      "description": "Due date in YYYY-MM-DD format"
    },
    "tags": {
      "type": "array",
      "items": {"type": "string"},
      "description": "List of tags for the task"
    }
  },
  "required": ["title"]
}
```

**Returns**:
```json
{
  "id": 42,
  "title": "buy groceries",
  "description": null,
  "completed": false,
  "created_at": "2025-12-15T10:30:00Z",
  "priority": 0,
  "tags": null,
  "due_date": null
}
```

**REST API Mapping**: `POST /api/tasks/`

**Example Usage**:
```python
result = add_task(
    title="Buy groceries",
    priority=2,
    due_date="2025-12-16"
)
# Returns: {"id": 42, "title": "Buy groceries", ...}
```

---

### 2. list_tasks

**Purpose**: Retrieve tasks with optional filtering

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "completed": {
      "type": "boolean",
      "description": "Filter by completion status (true=completed, false=pending, null=all)"
    },
    "limit": {
      "type": "integer",
      "description": "Maximum number of tasks to return",
      "default": 50
    }
  },
  "required": []
}
```

**Returns**:
```json
[
  {
    "id": 1,
    "title": "Buy groceries",
    "completed": false,
    "priority": 2,
    "due_date": "2025-12-16",
    "created_at": "2025-12-15T10:30:00Z"
  },
  {
    "id": 2,
    "title": "Call dentist",
    "completed": true,
    "priority": 1,
    "due_date": null,
    "created_at": "2025-12-14T09:00:00Z"
  }
]
```

**REST API Mapping**: `GET /api/tasks/?completed={completed}`

**Example Usage**:
```python
# Get all pending tasks
pending = list_tasks(completed=False)

# Get all tasks
all_tasks = list_tasks()
```

---

### 3. complete_task

**Purpose**: Mark a task as completed

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "task_id": {
      "type": "integer",
      "description": "ID of the task to complete"
    }
  },
  "required": ["task_id"]
}
```

**Returns**:
```json
{
  "id": 42,
  "title": "Buy groceries",
  "completed": true,
  "created_at": "2025-12-15T10:30:00Z"
}
```

**Error Response** (task not found):
```json
{
  "error": "Task not found",
  "task_id": 999
}
```

**REST API Mapping**: `POST /api/tasks/{task_id}/complete`

**Example Usage**:
```python
result = complete_task(task_id=42)
# Returns: {"id": 42, "completed": true, ...}
```

---

### 4. delete_task

**Purpose**: Delete a task permanently

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "task_id": {
      "type": "integer",
      "description": "ID of the task to delete"
    }
  },
  "required": ["task_id"]
}
```

**Returns**:
```json
{
  "message": "Task deleted successfully",
  "id": 42
}
```

**Error Response** (task not found):
```json
{
  "error": "Task not found",
  "task_id": 999
}
```

**REST API Mapping**: `DELETE /api/tasks/{task_id}`

**Example Usage**:
```python
result = delete_task(task_id=42)
# Returns: {"message": "Task deleted successfully", "id": 42}
```

---

### 5. update_task

**Purpose**: Update one or more attributes of a task

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "task_id": {
      "type": "integer",
      "description": "ID of the task to update"
    },
    "title": {
      "type": "string",
      "description": "New title for the task"
    },
    "description": {
      "type": "string",
      "description": "New description"
    },
    "priority": {
      "type": "integer",
      "description": "New priority level",
      "enum": [0, 1, 2, 3]
    },
    "due_date": {
      "type": "string",
      "format": "date",
      "description": "New due date"
    },
    "tags": {
      "type": "array",
      "items": {"type": "string"},
      "description": "New tags list"
    }
  },
  "required": ["task_id"]
}
```

**Returns**:
```json
{
  "id": 42,
  "title": "Buy groceries and milk",
  "description": "From the local store",
  "completed": false,
  "priority": 3,
  "due_date": "2025-12-16"
}
```

**REST API Mapping**: `PATCH /api/tasks/{task_id}`

**Example Usage**:
```python
result = update_task(
    task_id=42,
    title="Buy groceries and milk",
    priority=3
)
```

---

### 6. search_tasks

**Purpose**: Advanced search with keyword, status, and date filtering

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "keyword": {
      "type": "string",
      "description": "Search keyword to find in title or description"
    },
    "completed": {
      "type": "boolean",
      "description": "Filter by completion status"
    },
    "priority": {
      "type": "integer",
      "description": "Filter by priority level",
      "enum": [0, 1, 2, 3]
    },
    "date_from": {
      "type": "string",
      "format": "date",
      "description": "Filter tasks created on or after this date"
    },
    "date_to": {
      "type": "string",
      "format": "date",
      "description": "Filter tasks created on or before this date"
    },
    "due_before": {
      "type": "string",
      "format": "date",
      "description": "Filter tasks due on or before this date"
    },
    "tags": {
      "type": "array",
      "items": {"type": "string"},
      "description": "Filter tasks containing any of these tags"
    }
  },
  "required": []
}
```

**Returns**:
```json
{
  "count": 3,
  "tasks": [
    {
      "id": 5,
      "title": "Team meeting prep",
      "completed": false,
      "priority": 2,
      "due_date": "2025-12-16"
    },
    {
      "id": 8,
      "title": "Client meeting notes",
      "completed": true,
      "priority": 1
    },
    {
      "id": 12,
      "title": "Schedule meeting with HR",
      "completed": false,
      "priority": 3
    }
  ]
}
```

**REST API Mapping**: `GET /api/tasks/?keyword={keyword}&completed={completed}&...` (extended)

**Example Usage**:
```python
# Find all incomplete tasks with "meeting" in title
results = search_tasks(
    keyword="meeting",
    completed=False
)

# Find high-priority tasks due this week
results = search_tasks(
    priority=3,
    due_before="2025-12-22"
)
```

---

### 7. get_task_stats

**Purpose**: Get aggregated task statistics for analytics

**Parameters**:
```json
{
  "type": "object",
  "properties": {
    "days": {
      "type": "integer",
      "description": "Number of days to include in trends (default 7)",
      "default": 7
    }
  },
  "required": []
}
```

**Returns**:
```json
{
  "total_tasks": 25,
  "completed_tasks": 18,
  "pending_tasks": 7,
  "completion_rate": 72.0,
  "overdue_tasks": 2,
  "high_priority_pending": 3,
  "tasks_by_day": [
    {"date": "2025-12-09", "created": 3, "completed": 2},
    {"date": "2025-12-10", "created": 2, "completed": 4},
    {"date": "2025-12-11", "created": 5, "completed": 3},
    {"date": "2025-12-12", "created": 1, "completed": 2},
    {"date": "2025-12-13", "created": 4, "completed": 3},
    {"date": "2025-12-14", "created": 2, "completed": 2},
    {"date": "2025-12-15", "created": 3, "completed": 2}
  ]
}
```

**REST API Mapping**: `GET /api/tasks/stats?days={days}` (new endpoint)

**Example Usage**:
```python
stats = get_task_stats(days=7)
# Returns completion rates, trends, etc.
```

---

## Gemini Function Calling Format

For integration with Gemini API, tools are defined as:

```python
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "add_task",
            "description": "Create a new task for the user",
            "parameters": {
                "type": "object",
                "properties": {
                    "title": {"type": "string", "description": "Task title"},
                    "description": {"type": "string", "description": "Task description"},
                    "priority": {"type": "integer", "enum": [0,1,2,3]},
                    "due_date": {"type": "string", "format": "date"},
                    "tags": {"type": "array", "items": {"type": "string"}}
                },
                "required": ["title"]
            }
        }
    },
    # ... other tools
]
```

---

## Error Handling

All tools follow consistent error response format:

```json
{
  "error": "Human-readable error message",
  "code": "MACHINE_READABLE_CODE",
  "details": {
    "task_id": 999,
    "field": "title"
  }
}
```

**Error Codes**:
- `NOT_FOUND` - Resource doesn't exist
- `VALIDATION_ERROR` - Invalid parameters
- `PERMISSION_DENIED` - User can't access resource
- `INTERNAL_ERROR` - Server error

---

## Skill-to-Tool Mapping

| Skill | Tools Available |
|-------|-----------------|
| TaskManagementSkill | add_task, list_tasks, complete_task, delete_task, update_task |
| TaskSearchSkill | search_tasks, list_tasks |
| TaskAnalyticsSkill | get_task_stats, list_tasks |
| TaskRecommendationSkill | list_tasks, get_task_stats |

---

**MCP Tools Contract Status**: Complete

