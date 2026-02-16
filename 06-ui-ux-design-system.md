# UI/UX & Design System PRD

**Document Type:** Feature Product Requirements Document  
**Purpose:** Defines the design system, UI/UX specifications, testing requirements, and error handling patterns for the System Design Practice Application. All feature PRDs use this design system for consistent implementation.

**Version:** 2.0  
**Date:** February 13, 2026  
**Status:** Draft  

---

## References

See [Foundation PRD](00-foundation.md) for technical stack. This design system is used by all feature PRDs.

---

## 1. Feature Overview

This design system establishes the visual language, component specifications, and interaction patterns for the System Design Practice Application. It ensures consistency across the two-panel interface, whiteboard, progress indicators, and all interactive elements. The design system serves as the single source of truth for colors, typography, spacing, components, responsive behavior, accessibility, and error handling—enabling rapid, cohesive development across all feature PRDs.

---

## 2. Design Tokens

### 2.1 Colors
- **Primary:** Blue (#3B82F6)
- **Success:** Green (#10B981)
- **Warning:** Yellow (#F59E0B)
- **Error:** Red (#EF4444)
- **Background:** White (#FFFFFF)
- **Surface:** Light Gray (#F9FAFB)
- **Border:** Gray (#E5E7EB)
- **Text Primary:** Dark Gray (#111827)
- **Text Secondary:** Medium Gray (#6B7280)

### 2.2 Typography
- **Heading 1:** 32px, Semi-bold
- **Heading 2:** 24px, Semi-bold
- **Body:** 16px, Regular
- **Small:** 14px, Regular
- **Code:** Monospace, 14px

### 2.3 Spacing
- **Base unit:** 8px
- **Padding:** 16px, 24px, 32px
- **Margins:** 8px, 16px, 24px

---

## 3. Component Specifications

### 3.1 Progress Dots
- **Size:** 12px diameter
- **Spacing:** 8px between dots
- **Colors:**
  - Complete: Blue (#3B82F6)
  - Incomplete: Gray (#D1D5DB)
  - Current: Blue border, white fill
- **Hover:** Scale 1.2x, show tooltip

### 3.2 Buttons
- **Primary:** Blue background, white text, 40px height
- **Secondary:** White background, blue border and text
- **Disabled:** Gray background, gray text, 50% opacity
- **Hover:** Darken 10%
- **Active:** Darken 20%

### 3.3 Question Type Badges
- **Shape:** Pill shape, 8px padding
- **Color coded:**
  - Functional Req: Blue
  - Non-Functional Req: Purple
  - Entities: Green
  - API: Orange
  - High Level Design: Red
  - Deep Dive: Dark Red

### 3.4 Whiteboard Section Borders
- **Default:** 1px solid #E5E7EB
- **Active:** 2px solid #3B82F6, box-shadow
- **Inactive:** 1px solid #E5E7EB, 70% opacity

### 3.5 Loading States
- **Spinner:** Blue rotating circle
- **Skeleton screens** for content loading
- **Progress bar** for audio upload

---

## 4. Responsive Behavior

### 4.1 Desktop (1920x1080 and above)
- Full two-panel layout (20:80)
- All features visible

### 4.2 Laptop (1366x768)
- Same layout, slightly compressed
- Minimum whiteboard width: 800px

### 4.3 Tablet (768px - 1024px)
- Stacked layout: Left panel on top, whiteboard below
- Or tabs to switch between panels

### 4.4 Mobile (< 768px)
- Not recommended for whiteboard experience
- Show warning: "For best experience, use desktop"
- Alternative: Show question list only, practice on desktop

---

## 5. Animations and Transitions
- **Page transitions:** 200ms fade
- **Button hover:** 150ms ease
- **Tab switching:** 200ms slide
- **Modal open/close:** 250ms scale
- **Toast notifications:** 300ms slide from top
- **Section focus change:** 200ms border and shadow transition

---

## 6. Accessibility

- ARIA labels on all interactive elements
- Keyboard navigation: Tab, Shift+Tab, Enter, Escape
- Screen reader support for progress dots
- High contrast mode support
- Focus indicators (2px outline)
- Alt text for icons
- Minimum touch target size: 44x44px

---

## 7. Testing Requirements

### 7.1 Test Scenarios

**Practice Session Management:**
1. User starts new practice → Creates PracticeMain with status='practicing'
2. User returns to QuestionMain → Resumes at first incomplete question
3. User completes all questions → Status changes to 'completed', moves to history
4. Multiple users practice same QuestionMain → Isolated sessions

**Whiteboard Interaction:**
5. User draws in active section → Content saved in correct section
6. User tries to edit inactive section → Blocked with helpful message
7. User zooms whiteboard → All sections zoom together
8. User adds 100+ elements → Performance remains smooth
9. Auto-save triggers → Content persisted after 5 seconds inactivity
10. User navigates away mid-edit → Draft saved and recoverable

**Progress and Navigation:**
11. User clicks progress dot → Navigates to that question, focuses correct section
12. User clicks "Next Question" → Moves to next in sequence
13. User on last question clicks "Complete" → Shows review page
14. Progress dots update → Reflect answered vs unanswered questions

**Audio Recording:**
15. User records audio on High Level Design → Audio saves successfully
16. User tries to submit without audio on Deep Dive → Blocked with error message
17. Audio > 10 minutes → Recording stops, shows warning
18. Audio file > 50MB → Upload fails with helpful error
19. User re-records audio → Previous recording replaced

**Feedback Generation:**
20. User submits practice → Feedback generated within 30 seconds
21. User submits with audio → Feedback includes audio evaluation
22. LLM service fails → Graceful error message, retry option
23. User edits and resubmits → New feedback generated, old preserved in DB
24. Feedback includes score → Score displayed prominently

**Edge Cases:**
25. User loses internet connection → Shows offline message, queues saves
26. Browser crashes mid-practice → Can resume from last auto-save
27. Concurrent edits in different tabs → Last save wins, conflict warning
28. Very long feedback text → Scrollable, properly formatted
29. Empty whiteboard submission → Validation error
30. Malformed audio file → Upload rejected with clear message

### 7.2 Test Coverage Requirements
- Unit tests: 80%+ coverage
- Integration tests: All API endpoints
- E2E tests: Critical user flows
- Performance tests: Load testing for 1000 concurrent users
- Security tests: Authentication, authorization, input validation
- Accessibility tests: WCAG 2.1 Level AA compliance

### 7.3 Testing Tools
- Frontend: Jest, React Testing Library, Cypress
- Backend: pytest, unittest, Postman
- Performance: Apache JMeter, Lighthouse
- Accessibility: axe-core, WAVE
- Load: Locust, k6

---

## 8. Error Handling

### 8.1 Frontend Error Scenarios

**Network Errors:**
- Display: "Connection lost. Retrying..."
- Action: Auto-retry with exponential backoff
- User action: Manual retry button

**Validation Errors:**
- Display: Inline error messages below inputs
- Examples:
  - "Audio recording required for this question"
  - "Whiteboard cannot be empty"
  - "Audio file too large (max 50MB)"

**Server Errors (5xx):**
- Display: "Something went wrong. Please try again."
- Action: Log error details to monitoring service
- User action: Retry button, contact support link

**Session Expired:**
- Display: "Session expired. Please log in again."
- Action: Redirect to login page
- Preserve: Save draft before redirect

### 8.2 Backend Error Responses

**400 Bad Request:**
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

**401 Unauthorized:**
```json
{
  "error": "Authentication required",
  "message": "Please log in to continue"
}
```

**403 Forbidden:**
```json
{
  "error": "Access denied",
  "message": "You don't have permission to access this practice session"
}
```

**404 Not Found:**
```json
{
  "error": "Resource not found",
  "message": "QuestionMain with id 123 does not exist"
}
```

**429 Too Many Requests:**
```json
{
  "error": "Rate limit exceeded",
  "message": "You can only request feedback 10 times per hour",
  "retry_after": 1800
}
```

**500 Internal Server Error:**
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred. Our team has been notified."
}
```

### 8.3 AI Service Failures

**LLM Timeout:**
- Retry once after 30 seconds
- If still fails: "Feedback generation is taking longer than expected. We'll email you when it's ready."
- Queue for background processing

**LLM Service Down:**
- Display: "Feedback service temporarily unavailable. Please try again in a few minutes."
- Action: Save practice, allow user to continue to next question
- Background: Retry every 5 minutes for 1 hour

**Invalid LLM Response:**
- Fallback: Generic feedback based on question type
- Log: Alert engineering team
- Display: "Feedback generated with limited detail. Our team is working to improve this."

---

## Appendix A: Example Question Configuration

```json
{
  "question_main_id": 1,
  "name": "Design Twitter",
  "description": "Design a social media platform that allows users to post tweets, follow other users, and see a personalized feed.",
  "write_up": "# Twitter System Design - Sample Solution\n\n## Functional Requirements...",
  "questions": [
    {
      "question_id": 1,
      "order": 1,
      "type": "Functional Req",
      "name": "Define Functional Requirements",
      "description": "List and describe the core functional requirements for Twitter. Consider user actions, content creation, and consumption patterns.",
      "whiteboard_section": 1,
      "requires_recording": false
    },
    {
      "question_id": 2,
      "order": 2,
      "type": "Non-Functional Req",
      "name": "Define Non-Functional Requirements",
      "description": "Specify scalability, performance, availability, and consistency requirements.",
      "whiteboard_section": 2,
      "requires_recording": false
    },
    {
      "question_id": 3,
      "order": 3,
      "type": "Entities",
      "name": "Design Data Models",
      "description": "Define the core entities (User, Tweet, Follow, etc.) and their relationships.",
      "whiteboard_section": 3,
      "requires_recording": false
    },
    {
      "question_id": 4,
      "order": 4,
      "type": "API",
      "name": "Define API Endpoints",
      "description": "Specify the key API endpoints needed (POST /tweet, GET /feed, POST /follow, etc.).",
      "whiteboard_section": 4,
      "requires_recording": false
    },
    {
      "question_id": 5,
      "order": 5,
      "type": "High Level Design",
      "name": "Create High-Level Architecture",
      "description": "Design the overall system architecture including major components (API Gateway, Services, Databases, Caches, Message Queues).",
      "whiteboard_section": 5,
      "requires_recording": true
    },
    {
      "question_id": 6,
      "order": 6,
      "type": "Deep Dive",
      "name": "Deep Dive: Feed Generation",
      "description": "Explain in detail how you would implement the personalized feed generation system. Discuss fan-out strategies, caching, and ranking.",
      "whiteboard_section": 5,
      "requires_recording": true
    }
  ]
}
```
