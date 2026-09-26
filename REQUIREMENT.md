# requirements.md — AI Skill Assessment Backend

## 1. Purpose

Build the Node.js + Express backend for an AI-powered workforce skill assessment platform.

The system allows admins to create AI skill assessments, invite employees, run realtime voice assessments using LiveKit + Gemini Live, store transcripts and recordings, and generate evidence-backed competency reports using a separate evaluator.

The prototype must prioritize reliability and a complete end-to-end flow over breadth.

---

## 2. User Roles

The prototype supports two roles:

### ADMIN
Can:
- create and manage assessments
- configure scenarios, competencies, rubrics, and resources
- generate assessment content with AI assistance
- invite multiple employees to an assessment
- monitor active assessments
- listen to active sessions
- view current scenario / agent state
- review recordings and transcripts
- view evaluation reports

### EMPLOYEE
Can:
- enter through an assessment invitation link
- view assessment preparation information
- run a device / microphone check
- start an assigned assessment
- complete the voice assessment

Employees must never see assessment scores or evaluation reports.

---

## 3. Organization Model

The prototype runs with one organization internally.

The database must still remain tenant-ready by including `organizationId` on organization-owned entities.

No organization-management UI is required.

All organization-owned queries must be scoped by the trusted organization context.

---

## 4. Assessment Lifecycle

Assessment lifecycle:

```text
DRAFT
  ↓
PUBLISHED
  ↓
ARCHIVED
```

Rules:
- admins create assessments as drafts
- draft assessments can be edited
- publishing creates an immutable assessment version
- published versions cannot be modified
- changes require a new version
- assignments reference a specific published version

---

## 5. Assessment Creation

Admins can create assessments manually or use AI-assisted generation.

Each assessment contains:

- name
- description
- preparation instructions
- topics / competencies shown to employees
- scenarios
- competency mappings
- rubrics
- supporting PDF resources

### AI-Assisted Creation

Admins may upload training material and ask the system to generate:

- suggested scenarios
- competencies
- rubrics
- expected behaviours
- follow-up guidance

AI-generated content must remain editable before publishing.

---

## 6. Scenario Flow

Assessments use fixed scenarios in a configured order.

```text
Scenario 1
   ↓
adaptive follow-ups
   ↓
Scenario 2
   ↓
adaptive follow-ups
   ↓
Scenario 3
```

The realtime agent may ask adaptive follow-up questions within a scenario.

The agent must not reorder core scenarios unless explicitly supported later.

---

## 7. Employee Preview

Before starting, employees can see:

- assessment name
- description
- preparation instructions
- topics / competencies
- microphone check
- remaining attempts

Employees must NOT see:

- exact scenarios
- exact questions
- hidden scoring rubrics
- expected answers
- evaluator prompts

There is no hard assessment time limit.

---

## 8. Invitations

Admins can invite multiple employees to the same assessment.

Each invitation creates an assignment for an employee email.

The invitation itself acts as the employee's entry point.

The invitation link must:

- be unique
- expire
- identify the intended assignment
- authenticate or establish access for the invited employee
- not expose reusable secrets after acceptance

Resend is used for email delivery.

---

## 9. Attempt Rules

Default maximum attempts:

```text
3
```

The maximum is configured through an environment variable.

Example:

```env
MAX_ASSESSMENT_ATTEMPTS=3
```

Opening the invitation or preparation page does not consume an attempt.

An attempt is consumed when the realtime interview actually starts.

### Interrupted Attempt

If the employee:

- manually stops
- closes the interview
- loses connection
- abandons the session

the attempt is consumed.

Starting again uses the next available attempt.

There is no resume/reconnect into the same attempt for the prototype.

### Successful Completion

Once an employee successfully completes and submits an assessment:

- the assignment becomes completed
- the employee cannot start another attempt
- remaining unused attempts are no longer available

---

## 10. Attempt States

Use explicit states:

```text
READY
IN_PROGRESS
PROCESSING
COMPLETED
ABANDONED
FAILED
EXPIRED
```

Typical successful flow:

```text
READY
  ↓
IN_PROGRESS
  ↓
PROCESSING
  ↓
COMPLETED
```

Interrupted sessions should normally become:

```text
IN_PROGRESS
  ↓
ABANDONED
```

Attempt transitions must be controlled by the backend.

---

## 11. Realtime Assessment

Use:

- LiveKit for realtime media/session infrastructure
- Gemini Live for the realtime interviewer
- Node.js LiveKit agent worker

The agent must:

- load the assessment context
- follow the configured scenario order
- ask adaptive follow-up questions
- retrieve relevant uploaded resources when needed
- record assessment events
- mark scenario progression
- complete the assessment cleanly

The realtime interviewer must NOT be the authoritative evaluator.

---

## 12. Agent Tools

Initial tools:

```text
getAssessmentContext
getCurrentScenario
retrieveResource
recordAssessmentEvent
completeScenario
completeAssessment
```

Trusted identifiers such as:

- organizationId
- attemptId
- userId
- assessmentId

must be injected by the backend.

The model must not provide these as authoritative tool inputs.

---

## 13. Resources / RAG

Prototype supports PDF uploads only.

Pipeline:

```text
PDF upload
   ↓
store original
   ↓
extract text
   ↓
chunk
   ↓
generate embeddings
   ↓
store in PostgreSQL + pgvector
```

Resources are available to:

- the realtime interviewer
- the post-session evaluator

Retrieval must always be scoped by:

```text
organizationId
+
assessmentId
```

Do not use a separate vector database for the prototype.

---

## 14. Transcript

Persist transcript turns during the assessment.

Each turn should include:

- attemptId
- speaker
- content
- sequence
- timestamps where available

Speakers:

```text
EMPLOYEE
AGENT
SYSTEM
```

The database is the system of record.

---

## 15. Recording

Record audio only.

Recordings are stored privately in Cloudflare R2.

Recommended path:

```text
recordings/
  {organizationId}/
    {assessmentId}/
      {attemptId}/
        recording.ogg
```

The database stores recording metadata.

Recordings must never have permanent public URLs.

Admin playback must use short-lived signed URLs.

---

## 16. Admin Live Inspection

Admins must be able to inspect active assessments.

The live inspection screen needs backend support for:

- active attempt list
- employee identity
- current assessment
- current scenario
- current agent state
- live transcript
- listen-only audio access

Admin observer sessions must not publish microphone audio.

---

## 17. Independent Evaluation

Evaluation starts after a completed assessment.

Use a separate non-realtime Gemini model.

Evaluator inputs:

- assessment version
- scenarios
- competencies
- rubrics
- transcript
- relevant resource content

Evaluator output must contain:

- overall score
- competency scores
- strengths
- gaps
- transcript-backed evidence
- recommended training / next steps

---

## 18. Scoring

Internal competency scoring uses:

```text
1–5
```

Example:

```text
1 = very weak
2 = developing
3 = acceptable
4 = strong
5 = excellent
```

The frontend may convert this into a percentage for display.

Example:

```text
4 / 5 → 80%
```

The evaluator must return structured output validated by Zod.

Scores without evidence must not be accepted as valid evaluation results.

---

## 19. Evaluation Evidence

Each competency score should reference supporting transcript turns.

Example:

```json
{
  "competencyId": "privacy-awareness",
  "score": 4,
  "maxScore": 5,
  "reasoning": "The employee identified the need to verify company policy before uploading customer information.",
  "evidence": [
    {
      "turnId": "turn_17",
      "reason": "The employee explicitly mentioned checking whether the approved AI environment allowed customer data."
    }
  ]
}
```

---

## 20. Final Report

Admin-only report includes:

- overall result
- score per competency
- percentage representation if required
- strengths
- gaps
- recommended training
- evidence
- transcript
- audio recording

Employees do not receive this report.

---

## 21. Authentication and Authorization

Use Better Auth.

The invitation link acts as the employee entry/authentication flow.

Backend authorization must ensure:

- employees access only their assignment
- employees cannot exceed attempt limits
- employees cannot access reports
- admins can access organization assessments
- admins can inspect active attempts
- admins can access recordings

Frontend-provided `organizationId` must never be trusted for authorization.

---

## 22. Backend Stack

```text
Runtime        Node.js
Language       TypeScript
HTTP           Express
Database       PostgreSQL
ORM            Prisma
Vector Search  pgvector
Auth           Better Auth
Realtime       LiveKit
Realtime AI    Gemini Live
Evaluation AI  Gemini
Email          Resend
Storage        Cloudflare R2
Validation     Zod
Logging        Pino
Testing        Vitest + Supertest
```

---

## 23. Required Background Operations

Slow/retryable tasks should execute outside request handlers.

Initial background operations:

- PDF parsing
- embedding generation
- evaluation
- invitation email delivery
- recording completion processing

Do not introduce Redis / BullMQ unless required.

The prototype can use a simpler worker mechanism first.

---

## 24. Reliability Requirements

The backend must:

- validate all external input
- enforce attempt limits atomically
- prevent duplicate completed attempts
- prevent duplicate evaluations
- handle duplicate provider callbacks
- keep LiveKit tokens short-lived
- validate evaluator JSON
- store transcripts independently of the model
- keep R2 recordings private
- use structured logging
- expose a health endpoint

---

## 25. Security Requirements

Always:

- authorize organization resources
- generate LiveKit tokens server-side
- scope RAG retrieval
- use private R2 storage
- use signed recording URLs
- keep provider credentials server-side
- hide scoring rubrics from employees
- validate AI output
- expire invitation links
- avoid logging sensitive tokens or full transcripts at info level

---

## 26. Main Entities

Minimum conceptual entities:

```text
Organization
User

Assessment
AssessmentVersion
Scenario
Competency
Rubric

Resource
ResourceChunk

Invitation
Assignment

Attempt
AttemptScenario
AttemptTurn
AttemptEvent

Recording

Evaluation
CompetencyScore
EvaluationEvidence
```

---

## 27. Prototype End-to-End Flow

The prototype is complete when this works:

```text
Admin creates assessment
        ↓
Admin manually or AI-generates scenarios/rubrics
        ↓
Admin uploads PDF resources
        ↓
Admin publishes assessment
        ↓
Admin bulk-invites employees
        ↓
Employee opens invitation
        ↓
Employee sees preparation screen
        ↓
Employee checks microphone
        ↓
Employee starts attempt
        ↓
Attempt count is consumed
        ↓
Gemini Live conducts voice assessment
        ↓
Agent progresses fixed scenarios
        ↓
Agent retrieves resources when required
        ↓
Transcript is persisted
        ↓
Audio recording is stored in R2
        ↓
Employee completes assessment
        ↓
Assignment becomes locked
        ↓
Separate Gemini evaluator runs
        ↓
1–5 competency scores + evidence generated
        ↓
Admin views report, transcript, and recording
```

---

## 28. Out of Scope

Do not build for the prototype:

- MCP integrations
- SAML / SSO
- SCIM
- complex RBAC
- video assessments
- HRIS integrations
- Slack / Teams integrations
- custom WebRTC
- custom STT / TTS
- multiple realtime model providers
- advanced analytics
- employee-facing reports
- reconnect/resume of interrupted attempts
- organization management UI

---

## 29. Success Criteria

The prototype should demonstrate one thing clearly:

> A company can configure a realistic AI skill assessment, invite employees, run a reliable realtime voice assessment, and receive an evidence-backed report showing what the employee can actually do.
