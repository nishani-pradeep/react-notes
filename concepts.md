# React Concepts

## Virtual DOM

A Virtual DOM is a plain JavaScript object that mirrors the structure of the real DOM tree. It contains the same information (type, props, children) but exists only in memory.

Both the Current Tree and Work-in-Progress Tree are in React's memory. Reconciliation happens between these two trees as it's much cheaper than DOM manipulation.

### Relationship to Reconciliation:
- The Current Tree and Work-in-Progress Tree mentioned in reconciliation are both Virtual DOM trees
- React maintains these virtual representations in memory
- Reconciliation happens between two Virtual DOM trees (not the real DOM)

### Why it exists:
- **Performance**: Manipulating JavaScript objects in memory is much faster than manipulating the real DOM
- **Batching**: React can batch multiple updates to the Virtual DOM before touching the real DOM once
- **Diffing**: React can quickly compare two Virtual DOM trees to find minimal changes

### The Flow:
```
State Change → New Virtual DOM (Work-in-Progress Tree)
→ Diff with Old Virtual DOM (Current Tree)
→ Calculate minimal changes
→ Update Real DOM with only those changes
```

## React Reconciliation

Reconciliation is the algorithm React uses to determine which DOM elements need to be rendered or updated.

### How it works:

1. **Tree Structure**: React renders components as a tree structure where each node contains:
   - `type` (e.g., div, h1, or custom component)
   - `key`
   - `props`
   - `ref`

2. **Current Tree**: The existing tree that is currently rendered in the browser.

3. **Reconciliation Process**:
   - When state changes or user interactions occur, React creates a new tree called the **Work-in-Progress Tree**
   - React compares the Work-in-Progress Tree with the Current Tree to identify differences (diffing algorithm)
   - Only the differences are applied to the DOM (not the entire tree)
   - After the update, the Work-in-Progress Tree becomes the new Current Tree

### Benefits:
- Efficient updates by only modifying what changed
- Minimizes expensive DOM operations
- Improves application performance

## Render Phase vs Commit Phase

React's update process is split into two distinct phases:

### Render Phase

During the Render Phase, the Virtual DOM is built using the components and applying the newest state and props. This newly built tree is the **Work-in-Progress Tree**. In the Render Phase, the Current Tree and Work-in-Progress Tree are compared and React figures out what changes need to be done.

**What gets called:**
- Component function bodies
- `useState`, `useReducer` hooks and their updaters
- Lifecycle methods: `getDerivedStateFromProps`, `shouldComponentUpdate`

**Key characteristics:**
- **Asynchronous** - Can be paused, aborted, or restarted
- **Interruptible** - Can be interrupted if there is some high-priority task
- **Pure** - No side effects should occur during this phase
- **Works in memory** - Only manipulates Virtual DOM, doesn't touch real DOM

### Commit Phase

In the Commit Phase, React takes the list of changes from the Render Phase and applies them to the actual DOM.

**What happens:**
- Actually updates the real DOM
- Runs lifecycle methods and effect hooks
- Updates refs

**What gets called:**
- `useLayoutEffect` (runs synchronously after DOM mutations)
- `useEffect` (runs asynchronously after paint)
- Lifecycle methods: `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`
- Ref callbacks

**Key characteristics:**
- **Synchronous** - Cannot be interrupted, must complete in one go
- **Side effects allowed** - This is where side effects actually run
- **Touches the real DOM** - Makes actual changes visible to user
- **Fast** - Only applies minimal changes calculated in Render Phase

---

## React Hooks

For detailed documentation on React Hooks, see [react-hooks.md](./react-hooks.md)

**Covered hooks:**
- **useState** - State management in functional components
- **useEffect** - Side effects and lifecycle management
- **useMemo** - Memoization for performance optimization
- **useRef** - Refs and mutable values without re-renders
- **useContext** - Context consumption without prop drilling
- **useReducer** - Complex state management with reducer pattern
