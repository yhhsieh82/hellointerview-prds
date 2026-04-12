# PRD: Audio Recording

**Version:** 2.2  
**Date:** April 11, 2026  
**Status:** Draft  

See [Foundation PRD](00-foundation.md) for shared data models and platform requirements.

---

## Integration Points

This feature is used by the **AI Feedback System**. Spoken explanations are converted to transcript text in the browser, persisted as transcript segments on the related `practice`, and included in the LLM prompt when users submit High Level Design and Deep Dive answers.

---

## 1. Feature Overview

Audio recording enables users to provide verbal explanations of their system design solutions. For High Level Design and Deep Dive questions, the spoken explanation complements the diagram and provides rationale that is not obvious from shapes and arrows alone. In V2, the system uses frontend speech-to-text as the primary capture mechanism: transcript text and duration metadata are persisted, while raw audio files are not uploaded or stored.

**Purpose:**
- Capture verbal design rationale and walkthrough
- Enable AI to evaluate both diagram and spoken explanation
- Let users stop, leave the browser, and later continue adding spoken explanation to the same `practice`
- Show users the total captured speaking time for the current answer

---

## 2. Recording Interface

### 2.1 Recording Interface
**Priority:** P0 (Must Have)

**User Story:**  
As a user answering High Level Design or Deep Dive questions, I want to speak my explanation, stop when needed, and later continue from the same answer so my transcript is preserved across sessions.

**Acceptance Criteria:**
- Speech capture controls appear only for question types: "High Level Design", "Deep Dive"
- Button states:
  - Idle: Microphone icon, "Start Speaking"
  - Listening: Red dot, live timer, "Stop"
  - Saved: Checkmark, total captured duration, "Continue Speaking"
- Browser microphone and speech recognition permissions requested on first use
- Spoken explanation is encouraged for these question types, but submission is not blocked if transcript is empty or missing
- When the user clicks `Stop`, the frontend finalizes the current transcript segment and persists it to the related `practice`
- When the user opens or returns to a question, the frontend loads persisted transcript for the canonical `Practice` for that question in the current session (see API below) and the user can continue speaking on the same `practice`
- UI displays the total captured duration across all saved transcript segments for the current `practice`
- A read-only **“Your answer”** section (title exactly **Your answer**) appears **below** the speech capture controls and shows the persisted combined transcript (or an empty state). Layout order: controls first, then **Your answer**
- V1 does not provide delete-recording or segment-removal controls; users can only add more speech and submit
- UI shows a non-blocking warning when the current recording segment crosses 3 minutes
- Max cumulative speaking duration: 10 minutes per `practice`
- Frontend stops active recording when cumulative speaking duration reaches 10 minutes

**Technical Specifications:**
- API: Browser speech recognition (`SpeechRecognition` / Web Speech API or equivalent frontend speech-to-text provider)
- Persisted artifact: transcript text plus duration metadata
- Storage unit: ordered transcript segments per `practice`
- `total_duration_seconds` and `combined_transcript` are derived from ordered transcript segments in V1, not persisted as dedicated aggregate columns
- Language: English in V1
- Raw audio file upload/storage: not supported in V1

**API Flow:**
```
1. On question load -> GET /api/v1/practice-main/{practice_main_id}/practices?question_id={id}
   - 200: render transcript in "Your answer"; store practice_id
   - 404: empty "Your answer"; call POST /api/v1/practice-main/{practice_main_id}/practices { question_id } to create-or-get practice_id before first segment
2. User clicks Start Speaking -> Start browser speech recognition
3. Frontend accumulates transcript text and tracks elapsed duration
4. User clicks Stop -> Finalize transcript segment
5. POST /api/v1/practice/{practice_id}/transcript-segments
6. Backend stores transcript_text, duration_seconds, segment_order
7. Backend returns updated total_duration_seconds and combined_transcript; refresh "Your answer"
8. On Get Feedback (`POST /api/v1/practice`), backend computes merged transcript from stored transcript segments by `practice_id`
```

**Empty states (testable):**

- **GET returned 404** (no `Practice` row yet): **Your answer** is empty; client must create-or-get via **POST .../practices** before the first segment save.
- **GET returned 200** with **empty `transcript_segments`**: **Your answer** is empty but client **has** `practice_id` and may upload segments immediately.

---

## 3. API Endpoint

### 3.1 Save Transcript Segment

```
POST /api/v1/practice/{practice_id}/transcript-segments

Content-Type: application/json

Request Body:
{
  "transcript_text": "I would put a load balancer in front of stateless API servers...",
  "duration_seconds": 95
}

Response (200 OK):
{
  "practice_id": 789,
  "segment_order": 3,
  "duration_seconds": 95,
  "total_duration_seconds": 245,
  "combined_transcript": "I would start with... Next, I would put a load balancer..."
}

Error Responses:
- 400 Bad Request (empty transcript, duration exceeds max)
- 403 Forbidden
- 404 Not Found (invalid `practice_id`)
- 500 Internal Server Error
```

### 3.2 Load speech capture state (question load — primary)

Canonical contract: **`resource/prds/backend/01-practice-session-management.md`** §5.

```
GET /api/v1/practice-main/{practice_main_id}/practices?question_id={question_id}
```

Returns a single **`PracticeQuestionStateDto`**: `practice_id`, `practice_main_id`, `question_id`, `transcript_segments`, `total_duration_seconds`, `combined_transcript`. **404** if no row exists yet.

### 3.3 Create or get Practice before first segment

If §3.2 returns **404**, call (same doc §6):

```
POST /api/v1/practice-main/{practice_main_id}/practices
Content-Type: application/json

{ "question_id": 456 }
```

Response body matches §3.2 success shape (typically empty segments). **201** on first creation, **200** if row already existed.

### 3.4 Load by `practice_id` (optional)

When the client already has `practice_id`, it may use:

```
GET /api/v1/practice/{practice_id}
```

Response shape matches [Audio Recording – Backend APIs](backend/03-audio-recording.md) §4.

---

## 4. Error Handling

### 4.1 Speech-Capture Error Scenarios

**Validation and UX Rules:**
- Display: Inline error messages below inputs
- Examples:
  - "Whiteboard cannot be empty"
  - "No transcript was captured. Please try again."
  - "Total speaking time cannot exceed 10 minutes"
  - "Current segment has exceeded 3 minutes. Consider stopping and saving."
- Submission is not blocked for empty or missing speech transcript in V1

**Browser Capability Errors:**
- If speech recognition is unavailable in the current browser, show supported-browser guidance and allow user to continue without audio
- If microphone permission is denied, show recovery instructions and allow retry

**Backend 400 Bad Request (validation):**
```json
{
  "error": "Validation failed",
  "details": [
    {
      "field": "transcript_text",
      "message": "Transcript text cannot be empty"
    }
  ]
}
```

---

## 5. Testing Scenarios

**Speech Capture:**
- **15.** User speaks on High Level Design -> Transcript segment saves successfully
- **16.** User submits without speech on Deep Dive -> Submission succeeds, AI evaluates whiteboard-only input
- **17.** Current segment crosses 3 minutes -> Non-blocking warning is shown
- **18.** Total speaking duration reaches 10 minutes -> Capture stops, warning shown
- **19.** User stops speaking, closes browser, returns later -> GET practice-main/.../practices restores transcript and total duration in **Your answer**
- **32.** GET practices returns 404 -> Empty **Your answer**; after POST create-or-get, first segment saves successfully
- **33.** GET practices returns 200 with empty segments -> Empty **Your answer**; segment POST works without extra create
- **20.** User adds another speech segment -> Combined transcript and total duration update correctly
- **21.** User cannot delete prior speech segments -> UI offers continue-only flow

**Edge Cases:**
- **30.** Browser does not support speech recognition -> User sees clear guidance
- **31.** Speech recognition returns empty text -> Segment not saved, helpful retry message shown

---

## 6. UI Component Specifications

### 6.1 Speech Capture Button

**Visibility:**
- Only visible for "High Level Design" and "Deep Dive" questions

**Visual States:**
- **Idle:** Microphone icon, "Start Speaking" label
- **Listening:** Red dot, current segment timer, "Stop" label
- **Saved:** Checkmark icon, total duration display, "Continue Speaking" label

**Behavior:**
- Not required for submission in V1 on applicable question types
- Get Feedback button is not disabled solely due to missing spoken transcript
- Total captured duration is always visible after the first saved segment
- Prior saved speech is presented as part of the current answer state
- No delete or remove controls are shown for transcript segments in V1
- Show non-blocking warning when current segment exceeds 3 minutes; auto-stop capture at 10 minutes cumulative duration

**Design System Alignment:**
- Follows standard button specifications (see Foundation PRD)
- Primary action styling when capture is available
- Error/warning styling for active listening state and unsupported-browser states

### 6.2 Your answer (read-only transcript)

**Placement:** Directly **below** the speech capture control (§6.1).

**Title:** **Your answer** (section heading).

**Content:** Read-only display of **`combined_transcript`** (or equivalent text assembled from loaded segments). Long text may scroll within the region.

**Empty state:** When there is no saved speech yet, show a short placeholder (e.g. “No spoken answer saved yet.”) or equivalent; same region remains visible so layout does not jump.

**Updates:** After each successful segment save, refresh from the save response or refetch §3.2 so displayed text matches server truth.
