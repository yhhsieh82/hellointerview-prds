# PRD: Audio Recording

**Version:** 2.0  
**Date:** March 10, 2026  
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
- Spoken explanation is required for submission on these question types
- When the user clicks `Stop`, the frontend finalizes the current transcript segment and persists it to the related `practice`
- When the user closes the browser and later returns to the same question, previously saved transcript segments are loaded and the user can continue speaking on the same `practice`
- UI displays the total captured duration across all saved transcript segments for the current `practice`
- V1 does not provide delete-recording or segment-removal controls; users can only add more speech and submit
- Max cumulative speaking duration: 10 minutes per `practice`

**Technical Specifications:**
- API: Browser speech recognition (`SpeechRecognition` / Web Speech API or equivalent frontend speech-to-text provider)
- Persisted artifact: transcript text plus duration metadata
- Storage unit: ordered transcript segments per `practice`
- `total_duration_seconds` and `combined_transcript` are derived from ordered transcript segments in V1, not persisted as dedicated aggregate columns
- Language: English in V1
- Raw audio file upload/storage: not supported in V1

**API Flow:**
```
1. User clicks Start Speaking -> Start browser speech recognition
2. Frontend accumulates transcript text and tracks elapsed duration
3. User clicks Stop -> Finalize transcript segment
4. POST /api/v1/practice/{practice_id}/transcript-segments
5. Backend stores transcript_text, duration_seconds, segment_order
6. Backend returns updated total_duration_seconds and transcript summary
7. User later returns -> GET practice data includes saved transcript segments and total duration
```

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
- 400 Bad Request (empty transcript, duration exceeds max, invalid practice)
- 403 Forbidden
- 500 Internal Server Error
```

### 3.2 Load Existing Speech Capture State

The practice fetch endpoint should include the current transcript capture state for applicable questions so the frontend can restore the experience after reload or browser close.

```
GET /api/v1/practice/{practice_id}

Response excerpt:
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

---

## 4. Error Handling

### 4.1 Speech-Capture Error Scenarios

**Validation Errors:**
- Display: Inline error messages below inputs
- Examples:
  - "Spoken explanation is required for this question"
  - "Whiteboard cannot be empty"
  - "No transcript was captured. Please try again."
  - "Total speaking time cannot exceed 10 minutes"

**Browser Capability Errors:**
- If speech recognition is unavailable in the current browser, show a blocking message with supported-browser guidance
- If microphone permission is denied, show recovery instructions and allow retry

**Backend 400 Bad Request (validation):**
```json
{
  "error": "Validation failed",
  "details": [
    {
      "field": "combined_transcript",
      "message": "Spoken explanation is required for this question type"
    }
  ]
}
```

---

## 5. Testing Scenarios

**Speech Capture:**
- **15.** User speaks on High Level Design -> Transcript segment saves successfully
- **16.** User tries to submit without speech on Deep Dive -> Blocked with error message
- **17.** Total speaking duration exceeds 10 minutes -> Capture stops, shows warning
- **18.** User stops speaking, closes browser, returns later -> Saved transcript and total duration are restored
- **19.** User adds another speech segment -> Combined transcript and total duration update correctly
- **20.** User cannot delete prior speech segments -> UI offers continue-only flow

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
- Required for submission on applicable question types
- Get Feedback button disabled when spoken explanation is required but no saved transcript exists
- Total captured duration is always visible after the first saved segment
- Prior saved speech is presented as part of the current answer state
- No delete or remove controls are shown for transcript segments in V1

**Design System Alignment:**
- Follows standard button specifications (see Foundation PRD)
- Primary action styling when capture is available
- Error/warning styling for active listening state and unsupported-browser states
