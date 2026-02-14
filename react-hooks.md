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
