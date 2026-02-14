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
