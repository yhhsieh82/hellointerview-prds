### Whiteboard & Diagramming – Backend APIs (Phase 1)

This document specifies the backend APIs for the **Whiteboard & Diagramming** feature, as described in the product PRD [`resource/prds/02-whiteboard-diagramming.md`](../../02-whiteboard-diagramming.md).

These APIs extend the existing Practice Session Management backend (`resource/prds/backend/01-practice-session-management.md`) to support:

- **Loading** the canonical whiteboard for a practice session (5-section document, merged into a single Excalidraw canvas on the frontend)
- **Autosaving** the full 5-section whiteboard document (`PracticeMain.whiteboard_content`)
- **Persisting the final whiteboard snapshot** when a session is completed

All endpoints are versioned under `v1` and are served by the same Spring Boot service.

---

### 1. Common Models

#### 1.1 `whiteboard_content` (JSONB on `PracticeMain`)

The `PracticeMain.whiteboard_content` column stores the **canonical whiteboard for the entire practice session** as JSONB. It contains all 5 sections (Functional Req, Non-Functional Req, Entities, API, High Level Design), each with its own Excalidraw payload.

High-level structure (simplified):

```json
{
  "section_1": {
    "type": "diagram",
    "version": "1.0",
    "elements": [
      {
        "id": "elem1",
        "type": "rectangle",
        "x": 100,
        "y": 50,
        "width": 200,
        "height": 100,
        "text": "User Service",
        "fillColor": "#ffe7cc",
        "strokeColor": "#000000"
      }
    ],
    "appState": { },
    "files": { }
  },
  "section_2": { },
  "section_3": { },
  "section_4": { },
  "section_5": { }
}
```

Implementation notes:

- Mapped on the entity as a JSON-serializable field (e.g. `Map<String, Object>` or a dedicated `WhiteboardContent` type) with `@Column(columnDefinition = "jsonb")`.
- Exposed in JSON as `whiteboard_content` (snake_case) using `@JsonProperty("whiteboard_content")`.
- Treated as an **opaque document** by the backend; no server-side understanding of Excalidraw element schema is required for Phase 1.

#### 1.2 `PracticeMainResponseDto` (extended)

Returned by `GET /api/v1/practice-main` and used in some `PATCH` responses. This extends the existing definition in `01-practice-session-management.md` with the `whiteboard_content` field:

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "practicing",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": "2026-02-13T11:30:00Z",
  "question_ids_with_feedback": [10, 20],
  "whiteboard_content": {
    "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
  }
}
```

- **question_ids_with_feedback** (array\<number>): Same semantics as in `01-practice-session-management.md` (questions with at least one **`PracticeFeedback`** in this session).

- **whiteboard_content** (object, nullable):
  - When a session is newly created, it is initialized to the canonical empty 5-section structure.
  - When a session has been previously edited, this contains the full latest whiteboard state.

#### 1.3 `PracticeMain` (Create / Update Body & Response)

The raw `PracticeMain` entity (used for `POST` and some `PATCH` responses) is also extended to include `whiteboard_content`:

```json
{
  "practice_main_id": 123,
  "user_id": 456,
  "question_main_id": 1,
  "status": "practicing",
  "started_at": "2026-02-13T09:00:00Z",
  "completed_at": null,
  "whiteboard_content": {
    "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
  }
}
```

#### 1.4 `ErrorResponse`

Same as defined in `01-practice-session-management.md`:

```json
{
  "error": "Resource not found",
  "message": "PracticeMain with id 999 does not exist"
}
```

- Reused for all whiteboard-related errors:
  - Invalid `whiteboard_content` payload (400)
  - PracticeMain not found (404)
  - Unexpected server failures (500)

---

### 2. Get Active Practice Session (with Whiteboard)

This endpoint is an **extension** of the existing Practice Session Management GET endpoint. It now returns the full whiteboard for the active session.

#### 2.1 Endpoint

- **Method**: `GET`
- **Path**: `/api/v1/practice-main`

#### 2.2 Description

Retrieve the **active** practice session for a given user and `QuestionMain`, including:

- Which questions already have practice submissions (for progress dots)
- The **canonical 5-section whiteboard** (`whiteboard_content`) representing the current Excalidraw state

Typical frontend flow:

1. Call this endpoint to check whether the user has an active session (`status = practicing` by default).
2. If **200 OK**: resume this session, hydrate the whiteboard from `whiteboard_content`, and render progress dots.
3. If **404 Not Found**: create a new session via `POST /api/v1/practice-main`.

#### 2.3 Query Parameters

Unchanged from existing practice-session backend:

- **user_id** (number, required in Phase 1): User ID.
- **question_main_id** (number, required): `QuestionMain` ID.
- **status** (string, optional, default: `"practicing"`): Session status to search for.

#### 2.4 Responses

- **200 OK**

  Body (`PracticeMainResponseDto` with whiteboard):

  ```json
  {
    "practice_main_id": 123,
    "user_id": 456,
    "question_main_id": 1,
    "status": "practicing",
    "started_at": "2026-02-13T09:00:00Z",
    "completed_at": null,
    "question_ids_with_feedback": [10, 20],
    "whiteboard_content": {
      "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
    }
  }
  ```

  - If this is the **first time** the session is loaded and no autosave has occurred, the backend initializes `whiteboard_content` to the empty 5-section structure before returning it.

- **404 Not Found**

  Unchanged semantics; no active session exists for given parameters.

  ```json
  {
    "error": "Resource not found",
    "message": "No active practice session found"
  }
  ```

- **400 Bad Request** / **500 Internal Server Error**

  As defined in the existing practice-session backend doc.

#### 2.5 Example

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
  "question_ids_with_feedback": [10, 20],
  "whiteboard_content": {
    "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
  }
}
```

---

### 3. Create Practice Session (Initialize Whiteboard)

This endpoint is unchanged in shape but now **initializes** `whiteboard_content`.

#### 3.1 Endpoint

- **Method**: `POST`
- **Path**: `/api/v1/practice-main`

#### 3.2 Description

Create a **new** practice session for a user and `QuestionMain`. The new `PracticeMain` is created with:

- `status = "practicing"`
- `whiteboard_content` initialized to the empty 5-section diagram structure

Recommended frontend usage:

1. Call `GET /api/v1/practice-main`.
2. If that call returns **404**, call `POST /api/v1/practice-main` to create a new session.

#### 3.3 Request Body

Unchanged:

```json
{
  "question_main_id": 1,
  "user_id": 456
}
```

#### 3.4 Responses

- **201 Created**

  Body (raw `PracticeMain` with whiteboard):

  ```json
  {
    "practice_main_id": 123,
    "user_id": 456,
    "question_main_id": 1,
    "status": "practicing",
    "started_at": "2026-02-13T09:00:00Z",
    "completed_at": null,
    "whiteboard_content": {
      "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
    }
  }
  ```

- **400 / 404 / 500**

  Same as existing practice-session doc; no whiteboard-specific deviations.

#### 3.5 Example

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
  "completed_at": null,
  "whiteboard_content": {
    "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
  }
}
```

---

### 4. Autosave Whiteboard (Update `whiteboard_content`)

This extends the existing `PATCH /api/v1/practice-main/{id}` endpoint to support auto-saving the full whiteboard.

#### 4.1 Endpoint

- **Method**: `PATCH`
- **Path**: `/api/v1/practice-main/{id}`

#### 4.2 Description

Update an existing practice session. In addition to status updates already defined in `01-practice-session-management.md`, this endpoint now supports:

- **Autosave of the full canonical whiteboard** via `whiteboard_content` in the request body.

Behavior:

- Frontend always sends the **full 5-section `whiteboard_content` document** on autosave (no per-section patches).
- Backend performs a **last-write-wins** update of the JSONB field on the `PracticeMain` row.
- Updating `whiteboard_content` does **not** create or update `Practice` / `PracticeFeedback` rows; that is reserved for the “Get Feedback” flow.

#### 4.3 Path Parameters

- **id** (number, required): `practice_main_id` of the session to update.

#### 4.4 Request Body

Two usage patterns are supported:

1. **Autosave only** (no status change):

   ```json
   {
     "whiteboard_content": {
       "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
       "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
       "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
       "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
       "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
     }
   }
   ```

2. **Finalize + autosave** (completion and final snapshot in one call):

   ```json
   {
     "whiteboard_content": { },
     "status": "completed"
   }
   ```

- Fields:
  - **whiteboard_content** (object, optional): When present, overwrites the existing JSONB document for this `PracticeMain`.
  - **status** (string, optional): Same semantics as current implementation:
    - `"completed"` – triggers archiving behavior.
    - `"practicing"` – or other allowed values to update status and clear `completed_at`.

#### 4.5 Responses

- **200 OK**

  When only `whiteboard_content` is updated (autosave):

  ```json
  {
    "practice_main_id": 123,
    "user_id": 456,
    "question_main_id": 1,
    "status": "practicing",
    "started_at": "2026-02-13T09:00:00Z",
    "completed_at": null,
    "whiteboard_content": {
      "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
      "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
    }
  }
  ```

  When `status = "completed"` is set (with or without `whiteboard_content`):

  - Existing completion semantics still apply (archive to history tables, delete active row, return synthetic completed view).
  - If `whiteboard_content` is present, that final snapshot is persisted before archiving and is carried into `PracticeMainHistory.whiteboard_content`.

- **404 Not Found**, **400 Bad Request**, **500 Internal Server Error**

  Same as current practice-session doc, with additional 400s possible for malformed `whiteboard_content`.

#### 4.6 Examples

**Autosave only**

```http
PATCH /api/v1/practice-main/123
Content-Type: application/json
Accept: application/json

{
  "whiteboard_content": {
    "section_1": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_2": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_3": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_4": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} },
    "section_5": { "type": "diagram", "version": "1.0", "elements": [], "appState": {}, "files": {} }
  }
}
```

**Complete and Review with final snapshot**

```http
PATCH /api/v1/practice-main/123
Content-Type: application/json
Accept: application/json

{
  "whiteboard_content": { },
  "status": "completed"
}
```

---

### 5. Frontend Integration Notes (Whiteboard-Specific)

- **Initial load**
  - Frontend:
    1. Calls `GET /api/v1/practice-main?user_id={userId}&question_main_id={questionMainId}`.
    2. Merges all 5 section element arrays from `whiteboard_content` into one flat list and passes it to the single Excalidraw canvas as `initialData`.
    3. Uses `GET /api/v1/question-mains/{id}` to load questions and their `whiteboard_section` values.
    4. Sets **section focus** based on the current question’s `whiteboard_section`.

- **Autosave behavior**
  - Debounced (~5 seconds) after any whiteboard change:
    - Frontend sends **full** `whiteboard_content` to `PATCH /api/v1/practice-main/{id}`.
    - The backend stores this as the canonical last state; there is no per-section diffing or merging.
  - Autosave **does not** create/update `Practice` or `PracticeFeedback` records. To create a `Practice` row for speech capture or other per-question state without submitting for feedback, the client uses **`POST /api/v1/practice-main/{practice_main_id}/practices`** (see `resource/prds/backend/01-practice-session-management.md` §6).

- **Get Feedback interaction (high level)**
  - When the user clicks “Get Feedback” for a question:
    - The feedback subsystem reads the current `PracticeMain.whiteboard_content` plus question context and optional audio.
    - The whiteboard API’s role is only to ensure `whiteboard_content` is current and persisted.

- **Navigation between questions**
  - When moving to another question (via progress dots or “Next Question”):
    - Frontend should ensure any pending autosave has completed or trigger one before navigation.
    - Section focus on the whiteboard is updated purely on the frontend based on `Question.whiteboard_section`; the backend does **not** track active section.

- **Concurrency**
  - Last-write-wins semantics at the session level:
    - If the same user opens the same session in multiple browser tabs, the last successful autosave overwrites `whiteboard_content`.
    - Future enhancement: conflict detection or warnings are out of scope for Phase 1.

---

### 6. Summary

- **No new endpoints** are introduced; instead, the existing `PracticeMain` APIs are extended:
  - `GET /api/v1/practice-main` → returns `whiteboard_content`.
  - `POST /api/v1/practice-main` → initializes `whiteboard_content` to the empty 5-section document.
  - `PATCH /api/v1/practice-main/{id}` → accepts full-document `whiteboard_content` for autosave and combines it with existing completion behavior.
- The backend treats the whiteboard as an **opaque JSONB document** and focuses on reliable, low-latency persistence to support Excalidraw-based diagramming as specified in the 02 Whiteboard PRD.

