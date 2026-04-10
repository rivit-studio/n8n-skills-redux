# The Tier System

A deep dive into how the dynamic tier system works.

---

## Philosophy

**Not every workflow needs the same hardening.** 

A quick Slack notification for yourself doesn't need the same rigor as a payment processing system for clients. Over-engineering wastes time; under-engineering causes production failures.

The tier system right-sizes the investment.

---

## The Three Tiers

### Tier 1: Internal / Prototype

**Purpose:** Get something working fast. Prove a concept. Personal use.

**Characteristics:**
- Low stakes — failure is annoying, not costly
- Single user or small internal team
- Rapid iteration more important than robustness
- Often temporary or experimental

**What's included:**
- Basic null checks (`|| {}`, `|| ''`)
- Simple try-catch around external calls
- n8n's built-in Error Trigger → Slack/email notification

**What's skipped:**
- External logging database
- Comprehensive input validation
- HTTP status code handling
- Breaking tests

**Build time impact:** ~10% extra beyond core logic

**Examples:**
- "New GitHub issue → Slack notification"
- "Daily weather → personal email"
- "RSS feed → Discord channel"
- "Quick test of an API integration"

---

### Tier 2: Production / Client-Facing

**Purpose:** Reliable workflows for real users. Client projects. Business operations.

**Characteristics:**
- Real users depend on it
- Failures cost money, time, or trust
- Needs to be debuggable when things go wrong
- Will run for months/years

**What's included:**
- Full entry-point validation
- Explicit null vs empty string handling
- External logging (Supabase/Postgres)
- Proper HTTP status codes (400, 401, 404, 500)
- Error notifications to team
- Pre-deployment breaking tests
- Test database before production

**What's skipped:**
- Real-time monitoring dashboards
- Automated alerting/escalation
- Idempotency handling
- Rate limiting
- Rollback automation

**Build time impact:** ~80% extra beyond core logic

**Examples:**
- "Customer form → CRM + email sequence"
- "Client API integration"
- "Internal business process automation"
- "AI chatbot for client website"

---

### Tier 3: Mission-Critical / High-Volume

**Purpose:** Systems where failure has serious consequences. High throughput. Compliance requirements.

**Characteristics:**
- Financial transactions or sensitive data
- Contractual SLAs
- High request volume (10,000+/day)
- Regulatory/compliance requirements
- Significant revenue impact from downtime

**What's included:**
- Everything from Tier 2, plus:
- Real-time monitoring (Grafana, Datadog)
- Automated alerting with escalation (PagerDuty)
- Idempotency keys for duplicate handling
- Rate limiting
- Request queuing
- Rollback strategy
- Audit logging
- Chaos testing
- Runbooks

**Build time impact:** 2-3x the core logic development time

**Examples:**
- "Stripe payment → inventory + fulfillment + accounting"
- "HIPAA-compliant patient data sync"
- "High-traffic API gateway"
- "Financial reporting system"

---

## The 80/20 Rule

For Tier 2 workflows, the core logic is only 20% of the work:

| Component | Tier 1 | Tier 2 | Tier 3 |
|-----------|--------|--------|--------|
| **Core logic** | 70% | 20% | 10% |
| **Validation** | 10% | 20% | 15% |
| **Error handling** | 10% | 20% | 20% |
| **Logging** | 5% | 20% | 20% |
| **Testing** | 5% | 20% | 15% |
| **Monitoring/Ops** | — | — | 20% |

This is why "works on my machine" doesn't mean "ready for production."

---

## Tier Selection

### User-Driven Selection

After the initial prompt, ask once:

> "Quick question — what level of hardening does this workflow need?
> - **Tier 1** (Internal/Prototype)
> - **Tier 2** (Production)
> - **Tier 3** (Mission-Critical)
> 
> Or say **'autopilot'** and I'll figure it out."

If the user picks a tier, build to that spec.

### Autopilot Mode

If the user says "autopilot" or skips the question, infer from context:

| Context Clues | Tier |
|---------------|------|
| "quick", "test", "just for me", "prototype" | 1 |
| "client", "production", "deploy", "business" | 2 |
| "payment", "Stripe", "compliance", "SLA" | 3 |
| Ambiguous | Start at 1, monitor for escalation |

---

## Tier Changes

The tier isn't locked. Monitor the conversation and recommend changes when appropriate.

### Escalation Triggers (Moving Up)

**Tier 1 → Tier 2:**
- Debugging going in circles (3+ exchanges on same issue)
- "It works sometimes but not always"
- Can't see what data is coming in
- User mentions deployment or clients
- Silent failures with no error messages

**Tier 2 → Tier 3:**
- Volume concerns (1000+ requests/day)
- "What if [service] goes down?"
- Payment or compliance mentions
- SLA or uptime requirements

### De-escalation Triggers (Moving Down)

**Tier 3 → Tier 2:**
- Actually low volume after clarification
- No real compliance requirements
- User frustrated with complexity

**Tier 2 → Tier 1:**
- "This is just a test"
- "Just for internal use"
- Complexity blocking progress on core logic

### How to Recommend Changes

Be direct but not pushy:

1. State what you're observing
2. Explain why the other tier would help
3. Ask, don't mandate
4. Respect the answer

**Example (escalating):**
> "We've been debugging this for a while. I think Tier 2 logging would help us see what's actually happening. Want me to add it?"

**Example (de-escalating):**
> "This is getting complex for a test. Want me to simplify to Tier 1 and add hardening later?"

---

## Tier Checklist Summary

### Tier 1
- [ ] Basic null checks
- [ ] Try-catch around external calls
- [ ] Error Trigger → notification
- [ ] Manual happy-path test

### Tier 2
- [ ] Full input validation
- [ ] Null vs empty string handling
- [ ] External logging
- [ ] HTTP status codes
- [ ] Team notifications
- [ ] Breaking tests
- [ ] Test database
- [ ] Documentation

### Tier 3
- [ ] Everything from Tier 2
- [ ] Monitoring dashboard
- [ ] Alerting/escalation
- [ ] Idempotency
- [ ] Rate limiting
- [ ] Rollback plan
- [ ] Runbook
- [ ] Load test
- [ ] Chaos test

---

## FAQ

**Q: What if I'm not sure which tier?**
A: Start at Tier 1. It's easier to add hardening later than to strip it out. If you hit problems, escalate.

**Q: Can I mix tiers?**
A: Yes. You might have Tier 2 validation but skip external logging. The tiers are guidelines, not rigid rules.

**Q: What if the user insists on Tier 1 for something that should be Tier 2?**
A: Respect their choice but flag the concern: "That's fine for now, but you'll probably want logging before this goes to production."

**Q: How do I know when I've outgrown a tier?**
A: When you're debugging for hours without logs, or when "it works sometimes" is your status, or when a failure would cost real money — you've outgrown your current tier.
