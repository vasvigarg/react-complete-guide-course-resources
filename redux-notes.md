# Redux with React — Complete Learning Notes

> Generated from the codebase in `code/19 Redux Basics` and `code/20 Advanced Redux`.

---

## Table of Contents

1. [Project & Folder Structure](#1-project--folder-structure)
2. [Core Redux Concepts & Terminology](#2-core-redux-concepts--terminology)
3. [State Flow: UI → Actions → Reducers → Store → UI](#3-state-flow-ui--actions--reducers--store--ui)
4. [Integrating Redux with React](#4-integrating-redux-with-react)
5. [Classic Redux (createStore) Pattern](#5-classic-redux-createstore-pattern)
6. [Redux Toolkit Pattern](#6-redux-toolkit-pattern)
7. [Working with Multiple Slices](#7-working-with-multiple-slices)
8. [Handling Async Logic (Side Effects)](#8-handling-async-logic-side-effects)
9. [Patterns, Conventions & Best Practices](#9-patterns-conventions--best-practices)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Project & Folder Structure

### Section 19 – Counter + Auth App

```
src/
├── index.js            ← React entry point, wraps <App /> with <Provider>
├── App.js              ← Root component, reads auth state to toggle views
├── store/
│   ├── index.js        ← configureStore — combines counter + auth reducers
│   ├── counter.js      ← counterSlice (createSlice)
│   └── auth.js         ← authSlice (createSlice)
└── components/
    ├── Counter.js       ← Dispatches counter actions, reads counter state
    ├── Counter.module.css
    ├── Auth.js          ← Login form, dispatches login action
    ├── Auth.module.css
    ├── Header.js        ← Nav bar, dispatches logout, reads isAuthenticated
    ├── Header.module.css
    ├── UserProfile.js   ← Shown when logged in
    └── UserProfile.module.css
```

### Section 20 – Shopping Cart App

```
src/
├── index.js            ← Entry, <Provider store={store}>
├── App.js              ← Dispatches thunks (fetchCartData, sendCartData), reads UI state
├── store/
│   ├── index.js        ← configureStore — combines ui + cart slices
│   ├── ui-slice.js     ← UI state (cart visibility, notifications)
│   ├── cart-slice.js   ← Cart items, totalQuantity, changed flag
│   └── cart-actions.js ← Async thunk action creators (fetch + send)
└── components/
    ├── Cart/
    │   ├── Cart.js          ← Reads cart items from store
    │   ├── CartButton.js    ← Dispatches ui toggle action, reads totalQuantity
    │   └── CartItem.js      ← Dispatches add/remove cart actions
    ├── Shop/
    │   ├── Products.js      ← Renders product list
    │   └── ProductItem.js   ← Dispatches addItemToCart action
    ├── Layout/
    │   ├── Layout.js        ← Wrapper with MainHeader
    │   └── MainHeader.js    ← Contains CartButton
    └── UI/
        ├── Card.js          ← Reusable card wrapper
        └── Notification.js  ← Displays status/error/success messages
```

### Key Structural Observations

- **Store files live in a dedicated `store/` directory** — separated from component code
- **Each slice gets its own file** (`counter.js`, `auth.js`, `cart-slice.js`, `ui-slice.js`)
- **Async action creators are in a separate file** (`cart-actions.js`) — not inside the slice
- **Components are feature-grouped** (Section 20: `Cart/`, `Shop/`, `Layout/`, `UI/`)
- **CSS Modules** (`.module.css`) are used for scoped styling throughout

---

## 2. Core Redux Concepts & Terminology

### Store

- The **single source of truth** for application state
- Created with `configureStore()` (Redux Toolkit) or `createStore()` (classic)
- Holds the complete state tree and exposes `dispatch()`, `getState()`, and `subscribe()`

```js
const store = configureStore({
  reducer: { counter: counterReducer, auth: authReducer },
});
```

### Reducers

- **Pure functions** that take `(currentState, action)` and return a **new state**
- In classic Redux: must return a **brand-new object** (immutable updates)
- In Redux Toolkit: uses **Immer** internally, so you can write "mutating" syntax that actually produces immutable updates

```js
// Classic — must return new object
if (action.type === 'increment') {
  return { counter: state.counter + 1, showCounter: state.showCounter };
}

// Toolkit — looks like mutation, but Immer handles immutability
increment(state) {
  state.counter++;
}
```

### Actions

- Plain objects with a `type` property (and optionally a `payload`)
- Describe **what happened** — dispatched from the UI to trigger state changes
- In Redux Toolkit, action creators are **auto-generated** by `createSlice`

```js
// Auto-generated action creator
counterActions.increase(10)
// Produces: { type: 'counter/increase', payload: 10 }
```

### Slices

- A Redux Toolkit concept that bundles `name`, `initialState`, and `reducers` into one unit
- Created with `createSlice()` — generates action creators and action types automatically

```js
const counterSlice = createSlice({
  name: 'counter',
  initialState: { counter: 0, showCounter: true },
  reducers: {
    increment(state) { state.counter++; },
    increase(state, action) { state.counter += action.payload; },
  },
});
```

### Dispatch

- The **only way to trigger a state change** — sends an action to the store
- In React: obtained via the `useDispatch()` hook

```js
const dispatch = useDispatch();
dispatch(counterActions.increment());
```

### Selectors

- Functions that **extract specific pieces of state** from the store
- In React: used with the `useSelector()` hook
- Components automatically **re-render** when the selected state changes

```js
const counter = useSelector((state) => state.counter.counter);
const isAuth = useSelector((state) => state.auth.isAuthenticated);
```

### Payload

- The data carried by an action
- In Redux Toolkit, the argument passed to an action creator is automatically set as `action.payload`

```js
dispatch(counterActions.increase(10));
// Inside reducer: state.counter = state.counter + action.payload;
```

### Thunks (Action Creator Thunks)

- Functions that **return another function** instead of an action object
- The returned function receives `dispatch` as an argument
- Used for **async operations** (HTTP requests, timers) before dispatching real actions
- Redux Toolkit's `configureStore` includes thunk middleware by default

```js
export const sendCartData = (cart) => {
  return async (dispatch) => {
    dispatch(uiActions.showNotification({ status: 'pending', ... }));
    // async HTTP request
    await sendRequest();
    dispatch(uiActions.showNotification({ status: 'success', ... }));
  };
};
```

---

## 3. State Flow: UI → Actions → Reducers → Store → UI

```
┌─────────────────────────────────────────────────────────────┐
│                       REACT COMPONENT                       │
│                                                             │
│  useSelector(state => state.cart.items)    ← reads state    │
│  const dispatch = useDispatch()                             │
│  onClick → dispatch(cartActions.addItemToCart({id, price})) │
│                                     │                       │
└─────────────────────────────────────┼───────────────────────┘
                                      │ ACTION dispatched
                                      ▼
┌─────────────────────────────────────────────────────────────┐
│                     REDUX MIDDLEWARE                         │
│  (Thunk middleware intercepts functions,                    │
│   passes through plain action objects)                      │
└─────────────────────────────────────┬───────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────┐
│                         REDUCER                             │
│                                                             │
│  addItemToCart(state, action) {                              │
│    const newItem = action.payload;                          │
│    state.items.push({ id: newItem.id, ... });               │
│    state.totalQuantity++;                                   │
│  }                                                          │
└─────────────────────────────────────┬───────────────────────┘
                                      │ new state
                                      ▼
┌─────────────────────────────────────────────────────────────┐
│                         STORE                               │
│  { cart: { items: [...], totalQuantity: 3 },                │
│    ui: { cartIsVisible: true, notification: null } }        │
└─────────────────────────────────────┬───────────────────────┘
                                      │ state updated
                                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    REACT RE-RENDERS                          │
│  Components subscribed via useSelector re-render            │
│  with the new state values                                  │
└─────────────────────────────────────────────────────────────┘
```

### Concrete Example: Adding an Item to Cart

1. **User clicks** "Add to Cart" in `ProductItem.js`
2. **Component dispatches**: `dispatch(cartActions.addItemToCart({ id, title, price }))`
3. **Action flows** to the `cart-slice` reducer's `addItemToCart` method
4. **Reducer updates state**: pushes new item or increments existing item's quantity
5. **Store holds new state**: updated `items` array and `totalQuantity`
6. **`Cart.js` re-renders**: because it's subscribed to `state.cart.items` via `useSelector`
7. **`CartButton.js` re-renders**: because it's subscribed to `state.cart.totalQuantity`

### Async Flow (with Thunks)

1. **Cart state changes** → `useEffect` in `App.js` detects the change
2. **App dispatches thunk**: `dispatch(sendCartData(cart))`
3. **Thunk function executes**: dispatches `showNotification({ status: 'pending' })`, then makes an HTTP `PUT` request
4. **On success/failure**: dispatches another `showNotification` with `'success'` or `'error'`
5. **UI re-renders**: `Notification.js` appears based on `state.ui.notification`

---

## 4. Integrating Redux with React

### Step 1: Install Dependencies

```
npm install @reduxjs/toolkit react-redux
```

### Step 2: Create the Store

```js
// store/index.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counter';
import authReducer from './auth';

const store = configureStore({
  reducer: { counter: counterReducer, auth: authReducer },
});

export default store;
```

### Step 3: Provide the Store to React

```js
// index.js
import { Provider } from 'react-redux';
import store from './store/index';

root.render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

- `<Provider>` makes the Redux store available to **every component** in the tree
- Placed at the **top level** — wraps `<App />`

### Step 4: Read State with `useSelector`

```js
import { useSelector } from 'react-redux';

const counter = useSelector((state) => state.counter.counter);
```

- Accepts a **selector function** that receives the entire Redux state
- **Automatically subscribes** the component — re-renders only when the selected value changes
- With multiple slices, you access state via the key used in the `reducer` map (e.g., `state.counter`, `state.auth`)

### Step 5: Dispatch Actions with `useDispatch`

```js
import { useDispatch } from 'react-redux';
import { counterActions } from '../store/counter';

const dispatch = useDispatch();
dispatch(counterActions.increment());
dispatch(counterActions.increase(10)); // with payload
```

### Class-Based Components (Legacy Pattern)

The codebase includes a commented-out example showing the `connect()` higher-order component:

```js
import { connect } from 'react-redux';

const mapStateToProps = (state) => ({
  counter: state.counter,
});

const mapDispatchToProps = (dispatch) => ({
  increment: () => dispatch({ type: 'increment' }),
  decrement: () => dispatch({ type: 'decrement' }),
});

export default connect(mapStateToProps, mapDispatchToProps)(Counter);
```

- `mapStateToProps` — maps Redux state to component props
- `mapDispatchToProps` — maps dispatch calls to component props
- This is the **older pattern** — hooks (`useSelector`/`useDispatch`) are the modern approach

---

## 5. Classic Redux (createStore) Pattern

Demonstrated in early steps (steps 03–08) before migrating to Redux Toolkit:

```js
import { createStore } from 'redux';

const initialState = { counter: 0, showCounter: true };

const counterReducer = (state = initialState, action) => {
  if (action.type === 'increment') {
    return { counter: state.counter + 1, showCounter: state.showCounter };
  }
  if (action.type === 'increase') {
    return { counter: state.counter + action.amount, showCounter: state.showCounter };
  }
  if (action.type === 'toggle') {
    return { showCounter: !state.showCounter, counter: state.counter };
  }
  return state;
};

const store = createStore(counterReducer);
```

### Key Characteristics

- **String-based action types** — error-prone, no autocomplete
- **Must return entirely new state objects** — accidental mutation causes bugs
- **All state properties must be spread/copied** every time, even unchanged ones
- **Payload accessed by custom keys** (e.g., `action.amount`) — no standard convention
- **Single reducer function** — hard to scale for large apps

### Problems This Approach Creates

- Typo in action type string → silent bug (no error thrown)
- Growing reducer with many `if` blocks → unmaintainable
- Easy to accidentally mutate state → leads to rendering issues
- No built-in way to handle side effects

---

## 6. Redux Toolkit Pattern

Migration happens at step 09 and is the pattern used for all subsequent code:

### createSlice

```js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { counter: 0, showCounter: true },
  reducers: {
    increment(state) { state.counter++; },
    decrement(state) { state.counter--; },
    increase(state, action) { state.counter += action.payload; },
    toggleCounter(state) { state.showCounter = !state.showCounter; },
  },
});

export const counterActions = counterSlice.actions;
export default counterSlice.reducer;
```

### What `createSlice` Gives You

| Feature | Detail |
|---|---|
| **Auto-generated action types** | `'counter/increment'`, `'counter/increase'`, etc. |
| **Auto-generated action creators** | `counterActions.increment()`, `counterActions.increase(10)` |
| **Immer-based immutability** | Write `state.counter++` — Immer produces a new immutable state behind the scenes |
| **Standard payload** | Arguments passed to action creators automatically become `action.payload` |

### configureStore

```js
import { configureStore } from '@reduxjs/toolkit';

const store = configureStore({
  reducer: { counter: counterReducer, auth: authReducer },
});
```

- Replaces `createStore` + `combineReducers`
- Accepts a **reducer map** — keys become the state slice names
- **Automatically adds** Redux DevTools support and thunk middleware

---

## 7. Working with Multiple Slices

### Pattern: One File Per Slice

Each feature gets its own slice file:

```
store/
├── index.js        ← configureStore (combines all slices)
├── counter.js      ← counterSlice
├── auth.js         ← authSlice
```

Or in a larger app:

```
store/
├── index.js
├── cart-slice.js
├── ui-slice.js
└── cart-actions.js  ← async thunks for the cart feature
```

### Combining Slices

```js
const store = configureStore({
  reducer: {
    counter: counterReducer,   // state.counter
    auth: authReducer,          // state.auth
  },
});
```

- The **key** in the reducer map determines how you access state: `state.counter.counter`, `state.auth.isAuthenticated`
- Slices are **independent** — each manages its own piece of state
- **Cross-slice dispatching** is possible: a thunk in `cart-actions.js` dispatches actions from both `cartSlice` and `uiSlice`

### Cross-Slice Communication Example

In `cart-actions.js`, the `sendCartData` thunk dispatches actions from the `ui-slice`:

```js
import { uiActions } from './ui-slice';
import { cartActions } from './cart-slice';

export const fetchCartData = () => {
  return async (dispatch) => {
    try {
      const data = await fetchData();
      dispatch(cartActions.replaceCart({ items: data.items || [], ... }));
    } catch (error) {
      dispatch(uiActions.showNotification({ status: 'error', ... }));
    }
  };
};
```

---

## 8. Handling Async Logic (Side Effects)

### The Problem

- **Reducers must be pure** — no side effects (no HTTP requests, no timers)
- But real apps need to sync state with a backend

### Approach 1: useEffect in Components (Simple)

```js
// App.js — step 03
useEffect(() => {
  fetch('https://api.example.com/cart.json', {
    method: 'PUT',
    body: JSON.stringify(cart),
  });
}, [cart]);
```

- Pros: simple, quick
- Cons: no error handling, no loading states, no separation of concerns

### Approach 2: Action Creator Thunks (Recommended)

```js
// store/cart-actions.js
export const sendCartData = (cart) => {
  return async (dispatch) => {
    // 1. Dispatch "pending" notification
    dispatch(uiActions.showNotification({
      status: 'pending', title: 'Sending...', message: 'Sending cart data!',
    }));

    // 2. Perform async work
    const sendRequest = async () => {
      const response = await fetch('https://api.example.com/cart.json', {
        method: 'PUT',
        body: JSON.stringify({ items: cart.items, totalQuantity: cart.totalQuantity }),
      });
      if (!response.ok) throw new Error('Sending cart data failed.');
    };

    // 3. Handle success / error
    try {
      await sendRequest();
      dispatch(uiActions.showNotification({ status: 'success', ... }));
    } catch (error) {
      dispatch(uiActions.showNotification({ status: 'error', ... }));
    }
  };
};
```

### How Thunks are Dispatched

```js
// App.js
useEffect(() => {
  if (isInitial) { isInitial = false; return; }
  if (cart.changed) {
    dispatch(sendCartData(cart));
  }
}, [cart, dispatch]);

useEffect(() => {
  dispatch(fetchCartData());
}, [dispatch]);
```

- **`sendCartData`** — triggered whenever cart state changes (but not on first load)
- **`fetchCartData`** — triggered once on mount to load initial data from backend
- The `isInitial` flag prevents sending data back to the server when the app first loads and fetches initial data

### The `changed` Flag Pattern

```js
// cart-slice.js
addItemToCart(state, action) {
  state.changed = true;  // mark that this change came from user action
  state.totalQuantity++;
  // ...
},
replaceCart(state, action) {
  // No `changed = true` here — this is data from the server
  state.totalQuantity = action.payload.totalQuantity;
  state.items = action.payload.items;
},
```

- `changed` is set to `true` only by **user-initiated** actions (add/remove item)
- `replaceCart` (called when fetching from server) does **not** set `changed`
- This prevents an infinite loop: fetch → update state → state changed → send → fetch…

### UI Feedback with Notifications

The `ui-slice` manages notification state:

```js
const uiSlice = createSlice({
  name: 'ui',
  initialState: { cartIsVisible: false, notification: null },
  reducers: {
    toggle(state) { state.cartIsVisible = !state.cartIsVisible; },
    showNotification(state, action) {
      state.notification = {
        status: action.payload.status,   // 'pending' | 'success' | 'error'
        title: action.payload.title,
        message: action.payload.message,
      };
    },
  },
});
```

Components render feedback based on this state:

```js
{notification && (
  <Notification status={notification.status} title={notification.title} message={notification.message} />
)}
```

---

## 9. Patterns, Conventions & Best Practices

### ✅ Separation of Concerns

| Layer | Responsibility |
|---|---|
| **Slice files** (`*-slice.js`) | State shape, reducers, exported actions |
| **Action files** (`*-actions.js`) | Async thunks, side effects, HTTP logic |
| **Store index** (`store/index.js`) | Combining slices, creating store |
| **Components** | UI rendering, dispatching actions, reading state |

### ✅ Immutability

- Classic Redux: manually return new objects — **never mutate** `state` directly
- Redux Toolkit: write "mutating" code — **Immer handles immutability** under the hood
- **Never** mix approaches — if using Toolkit, rely on Immer consistently

### ✅ Named Exports for Actions, Default Export for Reducer

```js
export const counterActions = counterSlice.actions;   // named export
export default counterSlice.reducer;                    // default export
```

This convention makes imports clean:

```js
import counterReducer from './counter';           // the reducer
import { counterActions } from '../store/counter'; // action creators
```

### ✅ Minimal State Selection

- Select **only the data you need** with `useSelector`
- This limits re-renders to components that actually use the changed data

```js
// Good — specific selection
const counter = useSelector(state => state.counter.counter);
const show = useSelector(state => state.counter.showCounter);

// Avoid — selecting entire slice (causes unnecessary re-renders)
const counterState = useSelector(state => state.counter);
```

### ✅ Keep Reducers Lean

- Reducers should handle **synchronous state transformations only**
- All async work (API calls) goes in thunks or component-level `useEffect`
- Complex data transformations (finding items, calculating totals) are fine inside reducers

### ✅ Feature-Based File Organization

```
components/
├── Cart/          ← everything related to cart UI
├── Shop/          ← product listing
├── Layout/        ← app shell, headers
└── UI/            ← reusable presentational components
```

### ✅ Conditional Rendering Based on State

```js
// App.js — toggling views based on auth state
const isAuth = useSelector(state => state.auth.isAuthenticated);
return (
  <>
    <Header />
    {!isAuth && <Auth />}
    {isAuth && <UserProfile />}
    <Counter />
  </>
);
```

```js
// App.js — toggling cart visibility
const showCart = useSelector(state => state.ui.cartIsVisible);
return (
  <Layout>
    {showCart && <Cart />}
    <Products />
  </Layout>
);
```

### ✅ Preventing Redundant Side Effects

- Use **module-level flags** (`let isInitial = true`) to skip initial-render side effects
- Use **state flags** (`cart.changed`) to distinguish user changes from server-loaded data
- Both patterns prevent infinite loops and unnecessary API calls

---

## 10. Key Takeaways

1. **Redux provides predictable, centralized state management** — all state lives in one store and flows in one direction

2. **Redux Toolkit is the modern standard** — it eliminates boilerplate, auto-generates action creators, and uses Immer for safe "mutations"

3. **`createSlice` is the workhorse** — bundles initial state, reducers, and action creators into a single, clean API

4. **`configureStore` replaces `createStore`** — it sets up the store with middleware (thunks) and DevTools out of the box

5. **Hooks are the modern React-Redux API** — `useSelector` for reading state, `useDispatch` for triggering actions. The older `connect`/`mapStateToProps` pattern still works but is less ergonomic

6. **Reducers must stay pure** — no side effects, no async. All async logic belongs in thunks or component effects

7. **Thunks enable async workflows** — they return functions instead of action objects, receive `dispatch`, and can orchestrate multi-step async flows with loading/error states

8. **Each feature slice lives in its own file** — promotes modularity, testability, and clear ownership of state

9. **Cross-slice communication happens in thunks** — one thunk can dispatch actions from multiple slices (e.g., updating cart data and showing UI notifications)

10. **State access through `useSelector` is surgically precise** — select only what you need to avoid unnecessary re-renders

11. **Guard against infinite side-effect loops** — use flags (`isInitial`, `changed`) to prevent fetch-on-mount from triggering send-on-change

12. **Separation of concerns matters** — slice files define state shape, action files handle async logic, store/index combines everything, and components only care about rendering and dispatching

---

## 11. Redux Classic vs Redux Toolkit — Practical Usage Guide

### When to Use Classic Redux

Classic Redux (`createStore`, manual reducers, string action types) is rarely the right choice for new projects, but there are a few situations where it may still appear:

- **Legacy codebases** — existing projects built before Redux Toolkit was released may still use classic Redux; migrating all at once isn't always feasible
- **Extremely minimal use cases** — a tiny utility or prototype where pulling in `@reduxjs/toolkit` feels like unnecessary overhead (even then, RTK is lightweight)
- **Learning purposes** — understanding classic Redux first helps you appreciate _why_ Redux Toolkit exists and what problems it solves
- **Library/package authors** — if you're building a Redux middleware or plugin that needs zero dependencies beyond `redux` itself

> **Bottom line:** Unless you're maintaining legacy code or deliberately learning fundamentals, there's no practical reason to choose classic Redux over Redux Toolkit.

---

### When to Use Redux Toolkit (RTK)

Redux Toolkit is the **official, recommended way** to write Redux logic. Use it for:

- **All new projects** — RTK is the default starting point recommended by the Redux team
- **Apps with multiple state slices** — the slice pattern keeps things organized as your app grows
- **Apps with async data fetching** — built-in thunk support and `createAsyncThunk` streamline API interactions
- **Teams of any size** — RTK enforces consistent patterns, making onboarding easier
- **Migrating from classic Redux** — you can adopt RTK incrementally, one slice at a time, alongside existing classic reducers

#### Why RTK is the Preferred Modern Approach

- It is **maintained by the Redux core team** — it's not a third-party wrapper, it _is_ Redux
- The Redux documentation itself states: _"We recommend using Redux Toolkit for all Redux code"_
- `createStore` from classic Redux is **officially deprecated** in favor of `configureStore`

---

### Advantages of Redux Toolkit

#### 🔹 Reduction in Boilerplate

| Classic Redux | Redux Toolkit |
|---|---|
| Manually define action type constants | Auto-generated by `createSlice` |
| Write separate action creator functions | Auto-generated by `createSlice` |
| Write verbose `switch`/`if` reducer logic | Concise reducer methods in `createSlice` |
| Manually call `combineReducers` | `configureStore` handles it |
| Manually add middleware | `configureStore` includes thunk by default |

**Example — Classic (15+ lines):**

```js
const INCREMENT = 'INCREMENT';
const increment = () => ({ type: INCREMENT });

const reducer = (state = { count: 0 }, action) => {
  switch (action.type) {
    case INCREMENT:
      return { ...state, count: state.count + 1 };
    default:
      return state;
  }
};
```

**Example — RTK (7 lines):**

```js
const counterSlice = createSlice({
  name: 'counter',
  initialState: { count: 0 },
  reducers: {
    increment(state) { state.count++; },
  },
});
```

#### 🔹 Built-in Best Practices

- Enforces **immutable state updates** via Immer — you literally cannot make accidental mutation bugs
- Includes **serializable state checks** in development — warns if you put non-serializable values (functions, class instances) in the store
- Includes **immutability checks** in development — catches actual state mutations if they somehow bypass Immer

#### 🔹 Safer State Updates (Immutability Handling)

- Powered by **Immer** under the hood
- You write code that _looks_ like mutation (`state.counter++`) but Immer produces a **brand-new immutable state object**
- Eliminates the most common category of Redux bugs: forgetting to copy nested state

```js
// This is safe in RTK — Immer handles it
addItemToCart(state, action) {
  const existingItem = state.items.find(item => item.id === action.payload.id);
  if (existingItem) {
    existingItem.quantity++;              // looks like mutation — but it's safe
    existingItem.totalPrice += price;     // Immer tracks and clones
  } else {
    state.items.push({ ... });           // push is also safe here
  }
}
```

#### 🔹 Async Handling Support

- **Thunk middleware** is included automatically by `configureStore` — no extra setup
- **`createAsyncThunk`** provides a standardized pattern for async operations with `pending`, `fulfilled`, and `rejected` lifecycle actions
- Manual thunks (as seen in `cart-actions.js`) are also fully supported and sometimes preferred for complex flows

#### 🔹 Developer Experience Improvements

- **Redux DevTools** integration is automatic — no manual configuration
- **TypeScript support** is first-class — `createSlice` infers action types and state shapes
- **Fewer files to maintain** — no separate action type constants file, no separate action creators file
- **Consistent patterns** across the codebase — every feature follows the same slice structure

---

### Key Concepts Introduced by Redux Toolkit

#### `configureStore`

- **Replaces** `createStore` + `combineReducers` + `applyMiddleware`
- Accepts a `reducer` map — automatically combines them
- **Automatically includes**: thunk middleware, Redux DevTools extension, development-only checks (serializable + immutability)

```js
const store = configureStore({
  reducer: {
    ui: uiSlice.reducer,
    cart: cartSlice.reducer,
  },
});
```

#### `createSlice`

- **The core building block** of Redux Toolkit
- Bundles `name`, `initialState`, and `reducers` into a single declaration
- Auto-generates **action creators** and **action type strings**
- Reducer methods receive `(state, action)` and can use Immer-style "mutations"

```js
const authSlice = createSlice({
  name: 'authentication',
  initialState: { isAuthenticated: false },
  reducers: {
    login(state) { state.isAuthenticated = true; },
    logout(state) { state.isAuthenticated = false; },
  },
});

export const authActions = authSlice.actions;
export default authSlice.reducer;
```

#### `createAsyncThunk`

- Creates a **thunk action creator** that dispatches lifecycle actions automatically:
  - `pending` — dispatched when the async function starts
  - `fulfilled` — dispatched when it resolves successfully
  - `rejected` — dispatched when it throws or rejects
- Useful when you want **standardized loading/error state** without writing manual dispatch calls

```js
import { createAsyncThunk } from '@reduxjs/toolkit';

export const fetchCartData = createAsyncThunk('cart/fetchCart', async () => {
  const response = await fetch('https://api.example.com/cart.json');
  if (!response.ok) throw new Error('Failed to fetch');
  return response.json();
});

// Handle in slice with `extraReducers`:
const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], status: 'idle' },
  reducers: { /* ... */ },
  extraReducers: (builder) => {
    builder
      .addCase(fetchCartData.pending, (state) => { state.status = 'loading'; })
      .addCase(fetchCartData.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload.items;
      })
      .addCase(fetchCartData.rejected, (state) => { state.status = 'failed'; });
  },
});
```

> **Note:** The codebase uses **manual thunks** (in `cart-actions.js`) instead of `createAsyncThunk`. Both approaches are valid — manual thunks give more control over the dispatch flow; `createAsyncThunk` provides a more standardized, less verbose pattern.

#### Standardized Middleware and DevTools Setup

- `configureStore` automatically includes:
  - **redux-thunk** — enables dispatching functions
  - **Serializable check middleware** (dev only) — warns about non-serializable values in state/actions
  - **Immutability check middleware** (dev only) — detects accidental state mutations
  - **Redux DevTools Extension** — connects automatically if the browser extension is installed
- You can customize or extend middleware if needed, but the defaults cover most use cases

---

### Guidelines for Future Projects Using Redux Toolkit

#### Recommended Folder & File Structure

```
src/
├── store/
│   ├── index.js                  ← configureStore, combine all slices
│   ├── features/
│   │   ├── auth/
│   │   │   ├── auth-slice.js     ← createSlice for authentication
│   │   │   └── auth-actions.js   ← async thunks for auth (login API, etc.)
│   │   ├── cart/
│   │   │   ├── cart-slice.js     ← createSlice for cart state
│   │   │   └── cart-actions.js   ← async thunks for cart (fetch, send)
│   │   └── ui/
│   │       └── ui-slice.js       ← createSlice for UI state (modals, notifications)
├── components/
│   ├── Cart/                     ← components that use cart state
│   ├── Shop/                     ← components that use product/shop state
│   ├── Layout/                   ← structural components
│   └── UI/                       ← reusable, state-agnostic components
```

#### Feature-Based Slice Organization

- **Group by feature, not by type** — put a feature's slice + actions + selectors together
- Each feature folder contains:
  - `*-slice.js` — the slice definition (state shape, reducers, exported actions)
  - `*-actions.js` — async thunks / action creators with side effects
  - Optionally `*-selectors.js` — reusable selector functions for complex derived state

#### Where Async Logic Should Live

| Approach | When to Use |
|---|---|
| **Thunks in `*-actions.js`** | Multi-step async flows, cross-slice dispatching, complex error handling |
| **`createAsyncThunk`** | Standard CRUD operations with predictable loading/error states |
| **`useEffect` in components** | One-off side effects tightly coupled to a specific component's lifecycle |

- **Never put async logic in reducers** — reducers must remain pure and synchronous
- **Prefer thunks over component-level effects** for any logic that could be reused or tested independently

#### Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Slice file | `feature-slice.js` | `cart-slice.js`, `auth-slice.js` |
| Actions file | `feature-actions.js` | `cart-actions.js` |
| Slice name | camelCase noun | `name: 'cart'`, `name: 'authentication'` |
| Reducer methods | camelCase verb phrase | `addItemToCart`, `removeItemFromCart`, `toggle` |
| Exported actions | `featureActions` | `cartActions`, `uiActions`, `authActions` |
| State keys in store | camelCase noun matching slice | `state.cart`, `state.ui`, `state.auth` |

#### How to Keep Redux Scalable and Maintainable

- **One slice per feature** — never put unrelated state in the same slice
- **Keep slices small** — if a slice grows beyond ~100 lines, consider splitting it
- **Use selectors** — encapsulate state access logic so component code doesn't depend on the exact state shape
- **Co-locate related logic** — slice, thunks, and selectors for a feature live in the same folder
- **Avoid deeply nested state** — flatten state shape where possible; Immer helps, but flat state is easier to reason about
- **Use the `changed` flag pattern** (from the codebase) to distinguish user-driven changes from server-synced data

---

### Best Practices & Pitfalls to Avoid

#### ❌ What NOT to Store in Redux

Not everything belongs in the Redux store. Avoid storing:

- **Form input values** — use local component state (`useState`) or a form library; forms are typically local UI concerns
- **UI state limited to one component** — a modal's open/close state that only one component cares about should be local state
- **Derived/computed values** — don't store values that can be calculated from existing state; compute them in selectors or during render
- **Non-serializable values** — functions, class instances, Promises, DOM elements, and Symbols should never be in the store
- **Data that doesn't need to be shared** — if only one component reads and writes a piece of state, Redux is overkill for it

#### ✅ What SHOULD Be in Redux

- **State shared across multiple components** — auth status, cart contents, theme, notifications
- **State that needs to survive component unmounts** — global data that persists across routes
- **State that follows complex update logic** — when multiple actions can modify the same data
- **Server-cached data** — fetched data that should be globally accessible (though consider React Query / RTK Query for this)

#### ❌ How to Avoid Over-Engineering

- **Don't add Redux prematurely** — start with `useState` and `useContext`; add Redux when you actually feel the pain of prop drilling or scattered state
- **Don't create a slice for every piece of state** — group related state together logically (e.g., one `ui-slice` for all UI toggles, not separate slices for each modal)
- **Don't use thunks for simple sync actions** — if an action just updates state with no side effects, a regular `createSlice` reducer is sufficient
- **Don't over-normalize state** — normalization helps with large datasets but adds complexity; only normalize when you have relational data with frequent lookups

#### ✅ Tips for Clean State Design

- **Keep state minimal** — store the smallest amount of data needed; derive everything else
- **Keep state flat** — avoid nesting objects more than 2 levels deep
- **Use meaningful slice names** — `'cart'` not `'data'`, `'authentication'` not `'stuff'`
- **Initialize state with sensible defaults** — always provide safe initial values (`[]` for arrays, `null` for optional objects, `false` for booleans)
- **Separate domain state from UI state** — cart items (domain) live in `cart-slice`; notification visibility (UI) lives in `ui-slice`
- **Use `replaceX` reducers for server data** — when hydrating state from an API, use a dedicated reducer (like `replaceCart`) that clearly replaces server-sourced data without triggering sync flags
