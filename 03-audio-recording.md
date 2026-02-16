# PRD: Audio Recording

**Version:** 1.0  
**Date:** February 14, 2026  
**Status:** Draft  

See [Foundation PRD](00-foundation.md) for S3 configuration and data models.

---

## Integration Points

This feature is used by the **AI Feedback System**. Audio recordings are transcribed and included in the LLM prompt for evaluation when users submit practice answers for High Level Design and Deep Dive questions.

---

## 1. Feature Overview

Audio recording enables users to provide verbal explanations of their system design solutions. For High Level Design and Deep Dive questions—where users create architecture diagrams—the spoken explanation complements the visual diagram and provides context that cannot be conveyed through diagrams alone. The recording is uploaded to S3, transcribed, and sent to the AI feedback system for comprehensive evaluation.

**Purpose:**
- Capture verbal design rationale and walkthrough
- Enable AI to evaluate both diagram and explanation
- Improve feedback quality for complex architectural questions

---

## 2. Recording Interface

### 2.1 Recording Interface
**Priority:** P0 (Must Have)

**User Story:**  
As a user answering High Level Design or Deep Dive questions, I want to record an audio explanation of my design.

**Acceptance Criteria:**
- Record button appears only for question types: "High Level Design", "Deep Dive"
- Button states:
  - Idle: Microphone icon, "Record"
  - Recording: Red dot, timer, "Stop"
  - Recorded: Checkmark, duration, "Re-record"
- Browser permissions requested on first click
- Recording format: WebM audio
- Max duration: 10 minutes
- Recording required for submission on these question types

**Technical Specifications:**
- API: MediaRecorder API
- Format: audio/webm or audio/mp4
- Sample rate: 48kHz
- Max file size: 50MB
- Codec: Opus

**API Flow:**
```
1. User clicks Record → Start MediaRecorder
2. User clicks Stop → Stop recording, generate Blob
3. POST /api/v1/audio/upload with multipart/form-data
4. Backend uploads to S3
5. Returns audio_url
6. Store audio_url in practice.audio_url
```

---

## 3. API Endpoint

### 3.1 Upload Audio Recording

```
POST /api/v1/audio/upload

Content-Type: multipart/form-data

Request Body:
- file: audio file (max 50MB, formats: webm, mp3, wav)

Response (200 OK):
{
  "audio_url": "https://s3.amazonaws.com/bucket/recordings/user456_practice789_20260213.webm",
  "duration_seconds": 180,
  "file_size_bytes": 2500000
}

Error Responses:
- 400 Bad Request (file too large, invalid format)
- 500 Internal Server Error
```

---

## 4. Error Handling

### 4.1 Audio-Related Error Scenarios

**Validation Errors:**
- Display: Inline error messages below inputs
- Examples:
  - "Audio recording required for this question"
  - "Whiteboard cannot be empty"
  - "Audio file too large (max 50MB)"

**Backend 400 Bad Request (validation):**
```json
{
  "error": "Validation failed",
  "details": [
    {
      "field": "audio_url",
      "message": "Audio recording is required for this question type"
    }
  ]
}
```

---

## 5. Testing Scenarios

**Audio Recording:**
- **15.** User records audio on High Level Design → Audio saves successfully
- **16.** User tries to submit without audio on Deep Dive → Blocked with error message
- **17.** Audio > 10 minutes → Recording stops, shows warning
- **18.** Audio file > 50MB → Upload fails with helpful error
- **19.** User re-records audio → Previous recording replaced

**Edge Cases:**
- **30.** Malformed audio file → Upload rejected with clear message

---

## 6. UI Component Specifications

### 6.1 Record Button

**Visibility:**
- Only visible for "High Level Design" and "Deep Dive" questions

**Visual States:**
- **Idle:** Microphone icon, "Record" label
- **Recording:** Red dot, timer display, "Stop" label
- **Recorded:** Checkmark icon, duration display, "Re-record" label

**Behavior:**
- Required for submission on these question types
- Get Feedback button disabled when recording required but not provided

**Design System Alignment:**
- Follows standard button specifications (see Foundation PRD)
- Primary action styling when in Record state
- Error/warning styling for recording state (red dot indicator)
