# Whiteboard & Diagramming PRD

**See [Foundation PRD](00-foundation.md) for data models. Integrates with [Practice Session Management](01-practice-session-management.md) for state.**

---

## 1. Feature Overview

The Whiteboard & Diagramming feature provides an interactive canvas for users to practice system design problems. Users work through structured questions across five distinct whiteboard sections—Functional Requirements, Non-Functional Requirements, Entities, API, and High Level Design—each with identical diagramming capabilities. The interface supports blocks, text labels, arrows, connectors, zoom/pan, section focus management, and auto-save. All sections use Excalidraw-based canvases with content persisted as JSONB in the Practice model.

---

## 2. Two-Panel Interface

### 2.1 Screen Layout

**Priority:** P0 (Must Have)

**Layout Specification:**
```
┌────────────────────────────────────────────────────────────────────┐
│  ┌──────────┬─────────────────────────────────────────────────┐   │
│  │          │  Whiteboard (zoom-able)                         │   │
│  │          │  ┌──────────┬──────────────────────────────┐   │   │
│  │  Left    │  │ Sec 1    │                              │   │   │
│  │  Panel   │  │ (canvas) │                              │   │   │
│  │          │  ├──────────┤         Section 5            │   │   │
│  │ Progress │  │ Sec 2    │       (canvas)               │   │   │
│  │ Question │  │ (canvas) │                              │   │   │
│  │ Controls │  ├──────────┤                              │   │   │
│  │  Tabs    │  │ Sec 3    │                              │   │   │
│  │  Nav     │  │ (canvas) │                              │   │   │
│  │          │  ├──────────┤                              │   │   │
│  │          │  │ Sec 4    │                              │   │   │
│  │          │  │ (canvas) │                              │   │   │
│  │          │  └──────────┴──────────────────────────────┘   │   │
│  │          │       20%              80%                      │   │
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
- Whiteboard with 5 sections
- Internal 20:80 split (sections 1-4 left, section 5 right)
- Zoom controls
- Each section is an independent diagramming canvas

### 2.2 Left Panel Components

**Progress Indicators:**
- Display one dot per question
- Visual states:
  - Blue filled: Has practice submission
  - Grey filled: No submission
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
- 5 distinct section blocks
- Sections 1-4: Vertical stack on left (20% width)
- Section 5: Full height on right (80% width)
- Clear visual boundaries between sections
- Section labels: "Functional Req", "Non-Functional Req", "Entities", "API", "High Level Design"

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
- Each section = independent Excalidraw canvas instance
- Shared toolbar per section or global toolbar

**Element Types:**
```typescript
type DiagramElement = 
  | { type: 'rectangle', id: string, x: number, y: number, width: number, height: number, text?: string, fillColor?: string, strokeColor?: string }
  | { type: 'circle', id: string, x: number, y: number, radius: number, text?: string, fillColor?: string, strokeColor?: string }
  | { type: 'text', id: string, x: number, y: number, text: string, fontSize?: number, fontFamily?: string }
  | { type: 'arrow', id: string, startElementId?: string, endElementId?: string, points: [number, number][], label?: string }
```

### 3.3 Section Focus Management

**Priority:** P0 (Must Have)

**Behavior:**
- Only one section editable at a time (determined by current question)
- Active section has visual indicator:
  - Highlighted border (e.g., blue 2px border)
  - Slight shadow or glow effect
  - Higher z-index
- Inactive sections are view-only:
  - Slightly dimmed/grayed out
  - Click shows "This section is for Question X" tooltip
  - Content visible but tools disabled

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
- Zoom affects entire whiteboard (all 5 sections together)
- Zoom range: 50% to 200%
- Controls:
  - Zoom in button (+)
  - Zoom out button (-)
  - Zoom percentage display
  - Reset to 100% button
  - Mouse wheel zoom support
- Pan/scroll when zoomed in
- Minimap (optional) showing viewport position

### 3.5 Auto-Save

**Priority:** P1 (Should Have)

**Behavior:**
- Auto-save whiteboard content every 5 seconds (debounced)
- Save triggered on:
  - Element creation
  - Element modification
  - Element deletion
  - 5 seconds of inactivity after change
- Visual indicator: "Saving..." / "All changes saved"
- No save on read-only sections

---

## 4. Whiteboard Content Structure

The `whiteboard_content` field on Practice is stored as JSONB with the following structure:

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
      },
      {
        "id": "elem2",
        "type": "text",
        "x": 150,
        "y": 200,
        "text": "Handles authentication"
      },
      {
        "id": "elem3",
        "type": "arrow",
        "startElementId": "elem1",
        "endElementId": "elem4",
        "label": "queries",
        "points": [[300, 100], [400, 100]]
      }
    ]
  },
  "section_2": { /* similar structure */ },
  "section_3": { /* similar structure */ },
  "section_4": { /* similar structure */ },
  "section_5": { /* similar structure */ }
}
```

---

## 5. Excalidraw Integration

**Installation:**
```bash
npm install @excalidraw/excalidraw
```

**Component Usage:**
```typescript
import { Excalidraw } from "@excalidraw/excalidraw";

function WhiteboardSection({ sectionId, isEditable, initialData, onChange }) {
  return (
    <div className="whiteboard-section">
      <Excalidraw
        initialData={initialData}
        onChange={(elements, appState, files) => {
          if (isEditable) {
            onChange(sectionId, { elements, appState, files });
          }
        }}
        UIOptions={{
          canvasActions: {
            loadScene: false,
            export: false,
            saveAsImage: false
          }
        }}
        viewModeEnabled={!isEditable}
      />
    </div>
  );
}
```

**Data Structure:**
```typescript
interface WhiteboardContent {
  [key: string]: {
    type: "diagram";
    version: string;
    elements: ExcalidrawElement[];
    appState: AppState;
    files: BinaryFiles;
  };
}
```

---

## 6. Component Specifications (Whiteboard Section Borders)

**Whiteboard Section Borders:**
- Default: 1px solid #E5E7EB
- Active: 2px solid #3B82F6, box-shadow
- Inactive: 1px solid #E5E7EB, 70% opacity

---

## 7. Testing Scenarios

**Whiteboard Interaction:**
5. User draws in active section → Content saved in correct section
6. User tries to edit inactive section → Blocked with helpful message
7. User zooms whiteboard → All sections zoom together
8. User adds 100+ elements → Performance remains smooth
9. Auto-save triggers → Content persisted after 5 seconds inactivity
10. User navigates away mid-edit → Draft saved and recoverable

**Feedback Generation:**
20. User submits practice → Feedback generated within 30 seconds
