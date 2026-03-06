# Whiteboard & Diagramming PRD

**See [Foundation PRD](00-foundation.md) for data models. Integrates with [Practice Session Management](01-practice-session-management.md) for state.**

---

## 1. Feature Overview

The Whiteboard & Diagramming feature provides an interactive canvas for users to practice system design problems. Users work through structured questions across five distinct whiteboard sections—Functional Requirements, Non-Functional Requirements, Entities, API, and High Level Design—all on a **single Excalidraw canvas** divided into 5 named Frame regions. The interface supports blocks, text labels, arrows, connectors, zoom/pan, section focus management, and auto-save. When a question is active, the canvas automatically scrolls and zooms to the corresponding frame, and elements in all other frames are locked. Canvas content is persisted as JSONB on the `PracticeMain.whiteboard_content` field, which acts as the canonical whiteboard for the entire practice session.

---

## 2. Two-Panel Interface

### 2.1 Screen Layout

**Priority:** P0 (Must Have)

**Layout Specification:**
```
┌────────────────────────────────────────────────────────────────────┐
│  ┌──────────┬─────────────────────────────────────────────────┐   │
│  │          │  Single Excalidraw Canvas                        │   │
│  │  Left    │  ┌──────────┬──────────────────────────────┐   │   │
│  │  Panel   │  │[Sec 1]   │                              │   │   │
│  │          │  │  frame   │                              │   │   │
│  │          │  ├──────────┤     [Section 5] frame        │   │   │
│  │ Progress │  │[Sec 2]   │     (active → auto-focused)  │   │   │
│  │ Question │  │  frame   │                              │   │   │
│  │ Controls │  ├──────────┤                              │   │   │
│  │  Tabs    │  │[Sec 3]   │                              │   │   │
│  │  Nav     │  │  frame   │                              │   │   │
│  │          │  ├──────────┤                              │   │   │
│  │          │  │[Sec 4]   │                              │   │   │
│  │          │  │  frame   │                              │   │   │
│  │          │  └──────────┴──────────────────────────────┘   │   │
│  └──────────┴─────────────────────────────────────────────────┘   │
│      20%                        80%                                │
└────────────────────────────────────────────────────────────────────┘
```

**Components:**

**Left Panel (20% screen width):**
1. Progress dots (top)
2. Question type badge
3. Question description
4. Record button (conditional - only for High Level Design & Deep Dive)
5. "Get Feedback" button
6. Content tabs: "How to Answer" / "Feedback"
7. "Next Question" button (bottom)

**Right Panel (80% screen width):**
- One unified Excalidraw canvas with 5 named Frame regions; sections 1–4 stacked on the left, section 5 full-height on the right
- When a question is active, the canvas auto-focuses (scrolls and zooms) to the corresponding frame
- Zoom controls (manual +/- in addition to auto-focus)
- All sections share a single toolbar provided by Excalidraw

### 2.2 Left Panel Components

**Progress Indicators:**
- Display one dot per question
- Visual states:
  - Blue filled: Has at least one completed feedback attempt for this question in the current practice session (Practice + PracticeFeedback exist)
  - Grey filled: No completed feedback attempt yet for this question in the current practice session
  - Grey outline: Currently selected
- Hovering shows question name tooltip
- Clicking navigates to that question

**Question Display:**
- Type badge (color-coded by type)
- Full question description text
- Scrollable if content is long

**Action Buttons:**
- Record button:
  - Only visible for "High Level Design" and "Deep Dive" questions
  - Shows microphone icon
  - States: idle, recording, recorded
  - Required for submission on these question types
- Get Feedback button:
  - Primary action button
  - Disabled states:
    - Recording required but not provided
    - Whiteboard content empty
  - On click: Submits practice and retrieves feedback

**Content Tabs:**
- "How to Answer" tab:
  - Displays general tips and guidelines
  - Question-agnostic advice
  - Visible by default
- "Feedback" tab:
  - Displays AI-generated feedback
  - Auto-switches to this tab after feedback received
  - Shows loading state during generation

**Navigation:**
- "Next Question" button:
  - Moves to next question in sequence
  - On last question: Shows review page instead

---

## 3. Whiteboard Interface

### 3.1 Whiteboard Layout

**Priority:** P0 (Must Have)

**Structure:**
- One Excalidraw canvas containing 5 Frame elements as named section regions
- Sections 1–4: Vertical stack on the left side of the canvas
- Section 5: Full height on the right side of the canvas
- Clear visual boundaries between sections via Excalidraw Frame borders
- Section labels: "Functional Req", "Non-Functional Req", "Entities", "API", "High Level Design"

**Frame Coordinates (canvas units):**

| Frame | Canvas x | Canvas y | Width | Height |
|---|---|---|---|---|
| section_1 | 0 | 0 | 400 | 275 |
| section_2 | 0 | 295 | 400 | 275 |
| section_3 | 0 | 590 | 400 | 275 |
| section_4 | 0 | 885 | 400 | 275 |
| section_5 | 420 | 0 | 1100 | 1160 |

Frame elements carry stable IDs (`frame-section_1` … `frame-section_5`) and are created once on first load if not already present in stored data. All coordinates are in Excalidraw canvas units (not screen pixels).

### 3.2 Diagramming Capabilities

**Priority:** P0 (Must Have)

**All sections support identical features:**
- Create blocks/rectangles
- Create circles/ellipses
- Add text blocks and labels
- Draw arrows (straight, curved)
- Connect blocks with arrows
- Add labels to arrows
- Move elements
- Resize elements
- Layer management (bring forward/send backward)
- Undo/redo
- Clear section
- Selection and multi-select

**Recommended Implementation:**
- Library: Excalidraw (React components)
- Single Excalidraw canvas instance; sections are defined as Excalidraw `frame` elements with stable IDs (`frame-section_1` … `frame-section_5`)
- User-drawn elements inside a frame automatically receive a `frameId` property linking them to that section
- One shared Excalidraw toolbar for the entire canvas

**Element Types:**
Element shapes (rectangle, ellipse, arrow, text, image, etc.) are Excalidraw's native types. The frontend does not define a custom element schema; elements are stored and restored as raw `ExcalidrawElement[]`. Frame elements are of type `'frame'` with a `name` property used for the section label.

### 3.3 Section Focus Management

**Priority:** P0 (Must Have)

**Behavior:**
- Only one section editable at a time (determined by current question)
- When the active section changes, the canvas auto-focuses on the new section and locks all others

**Auto-focus on section change:**
- When `activeSectionKey` changes (due to question navigation), call `excalidrawAPI.scrollToContent(activeFrameAndContainedElements, { fitToContent: true, animate: true })`
- This smoothly pans and zooms the camera to fit the active frame within the viewport
- The user can then zoom/pan freely within the canvas after auto-focus

**Locking inactive sections:**
- Call `excalidrawAPI.updateScene({ elements: allElements.map(el => ({ ...el, locked: el.type !== 'frame' && el.frameId !== activeFrameId })) })`
- Locked elements are visible but cannot be selected, moved, or modified
- Frame elements themselves are never locked (preserving section label visibility)
- Locking is re-applied whenever `activeSectionKey` changes

**Visual indicator for active frame:**
- Active frame: blue stroke (`strokeColor: '#3B82F6'`, `strokeWidth: 2`) on the Excalidraw frame element
- Inactive frames: grey stroke (`strokeColor: '#E5E7EB'`, `strokeWidth: 1`); contained elements have `opacity: 0.7`
- These visual properties are applied via `updateScene` alongside the lock toggle

**Tooltip for inactive areas:**
- When the user hovers over a locked element in an inactive section, show a lightweight HTML overlay label reading "Locked — belongs to [Section Name]"
- The overlay is positioned using the Excalidraw canvas-to-screen coordinate transform
- This replaces any click-based tooltip approach; locked elements do not fire click events

**Section Mapping:**
```javascript
const questionTypeToSection = {
  'Functional Req': 1,
  'Non-Functional Req': 2,
  'Entities': 3,
  'API': 4,
  'High Level Design': 5,
  'Deep Dive': 5
};
```

### 3.4 Zoom and Pan Controls

**Priority:** P0 (Must Have)

**Features:**
- **Auto-focus on section change:** When the active section changes (via question navigation), the canvas automatically scrolls and zooms to fit the active frame in the viewport. This is the primary navigation mechanism and is animated. The zoom level after auto-focus is determined by `fitToContent` and may vary per frame size.
- Manual zoom controls remain available for user-driven zoom within a section after auto-focus
- Zoom range: 50% to 200% (for manual controls)
- Controls:
  - Zoom in button (+)
  - Zoom out button (-)
  - Zoom percentage display (reflects Excalidraw's internal viewport zoom from `appState.zoom`)
  - Reset to 100% button
  - Mouse wheel zoom support
- Pan/scroll when zoomed in
- Minimap (optional) showing viewport position

### 3.5 Auto-Save

**Priority:** P1 (Should Have)

**Behavior:**
- Auto-save session-level whiteboard content (`PracticeMain.whiteboard_content`) every 5 seconds (debounced)
- The entire canvas state (`elements`, `appState`, `files`) is saved as one document. Since inactive sections are locked (preventing changes), a change always originates in the active section.
- Save triggered on:
  - Element creation
  - Element modification
  - Element deletion
  - 5 seconds of inactivity after change
- Visual indicator: "Saving..." / "All changes saved"
- Lock/unlock `updateScene` calls (fired on section navigation) must be excluded from the autosave trigger. Guard by checking whether only `locked` properties changed before starting the save timer.

### 3.6 Submission Semantics (V1)

**Priority:** P1 (Should Have)

**Behavior:**
- There is at most one `Practice` record per question per `PracticeMain` in V1; repeated "Get Feedback" actions for the same question update the same logical attempt.
- Clicking "Get Feedback" captures the current state of the canonical session-level whiteboard (`PracticeMain.whiteboard_content`) together with the active question context to create or update the per-question submission and its feedback.
- Autosave operates only on `PracticeMain.whiteboard_content` and does not by itself create or update per-question `Practice` or `PracticeFeedback` records.

---

## 4. Whiteboard Content Structure

The `whiteboard_content` field on PracticeMain is stored as JSONB with the following structure and is shared across all questions in the session. Section editability is controlled by the current question type, but all sections are persisted together in one document.

**Backend storage format (unchanged — backward-compatible):**

```json
{
  "section_1": {
    "type": "diagram",
    "version": "1.0",
    "elements": [
      {
        "id": "frame-section_1",
        "type": "frame",
        "name": "Functional Req",
        "x": 0, "y": 0, "width": 400, "height": 275
      },
      {
        "id": "elem1",
        "type": "rectangle",
        "frameId": "frame-section_1",
        "x": 100, "y": 50, "width": 200, "height": 100
      }
    ],
    "appState": {},
    "files": {}
  },
  "section_2": { /* similar — includes its own frame element + user elements with frameId: "frame-section_2" */ },
  "section_3": { /* ... */ },
  "section_4": { /* ... */ },
  "section_5": { /* ... */ }
}
```

**Merge on load:** On load, elements from all 5 section buckets are merged into one flat array and passed to the single `<Excalidraw initialData>`. The `appState` from the most recently active section is used (or a default if none).

**Split on save:** On save, elements are partitioned back into the 5 section buckets by their `frameId`. Each section's bucket contains its own frame element (`type: 'frame'`) plus all user-drawn elements whose `frameId` matches. Elements with no `frameId` are assigned to `section_5` as a fallback.

**Frame element stability:** The 5 frame elements have stable, hardcoded IDs (`frame-section_1` … `frame-section_5`). If any frame is missing from stored data on load, it is recreated at its default canvas coordinates (see §3.1 table) before mounting the canvas.

---

## 5. Excalidraw Integration

**Installation:**
```bash
npm install @excalidraw/excalidraw
```

**Component Usage:**

One `<Excalidraw>` instance for the entire whiteboard. Section focus and locking are managed imperatively via the `excalidrawAPI` ref. Do **not** use `viewModeEnabled` — it triggers Excalidraw's internal Jotai store cleanup, causing infinite `setState` loops. Locking is done exclusively via `element.locked`.

```typescript
import { useState, useEffect } from "react";
import { Excalidraw } from "@excalidraw/excalidraw";
import type { ExcalidrawImperativeAPI } from "@excalidraw/excalidraw/types/types";

function WhiteboardCanvas({ initialData, activeSectionKey, onChange }) {
  const [excalidrawAPI, setExcalidrawAPI] = useState<ExcalidrawImperativeAPI | null>(null);

  // Auto-focus and lock when active section changes
  useEffect(() => {
    if (!excalidrawAPI) return;
    const allElements = excalidrawAPI.getSceneElements();
    const activeFrameId = `frame-${activeSectionKey}`;

    // Lock all non-frame elements not belonging to the active frame
    excalidrawAPI.updateScene({
      elements: allElements.map(el => ({
        ...el,
        locked: el.type !== 'frame' && el.frameId !== activeFrameId,
      })),
    });

    // Scroll and zoom to fit the active frame + its contents
    const activeElements = allElements.filter(
      el => el.id === activeFrameId || el.frameId === activeFrameId
    );
    if (activeElements.length > 0) {
      excalidrawAPI.scrollToContent(activeElements, { fitToContent: true, animate: true });
    }
  }, [activeSectionKey, excalidrawAPI]);

  return (
    <Excalidraw
      excalidrawAPI={setExcalidrawAPI}
      initialData={initialData}
      onChange={(elements, appState, files) => {
        onChange({ elements, appState, files });
      }}
      UIOptions={{
        canvasActions: {
          loadScene: false,
          export: false,
          saveAsImage: false,
        },
      }}
    />
  );
}
```

**Data Structure:**
```typescript
// Per-section bucket in storage (one per section_1..section_5 key)
interface WhiteboardSectionState {
  type: "diagram";
  version: string;
  elements: ExcalidrawElement[];  // includes the section's frame element + user elements
  appState: Record<string, unknown>;
  files: Record<string, unknown>;
}

// Top-level storage document
interface WhiteboardContent {
  section_1: WhiteboardSectionState;
  section_2: WhiteboardSectionState;
  section_3: WhiteboardSectionState;
  section_4: WhiteboardSectionState;
  section_5: WhiteboardSectionState;
}

// At runtime, all 5 sections are merged into one flat ExcalidrawElement[]
// before passing to <Excalidraw initialData>. On save, elements are
// partitioned back by frameId into the 5 section buckets.
```

---

## 6. Component Specifications (Whiteboard Section Frames)

Frame visual properties are applied to Excalidraw frame elements via `updateScene` whenever the active section changes.

**Active frame:**
- `strokeColor: '#3B82F6'`
- `strokeWidth: 2`

**Inactive frame:**
- `strokeColor: '#E5E7EB'`
- `strokeWidth: 1`
- All contained (user-drawn) elements: `opacity: 0.7`

**Locked element cursor:**
- `locked: true` on all non-frame elements in inactive sections
- Excalidraw renders a not-allowed cursor when the user hovers over a locked element; no additional CSS is needed

---

## 7. Testing Scenarios

**Whiteboard Interaction:**
5. User draws in active section → Element receives correct `frameId`; content saved to correct section bucket on autosave
6. User tries to interact with inactive section → Elements are locked; cursor shows not-allowed; hover tooltip identifies the section
7. User navigates to next question → Canvas auto-focuses (smooth scroll + zoom) to the new active frame; inactive frames locked
8. User manually zooms/pans after auto-focus → Viewport updates freely within the single canvas
9. User adds 100+ elements → Performance remains smooth (single canvas instance)
10. Auto-save triggers → Entire canvas state partitioned by frameId and persisted after 5 seconds inactivity
11. User navigates away mid-edit → Draft saved and recoverable

**Feedback Generation:**
20. User submits practice → Feedback generated within 30 seconds
