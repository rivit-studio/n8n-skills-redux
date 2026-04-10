# Pre-Deployment Checklists

Tiered checklists for workflow deployment. Use the checklist that matches your tier.

---

## Tier 1: Internal / Prototype

**Use for:** Personal tools, internal notifications, quick tests, prototypes

**Time to complete:** 5-10 minutes

### Before Deployment

- [ ] **Basic null checks** on critical fields
  ```javascript
  const email = body.email || '';
  const items = body.items || [];
  ```

- [ ] **Try-catch** around external API calls
  ```javascript
  try {
    const response = await $http.request({...});
  } catch (error) {
    return [{json: {error: error.message}}];
  }
  ```

- [ ] **Error Trigger** workflow configured
  - Sends Slack or email on failure
  - Includes workflow name and error message

- [ ] **Manual happy-path test** completed
  - Send valid data through
  - Verify expected output

### Optional (Nice to Have)

- [ ] Error message includes execution ID for debugging
- [ ] Workflow has descriptive name and notes

---

## Tier 2: Production / Client-Facing

**Use for:** Client projects, business workflows, external integrations

**Time to complete:** 30-60 minutes

### Input Validation

- [ ] **Every input field validated**
  - Required fields checked
  - Types verified (string, number, array, object)
  - Formats validated (email, URL, date)

- [ ] **Null vs empty string explicitly handled**
  ```javascript
  if (body.email === null) {
    return [{json: {error: 'email cannot be null', statusCode: 400}}];
  }
  ```

- [ ] **Validation node** placed immediately after Webhook
  - Returns `_validation.isValid` boolean
  - Returns `_validation.errors` array

- [ ] **IF node** routes invalid requests to error response

### Error Handling

- [ ] **Proper HTTP status codes** returned
  - 400 for validation errors
  - 401 for authentication failures
  - 403 for authorization failures
  - 404 for not found
  - 500 for internal errors

- [ ] **Error responses** include:
  - Status code
  - Error code (e.g., `VALIDATION_ERROR`)
  - Human-readable message
  - Specific errors array (for validation)

- [ ] **Error Trigger** workflow configured
  - Sends notification to team channel
  - Includes execution ID for debugging

### Logging

- [ ] **External logging** configured (Supabase/Postgres)
  - Log at entry (input received)
  - Log at key decision points
  - Log at output (response sent)
  - Log errors (in Error Trigger)

- [ ] **Sensitive data excluded** from logs
  - No API keys, tokens, passwords
  - No full credit card numbers
  - No PII beyond what's necessary

### Testing

- [ ] **Empty input test** — workflow handles `{}`
- [ ] **Null values test** — workflow rejects `{field: null}`
- [ ] **Wrong types test** — workflow rejects `{quantity: "five"}`
- [ ] **Missing auth test** — returns 401
- [ ] **Invalid auth test** — returns 401/403
- [ ] **Not found test** — returns 404 for missing resources

### Environment

- [ ] **Test database** used for all testing
- [ ] **Production credentials** only used after all tests pass
- [ ] **Environment variables** used for configuration (not hardcoded)

### Documentation

- [ ] **Workflow purpose** documented in notes
- [ ] **Expected input format** documented
- [ ] **Expected output format** documented
- [ ] **Error scenarios** documented

---

## Tier 3: Mission-Critical / High-Volume

**Use for:** Payments, compliance, high-traffic, SLA requirements

**Time to complete:** 2-4 hours (plus ongoing)

### All Tier 2 Items

- [ ] Complete all Tier 2 checklist items first

### Monitoring

- [ ] **Monitoring dashboard** configured (Grafana, Datadog, etc.)
  - Execution count over time
  - Error rate
  - Response times
  - Queue depth (if applicable)

- [ ] **Alerting** configured
  - Alert on error rate spike
  - Alert on response time degradation
  - Alert on queue backup

- [ ] **Escalation path** defined
  - PagerDuty or equivalent
  - On-call rotation if needed
  - Escalation after X minutes without acknowledgment

### Reliability

- [ ] **Idempotency** implemented
  - Idempotency key in request or generated from payload
  - Duplicate requests return cached response
  - Database has unique constraint on idempotency key

  ```javascript
  const idempotencyKey = body.idempotencyKey || 
    crypto.createHash('sha256').update(JSON.stringify(body)).digest('hex');
  ```

- [ ] **Rate limiting** configured
  - Per-client rate limits
  - Global rate limits
  - Graceful handling when limits exceeded (429 response)

- [ ] **Retry logic** for downstream services
  - Exponential backoff
  - Maximum retry count
  - Circuit breaker for failing services

- [ ] **Timeout handling**
  - Reasonable timeouts on all external calls
  - Graceful handling when timeouts occur

### Data Integrity

- [ ] **Transaction handling** for multi-step operations
  - Either all steps complete or none
  - Rollback on failure

- [ ] **Audit logging** for compliance
  - Who did what, when
  - Immutable log entries
  - Retention policy defined

### Deployment

- [ ] **Rollback plan** documented
  - How to revert to previous version
  - How to identify rollback need
  - Who can authorize rollback

- [ ] **Runbook** written
  - Common failure scenarios
  - Troubleshooting steps
  - Contact information

- [ ] **Load testing** completed
  - Tested at 2-3x expected volume
  - Identified breaking points
  - Performance baseline established

- [ ] **Chaos testing** completed
  - Simulated downstream service failures
  - Simulated database unavailability
  - Verified graceful degradation

### Compliance (if applicable)

- [ ] **Data handling** reviewed
  - PII handling documented
  - Data retention policies implemented
  - Data deletion capabilities exist

- [ ] **Security review** completed
  - Authentication mechanisms reviewed
  - Authorization logic reviewed
  - Input sanitization verified

---

## Quick Decision Guide

| Scenario | Tier |
|----------|------|
| "Just testing an idea" | 1 |
| "Internal tool for the team" | 1 or 2 |
| "Client project" | 2 |
| "Handles user data" | 2 |
| "Will be deployed for months" | 2 |
| "Handles payments" | 3 |
| "Compliance requirements" | 3 |
| "High volume (10k+/day)" | 3 |
| "Has SLA" | 3 |

---

## Checklist Template

Copy and paste for your workflow:

```markdown
## [Workflow Name] - Tier [X] Checklist

### Validation
- [ ] Required fields checked
- [ ] Types verified
- [ ] Null handling explicit

### Error Handling
- [ ] Status codes correct
- [ ] Error responses formatted
- [ ] Team notifications configured

### Logging
- [ ] External logging configured
- [ ] Sensitive data excluded

### Testing
- [ ] Empty input: ✓/✗
- [ ] Null values: ✓/✗
- [ ] Wrong types: ✓/✗
- [ ] Missing auth: ✓/✗

### Deployment
- [ ] Test database used
- [ ] Documentation complete
- [ ] Ready for production
```
