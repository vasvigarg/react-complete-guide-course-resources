# Redux Toolkit — Hands-On Practice Guide

> A generalized, implementation-first guide for using Redux Toolkit in any complex React application. Assumes you've read the [Redux with React — Complete Learning Notes](./redux-notes.md).

---

## Table of Contents

1. [Deciding What Goes in Redux](#1-deciding-what-goes-in-redux)
2. [Step-by-Step Setup](#2-step-by-step-setup)
3. [Folder & File Structure](#3-folder--file-structure)
4. [Writing Slices](#4-writing-slices)
5. [Async Logic](#5-async-logic)
6. [React Integration](#6-react-integration)
7. [Common Mistakes to Watch For](#7-common-mistakes-to-watch-for)
8. [Practice Checklist](#8-practice-checklist)

---

## 1. Deciding What Goes in Redux

Before writing any Redux code, categorize every piece of state in your app.

### ✅ Put in Redux (Global / Shared)

- State accessed by **multiple components** across the tree (auth status, user profile, cart, theme)
- State that must **survive component unmounts** (data that persists across route navigation)
- State with **complex update logic** (multiple actions can modify the same data)
- **Server-fetched data** that several components consume (product lists, notifications)

### ❌ Keep in Component State (Local / Transient)

- **Form input values** — managed by `useState` or a form library
- **UI state owned by one component** — dropdown open/close, tooltip visibility, hover effects
- **Derived/computed values** — calculate during render or in selectors, never store
- **Non-serializable values** — functions, class instances, Promises, DOM refs

### Quick Decision Rule

> _"Does more than one unrelated component need this? Does it need to survive a route change?"_
> **Yes → Redux.** **No → `useState` / `useContext`.**

---

## 2. Step-by-Step Setup

### 2.1 — Install Dependencies

```bash
npm install @reduxjs/toolkit react-redux
```

### 2.2 — Create the Store

```js
// src/store/index.js
import { configureStore } from '@reduxjs/toolkit';

import featureAReducer from './features/featureA/featureA-slice';
import featureBReducer from './features/featureB/featureB-slice';
import uiReducer from './features/ui/ui-slice';

const store = configureStore({
  reducer: {
    featureA: featureAReducer,   // → state.featureA
    featureB: featureBReducer,   // → state.featureB
    ui: uiReducer,               // → state.ui
  },
});

export default store;
```

`configureStore` automatically gives you:
- Combined reducers
- Thunk middleware
- Redux DevTools connection
- Serializable + immutability checks (dev only)

### 2.3 — Provide the Store to React

```js
// src/index.js
import { Provider } from 'react-redux';
import store from './store/index';

root.render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

### 2.4 — Plan Your Slices Before Coding

| Question | Why It Matters |
|---|---|
| What **features** need global state? | Each feature becomes a slice |
| What is the **initial shape** of each slice? | Flat state is easier to manage |
| Which slices involve **async operations**? | Those need thunks in a separate actions file |
| Do any slices need a **`changed` flag**? | To separate user edits from server-loaded data |

---

## 3. Folder & File Structure

```
src/
├── store/
│   ├── index.js                              ← configureStore
│   └── features/
│       ├── featureA/
│       │   ├── featureA-slice.js             ← createSlice
│       │   ├── featureA-actions.js           ← async thunks
│       │   └── featureA-selectors.js         ← (optional) reusable selectors
│       ├── featureB/
│       │   ├── featureB-slice.js
│       │   └── featureB-actions.js
│       └── ui/
│           └── ui-slice.js                   ← notifications, modals, global UI toggles
│
├── components/
│   ├── FeatureA/                             ← components consuming featureA state
│   ├── FeatureB/
│   ├── Layout/                               ← app shell, headers, navigation
│   └── UI/                                   ← reusable presentational components
│
├── App.js
└── index.js
```

### Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Slice file | `feature-slice.js` | `auth-slice.js`, `cart-slice.js` |
| Actions file | `feature-actions.js` | `cart-actions.js` |
| Slice name property | camelCase noun | `name: 'cart'` |
| Reducer methods | camelCase verb phrase | `addItem`, `removeItem`, `toggleVisibility` |
| Exported actions | `featureActions` | `cartActions`, `uiActions` |
| Exported reducer | default export | `export default slice.reducer` |

---

## 4. Writing Slices

### Slice Template

```js
// src/store/features/featureName/featureName-slice.js
import { createSlice } from '@reduxjs/toolkit';

const featureNameSlice = createSlice({
  name: 'featureName',
  initialState: {
    items: [],              // arrays default to []
    selectedId: null,       // optional refs default to null
    isActive: false,        // booleans default to false
    changed: false,         // tracks user-driven changes (for sync logic)
  },
  reducers: {
    // Server data — does NOT set changed
    replaceData(state, action) {
      state.items = action.payload.items || [];
    },

    // User action — sets changed = true
    addItem(state, action) {
      state.items.push(action.payload);  // Immer makes this safe
      state.changed = true;
    },

    // User action — mutating a found item is safe under Immer
    updateItem(state, action) {
      const item = state.items.find((i) => i.id === action.payload.id);
      if (item) {
        item.title = action.payload.title;
        state.changed = true;
      }
    },

    // User action — filtering to remove
    removeItem(state, action) {
      state.items = state.items.filter((i) => i.id !== action.payload);
      state.changed = true;
    },

    // Simple toggle
    toggleActive(state) {
      state.isActive = !state.isActive;
    },
  },
});

export const featureNameActions = featureNameSlice.actions;  // named export
export default featureNameSlice.reducer;                      // default export
```

### Key Rules

- **Immer is active inside `createSlice` reducers** — write `state.x = y` or `state.items.push(...)` freely
- **Always return nothing (mutate)** OR **return a new object** — never do both
- **`action.payload`** is the standard — whatever you pass to the action creator becomes `action.payload`
- **Keep each slice under ~100 lines** — split if it grows beyond that

### UI Slice (For Global Feedback)

```js
// src/store/features/ui/ui-slice.js
import { createSlice } from '@reduxjs/toolkit';

const uiSlice = createSlice({
  name: 'ui',
  initialState: {
    notification: null, // { status: 'pending'|'success'|'error', title, message }
  },
  reducers: {
    showNotification(state, action) {
      state.notification = {
        status: action.payload.status,
        title: action.payload.title,
        message: action.payload.message,
      };
    },
    clearNotification(state) {
      state.notification = null;
    },
  },
});

export const uiActions = uiSlice.actions;
export default uiSlice.reducer;
```

---

## 5. Async Logic

### Rule: Reducers Must Stay Pure

- ❌ No `fetch`, `async/await`, timers, or side effects inside reducers
- ✅ All async work goes in **thunks** (separate `*-actions.js` files)

---

### Approach A: Manual Thunks

Best for multi-step flows, cross-slice dispatching, and full control.

```js
// src/store/features/featureName/featureName-actions.js
import { uiActions } from '../ui/ui-slice';
import { featureNameActions } from './featureName-slice';

// ---- FETCH DATA ----
export const fetchData = () => {
  return async (dispatch) => {
    const load = async () => {
      const response = await fetch('https://api.example.com/data.json');
      if (!response.ok) throw new Error('Fetch failed.');
      return response.json();
    };

    try {
      const data = await load();
      dispatch(featureNameActions.replaceData({ items: data }));
    } catch (error) {
      dispatch(uiActions.showNotification({
        status: 'error', title: 'Error!', message: error.message,
      }));
    }
  };
};

// ---- SEND DATA ----
export const sendData = (dataToSend) => {
  return async (dispatch) => {
    dispatch(uiActions.showNotification({
      status: 'pending', title: 'Saving...', message: 'Syncing data.',
    }));

    try {
      const response = await fetch('https://api.example.com/data.json', {
        method: 'PUT',
        body: JSON.stringify(dataToSend),
      });
      if (!response.ok) throw new Error('Save failed.');

      dispatch(uiActions.showNotification({
        status: 'success', title: 'Done!', message: 'Data saved.',
      }));
    } catch (error) {
      dispatch(uiActions.showNotification({
        status: 'error', title: 'Error!', message: error.message,
      }));
    }
  };
};
```

---

### Approach B: `createAsyncThunk`

Best for standard CRUD with auto-generated loading/error lifecycle.

```js
import { createAsyncThunk } from '@reduxjs/toolkit';

export const fetchData = createAsyncThunk('featureName/fetch', async () => {
  const response = await fetch('https://api.example.com/data.json');
  if (!response.ok) throw new Error('Fetch failed.');
  return response.json(); // becomes action.payload in 'fulfilled'
});
```

Handle lifecycle in the slice:

```js
extraReducers: (builder) => {
  builder
    .addCase(fetchData.pending, (state) => { state.status = 'loading'; })
    .addCase(fetchData.fulfilled, (state, action) => {
      state.status = 'succeeded';
      state.items = action.payload;
    })
    .addCase(fetchData.rejected, (state, action) => {
      state.status = 'failed';
      state.error = action.error.message;
    });
},
```

### When to Choose Which

| Manual Thunks | `createAsyncThunk` |
|---|---|
| Cross-slice dispatching needed | Single slice owns the loading state |
| Multi-step async with custom dispatch order | Standard fetch → display pattern |
| Complex error handling with UI feedback | Auto pending/fulfilled/rejected is enough |

---

### Preventing Infinite Loops

When syncing Redux state with a backend, two guards are essential:

```js
// App.js
let isInitial = true; // module-level — persists across renders

function App() {
  const dispatch = useDispatch();
  const data = useSelector((state) => state.featureName);

  // 1. Fetch on mount — runs once
  useEffect(() => {
    dispatch(fetchData());
  }, [dispatch]);

  // 2. Send on change — but NOT on first load
  useEffect(() => {
    if (isInitial) { isInitial = false; return; }
    if (data.changed) {
      dispatch(sendData(data));
    }
  }, [data, dispatch]);
}
```

| Guard | Purpose |
|---|---|
| `isInitial` flag | Skips the send effect when the app first loads and fetch populates state |
| `changed` flag | Only `true` for user-driven actions (add/update/remove), never for `replaceData` |

---

## 6. React Integration

### Reading State — `useSelector`

```js
// ✅ Select only what you need — each call subscribes independently
const items = useSelector((state) => state.featureName.items);
const isActive = useSelector((state) => state.featureName.isActive);

// ❌ Avoid selecting the entire slice — causes re-renders on any change
const everything = useSelector((state) => state.featureName);
```

### Dispatching Actions — `useDispatch`

```js
import { useDispatch } from 'react-redux';
import { featureNameActions } from '../../store/features/featureName/featureName-slice';

const MyComponent = () => {
  const dispatch = useDispatch();

  const handleAdd = () => {
    dispatch(featureNameActions.addItem({ id: '1', title: 'New Item' }));
  };

  const handleRemove = (id) => {
    dispatch(featureNameActions.removeItem(id));
  };

  return ( /* UI with onClick handlers */ );
};
```

### Mixing Local State + Redux

```js
const MyForm = () => {
  const [inputValue, setInputValue] = useState(''); // local — form input
  const dispatch = useDispatch();

  const submitHandler = (e) => {
    e.preventDefault();
    dispatch(featureNameActions.addItem({ id: Date.now().toString(), title: inputValue }));
    setInputValue('');
  };

  return (
    <form onSubmit={submitHandler}>
      <input value={inputValue} onChange={(e) => setInputValue(e.target.value)} />
      <button type="submit">Add</button>
    </form>
  );
};
```

**The input lives in `useState`** because it's transient. Only the final submitted value gets dispatched to Redux.

### Derived Data — Compute, Don't Store

```js
const items = useSelector((state) => state.featureName.items);
const filter = useSelector((state) => state.featureName.filter);

// ✅ Compute during render — no extra state needed
const filtered = items.filter((item) => {
  if (filter === 'active') return item.isActive;
  if (filter === 'done') return !item.isActive;
  return true;
});
```

### Conditional Rendering from Redux State

```js
const isLoggedIn = useSelector((state) => state.auth.isLoggedIn);
const notification = useSelector((state) => state.ui.notification);

return (
  <>
    {notification && <Notification {...notification} />}
    {!isLoggedIn && <LoginForm />}
    {isLoggedIn && <Dashboard />}
  </>
);
```

---

## 7. Common Mistakes to Watch For

### ❌ Overusing Redux

> Every piece of state is in the store (form inputs, tooltips, hover effects).
>
> **Fix:** Ask _"Does more than one unrelated component need this?"_ If no → `useState`.

### ❌ Non-Serializable Data in the Store

```js
// ❌ Functions, Promises, class instances
dispatch(actions.save({ callback: () => {}, ref: domNode }));

// ✅ Plain data only
dispatch(actions.save({ id: '1', title: 'Item' }));
```

### ❌ Async Logic in Reducers

```js
// ❌ Never do this
reducers: {
  async loadData(state) { state.items = await fetch('/api'); }
}

// ✅ Async belongs in thunks
export const loadData = () => async (dispatch) => { /* ... */ };
```

### ❌ Mutating State Outside Reducers

```js
// ❌ Immer is NOT active here — this directly mutates Redux state
const items = useSelector((state) => state.feature.items);
items.push(newItem);

// ✅ Always dispatch
dispatch(featureActions.addItem(newItem));
```

### ❌ Overgrown Slices

> A single slice has 200+ lines and 15+ reducer methods.
>
> **Fix:** Split by sub-feature. Keep each slice focused on one concern.

### ❌ Missing Loop Guards

> App fetches on mount → state updates → triggers send → triggers fetch → infinite loop.
>
> **Fix:** Use `isInitial` flag + `changed` flag pattern.

### ❌ Selecting Too Much State

```js
// ❌ Re-renders on ANY change to the entire slice
const data = useSelector((state) => state.feature);

// ✅ Re-renders only when this specific value changes
const items = useSelector((state) => state.feature.items);
```

---

## 8. Practice Checklist

### Store Setup

- [ ] `@reduxjs/toolkit` and `react-redux` installed
- [ ] `configureStore` used (not `createStore`)
- [ ] All slices combined in `store/index.js`
- [ ] `<Provider store={store}>` wraps root `<App />`

### Slice Design

- [ ] One `*-slice.js` file per feature
- [ ] `createSlice` used for all reducers
- [ ] `initialState` has safe defaults (`[]`, `null`, `false`)
- [ ] Immer-style updates (no manual spreading/copying)
- [ ] Actions: named export — `export const featureActions = slice.actions`
- [ ] Reducer: default export — `export default slice.reducer`
- [ ] Slice `name` matches state key in `configureStore`
- [ ] No slice exceeds ~100 lines

### State Design

- [ ] Only shared, cross-component state is in Redux
- [ ] Local UI concerns use `useState`
- [ ] No derived values stored — computed during render or in selectors
- [ ] No non-serializable values in the store
- [ ] State is flat (max 2 levels of nesting)

### Async Logic

- [ ] All async operations in thunks (`*-actions.js`)
- [ ] Zero `async`/`fetch` inside reducer methods
- [ ] Loading/success/error feedback via UI slice notifications
- [ ] `isInitial` flag prevents send-on-first-load
- [ ] `changed` flag separates user edits from server data

### React Integration

- [ ] `useSelector` selects only specific values (no full-slice selections)
- [ ] `useDispatch` used for all state changes
- [ ] Form inputs stay in local `useState`
- [ ] Conditional rendering driven by Redux state

### Folder Structure

- [ ] `src/store/` contains all Redux files
- [ ] `store/features/featureName/` groups slice + actions + selectors
- [ ] `components/FeatureName/` groups related UI
- [ ] Consistent naming: `kebab-case` files, `camelCase` exports

### Developer Experience

- [ ] Redux DevTools connects automatically (no manual setup)
- [ ] No serialization warnings in console
- [ ] No mutation warnings in console

---

> **How to use this guide:** Start any new Redux Toolkit project by walking through sections 2–4 first. Add async logic (section 5) once your slices work in DevTools. Wire up components last (section 6). Run through the checklist (section 8) before shipping.
