# One-Day Prep Plan for Principal/Staff UI Engineer Interview

Based on the interview pattern you shared, here's a focused study plan:

---

## 🎯 Time Allocation (8-10 hours)

### **Morning (3-4 hours): React Fundamentals & Patterns**
### **Afternoon (3-4 hours): System Design & Performance**
### **Evening (2 hours): Algorithms & Behavioral Prep**

---

## 📚 Topic Breakdown

---

## **PART 1: React Core Concepts (3-4 hours)**

### **A. State Management & Lifecycle (1 hour)**

**Must Know:**
- `useState` with functional updates
- `useEffect` dependency arrays and cleanup
- `useRef` for persisting values without re-renders
- `useMemo` and `useCallback` for optimization
- When to use each hook

**Practice Problem:**
```
Build a timer that:
- Starts/stops on button click
- Persists when component unmounts/remounts
- Resets on page navigation
```

**Key Interview Questions:**
- How does useEffect cleanup work?
- When does React batch state updates?
- Difference between useRef and useState?
- How to prevent unnecessary re-renders?

---

### **B. Routing & Navigation (45 mins)**

**Must Know:**
- React Router: Routes, Route, Link, useNavigate, useParams
- Programmatic navigation
- Protected routes
- Route-based code splitting

**Practice Problem:**
```
Build a side nav with 3 routes:
- /timer (timer component)
- /about
- /settings

Timer should stop when navigating away and resume when coming back
```

**Key Concepts:**
- How to preserve component state across routes?
- useLocation, useNavigate hooks
- Route parameters and query strings

---

### **C. Component Lifecycle & Cleanup (30 mins)**

**Must Know:**
- Component mount/unmount/update cycle
- useEffect cleanup functions
- Memory leak prevention (timers, subscriptions)
- AbortController for fetch cancellation

**Practice Problem:**
```javascript
useEffect(() => {
  const timer = setInterval(() => {
    // Do something
  }, 1000);

  return () => clearInterval(timer); // Cleanup!
}, []);
```

---

### **D. Performance Optimization (1 hour)**

**Must Know:**
- Virtual scrolling (react-window, react-virtualized)
- Lazy loading and code splitting
- Debouncing and throttling
- Memoization (React.memo, useMemo, useCallback)
- Key props and list rendering

**Practice Problem:**
```
Render a list of 10,000 items efficiently
Add search/filter that doesn't block UI
```

**Key Questions:**
- How to optimize a slow re-rendering component?
- When to use React.memo?
- What causes unnecessary re-renders?

---

## **PART 2: System Design (3-4 hours)**

### **A. Large-Scale List/Grid Management (1.5 hours)**

**Must Know:**
- Virtual scrolling implementation
- Infinite scroll vs pagination
- Windowing technique
- Intersection Observer API
- Chunked data loading

**Practice Topics:**
```
1. Design a virtualized dropdown with 100K items
2. Implement infinite scroll with search
3. Handle chunked API responses
4. Cache and invalidation strategies
```

**Key Concepts:**
- Only render visible items (windowing)
- Use search-first approach for large datasets
- Server-side filtering for massive data
- Request cancellation and debouncing

---

### **B. Search & Filter Architecture (1.5 hours)**

**Must Know:**
- Debounced search implementation
- Filter state management
- URL sync (shareable filters)
- Client vs server-side filtering
- Request cancellation (AbortController)

**Design Pattern:**
```javascript
const [filters, setFilters] = useState({
  search: '',
  status: [],
  assignee: []
});

// Debounced API call
useEffect(() => {
  const controller = new AbortController();
  const timer = setTimeout(() => {
    fetchData(filters, { signal: controller.signal });
  }, 300);

  return () => {
    clearTimeout(timer);
    controller.abort();
  };
}, [filters]);
```

**Key Questions:**
- How to handle race conditions in search?
- How to make filters shareable (URL)?
- Client vs server filtering trade-offs?
- How to cache dropdown data?

---

### **C. Caching & Data Management (30 mins)**

**Must Know:**
- In-memory caching with TTL
- React Query / SWR concepts
- Stale-while-revalidate pattern
- Cache invalidation strategies

**Simple Cache Implementation:**
```javascript
class Cache {
  constructor(ttl = 300000) {
    this.cache = new Map();
    this.ttl = ttl;
  }

  get(key) {
    const item = this.cache.get(key);
    if (!item || Date.now() - item.time > this.ttl) {
      return null;
    }
    return item.value;
  }

  set(key, value) {
    this.cache.set(key, { value, time: Date.now() });
  }
}
```

---

## **PART 3: Algorithm & Problem Solving (2 hours)**

### **A. Recursion & Tree Traversal (1 hour)**

**Must Practice:**

**Problem 1: Nested List with Counts**
```javascript
// Input:
const list = [
  { name: 'Item 1', children: [
    { name: 'Item 1.1', children: [
      { name: 'Item 1.1.1' }
    ]},
    { name: 'Item 1.2' }
  ]},
  { name: 'Item 2' }
];

// Output with counts and indentation:
// Item 1 (3)
//   Item 1.1 (1)
//     Item 1.1.1 (0)
//   Item 1.2 (0)
// Item 2 (0)

function printNestedList(list, level = 0) {
  list.forEach(item => {
    const count = countChildren(item);
    const indent = '  '.repeat(level);
    console.log(`${indent}${item.name} (${count})`);

    if (item.children) {
      printNestedList(item.children, level + 1);
    }
  });
}

function countChildren(node) {
  if (!node.children || node.children.length === 0) {
    return 0;
  }

  let count = node.children.length;
  node.children.forEach(child => {
    count += countChildren(child);
  });

  return count;
}
```

**Problem 2: Flatten Nested Object**
```javascript
const nested = {
  a: 1,
  b: { c: 2, d: { e: 3 } }
};
// Output: { a: 1, 'b.c': 2, 'b.d.e': 3 }
```

**Problem 3: Find all paths in tree**

---

### **B. Array/Object Manipulation (30 mins)**

**Must Practice:**
- Deep clone
- Group by property
- Filter/map/reduce combinations
- Debounce/throttle implementation

---

### **C. Common Interview Patterns (30 mins)**

**Practice:**
```javascript
// 1. Debounce
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// 2. Throttle
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// 3. Promise.all implementation
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(value => {
        results[index] = value;
        completed++;
        if (completed === promises.length) {
          resolve(results);
        }
      }).catch(reject);
    });
  });
}
```

---

## **PART 4: Behavioral (30 mins prep)**

### **STAR Method Template**

**Common Questions:**
1. "Tell me about a time you optimized performance"
2. "Describe a challenging technical decision you made"
3. "How do you handle disagreements with team members?"
4. "Tell me about a project you're most proud of"
5. "Why are you looking for a new role?"
6. "What are your goals for the next role?"

**Prepare 3-4 Stories:**
- Performance optimization you did
- Complex system you designed
- Technical leadership example
- Conflict resolution

**For "Goals" question:**
- Technical: deeper architecture, scalability, mentoring
- Leadership: bigger impact, cross-team collaboration
- Growth: new technologies, best practices

---

## 🎯 **Priority Checklist (Must Know)**

### **High Priority:**
- ✅ useState, useEffect, useRef, useMemo
- ✅ useEffect cleanup and dependencies
- ✅ React Router navigation
- ✅ Virtual scrolling concept
- ✅ Debouncing search
- ✅ AbortController for fetch cancellation
- ✅ Recursion for nested structures
- ✅ Counting children in tree

### **Medium Priority:**
- ✅ useMemo vs useCallback
- ✅ React.memo
- ✅ Lazy loading and code splitting
- ✅ Caching strategies
- ✅ URL state management
- ✅ Promise handling

### **Good to Know:**
- ✅ React 18 features (Suspense, Transitions)
- ✅ Error boundaries
- ✅ Context API
- ✅ Custom hooks

---

## 📝 **Practice Problems (Must Do)**

### **React Coding:**
1. **Timer with navigation** (matches your interview)
   - Build side nav with 3 routes
   - Timer on page 1 that stops/resumes on nav

2. **Search with debouncing**
   - Input field that searches as you type
   - Debounce API calls
   - Show loading state

3. **Virtual list**
   - Render 10K items efficiently
   - Use windowing/virtual scroll

### **Algorithm:**
1. **Nested list with counts** (exactly your interview question)
2. **Tree traversal with level tracking**
3. **Implement debounce/throttle**

---

## 🚀 **Quick Review Before Interview**

### **5 Minutes Before:**
1. Review hooks cleanup patterns
2. Remember: useEffect deps cause infinite loops
3. Virtual scrolling = render only visible items
4. Debounce = delay execution, Throttle = limit rate
5. Recursion: base case + recursive case

### **Red Flags to Avoid:**
- ❌ Missing useEffect cleanup
- ❌ Infinite loops in useEffect
- ❌ Not handling loading/error states
- ❌ Rendering 10K items without virtualization
- ❌ No debouncing on search
- ❌ Memory leaks (intervals, subscriptions)

---

## 📖 **Resources (Quick Reference)**

- React Docs: https://react.dev
- React Router: https://reactrouter.com
- react-window: https://github.com/bvaughn/react-window
- Your concepts.md and react-hooks.md files!

---

Good luck! Focus on understanding concepts deeply rather than memorizing. They're looking for problem-solving approach and clean code, not perfect syntax.
