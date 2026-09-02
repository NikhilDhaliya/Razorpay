# Revenue Recovery Platform --- Production-Ready MVP Task Specification

## 0. Objective

Build a production-shaped, industry-standard MVP for an AI Revenue
Recovery platform.

The platform enables a merchant to connect their Razorpay account,
continuously ingest payment/revenue events, identify revenue at risk,
diagnose why revenue is at risk, determine a bounded recovery action,
execute that action, observe the outcome, and maintain a complete audit
trail.

### Core product loop

``` text
Detect → Understand → Decide → Act → Observe → Recover / Escalate / Stop
```

### Core architectural principle

AI is a decision-support component, not an unrestricted financial actor.

``` text
Razorpay Event
    ↓
Webhook Ingestion
    ↓
Verify + Validate + Deduplicate
    ↓
Persist
    ↓
Queue
    ↓
Risk Detection
    ↓
AI Diagnosis
    ↓
Deterministic Policy / Safety Gate
    ↓
Action Executor
    ↓
Razorpay
    ↓
Outcome Event
    ↓
Recovery Case + Audit Trail + Analytics
```

------------------------------------------------------------------------

# 1. Product Scope

## 1.1 MVP must demonstrate

A merchant can:

1.  Create an account.
2.  Sign in securely.
3.  Enter the dashboard.
4.  Connect a Razorpay account.
5.  See connection status persistently.
6.  Receive Razorpay payment events.
7.  Detect revenue at risk.
8.  Create a recovery case.
9.  Get an AI diagnosis.
10. Receive a recommended recovery action.
11. Pass that recommendation through deterministic policy checks.
12. Execute an allowed action.
13. Observe the resulting payment/event.
14. Continue, escalate, or stop based on policy.
15. See the complete case timeline.
16. See recovered/lost revenue and recovery metrics.
17. Run a simulation/demo batch without depending entirely on live
    payment traffic.

## 1.2 Explicit non-goals for the first MVP

Do not build:

-   Microservices.
-   A generic autonomous agent framework.
-   LangGraph unless workflow complexity genuinely requires it.
-   Arbitrary LLM-generated API calls.
-   Direct database access from the browser.
-   Merchant credential collection through frontend forms.
-   Unbounded automatic retries.
-   Production payment processing infrastructure unrelated to recovery.
-   A large multi-tenant enterprise permission system before core
    recovery works.

------------------------------------------------------------------------

# 2. Current Technology Stack

## Web

-   Next.js
-   TypeScript
-   App Router
-   Tailwind CSS
-   shadcn/ui
-   Better Auth
-   React client/server components

## API

-   Node.js
-   TypeScript
-   Express
-   Zod
-   jose
-   Prisma
-   PostgreSQL
-   Redis
-   BullMQ
-   OpenAI
-   Razorpay APIs / SDK

## Infrastructure

-   PostgreSQL: Supabase
-   Redis: production-managed Redis provider
-   Object storage if needed: Cloudflare R2
-   API deployment: production Node.js-compatible platform
-   Web deployment: Next.js-compatible platform
-   HTTPS required
-   Public webhook endpoint required

Prisma 7 uses driver adapters for database connectivity; keep the
existing PostgreSQL + `@prisma/adapter-pg` architecture. Prisma
CLI/database configuration should continue using the direct database URL
for migrations where appropriate.

------------------------------------------------------------------------

# 3. Repository Architecture

Two independent repositories remain mandatory.

``` text
Razorpay/
├── api/   # independent Git repository
└── web/   # independent Git repository
```

Do not convert this into a monorepo.

## API architecture

Use feature/domain-oriented modules:

``` text
api/src/
├── config/
├── Error/
├── middleware/
├── lib/
├── features/
│   ├── merchant/
│   ├── razorpay/
│   ├── webhook/
│   ├── recovery/
│   ├── risk/
│   ├── ai/
│   ├── policy/
│   ├── actions/
│   └── analytics/
└── index.ts
```

Each feature should contain only what it owns:

``` text
feature/
├── controller
├── routes
├── service
├── schemas
├── types
└── repository        # only where repository abstraction adds value
```

Do not create abstractions without a real use case.

## Web architecture

Use Next.js route-oriented architecture:

``` text
web/src/
├── app/
│   ├── api/
│   │   └── auth/
│   ├── login/
│   └── dashboard/
├── components/
│   ├── ui/
│   ├── dashboard/
│   ├── merchant/
│   ├── razorpay/
│   ├── recovery/
│   └── analytics/
├── lib/
│   ├── api/
│   ├── auth.ts
│   ├── auth-client.ts
│   ├── auth-session.ts
│   └── prisma.ts
└── types/
```

Do not create a generic `features/` directory on the web side unless the
application later grows enough to justify it.

------------------------------------------------------------------------

# 4. Authentication

## 4.1 Better Auth

Keep Better Auth as the authentication provider.

Requirements:

-   Email/password authentication for MVP.
-   Secure session management.
-   Server-side session checks for protected routes.
-   JWT plugin for Web → API authentication.
-   API validates JWT using Better Auth JWKS.
-   Never trust user IDs supplied by the browser.

Current flow:

``` text
Browser
  ↓
Better Auth
  ↓
Authenticated session
  ↓
JWT
  ↓
Express API
  ↓
jose JWT verification
```

## 4.2 Authorization

Every protected API endpoint must derive the authenticated user from the
verified JWT.

Never accept:

``` json
{
  "userId": "..."
}
```

as an authorization mechanism.

The authenticated identity must come from:

``` text
verified JWT → req.user.id
```

------------------------------------------------------------------------

# 5. Merchant Model

The first real domain entity is Merchant.

Relationship:

``` text
User
  │
  └── Merchant
        │
        └── RazorpayConnection
```

A user may eventually manage multiple merchants, but MVP can initially
enforce one merchant per user if that simplifies onboarding.

## Merchant fields

Recommended:

-   `id`
-   `ownerUserId`
-   `name`
-   `status`
-   `createdAt`
-   `updatedAt`

Use enums where appropriate.

Example statuses:

``` text
ACTIVE
SUSPENDED
ONBOARDING
```

Constraints:

-   Unique ownership relationship as defined by product requirements.
-   Foreign keys.
-   Indexed owner lookup.

------------------------------------------------------------------------

# 6. Razorpay Connection

## 6.1 Production architecture

Do NOT ask merchants to paste their Razorpay API Key ID and Secret into
our product.

For a platform accessing merchant/sub-merchant Razorpay accounts,
Razorpay documents OAuth as the required Technology Partner integration.
OAuth allows the merchant to authorize access without exposing their API
secret. citeturn0search6turn0search3

Production flow:

``` text
Merchant
   ↓
Connect Razorpay
   ↓
Razorpay OAuth authorization
   ↓
Merchant approves
   ↓
Razorpay redirects to our callback
   ↓
Backend exchanges authorization code
   ↓
Access token + refresh token
   ↓
Encrypted persistence
   ↓
RazorpayConnection
```

Razorpay's Technology Partner OAuth flow requires creating an
application through the Partner Dashboard and has separate
development/production clients. citeturn0search3turn0search7

## 6.2 Development fallback

Until Technology Partner OAuth access is available:

-   Keep Test Mode credentials in backend environment variables.
-   Never expose the secret to the browser.
-   Never store the secret in Git.
-   Never ask the merchant to provide it through the UI.
-   Clearly mark the current implementation as development/test
    infrastructure.

The existing `/api/razorpay/connection` endpoint is only a
credential/API connectivity check and is not the final merchant
connection architecture.

## 6.3 RazorpayConnection fields

Production-shaped model:

-   `id`
-   `merchantId`
-   `razorpayAccountId`
-   `status`
-   `accessTokenEncrypted`
-   `refreshTokenEncrypted` if supplied by the OAuth flow
-   `accessTokenExpiresAt`
-   `connectedAt`
-   `revokedAt`
-   `createdAt`
-   `updatedAt`

Never store OAuth secrets/tokens as plaintext.

Use application-level encryption with a key stored outside the database.

## 6.4 Connection statuses

``` text
PENDING
CONNECTED
EXPIRED
REVOKED
ERROR
```

------------------------------------------------------------------------

# 7. Razorpay OAuth Implementation

Implement:

``` text
GET /api/razorpay/connect
GET /api/razorpay/callback
POST /api/razorpay/disconnect
GET /api/razorpay/connection
```

## Connect

1.  Authenticate user.
2.  Resolve merchant.
3.  Generate OAuth authorization URL.
4.  Include a cryptographically secure state value.
5.  Persist state with expiry.
6.  Redirect user to Razorpay.

## Callback

1.  Validate OAuth state.
2.  Reject expired/invalid state.
3.  Exchange authorization code server-side.
4.  Obtain merchant account identity.
5.  Encrypt credentials/tokens.
6.  Upsert RazorpayConnection.
7.  Mark connection `CONNECTED`.
8.  Redirect to dashboard.

## Disconnect

1.  Authenticate user.
2.  Resolve merchant.
3.  Revoke/disconnect according to Razorpay's supported flow.
4.  Mark connection `REVOKED`.
5.  Stop recovery actions for that merchant.

## Security

-   Never accept OAuth client secrets from the browser.
-   Never log access tokens.
-   Never log client secrets.
-   Never put tokens in URLs except where the OAuth protocol explicitly
    requires them.
-   Validate `state`.
-   Enforce callback HTTPS in production.
-   Rotate encryption keys through a controlled key-management process.

------------------------------------------------------------------------

# 8. Razorpay API Client

Create one server-side Razorpay integration layer.

Responsibilities:

-   Authentication.
-   Merchant account context.
-   API request handling.
-   Typed response normalization.
-   Error normalization.
-   Rate-limit handling.
-   Request correlation/logging without secrets.
-   Retry only for safe/transient failures.

Do not scatter Razorpay SDK calls throughout controllers.

Preferred:

``` text
Controller
   ↓
Domain Service
   ↓
Razorpay Service
   ↓
Razorpay SDK/API
```

------------------------------------------------------------------------

# 9. Webhook Architecture

Webhooks are the primary event ingestion mechanism.

Razorpay describes webhooks as asynchronous, near-real-time
notifications and recommends webhook-driven automation.
citeturn0search15

## Endpoint

Example:

``` text
POST /api/webhooks/razorpay
```

This endpoint must be public over HTTPS in production.

## Critical requirement: raw body

Razorpay signature validation requires the original raw webhook request
body. Do not parse or cast the body before signature verification.
citeturn0search2

Express architecture must therefore preserve the raw body for this
route.

Do not blindly use global:

``` ts
app.use(express.json())
```

before the webhook route if that destroys access to the raw payload
required for signature verification.

Use route-specific raw-body handling or an equivalent verified raw-body
strategy.

------------------------------------------------------------------------

# 10. Webhook Security Pipeline

For every webhook:

``` text
Incoming Request
      ↓
Capture raw body
      ↓
Read signature header
      ↓
Verify Razorpay signature
      ↓
Read event ID
      ↓
Validate payload
      ↓
Persist raw event
      ↓
Return 2xx
      ↓
Queue processing
```

Requirements:

-   Verify signature.
-   Validate schema.
-   Extract `x-razorpay-event-id`.
-   Deduplicate.
-   Persist raw event before asynchronous processing.
-   Return success quickly after durable acceptance.
-   Never run an LLM synchronously inside webhook ingestion.

Razorpay uses at-least-once delivery, so duplicate events are expected;
`x-razorpay-event-id` should be used for deduplication. Events may also
arrive out of order. citeturn0search0turn0search2

------------------------------------------------------------------------

# 11. Webhook Event Model

Create an immutable event record.

Recommended fields:

-   `id`
-   `merchantId`
-   `provider`
-   `providerEventId`
-   `eventType`
-   `payload`
-   `signatureVerified`
-   `receivedAt`
-   `processedAt`
-   `processingStatus`
-   `processingError`
-   `createdAt`

Constraints:

``` text
UNIQUE(provider, providerEventId)
```

Processing statuses:

``` text
RECEIVED
QUEUED
PROCESSING
PROCESSED
FAILED
IGNORED
```

Never mutate the original raw payload.

------------------------------------------------------------------------

# 12. Supported MVP Events

Start with:

``` text
payment.failed
payment.authorized
payment.captured
```

Add subscription/mandate/invoice events only after payment recovery
works reliably.

Razorpay's payment webhook documentation includes payment failure,
authorization, and capture events.

------------------------------------------------------------------------

# 13. Queue Architecture

Use Redis + BullMQ.

Webhook handler:

``` text
Webhook
  ↓
Verify
  ↓
Persist
  ↓
Queue
  ↓
Return 2xx
```

Worker:

``` text
BullMQ Worker
  ↓
Load Event
  ↓
Determine Domain Effect
  ↓
Create/Update Recovery Case
  ↓
Risk Assessment
```

## Queue requirements

-   Durable jobs.
-   Retries with backoff.
-   Dead-letter handling.
-   Job idempotency.
-   Concurrency limits.
-   Graceful shutdown.
-   Structured logs.
-   No unbounded retry loops.

------------------------------------------------------------------------

# 14. Revenue-at-Risk Detection

Create a normalized risk model.

Initial risk sources:

``` text
PAYMENT_FAILED
CHECKOUT_ABANDONED
SUBSCRIPTION_PAYMENT_FAILED
MANDATE_FAILED
INVOICE_OVERDUE
```

MVP implementation priority:

1.  Payment failed.
2.  Checkout abandoned.
3.  Subscription payment failed.
4.  Others.

------------------------------------------------------------------------

# 15. Payment Risk Detection

When `payment.failed` arrives:

1.  Identify merchant.
2.  Identify payment.
3.  Extract amount/currency.
4.  Extract method information.
5.  Extract failure information.
6.  Look up related payment/customer/order context where available.
7.  Determine whether revenue is recoverable.
8.  Create RecoveryCase.
9.  Queue diagnosis.

Do not automatically retry every failure.

------------------------------------------------------------------------

# 16. Recovery Case

RecoveryCase is the central domain object.

Recommended fields:

-   `id`
-   `merchantId`
-   `sourceEventId`
-   `paymentId`
-   `customerReference`
-   `amount`
-   `currency`
-   `status`
-   `riskLevel`
-   `failureReason`
-   `currentStage`
-   `nextActionAt`
-   `resolvedAt`
-   `createdAt`
-   `updatedAt`

Statuses:

``` text
OPEN
DIAGNOSING
ACTION_PENDING
ACTION_EXECUTING
RECOVERED
ESCALATED
LOST
STOPPED
```

------------------------------------------------------------------------

# 17. Recovery Case Timeline

Every case must have an immutable timeline.

Example:

``` text
Payment failed
      ↓
Risk detected
      ↓
Context assembled
      ↓
AI diagnosis generated
      ↓
Recovery action recommended
      ↓
Policy approved
      ↓
Retry scheduled
      ↓
Retry executed
      ↓
Payment captured
      ↓
Revenue recovered
```

Timeline events should include:

-   event type
-   actor
-   reason
-   timestamp
-   metadata
-   correlation ID

Actors:

``` text
SYSTEM
AI
POLICY_ENGINE
MERCHANT
CUSTOMER
RAZORPAY
```

------------------------------------------------------------------------

# 18. AI Diagnosis Engine

AI receives a bounded context object.

Example:

``` json
{
  "payment": {
    "amount": 99900,
    "currency": "INR",
    "method": "card",
    "status": "failed"
  },
  "failure": {
    "reason": "..."
  },
  "customer": {
    "history": "..."
  },
  "previousAttempts": 1
}
```

AI must return structured output.

Example:

``` json
{
  "classification": "TEMPORARY_FAILURE",
  "confidence": 0.91,
  "reason": "Likely transient payment failure",
  "recommendedAction": "RETRY_PAYMENT",
  "recommendedDelayMinutes": 30
}
```

Use Zod to validate the model output.

Reject malformed/unknown outputs.

------------------------------------------------------------------------

# 19. AI Safety Boundary

The LLM must NEVER directly:

-   call Razorpay APIs;
-   choose arbitrary endpoints;
-   determine unlimited retry counts;
-   bypass policy;
-   alter financial amounts;
-   issue refunds;
-   modify merchant credentials;
-   send unrestricted customer messages.

Instead:

``` text
LLM
 ↓
Structured recommendation
 ↓
Zod validation
 ↓
Policy engine
 ↓
Allowed action
 ↓
Executor
```

------------------------------------------------------------------------

# 20. Policy Engine

The policy engine is deterministic.

Inputs:

-   AI recommendation.
-   Merchant configuration.
-   Recovery case.
-   Attempt count.
-   Amount.
-   Failure reason.
-   Previous actions.
-   Time since failure.
-   Customer/contact constraints.

Outputs:

``` text
ALLOW
DENY
ESCALATE
STOP
```

Example policy:

``` text
IF payment failed
AND failure appears transient
AND retry_count < max_retries
AND amount <= max_auto_recovery_amount
THEN allow retry
```

------------------------------------------------------------------------

# 21. Recovery Controls

Merchant-configurable controls:

-   Maximum retry attempts.
-   Maximum automatic recovery amount.
-   Minimum retry delay.
-   Allowed recovery actions.
-   Notification preferences.
-   Escalation threshold.
-   Stop-after duration.

Default conservative policy:

``` text
maxRetries = 2
maxAutomaticRecoveryAmount = merchant-defined
automaticRefunds = false
automaticCustomerMessaging = false
```

Do not optimize for maximum automation. Optimize for **safe measurable
recovery**.

------------------------------------------------------------------------

# 22. Action System

Create a normalized action model.

Action types:

``` text
RETRY_PAYMENT
SCHEDULE_RETRY
SEND_PAYMENT_LINK
SEND_NOTIFICATION
ESCALATE_TO_MERCHANT
STOP_RECOVERY
```

Each action has:

-   ID
-   recovery case ID
-   type
-   status
-   requestedBy
-   approvedByPolicy
-   parameters
-   idempotency key
-   execution time
-   result
-   error
-   createdAt
-   completedAt

------------------------------------------------------------------------

# 23. Action Executor

The executor is the only layer allowed to perform external recovery
actions.

``` text
Policy Engine
    ↓
Action
    ↓
Action Executor
    ↓
Razorpay Service
```

Requirements:

-   Idempotency.
-   Timeout.
-   Retry only where safe.
-   Provider error normalization.
-   Audit logging.
-   No direct LLM access.

------------------------------------------------------------------------

# 24. Retry Sequencer

Build deterministic retry scheduling.

Example:

``` text
Failure
  ↓
Attempt 1
  ↓
Wait
  ↓
Observe
  ↓
Still failed?
  ↓
AI reassessment
  ↓
Policy
  ↓
Attempt 2
  ↓
Observe
  ↓
Recovered / Escalate / Stop
```

Do not blindly execute:

``` text
retry forever
```

Every case must have a stopping condition.

------------------------------------------------------------------------

# 25. Stopping Rules

A case must stop when:

-   Payment succeeds.
-   Maximum attempts reached.
-   Merchant policy blocks further action.
-   Payment is no longer recoverable.
-   Customer/merchant escalation is required.
-   Recovery window expires.
-   Provider indicates permanent failure.
-   Fraud/risk signals require stopping.

------------------------------------------------------------------------

# 26. Idempotency

Idempotency is mandatory for financial actions.

Generate stable idempotency keys such as:

``` text
recovery:{caseId}:action:{actionId}
```

Before executing an action:

1.  Check existing action status.
2.  If already completed, return previous result.
3.  If already executing, do not duplicate.
4.  If failed and retryable, follow retry policy.
5.  If permanently failed, stop/escalate.

------------------------------------------------------------------------

# 27. Analytics

Track at minimum:

## Revenue metrics

-   Revenue at risk.
-   Revenue recovered.
-   Revenue lost.
-   Recovery rate.
-   Recovery amount by day.

## Operational metrics

-   Failed payments.
-   Active recovery cases.
-   Actions executed.
-   Retry success rate.
-   Average time to recovery.
-   Cases escalated.
-   Cases stopped.

## AI metrics

-   Diagnoses generated.
-   Recommendations accepted.
-   Recommendations rejected.
-   Policy rejection rate.
-   Recovery rate by recommendation.
-   AI confidence distribution.

------------------------------------------------------------------------

# 28. Dashboard

Build a professional merchant dashboard.

## Overview

Display:

``` text
Revenue at Risk
₹X

Recovered
₹Y

Recovery Rate
Z%

Active Cases
N
```

Charts:

-   Revenue at risk over time.
-   Revenue recovered over time.
-   Recovery rate.
-   Failure categories.
-   Recovery outcomes.

## Recent activity

Show:

-   Payment failed.
-   Case created.
-   AI diagnosis.
-   Action approved.
-   Action executed.
-   Revenue recovered.

------------------------------------------------------------------------

# 29. Recovery Cases UI

Create a case list.

Columns:

-   Customer/payment reference.
-   Amount.
-   Failure reason.
-   Risk.
-   Current action.
-   Status.
-   Time since failure.

Case detail:

``` text
Summary
Risk
AI Diagnosis
Recommended Action
Policy Decision
Action History
Payment Timeline
Audit Trail
```

The case timeline should be the primary proof that the system is
actually agentic and controlled.

------------------------------------------------------------------------

# 30. Razorpay Connection UI

Production UI should contain:

``` text
Razorpay
Connected

Account
acc_...

Status
Active

Connected
Sep 3, 2026
```

Actions:

``` text
Reconnect
Disconnect
```

During OAuth:

``` text
Connect Razorpay
    ↓
Redirect to Razorpay
    ↓
Authorize
    ↓
Return to platform
    ↓
Connected
```

Do not show API secrets.

------------------------------------------------------------------------

# 31. Simulation Mode

A simulation environment is mandatory for the hackathon demo.

Allow a merchant/admin to generate a batch such as:

``` text
100 payment attempts
20 failed
12 recoverable
8 permanently lost
```

Then run:

``` text
Detection
 ↓
Diagnosis
 ↓
Policy
 ↓
Recovery
```

Display:

``` text
₹50,000 at risk
₹31,000 recovered
62% recovery rate
```

Simulation must use the same domain services and state machine as
production wherever possible.

Do not create a completely separate fake implementation.

------------------------------------------------------------------------

# 32. Demo Scenarios

Prepare at least five deterministic scenarios.

## Scenario 1 --- Temporary card failure

``` text
Payment failed
→ AI identifies likely temporary failure
→ Retry allowed
→ Retry succeeds
→ Recovered
```

## Scenario 2 --- Retry fails

``` text
Payment failed
→ Retry
→ Retry fails
→ AI reassesses
→ Policy denies further retries
→ Escalated
```

## Scenario 3 --- High-value payment

``` text
Large amount
→ AI recommends recovery
→ Policy blocks automatic action
→ Merchant escalation
```

## Scenario 4 --- Permanent failure

``` text
Payment failed
→ AI identifies permanent failure
→ No retry
→ Stop
```

## Scenario 5 --- Duplicate webhook

``` text
Same Razorpay event delivered twice
→ First accepted
→ Second deduplicated
→ One recovery case
```

------------------------------------------------------------------------

# 33. Observability

Implement structured logging.

Every important operation should include:

``` text
requestId
merchantId
userId
recoveryCaseId
actionId
providerEventId
```

Never log:

-   API secrets.
-   OAuth client secrets.
-   Access tokens.
-   Refresh tokens.
-   Passwords.
-   Full sensitive payment/customer payloads unless explicitly safe.

------------------------------------------------------------------------

# 34. Error Handling

API errors must be normalized.

Categories:

``` text
VALIDATION_ERROR
AUTHENTICATION_ERROR
AUTHORIZATION_ERROR
NOT_FOUND
CONFLICT
PROVIDER_ERROR
RATE_LIMITED
INTERNAL_ERROR
```

Response format should be consistent.

Example:

``` json
{
  "error": {
    "code": "PROVIDER_ERROR",
    "message": "Unable to execute recovery action"
  }
}
```

Do not expose stack traces in production.

------------------------------------------------------------------------

# 35. API Standards

Use:

-   RESTful routes.
-   Request validation with Zod.
-   Response types.
-   Authentication middleware.
-   Authorization checks.
-   Consistent error responses.
-   Pagination for lists.
-   Filtering/sorting where needed.
-   Request IDs.
-   Proper HTTP status codes.

Example:

``` text
GET    /api/merchant
PATCH  /api/merchant

GET    /api/razorpay/connection
GET    /api/razorpay/connect
GET    /api/razorpay/callback
POST   /api/razorpay/disconnect

POST   /api/webhooks/razorpay

GET    /api/recovery/cases
GET    /api/recovery/cases/:id
POST   /api/recovery/cases/:id/actions/:actionId/execute

GET    /api/analytics/overview
GET    /api/analytics/recovery
```

------------------------------------------------------------------------

# 36. Database Design

Core models:

``` text
User
Merchant
RazorpayConnection

WebhookEvent

Payment
Customer
RecoveryCase
RecoveryCaseEvent

AIAnalysis
RecoveryAction
ActionExecution

MerchantPolicy

AuditLog
```

Do not duplicate provider payload data unnecessarily.

Store normalized fields needed for querying plus immutable raw events
where appropriate.

------------------------------------------------------------------------

# 37. Database Indexing

At minimum index:

``` text
Merchant.ownerUserId

RazorpayConnection.merchantId
RazorpayConnection.razorpayAccountId

WebhookEvent.merchantId
WebhookEvent.providerEventId
WebhookEvent.eventType

RecoveryCase.merchantId
RecoveryCase.status
RecoveryCase.createdAt
RecoveryCase.nextActionAt

RecoveryAction.recoveryCaseId
RecoveryAction.status
RecoveryAction.idempotencyKey

AuditLog.merchantId
AuditLog.recoveryCaseId
AuditLog.createdAt
```

Use unique constraints for:

-   Provider event IDs.
-   Action idempotency keys.
-   Merchant/provider account relationships where appropriate.

------------------------------------------------------------------------

# 38. Transactions

Use database transactions when multiple domain records must change
atomically.

Example:

``` text
Webhook Event
+
Recovery Case
+
Initial Case Event
```

should be committed consistently.

Do not hold long database transactions while calling Razorpay or OpenAI.

Correct:

``` text
DB transaction
  ↓
Commit
  ↓
External API call
```

or use an outbox pattern when reliable event publication is required.

------------------------------------------------------------------------

# 39. Outbox Pattern

For production reliability, introduce an outbox when cross-system
consistency becomes necessary.

Example:

``` text
DB Transaction
 ├── business state
 └── outbox event
          ↓
      Worker
          ↓
      External system
```

This prevents losing a queue/event publication after a successful
database transaction.

For the hackathon MVP, a durable webhook-event record + BullMQ job may
be sufficient, but the design should leave room for an outbox.

------------------------------------------------------------------------

# 40. Security

Mandatory:

-   HTTPS in production.
-   Secure cookies.
-   CSRF protection where applicable.
-   Strict CORS.
-   Rate limiting.
-   Input validation.
-   Output validation.
-   Secret management.
-   Encryption for stored OAuth tokens.
-   No secrets in logs.
-   No secrets in Git.
-   Least-privilege provider access.
-   Authorization on every merchant resource.
-   Webhook signature verification.
-   OAuth state verification.
-   Security headers.

------------------------------------------------------------------------

# 41. CORS

Development:

``` text
Web: http://localhost:3000
API: http://localhost:8080
```

Production:

``` text
https://app.example.com
https://api.example.com
```

Only allow known frontend origins.

Never use unrestricted production:

``` text
Access-Control-Allow-Origin: *
```

for authenticated application APIs.

------------------------------------------------------------------------

# 42. Rate Limiting

Protect:

-   Authentication routes.
-   OAuth routes.
-   Webhook endpoint where appropriate without interfering with provider
    retries.
-   Recovery action endpoints.
-   Simulation endpoints.
-   AI endpoints.

Use Redis-backed rate limiting when deployed across multiple API
instances.

------------------------------------------------------------------------

# 43. AI Reliability

Implement:

-   Model timeout.
-   Retry policy.
-   Structured output validation.
-   Maximum token/output constraints.
-   Fallback behavior.
-   Model version tracking.
-   Prompt version tracking.
-   AI decision audit.

Persist:

``` text
model
promptVersion
inputVersion
output
confidence
createdAt
```

Never make recovery depend on an unvalidated free-form response.

------------------------------------------------------------------------

# 44. AI Cost Controls

Avoid calling the model for every event.

First apply deterministic filtering:

``` text
Is revenue actually at risk?
Is this failure potentially recoverable?
Has this case already been diagnosed?
Is the case already stopped/recovered?
```

Only then invoke AI.

Cache/reuse diagnosis where valid.

------------------------------------------------------------------------

# 45. AI Decision Explainability

Every AI recommendation must have:

``` text
Classification
Confidence
Reason
Recommended action
Relevant context
Policy result
```

UI should show:

> AI recommended retry because the failure appears temporary and this
> payment has only had one previous attempt.

Then:

> Policy approved: maximum retries not exceeded.

This is more credible than displaying a generic "AI decided" label.

------------------------------------------------------------------------

# 46. Merchant Policies

Default policies must be deterministic and versioned.

Store:

``` text
policyVersion
maxRetryAttempts
maxAutoRecoveryAmount
allowedActions
retryDelays
escalationRules
stopRules
```

When a policy changes, existing cases should retain the policy version
under which their decisions were made.

------------------------------------------------------------------------

# 47. Audit Log

Audit all sensitive operations:

``` text
Merchant connected
OAuth completed
OAuth revoked
Webhook received
Recovery case created
AI diagnosis generated
Policy approved
Policy rejected
Action executed
Action failed
Case recovered
Case escalated
Case stopped
Merchant policy changed
```

Audit records should be append-only.

------------------------------------------------------------------------

# 48. Testing Strategy

## Unit tests

Test:

-   Policy engine.
-   Retry logic.
-   Stopping rules.
-   Risk classification.
-   Idempotency.
-   Webhook signature verification.
-   Webhook deduplication.
-   AI output schema.
-   Error mapping.

## Integration tests

Test:

``` text
Webhook
→ DB
→ Queue
→ Risk
→ AI
→ Policy
→ Action
```

## End-to-end tests

Test:

``` text
Login
→ Dashboard
→ Connect
→ Event
→ Recovery case
→ Recovery action
→ Recovered
```

------------------------------------------------------------------------

# 49. Webhook Tests

Mandatory cases:

1.  Valid signature.
2.  Invalid signature.
3.  Missing signature.
4.  Duplicate event.
5.  Out-of-order event.
6.  Malformed payload.
7.  Processing failure.
8.  Queue failure.
9.  Retry after provider redelivery.

Razorpay explicitly warns that duplicate and out-of-order webhook
delivery can occur. citeturn0search0turn0search2

------------------------------------------------------------------------

# 50. Recovery Tests

At minimum:

``` text
Temporary failure → retry → success

Temporary failure → retry → failure → stop

Permanent failure → no retry

High amount → escalation

Duplicate failure event → one case

Already recovered payment → no further action

Maximum attempts → stop
```

------------------------------------------------------------------------

# 51. Deployment Architecture

Production:

``` text
                    ┌───────────────┐
                    │    Browser    │
                    └───────┬───────┘
                            │
                         HTTPS
                            │
                            ▼
                    ┌───────────────┐
                    │   Next.js     │
                    │      Web      │
                    └───────┬───────┘
                            │
                         HTTPS
                            │
                            ▼
                    ┌───────────────┐
                    │    Express    │
                    │      API      │
                    └───┬─────┬─────┘
                        │     │
             ┌──────────┘     └──────────┐
             ▼                           ▼
       PostgreSQL                      Redis
             │                           │
             │                        BullMQ
             │                           │
             └──────────┬────────────────┘
                        ▼
                 Recovery Workers
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
         Razorpay               OpenAI
```

------------------------------------------------------------------------

# 52. Production Environment Variables

## API

``` env
NODE_ENV=production
PORT=8080

DATABASE_URL=
DIRECT_URL=

BETTER_AUTH_URL=

REDIS_URL=

RAZORPAY_CLIENT_ID=
RAZORPAY_CLIENT_SECRET=
RAZORPAY_WEBHOOK_SECRET=

OPENAI_API_KEY=

ENCRYPTION_KEY=

CORS_ORIGIN=
```

Development Test Mode credentials may additionally exist where required:

``` env
RAZORPAY_KEY_ID=
RAZORPAY_KEY_SECRET=
```

Never commit `.env`.

Maintain:

``` text
.env.example
```

with empty placeholders.

------------------------------------------------------------------------

# 53. Web Environment Variables

``` env
NEXT_PUBLIC_API_URL=
BETTER_AUTH_URL=
```

Only expose variables prefixed with `NEXT_PUBLIC_` when they are
genuinely safe for the browser.

Never expose:

-   Razorpay secret.
-   OAuth client secret.
-   OpenAI API key.
-   Database credentials.
-   Encryption key.

------------------------------------------------------------------------

# 54. Health Checks

API:

``` text
GET /health
```

Should return service health.

Add deeper readiness checks if required:

``` text
GET /health/live
GET /health/ready
```

Readiness may verify:

-   Database.
-   Redis.
-   Required configuration.

Do not expose secrets or sensitive dependency details.

------------------------------------------------------------------------

# 55. Graceful Shutdown

API must gracefully stop:

1.  Stop accepting new requests.
2.  Stop queue consumers.
3.  Finish or safely requeue active jobs.
4.  Close Redis.
5.  Close PostgreSQL.
6.  Exit.

Handle:

``` text
SIGTERM
SIGINT
```

------------------------------------------------------------------------

# 56. Background Worker Separation

Keep the API and worker logically separated even if they remain in the
same repository.

Example:

``` text
api/src/
├── index.ts
└── worker.ts
```

API process:

``` text
HTTP
```

Worker process:

``` text
BullMQ consumers
```

Do not run long-running queue workers inside request handlers.

------------------------------------------------------------------------

# 57. Performance

Targets for MVP:

-   Webhook acknowledgement: fast 2xx after durable persistence.
-   No LLM call during webhook HTTP request.
-   API endpoints paginated.
-   Database queries indexed.
-   No N+1 queries.
-   Dashboard metrics aggregated efficiently.
-   Background work processed asynchronously.

Razorpay considers webhook delivery unsuccessful when it receives a
non-2xx response or timeout; their documentation describes retry
behavior and a five-second response expectation for webhook consumption.
citeturn0search12turn0search0

------------------------------------------------------------------------

# 58. Production Webhook Configuration

Configure Razorpay webhooks for the required event set.

Production webhook URL:

``` text
https://api.example.com/api/webhooks/razorpay
```

Requirements:

-   HTTPS.
-   Secret configured.
-   Signature validation.
-   Event ID deduplication.
-   Monitoring.
-   Alerting.

Razorpay's documentation states production webhook URLs should use
supported HTTPS infrastructure and that webhook secrets should not be
exposed. citeturn0search15turn0search12

------------------------------------------------------------------------

# 59. Local Webhook Development

Localhost cannot directly receive Razorpay's production webhook
delivery.

Use a secure public tunnel during development.

Example:

``` text
Razorpay
   ↓
Public HTTPS tunnel
   ↓
localhost:8080
```

Never disable signature verification just because the environment is
local.

Use Razorpay Test Mode for testing.

------------------------------------------------------------------------

# 60. Merchant Onboarding UX

Final onboarding:

``` text
Sign up
  ↓
Create business profile
  ↓
Connect Razorpay
  ↓
Authorize on Razorpay
  ↓
Connection verified
  ↓
Enable monitoring
  ↓
Dashboard
```

Show clear status at every step.

Do not claim monitoring is active until the connection and required
webhook configuration are actually valid.

------------------------------------------------------------------------

# 61. Dashboard Information Architecture

``` text
Dashboard
├── Overview
├── Revenue at Risk
├── Recovery Cases
├── Activity
├── Analytics
├── Razorpay
└── Settings
```

Settings:

``` text
Merchant
Recovery Policies
Notifications
Razorpay Connection
Security
```

------------------------------------------------------------------------

# 62. Visual Design

Use a professional B2B SaaS visual system.

Principles:

-   Dark application shell.
-   High information density.
-   Clear typography.
-   Strong hierarchy.
-   Minimal decorative effects.
-   Consistent spacing.
-   Clear status badges.
-   Accessible contrast.
-   Responsive layout.
-   Loading/skeleton states.
-   Empty states.
-   Error states.

Do not build a marketing landing page as the primary MVP.

The product should look like a serious financial operations console.

------------------------------------------------------------------------

# 63. Empty States

Every major page needs an intentional empty state.

Example:

``` text
No recovery cases yet.

Once a payment becomes recoverable,
we'll automatically create a recovery case here.
```

Include a useful next action where appropriate.

------------------------------------------------------------------------

# 64. Failure States

Handle:

-   Razorpay disconnected.
-   OAuth failed.
-   Webhook unavailable.
-   AI unavailable.
-   Provider timeout.
-   Recovery action failed.
-   Session expired.
-   Merchant not found.
-   Queue unavailable.

Do not show raw exceptions to users.

------------------------------------------------------------------------

# 65. Demo Mode Safety

Simulation must never call live payment APIs accidentally.

Use an explicit environment/domain flag:

``` text
SIMULATION_MODE=true
```

and enforce it server-side.

Simulation actions should be marked:

``` text
SIMULATED
```

throughout the UI and database.

Never mix simulated recovered revenue into production analytics.

------------------------------------------------------------------------

# 66. Security Review Before Demo

Verify:

-   No secrets committed.
-   `.env` ignored.
-   No secrets in frontend bundle.
-   OAuth client secret server-only.
-   API authentication enforced.
-   Merchant authorization enforced.
-   Webhook signatures verified.
-   Raw body preserved.
-   Duplicate events handled.
-   Action idempotency enforced.
-   AI output validated.
-   Policy engine deterministic.
-   Production errors sanitized.
-   CORS restricted.
-   HTTPS configured.
-   Rate limiting enabled.
-   Audit logs active.

------------------------------------------------------------------------

# 67. Observability Before Demo

Minimum:

-   Structured API logs.
-   Request IDs.
-   Worker logs.
-   Error logs.
-   Recovery case IDs.
-   Action IDs.
-   Provider event IDs.
-   Basic metrics.

Track:

``` text
webhooks_received
webhooks_rejected
webhooks_duplicate
recovery_cases_created
ai_diagnoses
policy_approvals
policy_rejections
actions_executed
actions_failed
revenue_recovered
```

------------------------------------------------------------------------

# 68. Implementation Order

Build in this exact vertical order.

## Phase 1 --- Foundation

-   [ ] Clean env configuration.
-   [ ] Better Auth.
-   [ ] API JWT validation.
-   [ ] Root layout.
-   [ ] Dashboard shell.
-   [ ] Error handling.
-   [ ] API client.
-   [ ] Health checks.

## Phase 2 --- Merchant

-   [ ] Merchant Prisma model.
-   [ ] Merchant migration.
-   [ ] Merchant service.
-   [ ] Merchant API.
-   [ ] Merchant onboarding page.
-   [ ] Persistent merchant session/context.

## Phase 3 --- Razorpay connection

-   [ ] Razorpay integration abstraction.
-   [ ] Development Test Mode integration.
-   [ ] Razorpay OAuth application configuration.
-   [ ] OAuth connect route.
-   [ ] OAuth callback.
-   [ ] State validation.
-   [ ] Token encryption.
-   [ ] Connection persistence.
-   [ ] Connection status API.
-   [ ] Connect/disconnect UI.

## Phase 4 --- Webhooks

-   [ ] Raw-body handling.
-   [ ] Signature verification.
-   [ ] Event schema validation.
-   [ ] Event persistence.
-   [ ] Event ID uniqueness.
-   [ ] Deduplication.
-   [ ] Queue integration.
-   [ ] Worker.
-   [ ] Payment event normalization.

## Phase 5 --- Revenue risk

-   [ ] Payment model.
-   [ ] Failed-payment detector.
-   [ ] Revenue-at-risk calculation.
-   [ ] RecoveryCase.
-   [ ] Case timeline.
-   [ ] Recovery case UI.

## Phase 6 --- AI

-   [ ] Context builder.
-   [ ] AI diagnosis schema.
-   [ ] OpenAI service.
-   [ ] Structured output.
-   [ ] Confidence.
-   [ ] Prompt/model version tracking.
-   [ ] AI analysis persistence.

## Phase 7 --- Policy

-   [ ] Merchant policies.
-   [ ] Policy versioning.
-   [ ] Deterministic policy engine.
-   [ ] Approval/deny/escalate/stop.
-   [ ] Policy audit trail.

## Phase 8 --- Recovery actions

-   [ ] Action model.
-   [ ] Idempotency.
-   [ ] Action executor.
-   [ ] Retry scheduler.
-   [ ] Outcome processing.
-   [ ] Stopping rules.
-   [ ] Recovery completion.

## Phase 9 --- Analytics

-   [ ] Revenue at risk.
-   [ ] Recovered revenue.
-   [ ] Recovery rate.
-   [ ] Failure breakdown.
-   [ ] Action performance.
-   [ ] AI recommendation performance.
-   [ ] Dashboard charts.

## Phase 10 --- Simulation

-   [ ] Scenario generator.
-   [ ] Batch simulation.
-   [ ] Deterministic outcomes.
-   [ ] Simulation dashboard.
-   [ ] Demo scenarios.
-   [ ] Ensure simulation cannot affect live data.

## Phase 11 --- Production hardening

-   [ ] Security review.
-   [ ] Rate limiting.
-   [ ] Structured logging.
-   [ ] Health/readiness.
-   [ ] Graceful shutdown.
-   [ ] Queue reliability.
-   [ ] Error handling.
-   [ ] Database indexes.
-   [ ] Backups.
-   [ ] Monitoring.
-   [ ] Deployment.
-   [ ] HTTPS.
-   [ ] Production environment variables.

------------------------------------------------------------------------

# 69. Definition of Done

The MVP is not done when the UI looks complete.

It is done when this flow works:

``` text
Merchant signs in
       ↓
Merchant connects Razorpay
       ↓
Connection persists
       ↓
Razorpay sends payment.failed
       ↓
Webhook signature verified
       ↓
Event deduplicated and persisted
       ↓
Event queued
       ↓
Risk detected
       ↓
Recovery case created
       ↓
AI diagnoses failure
       ↓
Structured output validated
       ↓
Policy evaluates recommendation
       ↓
Allowed action created
       ↓
Action executes idempotently
       ↓
Razorpay outcome received
       ↓
Case updated
       ↓
Revenue marked recovered/lost
       ↓
Audit trail updated
       ↓
Dashboard metrics updated
```

A second identical webhook must not create a second recovery case.

An unsafe AI recommendation must not execute.

A second identical action request must not execute the financial action
twice.

A provider failure must produce a recoverable error state.

A completed recovery case must not continue acting.

------------------------------------------------------------------------

# 70. Hackathon Success Metrics

The final demo should lead with measurable outcomes.

Example:

``` text
100 payments simulated
₹1,20,000 revenue at risk

72 recoverable payments
₹86,000 recoverable revenue

₹61,000 recovered
71% recovery rate

142 actions evaluated
37 actions executed
0 duplicate financial actions
```

Use actual measured numbers from the implementation rather than invented
claims.

The strongest demo story is:

``` text
We detected ₹X at risk.
The agent diagnosed why.
The policy engine constrained what it could do.
It executed Y recovery actions.
₹Z was recovered.
Every decision and action is auditable.
```

------------------------------------------------------------------------

# 71. Engineering Principles to Preserve

1.  Webhook-first.
2.  Event-driven.
3.  At-least-once delivery aware.
4.  Idempotent financial actions.
5.  AI constrained by deterministic policy.
6.  No direct LLM control of money.
7.  OAuth for production merchant connection.
8.  Secrets remain server-side.
9.  Immutable audit trail.
10. Explicit stopping rules.
11. Persistent state, not frontend-only state.
12. Queue long-running work.
13. Validate every external input.
14. Normalize provider errors.
15. Keep domain logic out of controllers.
16. Keep provider integration behind a service boundary.
17. Prefer one well-structured API over premature microservices.
18. Build vertically and test end-to-end.
19. Measure actual recovered money.
20. Optimize for trust, safety, and explainability.

------------------------------------------------------------------------

# 72. Reference Documentation

Razorpay:

-   OAuth / Technology Partners:
    https://razorpay.com/docs/partners/technology-partners/onboard-businesses/integrate-oauth/
-   OAuth integration steps:
    https://razorpay.com/docs/partners/technology-partners/onboard-businesses/integrate-oauth/integration-steps/
-   Webhooks: https://razorpay.com/docs/webhooks/
-   Webhook best practices:
    https://razorpay.com/docs/webhooks/best-practices/
-   Webhook validation/testing:
    https://razorpay.com/docs/webhooks/validate-test/

Prisma:

-   Prisma ORM: https://docs.prisma.io/docs/orm/v7
-   PostgreSQL driver adapters:
    https://docs.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers

------------------------------------------------------------------------

# 73. Final Product Architecture

``` text
                         MERCHANT
                            │
                            ▼
                    ┌───────────────┐
                    │    Next.js    │
                    │   Dashboard   │
                    └───────┬───────┘
                            │
                         HTTPS/JWT
                            │
                            ▼
                    ┌───────────────┐
                    │    Express    │
                    │      API      │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
     PostgreSQL           Redis           Razorpay
          │              BullMQ              │
          │                 │                 │
          │                 ▼                 │
          │            Workers                │
          │                 │                 │
          └────────┬────────┴────────┬────────┘
                   │                 │
                   ▼                 ▼
             Recovery Engine      OpenAI
                   │
                   ▼
          ┌─────────────────────┐
          │ Deterministic       │
          │ Policy / Safety     │
          │ Gate                │
          └──────────┬──────────┘
                     │
                     ▼
               Action Executor
                     │
                     ▼
                  Razorpay
                     │
                     ▼
                  Outcome
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
       Audit Trail           Analytics
```

## Core system invariant

> **AI may recommend. Policy decides. Executors act. Webhooks prove
> outcomes. The database records the truth.**

That invariant should remain intact throughout the implementation.
