## Audio Recording – Backend APIs (Phase 1)

This document specifies the backend APIs for the **Transcript Segment Flow** of Audio Recording, aligned with the product PRD [`resource/prds/03-audio-recording.md`](../../03-audio-recording.md) and AI Feedback PRD [`resource/prds/04-ai-feedback-system.md`](../../04-ai-feedback-system.md).

The core behavior is:

- Users can **start/stop speaking multiple times** for a question.
- Every Stop persists one ordered transcript segment.
- Backend maintains a merged **`combined_transcript`** and cumulative **`total_duration_seconds`**.
- In V1, **`combined_transcript`** and **`total_duration_seconds`** are derived values computed from ordered transcript segments, and are not stored as dedicated aggregate columns.
- On Get Feedback, frontend sends the merged transcript into `POST /api/v1/practice`.

All endpoints are versioned under `v1` and are served by the same Spring Boot service.

---

## 1. Scope and Integration

### 1.1 In scope

- Persist transcript segments per `practice`.
- Return updated merged transcript and cumulative duration after each segment save.
- Expose transcript state when loading a practice so users can continue speaking later.
- Define how merged transcript is passed into `POST /api/v1/practice` for AI feedback.

### 1.2 Out of scope

- Raw audio file upload/storage and file-serving endpoints.
- Transcript edit/delete for individual segments in V1.
- Authentication redesign (follow current Phase-1 pattern).

### 1.3 Integration contract

- Audio Recording API provides `combined_transcript` and `total_duration_seconds`.
- Frontend passes those fields to `POST /api/v1/practice`.
- AI feedback generation uses `combined_transcript` as prompt input.

---

## 2. Common Models

### 2.1 `TranscriptSegmentSaveRequest`

Request body for saving one segment:

```json
{
  "transcript_text": "I would put a load balancer in front of stateless API servers...",
  "duration_seconds": 95
}
```

- **transcript_text** (string, required): Non-empty recognized speech for this stop event.
- **duration_seconds** (number, required): Duration of this segment in seconds.

### 2.2 `TranscriptSegmentSaveResponse`

Response from segment save:

```json
{
  "practice_id": 789,
  "segment_order": 3,
  "duration_seconds": 95,
  "total_duration_seconds": 245,
  "combined_transcript": "I would start with... Next, I would put a load balancer..."
}
```

- **practice_id** (number): Target practice.
- **segment_order** (number): 1-based insertion order for this saved segment.
- **duration_seconds** (number): Echo of the saved segment duration.
- **total_duration_seconds** (number): Cumulative duration across all segments on this practice.
- **combined_transcript** (string): Ordered concatenation of all saved transcript segments for this practice.

### 2.3 `TranscriptSegmentDto`

Used when loading practice state:

```json
{
  "segment_order": 2,
  "transcript_text": "For data storage I would separate hot and cold paths...",
  "duration_seconds": 70
}
```

### 2.4 `ErrorResponse`

Standard backend error shape:

```json
{
  "error": "Validation failed",
  "message": "One or more fields are invalid"
}
```

Optional validation detail extension:

```json
{
  "error": "Validation failed",
  "message": "One or more fields are invalid",
  "details": [
    {
      "field": "transcript_text",
      "message": "Transcript text cannot be empty"
    }
  ]
}
```

---

## 3. Save Transcript Segment

### 3.1 Endpoint

- **Method:** `POST`
- **Path:** `/api/v1/practice/{practice_id}/transcript-segments`
- **Content-Type:** `application/json`

### 3.2 Description

Persists one transcript segment produced when the user presses Stop. This endpoint can be called repeatedly for the same `practice_id`, enabling multiple recordings per question.

Prerequisite rule: `practice_id` is obtained by create-or-get practice **before recording starts**. Therefore, this transcript segment API requires an existing `practice_id`.

### 3.3 Path parameters

- **practice_id** (number, required): Practice record to which this segment belongs.

### 3.4 Request body

`TranscriptSegmentSaveRequest` (see §2.1).

### 3.5 Behavior

1. Validate that `practice_id` exists and is writable.
2. Validate `transcript_text` is not blank.
3. Validate `duration_seconds` is positive and cumulative duration does not exceed V1 max (10 minutes / 600 seconds).
4. Compute next `segment_order` (append-only).
5. Persist segment row.
6. Recompute `total_duration_seconds`.
7. Rebuild `combined_transcript` by concatenating all segments in `segment_order`.
8. Return `TranscriptSegmentSaveResponse`.

Storage note: `total_duration_seconds` and `combined_transcript` are computed from persisted segment rows for the target `practice_id` (derived-on-read/response), not persisted as separate aggregate fields in V1.

### 3.6 Responses

- **200 OK** — `TranscriptSegmentSaveResponse`.
- **400 Bad Request** — Empty transcript, invalid duration, duration limit exceeded, malformed payload.
- **403 Forbidden** — User not allowed to write this practice (when auth is enforced).
- **404 Not Found** — `practice_id` not found.
- **500 Internal Server Error** — Unexpected server failure.

### 3.7 Example

**Request**

```http
POST /api/v1/practice/789/transcript-segments
Content-Type: application/json
Accept: application/json

{
  "transcript_text": "I would put a load balancer in front of stateless API servers...",
  "duration_seconds": 95
}
```

**Response – 200 OK**

```json
{
  "practice_id": 789,
  "segment_order": 3,
  "duration_seconds": 95,
  "total_duration_seconds": 245,
  "combined_transcript": "I would start with... Next, I would put a load balancer..."
}
```

---

## 4. Load Practice Speech Capture State

### 4.1 Endpoint

- **Method:** `GET`
- **Path:** `/api/v1/practice/{practice_id}`

### 4.2 Description

Returns current speech capture state so the frontend can restore prior work and continue speaking after refresh or returning later.

### 4.3 Response excerpt

```json
{
  "practice_id": 789,
  "question_id": 456,
  "transcript_segments": [
    {
      "segment_order": 1,
      "transcript_text": "First I would define the core services...",
      "duration_seconds": 80
    },
    {
      "segment_order": 2,
      "transcript_text": "For data storage I would separate hot and cold paths...",
      "duration_seconds": 70
    }
  ],
  "total_duration_seconds": 150,
  "combined_transcript": "First I would define..."
}
```

- **transcript_segments** is ordered by `segment_order`.
- `total_duration_seconds` and `combined_transcript` represent all persisted segments.
- `total_duration_seconds` and `combined_transcript` are computed from `transcript_segments` and are not stored as dedicated aggregate columns in V1.

---

## 5. Get Feedback Integration (`POST /api/v1/practice`)

### 5.1 Required behavior

When the user clicks Get Feedback, frontend sends merged speech context from transcript segments:

```json
{
  "practice_id": 123,
  "practice_main_id": 456,
  "question_id": 789,
  "whiteboard_content": {},
  "combined_transcript": "I would start with a load balancer in front of stateless API servers...",
  "total_duration_seconds": 180
}
```

- `combined_transcript` must represent **all saved segments** for that question/practice in order.
- Backend AI prompt construction uses this merged transcript.

### 5.2 Multiple recordings semantics

- Each stop event creates a new segment row.
- Users can continue recording repeatedly (append-only in V1).
- Effective transcript for AI feedback is always the merge of all saved segments, not just the most recent segment.

---

## 6. Error Handling

| Scenario | HTTP | Notes |
|----------|------|-------|
| Empty `transcript_text` | 400 | Validation failure; optional `details` entry for field. |
| `duration_seconds` missing/invalid | 400 | Validation failure. |
| Cumulative duration exceeds 600s | 400 | Return clear limit-exceeded message. |
| Invalid or missing `practice_id` | 404 | Practice not found. |
| Unauthorized practice access | 403 | When auth is available/enforced. |
| Unexpected server/database error | 500 | Global exception fallback. |

---

## 7. Frontend Integration Notes

1. Before recording starts, frontend create-or-get a practice and stores `practice_id`.
2. User clicks Start Speaking and browser speech recognition begins.
3. User clicks Stop and frontend finalizes one transcript segment.
4. Frontend calls `POST /api/v1/practice/{practice_id}/transcript-segments`.
5. Backend returns updated `total_duration_seconds` and `combined_transcript`.
6. User may click Continue Speaking and repeat steps 3-5 multiple times.
7. On Get Feedback, frontend submits `combined_transcript` and `total_duration_seconds` in `POST /api/v1/practice`.

No transcript segment delete/edit is provided in V1; users only append and submit.

---

## 8. Summary

- Supports multiple recording cycles per question via append-only transcript segments.
- Maintains deterministic merged `combined_transcript` for AI feedback.
- Uses `POST /api/v1/practice/{practice_id}/transcript-segments` and `GET /api/v1/practice/{practice_id}` as the core API surface.
- Aligns with product PRD recording behavior and AI Feedback request contract.
