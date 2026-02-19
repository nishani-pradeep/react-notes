# React Hooks

## useState

`useState` is a hook that lets you add state to functional components.

### Basic Syntax:
```javascript
const [state, setState] = useState(initialValue);
```

### How it works:
- Returns an array with two elements: current state value and a function to update it
- When `setState` is called, React schedules a re-render with the new state value
- The component re-renders with the updated state

### Example:
```javascript
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### Functional Updates:
When new state depends on previous state, use the functional form:
```javascript
setCount(prevCount => prevCount + 1);
```

### Key Points:
- State updates are asynchronous and batched by React
- Calling `setState` with the same value doesn't trigger re-render
- Initial state is only used on first render
- For expensive calculations, use lazy initialization: `useState(() => expensiveComputation())`

---

## useEffect

`useEffect` lets you perform side effects in functional components. Side effects include data fetching, subscriptions, DOM manipulation, etc.

### Basic Syntax:
```javascript
useEffect(() => {
  // Side effect code here

  return () => {
    // Cleanup code (optional)
  };
}, [dependencies]);
```

### How it works:
- Runs after the Commit Phase (after DOM is updated)
- Takes two arguments: a function and a dependency array
- The function runs after every render where dependencies have changed
- Returns an optional cleanup function that runs before the next effect or when component unmounts

### Dependency Array Behavior:

**No dependency array - Runs after every render:**
```javascript
useEffect(() => {
  console.log('Runs after every render');
});
```

**Empty dependency array - Runs once after mount:**
```javascript
useEffect(() => {
  console.log('Runs once after component mounts');
}, []);
```

**With dependencies - Runs when dependencies change:**
```javascript
useEffect(() => {
  console.log('Runs when count changes');
}, [count]);
```

### Example - Data Fetching:
```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);

    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]); // Re-fetch when userId changes

  if (loading) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}
```

### Example - Cleanup:
```javascript
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);

    // Cleanup function - clears interval when component unmounts
    return () => {
      clearInterval(interval);
    };
  }, []);

  return <div>{count}</div>;
}
```

### Key Points:
- useEffect runs in the Commit Phase (after render)
- Always runs after the first render
- Cleanup function runs before the next effect or when component unmounts
- Missing dependencies can cause bugs (use ESLint plugin to catch them)
- Don't put state variables in dependencies if you're updating them inside the effect (causes infinite loops)

---

## useMemo

`useMemo` caches the result of an expensive calculation. It only recalculates when dependencies change.

### Basic Syntax:
```javascript
const memoizedValue = useMemo(() => expensiveCalculation(a, b), [a, b]);
```

### How it works:
- Takes a function and a dependency array
- Runs the function and caches the result
- Returns cached result on next renders
- Re-runs the function only if dependencies change

### Example:
```javascript
function ProductList({ products, filterText }) {
  const filteredProducts = useMemo(() => {
    return products.filter(p => p.name.includes(filterText));
  }, [products, filterText]);

  return (
    <ul>
      {filteredProducts.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

### When to Use:
- **Use when:**
  - Calculation is expensive (like filtering large arrays)
  - Complex transformations
  - Preventing unnecessary re-renders

- **Don't use when:**
  - Calculation is simple and fast
  - Premature optimization

### Memoizing Objects to Prevent useEffect Re-runs:

React uses reference comparison for objects. Every time a component re-renders, any object created in the component is a new object (new reference), even if the values inside are the same. This causes useEffect to run again and again because it sees a "different" object each time.

**Problem without useMemo:**
```javascript
function UserProfile({ userId }) {
  // This object is recreated on every render (new reference each time)
  const filters = { userId: userId, active: true };

  useEffect(() => {
    console.log('Fetching with filters...');
    fetchUsers(filters);
  }, [filters]); // This runs on EVERY render because filters is always a new object!

  return <div>User Profile</div>;
}
```

**Solution with useMemo:**
```javascript
function UserProfile({ userId }) {
  // useMemo returns the same object reference if userId hasn't changed
  const filters = useMemo(() => {
    return { userId: userId, active: true };
  }, [userId]);

  useEffect(() => {
    console.log('Fetching with filters...');
    fetchUsers(filters);
  }, [filters]); // This only runs when userId changes, not on every render!

  return <div>User Profile</div>;
}
```

**Why this happens:**
```javascript
// Every render creates a new object
const obj1 = { userId: 1 };
const obj2 = { userId: 1 };
console.log(obj1 === obj2); // false - different references!

// useMemo keeps the same reference
const memoized = useMemo(() => ({ userId: 1 }), []);
// Returns same object reference on every render
```

### Key Points:
- useMemo runs during Render Phase (must be pure, no side effects)
- Only use for performance optimization, not for correctness
- React may discard cached values in some cases
- Use useMemo to memoize objects when passing them to useEffect dependencies to prevent unnecessary re-runs

---

## useRef

`useRef` is a hook that creates a mutable reference that persists across renders. Updating it does NOT cause re-renders.

### Basic Syntax:
```javascript
const ref = useRef(initialValue);
```

### How it works:
- Returns an object with a `.current` property
- Changing `.current` does NOT trigger a re-render
- The same ref object persists for the entire lifetime of the component
- The value inside `.current` can be mutated directly

### Common Use Cases:

#### 1. Accessing DOM Elements:
```javascript
function TextInput() {
  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus(); // Direct DOM manipulation
  };

  return (
    <div>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus Input</button>
    </div>
  );
}
```

#### 2. Storing Values That Don't Need Re-renders:
```javascript
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null); // Store interval ID

  const startTimer = () => {
    intervalRef.current = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
  };

  const stopTimer = () => {
    clearInterval(intervalRef.current); // Access stored interval ID
  };

  useEffect(() => {
    return () => clearInterval(intervalRef.current); // Cleanup
  }, []);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={startTimer}>Start</button>
      <button onClick={stopTimer}>Stop</button>
    </div>
  );
}
```

#### 3. Keeping Track of Previous Values:
```javascript
function Counter() {
  const [count, setCount] = useState(0);
  const prevCountRef = useRef();

  useEffect(() => {
    prevCountRef.current = count; // Store current value after render
  });

  const prevCount = prevCountRef.current; // Get previous value

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### useState vs useRef:

| Feature | useState | useRef |
|---------|----------|--------|
| Triggers re-render | Yes | No |
| Value persists across renders | Yes | Yes |
| Mutable | No (use setState) | Yes (.current is mutable) |
| Use case | UI data that affects rendering | Data that doesn't affect UI |
| When it updates | Asynchronously (next render) | Immediately |

### Example - When to use what:
```javascript
function Component() {
  // ✓ Use useState - affects UI, needs re-render
  const [visible, setVisible] = useState(true);

  // ✓ Use useRef - doesn't affect UI, no re-render needed
  const clickCount = useRef(0);

  const handleClick = () => {
    clickCount.current++; // Updated immediately, no re-render
    console.log('Clicked', clickCount.current, 'times');
  };

  return (
    <div>
      {visible && <p>Content</p>}
      <button onClick={handleClick}>Click me</button>
    </div>
  );
}
```

### Key Points:
- useRef doesn't trigger re-renders when you change `.current`
- Perfect for storing values that need to persist but don't affect rendering
- Common for DOM access, storing timers, tracking previous values
- Changes to `.current` are immediate (not batched like setState)

---

## useContext

`useContext` is a hook that lets you read and subscribe to context from your component. It's used to pass data through the component tree without passing props manually at every level.

### Basic Syntax:
```javascript
const value = useContext(MyContext);
```

### How it works:
- Takes a context object (created with `React.createContext`)
- Returns the current context value for that context
- The value is determined by the nearest context provider above the component
- When the provider updates, this hook triggers a re-render with the latest context value

### Example:
```javascript
// Create context
const ThemeContext = React.createContext('light');

// Provider component
function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <Toolbar />
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </ThemeContext.Provider>
  );
}

// Consumer component (deep in the tree)
function ThemedButton() {
  const theme = useContext(ThemeContext);

  return (
    <button className={theme}>
      I am styled by theme context!
    </button>
  );
}
```

### When to use:
- Sharing data that can be considered "global" (user info, theme, language)
- Avoiding prop drilling (passing props through many levels)
- When multiple components at different levels need the same data

### Key Points:
- Component calling `useContext` will re-render when context value changes
- Always re-renders when context changes (even if only using part of the value)
- For optimization, split context or use `useMemo` on context value
- Context is not for state management (it just passes data down)

---

## useReducer

`useReducer` is a hook for managing complex state logic. It's an alternative to `useState` and works like Redux reducers.

### Basic Syntax:
```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

### How it works:
- Takes a reducer function and initial state
- Returns current state and a dispatch function
- Call `dispatch` with an action to update state
- Reducer function determines how state changes based on the action

### Example:
```javascript
// Reducer function
function counterReducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    default:
      return state;
  }
}

// Component using useReducer
function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

### useState vs useReducer:

| Feature | useState | useReducer |
|---------|----------|------------|
| Complexity | Simple state | Complex state logic |
| Related updates | Independent | Multiple related updates |
| Next state | Directly set | Computed from action |
| Testing | Harder | Easier (pure function) |
| Use case | Single values | Objects with multiple sub-values |

### When to use useReducer:
- State has complex update logic
- Multiple sub-values that depend on each other
- Next state depends on previous state
- Want to separate state logic from component
- Need to optimize performance (dispatch is stable, doesn't change)

### Key Points:
- Reducer function must be pure (no side effects)
- Action is just an object (usually with `type` property)
- Can pass payload in action: `dispatch({ type: 'add', payload: value })`
- `dispatch` function identity is stable (won't change on re-renders)
- Good for form handling with multiple fields

---

## useCallback

`useCallback` is used to memoize functions so they keep the same reference across re-renders. This prevents unnecessary re-renders of child components.

### Basic Syntax:
```javascript
const memoizedCallback = useCallback(() => {
  // Your function logic
}, [dependencies]);
```

### The Problem Without useCallback:

```javascript
function Parent() {
  const [count, setCount] = useState(0);

  // ❌ New function created on EVERY render
  const handleClick = () => {
    console.log('Clicked!');
  };

  return (
    <div>
      <Child onClick={handleClick} />
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
    </div>
  );
}

const Child = React.memo(({ onClick }) => {
  console.log('Child rendered'); // Logs on every parent re-render!
  return <button onClick={onClick}>Click me</button>;
});
```

**Problem:** Even though `Child` is wrapped with `React.memo`, it still re-renders every time because `handleClick` is a new function on each render (different reference).

### The Solution With useCallback:

```javascript
function Parent() {
  const [count, setCount] = useState(0);

  // ✅ Same function reference across re-renders
  const handleClick = useCallback(() => {
    console.log('Clicked!');
  }, []); // Dependencies array

  return (
    <div>
      <Child onClick={handleClick} />
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
    </div>
  );
}

const Child = React.memo(({ onClick }) => {
  console.log('Child rendered'); // Only logs when onClick actually changes
  return <button onClick={onClick}>Click me</button>;
});
```

**Solution:** `useCallback` returns the same function reference unless dependencies change, so `Child` doesn't re-render unnecessarily.

### Key Use Cases:

**1. Passing callbacks to memoized child components:**
```javascript
const MemoizedChild = React.memo(Child);

function Parent() {
  const handleClick = useCallback(() => {
    // Do something
  }, []);

  return <MemoizedChild onClick={handleClick} />;
}
```

**2. Function is a dependency of useEffect:**
```javascript
function SearchComponent() {
  const [query, setQuery] = useState('');

  const fetchData = useCallback(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => console.log(data));
  }, [query]); // Only recreate when query changes

  useEffect(() => {
    fetchData();
  }, [fetchData]); // Won't cause infinite loops
}
```

**3. Function passed to custom hooks:**
```javascript
function useDebounce(callback, delay) {
  useEffect(() => {
    const timer = setTimeout(callback, delay);
    return () => clearTimeout(timer);
  }, [callback, delay]); // callback should be stable
}

function Component() {
  const handleSearch = useCallback(() => {
    // Search logic
  }, []);

  useDebounce(handleSearch, 500);
}
```

### When NOT to Use useCallback:

- Function is not passed to child components
- Function is not used in dependencies
- Premature optimization (adds overhead)
- Simple event handlers that aren't causing performance issues

```javascript
// ❌ Unnecessary - not passed to children or used in deps
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);

// ✅ Just use regular function
const handleClick = () => {
  console.log('clicked');
};
```

### Key Points:
- useCallback memoizes the function itself
- Returns same function reference if dependencies haven't changed
- Prevents unnecessary re-renders of child components
- Use with React.memo for optimization
- Don't overuse - has its own overhead
- Different from useMemo: `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`
