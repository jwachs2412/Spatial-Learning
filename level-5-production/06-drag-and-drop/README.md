`Level 5` **Step 6 of 9** — Drag & Drop

# 06 — Drag & Drop: dnd-kit with Optimistic Updates

## Spatial Orientation

This is the interaction that makes CollabBoard feel like a real product. Users drag cards between lists — "To Do" to "In Progress" — and the UI updates **instantly**. The backend confirms the move in the background. If it fails, the card snaps back.

The drag-and-drop system has three layers: the **dnd-kit library** handles pointer/touch/keyboard events, the **Redux store** manages optimistic state, and the **API layer** persists the move to the database.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DRAG & DROP ARCHITECTURE                          │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                        dnd-kit Layer                           │  │
│  │                                                                │  │
│  │  Sensors (pointer, keyboard) → Events (start, over, end)      │  │
│  │  DndContext → SortableContext → useSortable → DragOverlay      │  │
│  └──────────────────────┬─────────────────────────────────────────┘  │
│                         │ onDragEnd                                   │
│  ┌──────────────────────▼─────────────────────────────────────────┐  │
│  │                      Redux Layer                               │  │
│  │                                                                │  │
│  │  1. Snapshot current state                                     │  │
│  │  2. dispatch(moveCardOptimistic) → UI updates instantly        │  │
│  │  3. On failure → dispatch(revertBoard(snapshot))               │  │
│  └──────────────────────┬─────────────────────────────────────────┘  │
│                         │ API call                                    │
│  ┌──────────────────────▼─────────────────────────────────────────┐  │
│  │                       API Layer                                │  │
│  │                                                                │  │
│  │  PUT /api/cards/:id/move  →  { targetListId, targetPosition } │  │
│  │  Server runs transaction: close gap → make room → move card   │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## How dnd-kit Works

dnd-kit is a modular drag-and-drop toolkit for React. Unlike monolithic libraries, it lets you compose only the pieces you need.

### Core Concepts

| Concept | What It Does |
|---------|-------------|
| **DndContext** | Wraps the entire drag area. Provides drag events (`onDragStart`, `onDragOver`, `onDragEnd`). |
| **useDroppable** | Marks a container as a valid drop target. |
| **useDraggable** | Marks an element as draggable. |
| **useSortable** | Combines draggable + droppable for items in a sorted list. Handles reordering automatically. |
| **SortableContext** | Wraps a list of sortable items. Tracks their order and provides sorting strategy. |
| **DragOverlay** | Renders a floating preview of the dragged item. Separate from the original element. |
| **Sensors** | Detect input types: mouse/pointer, touch, keyboard. Each sensor has activation constraints. |

### Component Tree

```
DndContext
├── SortableContext (List: "To Do")
│   ├── SortableCard (Card A)
│   ├── SortableCard (Card B)
│   └── SortableCard (Card C)
├── SortableContext (List: "In Progress")
│   ├── SortableCard (Card D)
│   └── SortableCard (Card E)
└── DragOverlay (floating card while dragging)
```

Each `SortableContext` wraps one list column. Inside, each card uses `useSortable` which makes it both draggable and a drop target. The `DragOverlay` renders outside the list hierarchy so it can float freely across the board.

> [!NOTE]
> **Technical:** `DragOverlay` uses a React portal to render outside the normal DOM hierarchy. This avoids `overflow: hidden` clipping and `transform` stacking context issues that would trap the dragged element inside its parent list.
>
> **Plain English:** If the floating card were rendered inside its list column, scrolling and CSS clipping would make it disappear when dragged outside. The overlay renders at the top level of the page, so it floats freely.

---

## Implementation

### Install Note

> [!IMPORTANT]
> The dnd-kit packages were already installed in Step 2:
> - `@dnd-kit/core`
> - `@dnd-kit/sortable`
> - `@dnd-kit/utilities`
>
> No new packages needed for this step.

---

### Step 1: Create SortableCard.tsx

This component wraps `CardItem` with drag-and-drop capabilities.

> [!IMPORTANT]
> **You should be in:** `collab-board/`

Create `client/src/features/board/SortableCard.tsx`:

```typescript
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import CardItem from './CardItem';
import type { Card } from '../../types';

interface SortableCardProps {
  card: Card;
  onCardClick: (card: Card) => void;
}

export default function SortableCard({ card, onCardClick }: SortableCardProps) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({
    id: card.id,
    data: {
      type: 'card',
      card,
    },
  });

  const style: React.CSSProperties = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.4 : 1,
  };

  return (
    <div
      ref={setNodeRef}
      style={style}
      className={`sortable-card ${isDragging ? 'card-item--dragging' : ''}`}
      {...attributes}
      {...listeners}
      aria-roledescription="draggable card"
    >
      <CardItem card={card} onClick={() => onCardClick(card)} />
    </div>
  );
}
```

**What each hook return does:**

| Return Value | Purpose |
|-------------|---------|
| `setNodeRef` | Attaches to the DOM element so dnd-kit can track its position |
| `attributes` | ARIA attributes for accessibility (`role`, `tabIndex`, etc.) |
| `listeners` | Event handlers for drag initiation (pointer down, keyboard) |
| `transform` | The x/y offset while dragging — applied as CSS transform |
| `transition` | CSS transition string for smooth animations |
| `isDragging` | Boolean — true while this item is being dragged |

The `data` object in `useSortable` lets us pass metadata through drag events — we use it to identify the card being dragged and its source list.

---

> [!TIP]
> **Session Break** — You've learned dnd-kit concepts and built the SortableCard component. Save your work and take a break.
> When you return, you'll update ListColumn and BoardView to complete the drag-and-drop flow.

---

### Step 2: Update ListColumn.tsx

Wrap the cards inside each list with `SortableContext` so dnd-kit knows they are sortable.

Update `client/src/features/board/ListColumn.tsx`:

```typescript
import { useDroppable } from '@dnd-kit/core';
import {
  SortableContext,
  verticalListSortingStrategy,
} from '@dnd-kit/sortable';
import SortableCard from './SortableCard';
import type { ListWithCards, Card } from '../../types';

interface ListColumnProps {
  list: ListWithCards;
  onAddCard: (listId: number) => void;
  onCardClick: (card: Card) => void;
}

export default function ListColumn({ list, onAddCard, onCardClick }: ListColumnProps) {
  const { setNodeRef, isOver } = useDroppable({
    id: `list-${list.id}`,
    data: {
      type: 'list',
      listId: list.id,
    },
  });

  return (
    <div className={`list-column ${isOver ? 'list-column--over' : ''}`}>
      <div className="list-column__header">
        <h3>{list.name}</h3>
        <span className="list-column__count">{list.cards.length}</span>
      </div>

      <div ref={setNodeRef} className="list-column__cards">
        <SortableContext
          items={list.cards.map((c) => c.id)}
          strategy={verticalListSortingStrategy}
        >
          {list.cards.map((card) => (
            <SortableCard
              key={card.id}
              card={card}
              onCardClick={onCardClick}
            />
          ))}
        </SortableContext>
      </div>

      <button
        className="list-column__add-btn"
        onClick={() => onAddCard(list.id)}
      >
        + Add Card
      </button>
    </div>
  );
}
```

**Key details:**

- `useDroppable` makes the list container a drop target — when a card is dragged over it, `isOver` becomes `true` (used for visual highlighting).
- `SortableContext` receives `items` as an array of card IDs. The order of this array determines the visual sort order.
- `verticalListSortingStrategy` tells dnd-kit these items are stacked vertically, so it calculates insertion points along the Y axis.
- The `id` for the droppable is prefixed with `list-` to distinguish list containers from card IDs in drag events.

---

### Step 3: Update BoardView.tsx

This is where all the drag logic comes together.

Update `client/src/features/board/BoardView.tsx`:

```typescript
import { useState, useCallback } from 'react';
import {
  DndContext,
  DragOverlay,
  closestCorners,
  PointerSensor,
  KeyboardSensor,
  useSensor,
  useSensors,
  type DragStartEvent,
  type DragEndEvent,
  type DragOverEvent,
} from '@dnd-kit/core';
import { sortableKeyboardCoordinates } from '@dnd-kit/sortable';
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import {
  moveCardOptimistic,
  revertBoard,
  selectCurrentBoard,
} from './boardSlice';
import { moveCard, fetchBoard } from '../../services/api';
import ListColumn from './ListColumn';
import CardItem from './CardItem';
import type { Card, BoardWithLists } from '../../types';

export default function BoardView() {
  const dispatch = useAppDispatch();
  const board = useAppSelector(selectCurrentBoard);
  const [activeCard, setActiveCard] = useState<Card | null>(null);

  // --- Sensors ---
  const pointerSensor = useSensor(PointerSensor, {
    activationConstraint: {
      distance: 5, // 5px movement before drag starts (prevents accidental drags)
    },
  });

  const keyboardSensor = useSensor(KeyboardSensor, {
    coordinateGetter: sortableKeyboardCoordinates,
  });

  const sensors = useSensors(pointerSensor, keyboardSensor);

  // --- Drag Handlers ---
  const handleDragStart = useCallback((event: DragStartEvent) => {
    const { active } = event;
    const card = active.data.current?.card as Card | undefined;
    if (card) {
      setActiveCard(card);
    }
  }, []);

  const handleDragEnd = useCallback(
    async (event: DragEndEvent) => {
      const { active, over } = event;
      setActiveCard(null);

      if (!over || !board) return;

      const cardId = active.id as number;
      const activeData = active.data.current;
      const overData = over.data.current;

      if (!activeData) return;

      // Determine the target list and position
      let targetListId: number;
      let targetPosition: number;

      if (overData?.type === 'list') {
        // Dropped on an empty list
        targetListId = overData.listId;
        targetPosition = 0;
      } else if (overData?.type === 'card') {
        // Dropped on another card — find that card's list and position
        const overCard = overData.card as Card;
        targetListId = overCard.list_id;
        targetPosition = overCard.position;
      } else {
        // Dropped on a list container (via SortableContext)
        const overId = String(over.id);
        if (overId.startsWith('list-')) {
          targetListId = Number(overId.replace('list-', ''));
          targetPosition = 0;
        } else {
          return;
        }
      }

      // Find source list
      const sourceList = board.lists.find((l) =>
        l.cards.some((c) => c.id === cardId)
      );
      if (!sourceList) return;

      const sourceCard = sourceList.cards.find((c) => c.id === cardId);
      if (!sourceCard) return;

      // Skip if dropped in the same position
      if (
        sourceCard.list_id === targetListId &&
        sourceCard.position === targetPosition
      ) {
        return;
      }

      // --- Optimistic Update Pattern ---
      // 1. Snapshot the current board state
      const snapshot: BoardWithLists = JSON.parse(JSON.stringify(board));

      // 2. Dispatch optimistic update (UI moves instantly)
      dispatch(
        moveCardOptimistic({
          cardId,
          sourceListId: sourceList.id,
          targetListId,
          targetPosition,
        })
      );

      try {
        // 3. Call the API
        await moveCard(cardId, targetListId, targetPosition);

        // 4. On success — fetch fresh board to reconcile with server truth
        dispatch(fetchBoard(board.id));
      } catch (error) {
        // 5. On failure — revert to snapshot
        console.error('Move failed, reverting:', error);
        dispatch(revertBoard(snapshot));
      }
    },
    [board, dispatch]
  );

  if (!board) return null;

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCorners}
      onDragStart={handleDragStart}
      onDragEnd={handleDragEnd}
    >
      <div className="board-view">
        <div className="board-view__header">
          <h2>{board.name}</h2>
        </div>

        <div className="board-view__lists">
          {board.lists.map((list) => (
            <ListColumn
              key={list.id}
              list={list}
              onAddCard={(listId) => {
                /* open add card modal */
              }}
              onCardClick={(card) => {
                /* open card detail modal */
              }}
            />
          ))}
        </div>
      </div>

      {/* DragOverlay renders the floating card preview */}
      <DragOverlay dropAnimation={null}>
        {activeCard ? (
          <div className="drag-overlay">
            <CardItem card={activeCard} onClick={() => {}} />
          </div>
        ) : null}
      </DragOverlay>
    </DndContext>
  );
}
```

**Walk through the flow:**

```
1. User presses and moves pointer 5px on Card D
   └── PointerSensor fires → onDragStart
       └── setActiveCard(Card D) → DragOverlay shows floating Card D

2. User drags Card D over the "Done" list
   └── closestCorners detects nearest drop target

3. User releases Card D
   └── onDragEnd fires with active (Card D) and over (target)
       ├── Snapshot current board state (deep copy)
       ├── dispatch(moveCardOptimistic) → Card D moves in UI instantly
       ├── API call: PUT /api/cards/4/move { targetListId: 3, targetPosition: 0 }
       ├── SUCCESS → dispatch(fetchBoard) to reconcile
       └── FAILURE → dispatch(revertBoard(snapshot)) → Card D snaps back
```

> [!NOTE]
> **Technical:** `closestCorners` is the collision detection algorithm. It calculates which droppable container's corners are closest to the pointer. Other options include `closestCenter`, `rectIntersection`, and `pointerWithin`. `closestCorners` works best for Kanban boards because it handles both reordering within a list and moving between lists.
>
> **Plain English:** When you drag a card, dnd-kit needs to figure out "where is the user trying to drop this?" It measures the distance between your pointer and the corners of each potential drop zone, then picks the closest one.

---

### Step 4: Update boardSlice.ts

Add the `moveCardOptimistic` and `revertBoard` reducers to handle instant UI updates and rollback.

Update `client/src/features/board/boardSlice.ts` — add these reducers to the existing slice:

```typescript
import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../../app/store';
import type { BoardWithLists } from '../../types';
import * as api from '../../services/api';

interface BoardState {
  currentBoard: BoardWithLists | null;
  loading: boolean;
  error: string | null;
}

const initialState: BoardState = {
  currentBoard: null,
  loading: false,
  error: null,
};

// Async thunk to fetch a board
export const fetchBoard = createAsyncThunk(
  'board/fetchBoard',
  async (boardId: number) => {
    return await api.getBoard(boardId);
  }
);

const boardSlice = createSlice({
  name: 'board',
  initialState,
  reducers: {
    // --- Optimistic card move ---
    moveCardOptimistic(
      state,
      action: PayloadAction<{
        cardId: number;
        sourceListId: number;
        targetListId: number;
        targetPosition: number;
      }>
    ) {
      if (!state.currentBoard) return;

      const { cardId, sourceListId, targetListId, targetPosition } = action.payload;

      // Find the source list
      const sourceList = state.currentBoard.lists.find(
        (l) => l.id === sourceListId
      );
      if (!sourceList) return;

      // Find and remove the card from the source list
      const cardIndex = sourceList.cards.findIndex((c) => c.id === cardId);
      if (cardIndex === -1) return;
      const [card] = sourceList.cards.splice(cardIndex, 1);

      // Update positions in the source list (close the gap)
      sourceList.cards.forEach((c, i) => {
        c.position = i;
      });

      // Find the target list
      const targetList = state.currentBoard.lists.find(
        (l) => l.id === targetListId
      );
      if (!targetList) return;

      // Insert the card at the target position
      card.list_id = targetListId;
      card.position = targetPosition;
      targetList.cards.splice(targetPosition, 0, card);

      // Update positions in the target list
      targetList.cards.forEach((c, i) => {
        c.position = i;
      });
    },

    // --- Revert to a previous board state (rollback on API failure) ---
    revertBoard(state, action: PayloadAction<BoardWithLists>) {
      state.currentBoard = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchBoard.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchBoard.fulfilled, (state, action) => {
        state.loading = false;
        state.currentBoard = action.payload;
      })
      .addCase(fetchBoard.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to load board';
      });
  },
});

export const { moveCardOptimistic, revertBoard } = boardSlice.actions;

export const selectCurrentBoard = (state: RootState) => state.board.currentBoard;
export const selectBoardLoading = (state: RootState) => state.board.loading;
export const selectBoardError = (state: RootState) => state.board.error;

export default boardSlice.reducer;
```

**The `moveCardOptimistic` reducer step by step:**

```
Before:
  "To Do":      [Card A (pos 0), Card B (pos 1), Card C (pos 2)]
  "In Progress": [Card D (pos 0), Card E (pos 1)]

Action: moveCardOptimistic({ cardId: B, sourceListId: 1, targetListId: 2, targetPosition: 1 })

Step 1 — Remove Card B from source:
  "To Do":      [Card A, Card C]           ← splice removes Card B

Step 2 — Reindex source:
  "To Do":      [Card A (pos 0), Card C (pos 1)]   ← close the gap

Step 3 — Insert Card B into target at position 1:
  "In Progress": [Card D, Card B, Card E]   ← splice inserts at index 1

Step 4 — Reindex target:
  "In Progress": [Card D (pos 0), Card B (pos 1), Card E (pos 2)]

After:
  "To Do":      [Card A (pos 0), Card C (pos 1)]
  "In Progress": [Card D (pos 0), Card B (pos 1), Card E (pos 2)]
```

> [!NOTE]
> **Technical:** Redux Toolkit uses Immer under the hood, so the `splice` and direct property assignments (`c.position = i`) look like mutations but actually produce a new immutable state. Without Immer, you would need to deep-clone the lists and cards arrays before modifying them.
>
> **Plain English:** Normally in Redux you cannot change state directly — you must return a new object. Redux Toolkit has a built-in library (Immer) that lets you write code that *looks* like it changes state directly, but secretly creates a safe copy behind the scenes.

---

### Step 5: The Optimistic Update Pattern

Here is the complete flow in one diagram:

```
┌──────────────────────────────────────────────────────────────────┐
│                   OPTIMISTIC UPDATE PATTERN                       │
│                                                                   │
│  1. SNAPSHOT                                                      │
│     const snapshot = JSON.parse(JSON.stringify(board));            │
│     └── Deep copy of current state (our safety net)               │
│                                                                   │
│  2. OPTIMISTIC DISPATCH                                           │
│     dispatch(moveCardOptimistic({ ... }));                        │
│     └── UI updates INSTANTLY — card moves to new list             │
│                                                                   │
│  3. API CALL                                                      │
│     await moveCard(cardId, targetListId, targetPosition);         │
│     │                                                             │
│     ├── SUCCESS                                                   │
│     │   dispatch(fetchBoard(board.id));                            │
│     │   └── Fetch fresh data from server to reconcile             │
│     │       (server is the source of truth)                       │
│     │                                                             │
│     └── FAILURE                                                   │
│         dispatch(revertBoard(snapshot));                           │
│         └── Restore the snapshot — card snaps back                │
│             User sees the card return to its original position    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Why this pattern matters:**

Without optimistic updates, dragging a card would:
1. User drops card
2. UI freezes or shows a spinner (waiting for server)
3. Server responds (200-500ms later)
4. Card finally moves in the UI

With optimistic updates:
1. User drops card
2. Card moves **immediately** (0ms perceived latency)
3. Server confirms in background
4. If the server says "no" — card quietly snaps back

This is the same pattern used by Trello, Notion, Linear, and every modern drag-and-drop app.

> [!WARNING]
> **`JSON.parse(JSON.stringify(board))` creates a deep clone** of the board state for the snapshot. This works correctly for serializable data (strings, numbers, arrays, objects) but would break if the state contained functions, Dates, or class instances. Redux state should always be serializable, so this is safe here. For large boards with hundreds of cards, consider a more targeted snapshot (just the affected lists).

---

> [!TIP]
> **Session Break** — You've built the optimistic update pattern with snapshot, dispatch, and rollback. Save your work and take a break.
> When you return, you'll add the API service update, accessibility features, and drag-and-drop styles.

---

### Step 6: API Service Update

Add the `moveCard` function to `client/src/services/api.ts` (if not already present from Step 5):

```typescript
// Add to existing api.ts

export async function moveCard(
  cardId: number,
  targetListId: number,
  targetPosition: number
): Promise<Card> {
  const response = await fetch(`${API_BASE}/api/cards/${cardId}/move`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${getToken()}`,
    },
    body: JSON.stringify({ targetListId, targetPosition }),
  });

  if (!response.ok) {
    throw new Error('Failed to move card');
  }

  return response.json();
}
```

---

### Step 7: Accessibility

Drag-and-drop must work for keyboard users and screen readers.

#### Keyboard Navigation

The `KeyboardSensor` with `sortableKeyboardCoordinates` enables:

| Key | Action |
|-----|--------|
| `Tab` | Focus a card |
| `Space` / `Enter` | Pick up the focused card |
| `Arrow Up` / `Arrow Down` | Move card within the list |
| `Arrow Left` / `Arrow Right` | Move card to adjacent list |
| `Space` / `Enter` | Drop the card at current position |
| `Escape` | Cancel the drag |

#### Screen Reader Announcements

Add announcement callbacks to `DndContext` in `BoardView.tsx`:

```typescript
const announcements = {
  onDragStart({ active }: { active: { id: string | number } }) {
    const card = board?.lists
      .flatMap((l) => l.cards)
      .find((c) => c.id === active.id);
    if (card) {
      return `Picked up card: ${card.title}`;
    }
    return 'Picked up card';
  },
  onDragEnd({ active, over }: { active: { id: string | number }; over: { id: string | number } | null }) {
    const card = board?.lists
      .flatMap((l) => l.cards)
      .find((c) => c.id === active.id);
    if (!card || !over) {
      return 'Card dropped';
    }

    const targetList = board?.lists.find(
      (l) => `list-${l.id}` === String(over.id) || l.cards.some((c) => c.id === over.id)
    );
    if (targetList) {
      return `Dropped card "${card.title}" into list "${targetList.name}"`;
    }
    return 'Card dropped';
  },
  onDragCancel() {
    return 'Drag cancelled. Card returned to original position.';
  },
};
```

Then pass it to `DndContext`:

```typescript
<DndContext
  sensors={sensors}
  collisionDetection={closestCorners}
  onDragStart={handleDragStart}
  onDragEnd={handleDragEnd}
  accessibility={{
    announcements,
  }}
>
```

> [!NOTE]
> **Technical:** dnd-kit uses a live region (`aria-live="assertive"`) to announce drag events to screen readers. The `announcements` object provides custom messages for each lifecycle event. Without these, a screen reader user would have no feedback about what the drag operation is doing.
>
> **Plain English:** Sighted users see the card move across the screen. Blind users using screen readers hear announcements like "Picked up card: Fix login bug" and "Dropped card into list Done." Without these announcements, drag-and-drop would be completely invisible to screen reader users.

---

## CSS Additions

Add these styles to your board stylesheet (e.g., `client/src/features/board/Board.css`):

```css
/* --- Sortable Card --- */
.sortable-card {
  cursor: grab;
  touch-action: none; /* Prevents scroll on touch devices during drag */
}

.sortable-card:active {
  cursor: grabbing;
}

/* Card becomes translucent when being dragged (the original stays in place) */
.card-item--dragging {
  opacity: 0.4;
  border: 2px dashed var(--color-border, #d1d5db);
  background: var(--color-surface-dim, #f3f4f6);
}

/* --- Drag Overlay (the floating preview) --- */
.drag-overlay {
  transform: scale(1.03);
  box-shadow:
    0 12px 28px rgba(0, 0, 0, 0.15),
    0 4px 10px rgba(0, 0, 0, 0.08);
  border-radius: 8px;
  cursor: grabbing;
  pointer-events: none;
}

.drag-overlay .card-item {
  background: var(--color-surface, #ffffff);
  border: 1px solid var(--color-primary, #6366f1);
}

/* --- List Column drop target highlight --- */
.list-column--over {
  background: var(--color-primary-light, #eef2ff);
  border-color: var(--color-primary, #6366f1);
  transition: background-color 150ms ease, border-color 150ms ease;
}

.list-column--over .list-column__cards {
  min-height: 60px; /* Ensures empty lists have a visible drop area */
}

/* --- Smooth transitions for cards being pushed aside --- */
.sortable-card {
  transition: transform 200ms ease;
}
```

**What each class does:**

```
.card-item--dragging       → The "ghost" left behind where the card was
                              (translucent placeholder so the user sees the gap)

.drag-overlay              → The floating card following the pointer
                              (slightly scaled up with a shadow for depth)

.list-column--over         → The list column the user is hovering over
                              (highlighted border/background as a drop hint)
```

---

## Step 8: Commit

> [!IMPORTANT]
> **You should be in:** `collab-board/`

```bash
git add .
git commit -m "feat: add drag-and-drop with dnd-kit and optimistic updates"
```

---

> [!TIP]
> ## Spatial Check-In

1. **Why do we use optimistic updates instead of waiting for the API response before moving the card in the UI?**

<details><summary>Answer</summary>

API calls take 100-500ms depending on network speed. If we waited for the server to respond before moving the card, the user would see a noticeable delay after dropping — the card would hang in its old position, then jump to the new one. Optimistic updates move the card instantly (0ms perceived latency), making the app feel responsive and native. The tradeoff is that we need rollback logic in case the server rejects the move.

</details>

2. **What happens if the API call fails after the optimistic update has already moved the card?**

<details><summary>Answer</summary>

The card snaps back to its original position. Before dispatching the optimistic update, we take a deep-copy snapshot of the board state (`JSON.parse(JSON.stringify(board))`). If the API call throws an error, we `dispatch(revertBoard(snapshot))` which replaces the current board state with the saved snapshot. The user sees the card animate back to where it was, and we could show an error notification ("Failed to move card. Please try again.").

</details>

3. **Why is `DragOverlay` a separate component instead of just styling the card being dragged?**

<details><summary>Answer</summary>

Three reasons. First, the original card stays in the DOM as a translucent placeholder (showing the gap where the card was), while the overlay is a separate copy that follows the pointer. Second, the overlay renders via a portal at the top of the DOM tree, so it is not clipped by `overflow: hidden` or affected by CSS transforms on parent containers — it can float freely across list boundaries. Third, it allows different rendering: the overlay can have a shadow and scale effect while the original has a dashed-border ghost appearance.

</details>

4. **How does keyboard accessibility work for drag-and-drop, and why is it important?**

<details><summary>Answer</summary>

The `KeyboardSensor` allows users to pick up a card with Space/Enter, move it with arrow keys, drop it with Space/Enter, or cancel with Escape. The `sortableKeyboardCoordinates` function maps arrow keys to movement within and between sortable contexts (up/down within a list, left/right between lists). This is important because not all users can use a mouse — people with motor disabilities, repetitive strain injuries, or who simply prefer keyboard navigation need a non-pointer way to reorder cards. Screen reader announcements complement this by providing audio feedback for what is happening during the drag.

</details>

---

| | | |
|:---|:---:|---:|
| [← 05 — Frontend](../05-frontend/) | [Level 5 Overview](../) | [07 — CI/CD & Monitoring →](../07-cicd-monitoring/) |
