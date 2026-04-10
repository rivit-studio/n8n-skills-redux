# Production Patterns

All production hardening patterns in one reference document.

---

## Table of Contents

1. [The Silent Failure Problem](#the-silent-failure-problem)
2. [Entry-Point Validation](#entry-point-validation)
3. [Null vs Empty String Handling](#null-vs-empty-string-handling)
4. [External Logging](#external-logging)
5. [HTTP Status Codes](#http-status-codes)
6. [Error Response Patterns](#error-response-patterns)
7. [Pre-Deployment Testing](#pre-deployment-testing)

---

## The Silent Failure Problem

Workflows don't fail loudly by default. They continue processing bad data silently.

### Real Failure Scenario

A workflow worked in 50+ local tests. In production, it sent blank responses.

**Root cause:** Frontend sent `null` instead of `""` for one field. No validation. No logs. The workflow processed garbage and returned garbage.

### The Pattern

```javascript
// ❌ SILENT FAILURE
const email = $json.body.email;  // Could be null!
return [{json: {email}}];        // Returns garbage

// ✅ EXPLICIT VALIDATION
const body = $json.body || {};
const email = body.email;

if (!email) {
  return [{
    json: {
      status: 'error',
      statusCode: 400,
      message: 'email is required'
    }
  }];
}

return [{json: {email, validated: true}}];
```

---

## Entry-Point Validation

Check everything at the entry point. If any answer is "no", stop immediately.

### The Three Questions

1. **Does this user/entity exist?**
2. **Is the request authenticated/authorized?**
3. **Is the data shaped correctly?**

### Complete Validation Node

```javascript
// Place immediately after Webhook
const body = $json.body || {};
const headers = $json.headers || {};

const validation = {
  isValid: true,
  errors: [],
  warnings: []
};

// === REQUIRED FIELDS ===
const requiredFields = ['email', 'orderId', 'items'];
for (const field of requiredFields) {
  if (body[field] === undefined || body[field] === null) {
    validation.isValid = false;
    validation.errors.push(`Missing required field: ${field}`);
  }
}

// === TYPE CHECKS ===
if (body.email && typeof body.email !== 'string') {
  validation.isValid = false;
  validation.errors.push('email must be a string');
}

if (body.items && !Array.isArray(body.items)) {
  validation.isValid = false;
  validation.errors.push('items must be an array');
}

// === NULL vs EMPTY STRING ===
if (body.email === null) {
  validation.isValid = false;
  validation.errors.push('email is null (expected string)');
}

// === FORMAT VALIDATION ===
if (body.email && typeof body.email === 'string' && !body.email.includes('@')) {
  validation.isValid = false;
  validation.errors.push('email format invalid');
}

// === AUTHENTICATION ===
if (!headers['x-api-key']) {
  validation.isValid = false;
  validation.errors.push('Missing API key header');
}

return [{
  json: {
    ...body,
    _validation: validation,
    _validatedAt: new Date().toISOString()
  }
}];
```

Then use IF node:
- True: `{{$json._validation.isValid}}` equals `true` → Continue
- False: → Error Response node

---

## Null vs Empty String Handling

The #1 cause of silent production failures.

### The Problem

A field can be:
- `undefined` — key doesn't exist
- `null` — key exists, value is null
- `""` — key exists, value is empty string
- `"value"` — key exists, has value

Each requires different handling.

### Comparison Table

| Value | `!value` | `value == null` | `value === null` | `value === ''` |
|-------|----------|-----------------|------------------|----------------|
| `undefined` | `true` | `true` | `false` | `false` |
| `null` | `true` | `true` | `true` | `false` |
| `''` | `true` | `false` | `false` | `true` |
| `'value'` | `false` | `false` | `false` | `false` |

### Defensive Patterns

```javascript
// Reject null explicitly
if (body.field === null) {
  throw new Error('field cannot be null');
}

// Normalize null to empty string
const field = body.field ?? '';

// Check for "real" value
const hasField = body.field != null && body.field !== '';
```

---

## External Logging

n8n's internal logs won't help at 2 AM. Log to an external database.

### Database Schema

```sql
CREATE TABLE workflow_logs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  workflow_id TEXT NOT NULL,
  workflow_name TEXT,
  execution_id TEXT,
  log_type TEXT NOT NULL,  -- 'input', 'decision', 'output', 'error'
  stage TEXT,              -- 'entry', 'validation', 'processing', 'response'
  payload JSONB,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_workflow_logs_workflow_id ON workflow_logs(workflow_id);
CREATE INDEX idx_workflow_logs_execution_id ON workflow_logs(execution_id);
CREATE INDEX idx_workflow_logs_created_at ON workflow_logs(created_at);
```

### Logging Code Node

```javascript
const logEntry = {
  workflow_id: $workflow.id,
  workflow_name: $workflow.name,
  execution_id: $execution.id,
  log_type: 'input',  // Change: 'input', 'decision', 'output', 'error'
  stage: 'entry',     // Change: 'entry', 'validation', 'processing', 'response'
  payload: {
    body: $json.body,
    headers: {
      'content-type': $json.headers?.['content-type'],
      'user-agent': $json.headers?.['user-agent'],
      // Don't log: authorization, x-api-key
    }
  },
  metadata: {
    timestamp: new Date().toISOString(),
    nodeId: $node.id
  }
};

return [{json: {...$json, _logEntry: logEntry}}];
```

### What to Log at Each Stage

| Stage | Log Type | What to Capture |
|-------|----------|-----------------|
| Entry | `input` | Request body, headers (not auth), query params |
| Validation | `decision` | Validation result, errors, warnings |
| Processing | `decision` | Branches taken, key decisions |
| External Calls | `output` | Requests sent, responses received |
| Response | `output` | What was returned |
| Errors | `error` | Message, stack, state at failure |

### Error Logging (Error Trigger)

```javascript
const errorLog = {
  workflow_id: $json.workflow?.id,
  execution_id: $json.execution?.id,
  log_type: 'error',
  stage: 'error_handler',
  payload: {
    error_message: $json.execution?.error?.message,
    error_node: $json.execution?.error?.node?.name,
    error_stack: $json.execution?.error?.stack
  },
  metadata: {
    timestamp: new Date().toISOString()
  }
};

return [{json: errorLog}];
```

---

## HTTP Status Codes

Return proper status codes, not just "success" or "error".

### Reference

| Code | Meaning | When to Use |
|------|---------|-------------|
| `200` | Success | Request processed |
| `201` | Created | Resource created |
| `400` | Bad Request | Invalid input |
| `401` | Unauthorized | Missing/invalid auth |
| `403` | Forbidden | Valid auth, no permission |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Duplicate/state conflict |
| `422` | Unprocessable | Valid syntax, semantic error |
| `500` | Internal Error | Unexpected failure |
| `503` | Unavailable | Downstream service down |

---

## Error Response Patterns

### 400 - Validation Error

```javascript
return [{
  json: {
    statusCode: 400,
    body: {
      status: 'error',
      code: 'VALIDATION_ERROR',
      message: 'Invalid request data',
      errors: ['email is required', 'items must be array']
    }
  }
}];
```

### 401 - Authentication Error

```javascript
return [{
  json: {
    statusCode: 401,
    body: {
      status: 'error',
      code: 'UNAUTHORIZED',
      message: 'Invalid or missing API key'
    }
  }
}];
```

### 404 - Not Found

```javascript
return [{
  json: {
    statusCode: 404,
    body: {
      status: 'error',
      code: 'NOT_FOUND',
      message: 'User not found',
      resourceId: $json.userId
    }
  }
}];
```

### 500 - Internal Error

```javascript
return [{
  json: {
    statusCode: 500,
    body: {
      status: 'error',
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      referenceId: $execution.id  // For support tickets
    }
  }
}];
```

### Webhook Response Node Config

```
Respond With: JSON
Status Code: ={{$json.statusCode}}
Response Body: ={{JSON.stringify($json.body)}}
```

---

## Pre-Deployment Testing

### The "Try to Break It" Methodology

Test with malicious intent. If you don't break it, production will.

### Test Categories

#### 1. Empty/Missing Data

```bash
# Missing body
curl -X POST ... -H "Content-Type: application/json"

# Empty object
curl -X POST ... -d '{}'

# Null values
curl -X POST ... -d '{"email": null}'

# Empty strings
curl -X POST ... -d '{"email": ""}'
```

#### 2. Wrong Types

```bash
# String instead of number
curl -X POST ... -d '{"quantity": "five"}'

# Number instead of string
curl -X POST ... -d '{"email": 12345}'

# Object instead of array
curl -X POST ... -d '{"items": {"id": 1}}'
```

#### 3. Malformed Data

```bash
# Invalid JSON
curl -X POST ... -d 'not json'

# Incomplete JSON
curl -X POST ... -d '{"email": "test@example.com"'
```

#### 4. Boundary Tests

```bash
# Very long strings
curl -X POST ... -d '{"email": "aaaa...10000 chars...@test.com"}'

# Special characters
curl -X POST ... -d '{"name": "Test 🔥 <script>alert(1)</script>"}'
```

#### 5. Authentication Tests

```bash
# Missing API key
curl -X POST ... -d '{"email": "test@example.com"}'

# Invalid API key
curl -X POST ... -H "x-api-key: wrong" -d '{"email": "test@example.com"}'
```

### Test Database Strategy

Always test against a separate database:

```javascript
const tableName = $env.NODE_ENV === 'production'
  ? 'orders'
  : 'test_orders';
```

---

## Quick Reference

### Production Workflow Structure

```
Webhook
    ↓
Log Input → [External DB]
    ↓
Validate Input → [Invalid] → 400 Response
    ↓ [Valid]
Check Auth → [Failed] → 401 Response
    ↓ [Success]
Check Resource → [Not Found] → 404 Response
    ↓ [Found]
Process (with try-catch)
    ↓
Log Output → [External DB]
    ↓
Success Response (200)

Error Trigger → Log Error → Notify Team → 500 Response
```

### The 80/20 Rule

For production workflows:
- 20% = Core logic
- 80% = Validation + Error handling + Logging + Testing
