## Practice Session Management – Backend APIs (Phase 1)

This document specifies the backend APIs implemented for the **Practice Session Management** feature, as described in the product PRD [`resource/prds/01-practice-session-management.md`](../../resource/prds/01-practice-session-management.md).

These APIs support:
- **Start / Resume** a practice session
- **Progress tracking** (which questions have at least one **`PracticeFeedback`** — see §1.1 and [AI Feedback PRD §2.4](../../04-ai-feedback-system.md))
- **Complete** a practice session and archive it to history

All endpoints are versioned under `v1` and are served by the same Spring Boot service.

---

## 1. Common Models

### 1.0 Data integrity constraint (authoritative)

For active rows, `practice` MUST enforce:

```sql
UNIQUE (practice_main_id, question_id)
```

Constraint intent:

- Guarantees exactly one canonical `practice_id` per question within a `PracticeMain`.
- Makes `GET /api/v1/practice-main/{practice_main_id}/practices?question_id=...` deterministic (0 or 1 row, never many).
- Makes `POST /api/v1/practice-main/{practice_main_id}/practices` truly idempotent under concurrent create races.
- Preserves product semantics where resubmits append `PracticeFeedback` rows instead of creating duplicate active `Practice` rows.

This uniqueness requirement applies to the active `practice` table; history/archive tables are out of scope for this specific constraint.

### 1.1 `PracticeMainResponseDto` (GET Response)

Returned by `GET /api/v1/practice-main`:

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "practicing",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": "2026-02-13T11:30:00Z",
  "question_ids_with_feedback": [10, 20]
}
```

- **practice_main_id** (number): Unique ID of the practice session.
- **user_id** (number): ID of the user (`app_user.user_id`).
- **question_main_id** (number): ID of the `QuestionMain` being practiced.
- **status** (string): Session status; currently `"practicing"` or `"completed"`.
- **started_at** (string, ISO‑8601): When the session started.
- **completed_at** (string, ISO‑8601, optional): Present only when the session is completed.
- **question_ids_with_feedback** (array\<number>): Question IDs for which at least one **`PracticeFeedback`** row exists in this session (i.e. the user completed Get Feedback at least once for that question). Used for **progress dots**. A `Practice` row is not required for every question until the user records speech or submits feedback; progress reflects **feedback received**, not draft work or transcript-only state.

### 1.2 `PracticeMain` (Create / Update Body & Response)

For create and update operations the raw `PracticeMain` entity is used:

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "practicing",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": null
}
```

Fields:

- **practice_main_id** (number): Unique ID of the practice session.
- **user_id** (number): User ID.
- **question_main_id** (number): `QuestionMain` ID.
- **status** (string): `"practicing"` or `"completed"`.
- **started_at** (string, ISO‑8601): Session start time (server‑generated).
- **completed_at** (string, ISO‑8601, nullable): Completion time when status is `"completed"`.

### 1.3 `ErrorResponse`

Standard error format returned by GlobalExceptionHandler:

```json
{
  "error": "Resource not found",
  "message": "PracticeMain with id 999 does not exist"
}
```

- **error** (string): High‑level error category (`"Resource not found"`, `"Bad request"`, `"Internal server error"`).
- **message** (string): Human‑readable explanation.

---

## 2. Get Active Practice Session (with Progress)

### 2.1 Endpoint

- **Method**: `GET`
- **Path**: `/api/v1/practice-main`

### 2.2 Description

Retrieve the **active** practice session for a given user and `QuestionMain`, including which questions already have **AI feedback** (see `question_ids_with_feedback`).

The typical frontend flow:

1. Call this endpoint to check whether the user has an active session (`status = practicing` by default).
2. If **200 OK**: resume this session.
3. If **404 Not Found**: create a new session via `POST /api/v1/practice-main`.

### 2.3 Query Parameters

- **user_id** (number, required in Phase 1): User ID. Until authentication is implemented, this is passed explicitly; once auth is in place, `user_id` will be derived from the access token and this parameter will be removed.
- **question_main_id** (number, required): `QuestionMain` ID.
- **status** (string, optional, default: `"practicing"`): Session status to search for (usually `"practicing"`).

### 2.4 Responses

- **200 OK**

  Body (`PracticeMainResponseDto`):

  ```json
  {
    "practice_main_id": 123,
    "user_id": 456,
    "question_main_id": 1,
    "status": "practicing",
    "started_at": "2026-02-13T09:00:00Z",
    "completed_at": null,
    "question_ids_with_feedback": [10, 20]
  }
  ```

- **404 Not Found**

  No active session exists for the provided `user_id`, `question_main_id`, and `status`.

  ```json
  {
    "error": "Resource not found",
    "message": "No active practice session found"
  }
  ```

- **400 Bad Request**

  Parameter type conversion errors (e.g., `user_id=invalid`) are handled globally:

  ```json
  {
    "error": "Bad request",
    "message": "Invalid value 'invalid' for parameter 'user_id'"
  }
  ```

- **500 Internal Server Error**

  Fallback for unexpected errors:

  ```json
  {
    "error": "Internal server error",
    "message": "An unexpected error occurred. Please try again later."
  }
  ```

### 2.5 Example

**Request**

```http
GET /api/v1/practice-main?user_id=456&question_main_id=1&status=practicing
Accept: application/json
```

**Response – 200 OK**

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "practicing",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": null,
  "question_ids_with_feedback": [10, 20]
}
```

---

## 3. Create Practice Session

### 3.1 Endpoint

- **Method**: `POST`
- **Path**: `/api/v1/practice-main`

### 3.2 Description

Create a **new** practice session for a user and `QuestionMain`.

Recommended frontend usage:

1. Call `GET /api/v1/practice-main` to check for an existing active session.
2. If that call returns **404**, call this endpoint to create a new `PracticeMain`.

### 3.3 Request Body

```json
{
  "question_main_id": 1,
  "user_id": 456
}
```

- **question_main_id** (number, required): ID of the `QuestionMain` the user is practicing.
- **user_id** (number, required in Phase 1): User ID. This field is a temporary workaround before auth; once authentication is implemented, `user_id` will be derived from the access token and this field will be removed from the request body.

### 3.4 Responses

- **201 Created**

  Body (`PracticeMain`):

  ```json
  {
    "practice_main_id": 123,
    "user_id": 456,
    "question_main_id": 1,
    "status": "practicing",
    "started_at": "2026-02-13T09:00:00Z",
    "completed_at": null
  }
  ```

- **400 Bad Request**

  Malformed JSON or invalid types in the body:

  ```json
  {
    "error": "Bad request",
    "message": "Invalid request payload"
  }
  ```

  (Exact message may vary depending on validation/parsing.)

- **404 Not Found**

  If referenced resources (e.g., `QuestionMain` or `app_user`) are validated and do not exist:

  ```json
  {
    "error": "Resource not found",
    "message": "QuestionMain with id 1 does not exist"
  }
  ```

- **500 Internal Server Error**

  For unexpected server‑side failures.

### 3.5 Example

**Request**

```http
POST /api/v1/practice-main
Content-Type: application/json
Accept: application/json

{
  "question_main_id": 1,
  "user_id": 456
}
```

**Response – 201 Created**

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "practicing",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": null
}
```

---

## 4. Update Practice Session Status (Complete Session)

### 4.1 Endpoint

- **Method**: `PATCH`
- **Path**: `/api/v1/practice-main/{id}`

### 4.2 Description

Update the status of an existing practice session.

Special handling for `"completed"`:

- If the session is currently `"practicing"`:
  - Archive `PracticeMain` and all associated `Practice` records into history tables.
  - Delete the active `PracticeMain` record.
  - Return a synthetic `PracticeMain` view with `status = "completed"` and `completed_at` set.
- If the session was already archived:
  - Load from history and return the completed metadata (idempotent behavior).
- For any other status (e.g., `"practicing"`):
  - Update the status on the active row and clear `completed_at`.

### 4.3 Path Parameters

- **id** (number, required): `practice_main_id` of the session to update.

### 4.4 Request Body

```json
{
  "status": "completed"
}
```

- **status** (string, required):
  - `"completed"` – triggers completion + archiving.
  - `"practicing"` – or other allowed values to update status and clear `completed_at`.

### 4.5 Responses

- **200 OK**

  When status is set to `"completed"`:

  ```json
  {
    "practice_main_id": 123,
    "user_id": 456,
    "question_main_id": 1,
    "status": "completed",
    "started_at": "2026-02-13T09:00:00Z",
    "completed_at": "2026-02-13T11:30:00Z"
  }
  ```

- **404 Not Found**

  No active or archived session with the given `id`:

  ```json
  {
    "error": "Resource not found",
    "message": "PracticeMain with id 999 does not exist"
  }
  ```

- **400 Bad Request**

  Invalid `{id}` path variable (e.g., non‑numeric):

  ```json
  {
    "error": "Bad request",
    "message": "Invalid value 'invalid' for parameter 'id'"
  }
  ```

- **500 Internal Server Error**

  For example, attempting to complete a session in an unsupported status:

  ```json
  {
    "error": "Internal server error",
    "message": "An unexpected error occurred. Please try again later."
  }
  ```

### 4.6 Example

**Request**

```http
PATCH /api/v1/practice-main/123
Content-Type: application/json
Accept: application/json

{
  "status": "completed"
}
```

**Response – 200 OK**

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "completed",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": "2026-02-13T11:30:00Z"
}
```

---

## 5. Get canonical Practice for a question

### 5.1 Endpoint

- **Method:** `GET`
- **Path:** `/api/v1/practice-main/{practice_main_id}/practices`
- **Query parameters:** `question_id` (number, required)

### 5.2 Description

Returns the **single** canonical `Practice` for the given `practice_main_id` and `question_id`, including speech-capture state for that answer. The active `practice` table enforces **`UNIQUE (practice_main_id, question_id)`** (see Foundation PRD). This endpoint does **not** return `PracticeMain.whiteboard_content`; the canonical diagram remains on `PracticeMain`.

### 5.3 Responses

- **200 OK** — Body: `PracticeQuestionStateDto` (single object):

```json
{
  "practice_id": 789,
  "practice_main_id": 123,
  "question_id": 456,
  "transcript_segments": [
    {
      "segment_order": 1,
      "transcript_text": "First I would define the core services...",
      "duration_seconds": 80
    }
  ],
  "total_duration_seconds": 80,
  "combined_transcript": "First I would define the core services..."
}
```

- **transcript_segments** (array): Ordered by `segment_order`. May be empty if no speech has been saved yet.
- **total_duration_seconds** and **combined_transcript**: Derived from segments in V1 (see [Audio Recording – Backend APIs](03-audio-recording.md)).

- **404 Not Found** — No `Practice` row exists for this `(practice_main_id, question_id)` pair. Client may call **§6 Create or get Practice** before the first transcript segment upload.

- **409 Conflict** — More than one row matches (data integrity violation). Use a stable `error` value such as `practice_duplicate_for_question` and a clear `message`. This state MUST NOT occur when the unique constraint is enforced.

- **403 Forbidden** — Caller cannot access this `practice_main_id` (when auth is enforced).

### 5.4 Example

```http
GET /api/v1/practice-main/123/practices?question_id=456
Accept: application/json
```

---

## 6. Create or get Practice for a question

### 6.1 Endpoint

- **Method:** `POST`
- **Path:** `/api/v1/practice-main/{practice_main_id}/practices`
- **Content-Type:** `application/json`

### 6.2 Description

**Idempotent create-or-get:** Ensures exactly one `Practice` row exists for `(practice_main_id, question_id)` so the client can obtain a `practice_id` before **`POST /api/v1/practice/{practice_id}/transcript-segments`** (see [Audio Recording – Backend APIs](03-audio-recording.md)). Does not create `PracticeFeedback`; progress dots still follow **`question_ids_with_feedback`**.

### 6.3 Request body

```json
{
  "question_id": 456
}
```

### 6.4 Responses

- **201 Created** — New row inserted. Body: same `PracticeQuestionStateDto` as **§5.3** (typically empty `transcript_segments`).

- **200 OK** — Row already existed. Body: current `PracticeQuestionStateDto` (same shape as GET **200**).

- **Concurrency:** If two requests race to create the same row, the unique constraint MUST ensure only one row survives. The losing insert SHOULD be handled idempotently: **load the existing row and return `200 OK`** with its `PracticeQuestionStateDto` (do not surface a user-facing error for a benign race).

- **400 Bad Request** — Missing or invalid `question_id`.

- **403 Forbidden** — Caller cannot write this session.

- **404 Not Found** — `practice_main_id` or `question_id` parent entities invalid, if validated.

### 6.5 Example

```http
POST /api/v1/practice-main/123/practices
Content-Type: application/json

{
  "question_id": 456
}
```

---

## 7. Frontend Integration Notes

- **Start / Resume Flow**
  - Call `GET /api/v1/practice-main?user_id={userId}&question_main_id={questionMainId}`.
    - **200** → Resume existing session; use `question_ids_with_feedback` to render progress dots.
    - **404** → No active session; call `POST /api/v1/practice-main` to create one, then load questions via `GET /api/v1/question-mains/{id}`.

- **Progress Dots**
  - For a given `QuestionMain`, frontends already have the ordered list of `Question` IDs from `GET /api/v1/question-mains/{id}`.
  - Compare that list against `question_ids_with_feedback` from the GET practice main response to determine:
    - **Blue dot**: question id is present in `question_ids_with_feedback` (at least one **`PracticeFeedback`** for that question in this session).
    - **Grey dot**: question id is not present (no feedback yet; draft whiteboard or transcript work does not change the dot).

- **Speech capture / question load**
  - To load persisted transcript and `practice_id` for a question: **`GET /api/v1/practice-main/{practice_main_id}/practices?question_id={id}`** (§5). If **404**, call **`POST /api/v1/practice-main/{practice_main_id}/practices`** with `{ "question_id" }` (§6), then use returned `practice_id` for segment uploads.
  - Feedback creation is separate from create-or-get: call **`POST /api/v1/practices/{practice_id}/feedbacks`** to generate and persist a `PracticeFeedback` row.

- **Complete and Review Flow**
  - On “Complete and Review”, call `PATCH /api/v1/practice-main/{id}` with `{ "status": "completed" }`.
  - Use the returned `status` and `completed_at` to drive navigation to the review page, as specified in the Review PRD (`resource/prds/05-review-history.md`).

---

## 8. Deprecation note (`POST /api/v1/practice`)

- `POST /api/v1/practice` is deprecated.
- Use `POST /api/v1/practice-main/{practice_main_id}/practices` as the idempotent create-or-get API for canonical `practice_id`.
- Use `POST /api/v1/practices/{practice_id}/feedbacks` for submit-for-feedback (empty JSON body; optional `Idempotency-Key` header; server derives diagram + transcript per [AI Feedback PRD](../04-ai-feedback-system.md) **§4.1**).

**Backend behavior (idempotency):** Durable `practice_feedback_request` row with `UNIQUE (user_id, idempotency_key)`, states `CLAIMED` / `COMPLETED` / `FAILED`, replay on `COMPLETED`, `503` + `code: feedback_in_progress` while `CLAIMED`, stale abandoned claims, and `FAILED` + same key + same input fingerprint may re-run LLM — see [AI Feedback PRD](../04-ai-feedback-system.md) **§4.1.1–4.1.3**.

