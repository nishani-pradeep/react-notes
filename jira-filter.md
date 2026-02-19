# Jira Filter System - Complete Interview Guide

## 📚 The Problem (Exact Palo Alto Networks Question)

> **"Design the search and filter section of a ticket management system like Jira or ClickUp. The system receives data in CHUNKS, and dropdowns may contain HUNDREDS OF THOUSANDS of items."**

**Asked in:** Palo Alto Networks Senior Staff UI Engineer interviews (2024-2025)

---

## 🎯 Step 1: Understanding the Challenge

### Why is this hard?

**Normal dropdown:** 10-50 items
```javascript
// Easy - just render all
<select>
  {users.map(user => <option>{user.name}</option>)}
</select>
```

**This question:** 100,000 items
```javascript
// ❌ This will CRASH the browser!
<select>
  {100000users.map(user => <option>{user.name}</option>)}
</select>
```

### The Core Problems:

1. **DOM Performance** - Browser can't handle 100K DOM nodes
2. **Memory** - Would consume gigabytes of RAM
3. **Network** - Can't send 100K items in one response
4. **Search** - User can't scroll through 100K items
5. **Render Time** - Would take 10+ seconds to render

**Real-World Numbers:**
```
Each DOM node: ~200 bytes
100,000 nodes × 200 bytes = 20 MB just for DOM
Plus React overhead: ~100 MB
Plus event listeners: ~50 MB
Total: 170+ MB for ONE dropdown!

Browser becomes unresponsive at ~10K nodes
JavaScript execution time: 5-10 seconds
User experience: TERRIBLE ❌
```

---

## 🏗️ Step 2: The Solution Strategy

### Three Key Techniques:

#### **1. Virtual Scrolling (Windowing)**
```
Instead of rendering all 100K items:
┌─────────────────┐
│ Item 1          │ ← Rendered (visible)
│ Item 2          │ ← Rendered (visible)
│ Item 3          │ ← Rendered (visible)
│ ...             │
│ Item 99998      │ ← NOT rendered
│ Item 99999      │ ← NOT rendered
│ Item 100000     │ ← NOT rendered
└─────────────────┘

Only render ~20 visible items at a time!
Memory: 170 MB → 340 KB (500x reduction!)
```

#### **2. Server-Side Filtering**
```
❌ DON'T: Download all 100K items, filter on client
✅ DO: Send search query to server, get filtered results

User types "John" → API request → Server filters → Returns 50 matching results

Network: 20 MB download → 10 KB download (2000x reduction!)
```

#### **3. Chunked Data Loading**
```
❌ DON'T: Load all tickets at once
✅ DO: Load in pages (chunks) of 50

Initial load: Tickets 1-50
Scroll down: Tickets 51-100
Scroll down: Tickets 101-150

Initial load: 5 seconds → 200ms (25x faster!)
```

---

## 💻 Step 3: Implementation Walkthrough

### Part A: Virtual Scrolling Dropdown

Let me explain how virtual scrolling works:

```javascript
// Imagine dropdown with 100,000 items
// Container height: 400px
// Each item height: 50px
// Visible items: 400/50 = 8 items

function VirtualDropdown({ items }) {
  const [scrollTop, setScrollTop] = useState(0);

  // Calculate which items are visible
  const itemHeight = 50;
  const containerHeight = 400;

  // If scrolled 500px down, start at item 10 (500/50)
  const startIndex = Math.floor(scrollTop / itemHeight);
  // Show next 8 items
  const visibleCount = Math.ceil(containerHeight / itemHeight);
  const endIndex = startIndex + visibleCount;

  // Only render these items!
  const visibleItems = items.slice(startIndex, endIndex);

  // But trick: make scrollbar think there are 100K items
  const totalHeight = items.length * itemHeight;

  return (
    <div
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.target.scrollTop)}
    >
      {/* Fake div to create scroll space */}
      <div style={{ height: totalHeight, position: 'relative' }}>
        {/* Actual items, positioned correctly */}
        <div style={{ transform: `translateY(${startIndex * itemHeight}px)` }}>
          {visibleItems.map((item, i) => (
            <div key={startIndex + i} style={{ height: itemHeight }}>
              {item.name}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

**Visual Explanation:**
```
Scrollbar shows 100K items (fake)
     ↓
┌──────────────────────┐
│ [Scroll Bar]         │ ← Thinks there are 100K items
│                      │
│ Item 95  ← Rendered  │ ← Only these 8 items
│ Item 96  ← Rendered  │   actually exist in DOM
│ Item 97  ← Rendered  │
│ Item 98  ← Rendered  │
│ Item 99  ← Rendered  │
│ Item 100 ← Rendered  │
│ Item 101 ← Rendered  │
│ Item 102 ← Rendered  │
│                      │
└──────────────────────┘

DOM nodes: 100,000 → 8
Memory used: 8 items instead of 100K! 🎉
Render time: 5000ms → 5ms (1000x faster!)
```

**How the Math Works:**
```javascript
// User scrolls to position 2500px
scrollTop = 2500;
itemHeight = 50;

// Calculate visible range
startIndex = Math.floor(2500 / 50) = 50
visibleCount = Math.ceil(400 / 50) = 8
endIndex = 50 + 8 = 58

// Only render items 50-58 (8 items total)
visibleItems = items.slice(50, 58);

// Position them correctly
offset = 50 × 50 = 2500px
// Use transform for GPU acceleration
transform: translateY(2500px)
```

---

### Part B: Server-Side Search

```javascript
// ❌ BAD: Client-side filtering with 100K items
function ClientSideSearch() {
  const [allUsers] = useState(/* 100K users */);
  const [search, setSearch] = useState('');

  // This is SLOW - loops through 100K items every keystroke!
  const filtered = allUsers.filter(user =>
    user.name.toLowerCase().includes(search.toLowerCase())
  );

  return <Dropdown options={filtered} />;
}

/*
Performance Analysis:
- Initial load: Download 100K users (20 MB, 5 seconds)
- Every keystroke: Loop through 100K items (50-100ms)
- Memory: 100K objects in memory (~100 MB)
- User types "John": 4 loops = 400ms lag ❌
*/

// ✅ GOOD: Server-side filtering
function ServerSideSearch() {
  const [search, setSearch] = useState('');
  const debouncedSearch = useDebounce(search, 300);

  // Only fetch matching results from server
  const { data } = useQuery({
    queryKey: ['users', debouncedSearch],
    queryFn: () => fetch(`/api/users?search=${debouncedSearch}&limit=50`)
      .then(r => r.json())
  });

  // Only 50 results, not 100K!
  return <Dropdown options={data || []} />;
}

/*
Performance Analysis:
- Initial load: Nothing! Just render empty dropdown
- User types: Wait 300ms, then one API call
- Server filters: Database index makes it instant (<10ms)
- Network: Download 50 results (1 KB, 50ms)
- Memory: Only 50 objects (~10 KB)
- Total time: 350ms (includes network) ✅
*/
```

**Why Debouncing?**
```
Without debounce:
User types "John"
J → API call (50ms)
Jo → API call (50ms) ← Cancels previous
Joh → API call (50ms) ← Cancels previous
John → API call (50ms) ← Cancels previous
= 4 API calls, 3 wasted! Server load 4x ❌

With 300ms debounce:
J → wait...
Jo → wait...
Joh → wait...
John → wait 300ms → API call (50ms)
= 1 API call! 🎉
User feels: 350ms response time
Server load: 75% reduction
```

**Debounce Implementation:**
```javascript
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    // Set timeout to update debounced value
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // Cleanup: cancel timeout if value changes
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function SearchInput() {
  const [input, setInput] = useState('');
  const debouncedInput = useDebounce(input, 300);

  // This only changes 300ms after user stops typing
  useEffect(() => {
    if (debouncedInput) {
      searchAPI(debouncedInput);
    }
  }, [debouncedInput]);

  return <input value={input} onChange={e => setInput(e.target.value)} />;
}
```

---

### Part C: Complete Jira Filter System

Now let's put it all together:

```javascript
function JiraFilterSystem() {
  // State for all filters
  const [filters, setFilters] = useState({
    search: '',
    status: [],      // ['open', 'in-progress']
    priority: [],    // ['high', 'critical']
    assignee: [],    // ['user-123']
    tags: []         // ['frontend', 'bug']
  });

  // Debounce to avoid excessive API calls
  const debouncedFilters = useDebounce(filters, 300);

  // Fetch filtered tickets from server
  const { data, isLoading } = useQuery({
    queryKey: ['tickets', debouncedFilters],
    queryFn: () => fetchTickets(debouncedFilters)
  });

  return (
    <div className="jira-filter-system">
      {/* Search Input */}
      <input
        value={filters.search}
        onChange={(e) => setFilters({
          ...filters,
          search: e.target.value
        })}
        placeholder="Search tickets..."
      />

      {/* Status Dropdown - Small list (3-5 items), can be client-side */}
      <Dropdown
        options={['open', 'closed', 'in-progress', 'resolved']}
        value={filters.status}
        onChange={(val) => setFilters({...filters, status: val})}
      />

      {/* Priority Dropdown - Small list (4-5 items), can be client-side */}
      <Dropdown
        options={['low', 'medium', 'high', 'critical']}
        value={filters.priority}
        onChange={(val) => setFilters({...filters, priority: val})}
      />

      {/* Assignee Dropdown - 50K users, needs virtual scroll + server search */}
      <VirtualizedSearchableDropdown
        searchEndpoint="/api/users/search"
        value={filters.assignee}
        onChange={(val) => setFilters({...filters, assignee: val})}
        placeholder="Search assignees..."
        itemHeight={48}
        maxHeight={400}
      />

      {/* Tags Multi-Select - 10K tags, needs virtual scroll + server search */}
      <VirtualizedSearchableDropdown
        searchEndpoint="/api/tags/search"
        value={filters.tags}
        onChange={(val) => setFilters({...filters, tags: val})}
        placeholder="Search tags..."
        multiple
        itemHeight={48}
        maxHeight={400}
      />

      {/* Active Filters Display */}
      <div className="active-filters">
        {filters.status.map(status => (
          <Chip
            key={status}
            label={`Status: ${status}`}
            onRemove={() => {/* remove filter */}}
          />
        ))}
        {/* Similar for other filters */}
      </div>

      {/* Results Grid - Also needs virtual scrolling */}
      <VirtualizedTicketGrid
        tickets={data?.tickets || []}
        loading={isLoading}
      />
    </div>
  );
}
```

**Filter Decision Tree:**
```
For each filter, ask:
1. How many items?
   - < 10: Regular dropdown ✅
   - 10-1000: Regular dropdown with search
   - > 1000: Virtual scrolling + server search

2. Static or dynamic?
   - Static: Can hardcode options
   - Dynamic: Fetch from API

3. Searchable?
   - No: Simple dropdown
   - Yes: Need search input + debouncing

Status/Priority: ~5 items → Regular dropdown
Assignee: 50K items → Virtual + Server + Search
Tags: 10K items → Virtual + Server + Search
```

---

### Part D: VirtualizedSearchableDropdown Component

Complete implementation with all features:

```javascript
import { FixedSizeList } from 'react-window';

function VirtualizedSearchableDropdown({
  searchEndpoint,
  value,
  onChange,
  placeholder,
  multiple = false,
  itemHeight = 48,
  maxHeight = 400
}) {
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');
  const debouncedQuery = useDebounce(searchQuery, 300);

  // Fetch results from server
  const { data, isLoading } = useQuery({
    queryKey: ['dropdown', searchEndpoint, debouncedQuery],
    queryFn: () =>
      fetch(`${searchEndpoint}?search=${debouncedQuery}&limit=100`)
        .then(r => r.json()),
    enabled: isOpen, // Only fetch when dropdown is open
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
  });

  const options = data?.data || [];
  const visibleCount = Math.min(options.length, Math.floor(maxHeight / itemHeight));

  // Render each row in virtual list
  const Row = ({ index, style }) => {
    const option = options[index];
    const isSelected = multiple
      ? value.includes(option.id)
      : value === option.id;

    return (
      <div
        style={style}
        className={`dropdown-option ${isSelected ? 'selected' : ''}`}
        onClick={() => {
          if (multiple) {
            const newValue = isSelected
              ? value.filter(id => id !== option.id)
              : [...value, option.id];
            onChange(newValue);
          } else {
            onChange(option.id);
            setIsOpen(false);
          }
        }}
        role="option"
        aria-selected={isSelected}
      >
        {multiple && (
          <input
            type="checkbox"
            checked={isSelected}
            readOnly
          />
        )}
        <span className="option-label">{option.label}</span>
        {option.count && (
          <span className="option-count">({option.count})</span>
        )}
      </div>
    );
  };

  return (
    <div className="virtualized-dropdown">
      {/* Trigger Button */}
      <button
        className="dropdown-trigger"
        onClick={() => setIsOpen(!isOpen)}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
      >
        {value.length > 0 ? `${value.length} selected` : placeholder}
      </button>

      {/* Dropdown Menu */}
      {isOpen && (
        <div className="dropdown-menu" role="listbox">
          {/* Search Input */}
          <div className="dropdown-search">
            <input
              type="text"
              placeholder="Search..."
              value={searchQuery}
              onChange={(e) => setSearchQuery(e.target.value)}
              autoFocus
            />
            {isLoading && <Spinner size="small" />}
          </div>

          {/* Virtual List */}
          {options.length > 0 ? (
            <FixedSizeList
              height={Math.min(maxHeight, options.length * itemHeight)}
              itemCount={options.length}
              itemSize={itemHeight}
              width="100%"
            >
              {Row}
            </FixedSizeList>
          ) : (
            <div className="dropdown-empty">
              {isLoading ? 'Loading...' : 'No results found'}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

---

## 🎯 Step 4: Key Interview Points to Mention

### 1. **Why Virtual Scrolling?**
```
"For 100K items, rendering all would create 100K DOM nodes.
Modern browsers struggle after ~10K nodes.

Performance Impact:
- 100K nodes: ~5-10 seconds render time
- Browser becomes unresponsive
- Memory usage: ~170 MB per dropdown
- JavaScript execution: Main thread blocked

Virtual scrolling only renders visible items - typically 10-20.
This reduces:
- Render time: 5000ms → 5ms (1000x improvement)
- Memory: 170 MB → 340 KB (500x reduction)
- DOM nodes: 100K → 20 (5000x reduction)"
```

### 2. **Client vs Server Filtering Trade-off**

```
Client-Side Filtering:
✅ Instant filtering (no network delay)
✅ Works offline
✅ No server load
❌ Must download all data first (20 MB, 5 seconds)
❌ Memory intensive (100 MB+ in RAM)
❌ Slow initial load
❌ Filtering still takes 50-100ms for 100K items

Server-Side Filtering:
✅ Handles millions of items
✅ Fast initial load (no download needed)
✅ Fresh data always
✅ Database indexes make search instant (<10ms)
✅ Low memory usage (only current results)
❌ Network latency (~300ms per search with debounce)
❌ Requires backend support
❌ Doesn't work offline

For 100K items: Use server-side with caching
- Store last search results in memory
- 5-minute cache TTL
- Prefetch popular searches
- Result: Fast + Scalable 🎉
```

### 3. **Debouncing Strategy**

```
"I'd use 300ms debounce for search input.

Why 300ms?
- Too short (50ms): Too many API calls, server overload
- Too long (1000ms): Feels sluggish, poor UX
- 300ms: Sweet spot - feels instant, reduces API calls by 75%

What it does:
- User types 'bug fix' (7 characters)
- Without debounce: 7 API calls
- With 300ms debounce: 1 API call
- Reduction: 85% fewer requests

Implementation:
- Use setTimeout in useEffect
- Clear timeout on every keystroke
- Only fire API call after 300ms of silence

Alternative considered: Throttling
- Throttling: Execute every 300ms while typing
- Debouncing: Execute 300ms after stopped typing
- For search, debouncing is better (fewer API calls)
- For scroll events, throttling is better (continuous feedback)"
```

### 4. **Caching Strategy**

```javascript
// Three-level caching strategy

// Level 1: React Query in-memory cache
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes - data is fresh
      gcTime: 10 * 60 * 1000, // 10 minutes - keep in cache
    },
  },
});

// Level 2: Session Storage for dropdown options
// (Survives page refresh within session)
function useCachedDropdownOptions(key, fetcher) {
  const [data, setData] = useState(() => {
    const cached = sessionStorage.getItem(key);
    return cached ? JSON.parse(cached) : null;
  });

  useEffect(() => {
    if (!data) {
      fetcher().then(newData => {
        setData(newData);
        sessionStorage.setItem(key, JSON.stringify(newData));
      });
    }
  }, [data, fetcher, key]);

  return data;
}

// Level 3: Service Worker for offline support
// (Caches API responses)
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});

// Cache invalidation strategies:
// 1. Time-based: 5-minute TTL
// 2. Event-based: Invalidate on create/update/delete
// 3. Manual: "Refresh" button

// Example: Invalidate assignee cache when new user added
function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createUser,
    onSuccess: () => {
      // Invalidate all assignee searches
      queryClient.invalidateQueries({ queryKey: ['assignees'] });
      sessionStorage.removeItem('assignees-cache');
    },
  });
}
```

### 5. **Request Cancellation**

```javascript
// Problem: User types fast, multiple API calls in flight
// Solution: Cancel previous requests

// React Query handles this automatically!
function useAssigneeSearch(query) {
  return useQuery({
    queryKey: ['assignees', query],
    queryFn: ({ signal }) =>
      fetch(`/api/users?search=${query}`, {
        signal // AbortSignal passed by React Query
      }).then(r => r.json()),
    enabled: query.length > 0
  });
}

// What happens:
// 1. User types "J" → Request 1 starts
// 2. User types "Jo" → Request 1 CANCELLED, Request 2 starts
// 3. User types "Joh" → Request 2 CANCELLED, Request 3 starts
// 4. User types "John" → Request 3 CANCELLED, Request 4 starts
// 5. Only Request 4 completes and updates UI

// Manual implementation (without React Query):
function useSearchWithCancellation(query) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;

    // Create AbortController for this request
    const controller = new AbortController();
    setLoading(true);

    fetch(`/api/search?q=${query}`, {
      signal: controller.signal
    })
      .then(r => r.json())
      .then(data => setData(data))
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error('Search failed:', err);
        }
      })
      .finally(() => setLoading(false));

    // Cleanup: abort request if query changes
    return () => controller.abort();
  }, [query]);

  return { data, loading };
}

// Benefits:
// - Saves bandwidth (cancelled requests stop downloading)
// - Prevents race conditions (old results don't overwrite new)
// - Reduces server load (aborted requests stop processing)
// - Better UX (no outdated results flashing)
```

---

## 📊 Step 5: Architecture Diagram

Draw this during your interview:

```
┌─────────────────────────────────────────────────────────┐
│                   User Interface                        │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────┐   │
│  │  Search Input                                   │   │
│  │  (debounced 300ms)                             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ Status       │  │ Priority     │  │ Assignee    │  │
│  │ Dropdown     │  │ Dropdown     │  │ Dropdown    │  │
│  │ (client)     │  │ (client)     │  │ (virtual +  │  │
│  │ 4 items      │  │ 4 items      │  │  server)    │  │
│  │              │  │              │  │ 50K items   │  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Active Filters Chips                          │   │
│  │  [Status: Open ✕] [Priority: High ✕]          │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Virtual Scrolling Grid                        │   │
│  │  (renders 20 rows out of 100K)                │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────┬──────────────────────────────────────────┘
               │
               │ HTTP GET /api/tickets?
               │   search=bug&
               │   status=open&
               │   assignee=user-123&
               │   page=1&limit=50
               │
               │ (AbortController for cancellation)
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│              Backend API Server                         │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────┐   │
│  │  GET /api/tickets                              │   │
│  │  - Parse query parameters                      │   │
│  │  - Validate inputs                             │   │
│  │  - Build database query                        │   │
│  │  - Return paginated results                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  GET /api/users/search?query=john              │   │
│  │  - Full-text search on name/email              │   │
│  │  - Return top 50 matches                       │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────┬──────────────────────────────────────────┘
               │
               │ SQL Query with indexes
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL Database                        │
├─────────────────────────────────────────────────────────┤
│  Tickets Table:                                         │
│  - id (PRIMARY KEY, indexed)                           │
│  - title (INDEXED for full-text search)               │
│  - status (INDEXED for filtering)                      │
│  - priority (INDEXED)                                  │
│  - assignee_id (FOREIGN KEY, INDEXED)                 │
│  - created_at (INDEXED for sorting)                    │
│                                                         │
│  Users Table:                                          │
│  - id (PRIMARY KEY)                                    │
│  - name (INDEXED for search)                          │
│  - email (INDEXED)                                     │
│                                                         │
│  Query Example:                                        │
│  SELECT * FROM tickets                                 │
│  WHERE title ILIKE '%bug%'                            │
│    AND status = 'open'                                │
│    AND assignee_id = 'user-123'                       │
│  ORDER BY created_at DESC                              │
│  LIMIT 50 OFFSET 0;                                    │
│                                                         │
│  Execution time: ~10ms (with indexes)                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              Redis Cache (Optional)                     │
├─────────────────────────────────────────────────────────┤
│  - Cache frequent searches                              │
│  - Store aggregated filter options                      │
│  - TTL: 5 minutes                                       │
│  - Reduces database load by 80%                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Step 6: Performance Optimizations

### Optimization 1: Prefetching

```javascript
// Prefetch next page when user scrolls to 80%
function TicketGrid({ tickets }) {
  const { fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey: ['tickets'],
    queryFn: ({ pageParam = 1 }) =>
      fetchTickets({ page: pageParam, limit: 50 }),
    getNextPageParam: (lastPage) =>
      lastPage.hasMore ? lastPage.page + 1 : undefined,
  });

  const handleScroll = (e) => {
    const { scrollTop, scrollHeight, clientHeight } = e.target;
    const scrollPercentage = (scrollTop + clientHeight) / scrollHeight;

    // Start loading next page at 80% scroll
    if (scrollPercentage > 0.8 && hasNextPage) {
      fetchNextPage();
    }
  };

  return (
    <div
      onScroll={handleScroll}
      style={{ height: '600px', overflow: 'auto' }}
    >
      {tickets.map(ticket => (
        <TicketRow key={ticket.id} ticket={ticket} />
      ))}
    </div>
  );
}

// Advanced: Prefetch on hover (predictive loading)
function TicketRow({ ticket }) {
  const queryClient = useQueryClient();

  const handleMouseEnter = () => {
    // Prefetch ticket details before user clicks
    queryClient.prefetchQuery({
      queryKey: ['ticket', ticket.id],
      queryFn: () => fetchTicketDetails(ticket.id),
    });
  };

  return (
    <div onMouseEnter={handleMouseEnter}>
      {ticket.title}
    </div>
  );
}

// Benefits:
// - Next page loads seamlessly (no loading spinner)
// - Ticket details appear instantly on click
// - Feels like native app
```

### Optimization 2: Memoization

```javascript
// Prevent unnecessary re-renders of ticket rows
const TicketRow = memo(({ ticket, onSelect }) => {
  console.log('Rendering ticket:', ticket.id);

  return (
    <div onClick={() => onSelect(ticket.id)}>
      <h3>{ticket.title}</h3>
      <p>{ticket.description}</p>
    </div>
  );
}, (prevProps, nextProps) => {
  // Custom comparison: only re-render if ticket data changed
  return (
    prevProps.ticket.id === nextProps.ticket.id &&
    prevProps.ticket.title === nextProps.ticket.title &&
    prevProps.ticket.status === nextProps.ticket.status
  );
});

// Memoize expensive filter computations
function FilterBar({ allTickets, filters }) {
  // This only recalculates when tickets or filters change
  const filteredCount = useMemo(() => {
    return allTickets.filter(ticket =>
      matchesFilters(ticket, filters)
    ).length;
  }, [allTickets, filters]);

  return <div>{filteredCount} tickets match filters</div>;
}

// Memoize callbacks to prevent child re-renders
function TicketList({ tickets }) {
  // Without useCallback, this creates new function every render
  // causing all TicketRow components to re-render
  const handleSelect = useCallback((ticketId) => {
    console.log('Selected:', ticketId);
  }, []);

  return tickets.map(ticket => (
    <TicketRow
      key={ticket.id}
      ticket={ticket}
      onSelect={handleSelect} // Same reference every time
    />
  ));
}

// Performance impact:
// Without memo: 100 ticket rows re-render on every filter change
// With memo: Only changed rows re-render (typically 0-5)
// Time saved: 200ms → 5ms per filter change
```

### Optimization 3: URL Synchronization

```javascript
// Make filters shareable via URL
function useFilterState() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Read filters from URL
  const filters = useMemo(() => ({
    search: searchParams.get('search') || '',
    status: searchParams.getAll('status'),
    priority: searchParams.getAll('priority'),
    assignee: searchParams.getAll('assignee'),
  }), [searchParams]);

  // Update URL when filters change
  const updateFilters = useCallback((newFilters) => {
    const params = new URLSearchParams();

    if (newFilters.search) {
      params.set('search', newFilters.search);
    }

    newFilters.status.forEach(s => params.append('status', s));
    newFilters.priority.forEach(p => params.append('priority', p));
    newFilters.assignee.forEach(a => params.append('assignee', a));

    setSearchParams(params, { replace: true });
  }, [setSearchParams]);

  return [filters, updateFilters];
}

// Usage
function FilterBar() {
  const [filters, setFilters] = useFilterState();

  // URL updates automatically!
  // /tickets?search=bug&status=open&priority=high

  // Users can now:
  // 1. Bookmark filtered views
  // 2. Share links with filters applied
  // 3. Use browser back/forward buttons
  // 4. Refresh page without losing filters
}

// Example URLs:
// /tickets?search=login+bug&status=open
// /tickets?priority=high&priority=critical&assignee=user-123
// /tickets?search=performance&status=open&status=in-progress
```

### Optimization 4: Optimistic Updates

```javascript
// Update UI immediately, sync with server in background
function useUpdateTicketStatus() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ ticketId, newStatus }) =>
      updateTicketStatus(ticketId, newStatus),

    // Before API call: Update UI optimistically
    onMutate: async ({ ticketId, newStatus }) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['tickets'] });

      // Snapshot current state (for rollback)
      const previousTickets = queryClient.getQueryData(['tickets']);

      // Optimistically update cache
      queryClient.setQueryData(['tickets'], (old) => {
        return old.map(ticket =>
          ticket.id === ticketId
            ? { ...ticket, status: newStatus }
            : ticket
        );
      });

      // Return context for rollback
      return { previousTickets };
    },

    // On error: Rollback
    onError: (err, variables, context) => {
      queryClient.setQueryData(['tickets'], context.previousTickets);
      toast.error('Failed to update ticket');
    },

    // On success: Refetch to ensure consistency
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tickets'] });
      toast.success('Ticket updated');
    },
  });
}

// User experience:
// Without optimistic: Click → Wait 300ms → UI updates
// With optimistic: Click → UI updates instantly → Syncs in background
// Feels: 3x faster!
```

### Optimization 5: Intersection Observer

```javascript
// Load ticket details only when visible on screen
function TicketRow({ ticket }) {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect(); // Only observe once
        }
      },
      { threshold: 0.1 } // Trigger when 10% visible
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, []);

  // Only fetch details when visible
  const { data } = useQuery({
    queryKey: ['ticket-details', ticket.id],
    queryFn: () => fetchTicketDetails(ticket.id),
    enabled: isVisible,
  });

  return (
    <div ref={ref}>
      {isVisible ? (
        <TicketDetails data={data} />
      ) : (
        <TicketSkeleton />
      )}
    </div>
  );
}

// Performance impact:
// Without: Load 100 tickets' details on page load (100 API calls)
// With: Load only visible ~10 tickets (10 API calls)
// Network: 90% reduction
// Initial load: 5s → 500ms
```

---

## ✅ Interview Answer Template

Here's how to answer in the interview:

### **1. Clarify Requirements (2 min)**
```
"Let me clarify a few things:

Functional:
- What are we filtering? Tickets, issues, tasks?
- What filter types? Search, dropdowns, date ranges?
- Single or multi-select for dropdowns?

Scale:
- How many total tickets? (Assume 100K)
- How many users/assignees? (Assume 50K)
- How many tags? (Assume 10K)
- Concurrent users? (Assume 1000+)

Performance:
- Expected response time? (< 100ms for interactions)
- Initial load time acceptable? (< 2 seconds)

Other:
- Browser support? (Modern browsers)
- Mobile responsive? (Yes)
- Accessibility requirements? (WCAG AA)
- Offline support needed? (No)"
```

### **2. Identify Key Challenges (2 min)**
```
"The main challenges are:

1. DOM Performance
   - Can't render 100K items in dropdown
   - Browser becomes unresponsive after 10K nodes
   - Would use 170 MB+ memory per dropdown

2. Network Performance
   - Can't download 100K items (20 MB)
   - Would take 5+ seconds initial load
   - User can't wait that long

3. Search Experience
   - User can't scroll through 100K items
   - Need instant search results
   - Can't filter 100K items on every keystroke

4. Chunked Data
   - Backend sends data in pages
   - Need infinite scroll or pagination
   - Must handle loading states"
```

### **3. Propose High-Level Solution (3 min)**
```
"I'd use a three-part approach:

1. Virtual Scrolling (Windowing)
   - Only render visible items (~10-20)
   - Use react-window library
   - Reduces 100K DOM nodes to 20
   - Memory: 170 MB → 340 KB

2. Server-Side Filtering
   - Send search query to backend
   - Database uses indexes for fast filtering
   - Return top 50 results
   - Much faster than client-side filtering

3. Debouncing + Request Cancellation
   - Wait 300ms after user stops typing
   - Cancel previous requests
   - Prevents API spam
   - Reduces server load 75%

This gives us:
- Fast initial load (<200ms)
- Instant feeling search (<350ms with network)
- Scalable to millions of items
- Low memory footprint"
```

### **4. Draw Architecture (2 min)**
```
[Draw the architecture diagram from Step 5]

"Key components:
- Search input with 300ms debounce
- Small dropdowns (status, priority) - client-side
- Large dropdowns (assignee, tags) - virtual + server
- Virtual scrolling grid for results
- URL state for shareable links
- React Query for caching and deduplication"
```

### **5. Dive Into Implementation (5 min)**
```
"Let me explain virtual scrolling in detail:

[Explain with code example from Part A]

For server-side search:
[Explain with code example from Part B]

For debouncing:
[Explain the debounce implementation]"
```

### **6. Discuss Optimizations (5 min)**
```
"Several optimizations I'd add:

1. Caching
   - React Query with 5-minute stale time
   - Session storage for dropdown options
   - 80% reduction in API calls

2. Prefetching
   - Load next page at 80% scroll
   - Prefetch on hover
   - Seamless infinite scroll

3. Memoization
   - memo() for ticket rows
   - useMemo() for expensive filters
   - 10x faster re-renders

4. Optimistic Updates
   - Update UI before server confirms
   - Rollback on error
   - Feels instant to user

5. Intersection Observer
   - Load ticket details when visible
   - 90% fewer initial API calls
   - Faster page load"
```

### **7. Discuss Trade-offs (3 min)**
```
"Key trade-offs:

1. Client vs Server Filtering
   - Client: Instant but can't scale
   - Server: Slight delay but handles millions
   - Choice: Server-side with caching

2. Debounce Delay
   - 50ms: Too many API calls
   - 1000ms: Feels sluggish
   - 300ms: Sweet spot

3. Virtual Scroll Library
   - react-window: Battle-tested, 2KB
   - Custom: More control, more code
   - Choice: react-window for reliability

4. Caching Duration
   - 30s: Fresh data, more API calls
   - 30min: Stale risk, fewer calls
   - 5min: Good balance"
```

### **8. Monitoring & Observability (2 min)**
```
"I'd track these metrics:

1. Performance
   - Search response time (p50, p95, p99)
   - Time to interactive
   - Virtual scroll frame rate

2. Usage
   - Most searched terms
   - Most used filters
   - Conversion rate (search → select)

3. Errors
   - Failed API calls
   - Timeout rate
   - Client-side errors

4. Business
   - Tickets filtered per session
   - Time to find ticket
   - User satisfaction"
```

---

## 🎓 Practice Exercises

### Exercise 1: Handle 1 Million Items

**Q:** "What if we have 1 million assignees instead of 50K?"

<details>
<summary>Click for answer</summary>

```
Changes needed:

1. Require Minimum Search Length
   - Don't search until user types 3+ characters
   - Reduces search space significantly
   - "j" → No search
   - "joh" → Search (much fewer matches)

2. Add Search Suggestions
   - Show recent searches
   - Show popular assignees
   - Guide user to be specific

3. Limit Results
   - Return max 100 results even if 1000 match
   - Add message: "Showing top 100 results, refine search"
   - Force user to narrow down

4. Use Elasticsearch
   - PostgreSQL full-text search may be slow
   - Elasticsearch optimized for this
   - Sub-10ms search even on millions

5. Database Optimization
   - Partition users table by first letter
   - More aggressive indexing
   - Read replicas for search

Code example:
```javascript
function AssigneeDropdown() {
  const [query, setQuery] = useState('');

  // Only search with 3+ characters
  const shouldSearch = query.length >= 3;

  const { data } = useQuery({
    queryKey: ['assignees', query],
    queryFn: () => searchAssignees(query, { limit: 100 }),
    enabled: shouldSearch,
  });

  return (
    <VirtualDropdown
      options={data || []}
      placeholder="Type at least 3 characters..."
      emptyMessage={
        query.length < 3
          ? "Type at least 3 characters to search"
          : "No results. Try different search terms."
      }
    />
  );
}
```
</details>

---

### Exercise 2: Offline Mode

**Q:** "How would you handle offline mode?"

<details>
<summary>Click for answer</summary>

```
Strategy:

1. Service Worker
   - Cache API responses
   - Serve cached data when offline
   - Show "Offline" indicator

2. IndexedDB Storage
   - Store last 100 viewed tickets
   - Store last 10 search queries
   - Store dropdown options

3. Queue Mutations
   - Store filter changes locally
   - Sync when back online
   - Show "Syncing..." indicator

4. Optimistic UI
   - Show cached data immediately
   - Update when back online
   - Resolve conflicts (last-write-wins)

Implementation:
```javascript
// Check online status
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return isOnline;
}

// Use cached data when offline
function useTicketsWithOffline(filters) {
  const isOnline = useOnlineStatus();

  return useQuery({
    queryKey: ['tickets', filters],
    queryFn: () => fetchTickets(filters),

    // Use cache when offline
    networkMode: isOnline ? 'online' : 'offlineFirst',

    // Cache for 1 hour
    staleTime: 60 * 60 * 1000,
    gcTime: 60 * 60 * 1000,
  });
}

// Show offline indicator
function OfflineIndicator() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div className="offline-banner">
      ⚠️ You're offline. Showing cached data.
    </div>
  );
}
```
</details>

---

### Exercise 3: Multi-Select with 50K Items

**Q:** "User wants to select multiple assignees. How do you handle 50K items?"

<details>
<summary>Click for answer</summary>

```javascript
function MultiSelectAssigneeDropdown() {
  const [selected, setSelected] = useState([]);
  const [searchQuery, setSearchQuery] = useState('');

  const { data } = useQuery({
    queryKey: ['assignees', searchQuery],
    queryFn: () => searchAssignees(searchQuery, { limit: 100 }),
  });

  const handleToggle = (userId) => {
    setSelected(prev =>
      prev.includes(userId)
        ? prev.filter(id => id !== userId)
        : [...prev, userId]
    );
  };

  // Fetch full details for selected users
  const { data: selectedUsers } = useQuery({
    queryKey: ['users', selected],
    queryFn: () => fetchUsersByIds(selected),
    enabled: selected.length > 0,
  });

  return (
    <div>
      {/* Show selected as chips */}
      <div className="selected-chips">
        {selectedUsers?.map(user => (
          <Chip
            key={user.id}
            label={user.name}
            onRemove={() => handleToggle(user.id)}
          />
        ))}
      </div>

      {/* Dropdown with checkboxes */}
      <VirtualDropdown
        options={data || []}
        selected={selected}
        onToggle={handleToggle}
        searchQuery={searchQuery}
        onSearchChange={setSearchQuery}
        renderItem={(user, isSelected) => (
          <div className="dropdown-item">
            <input
              type="checkbox"
              checked={isSelected}
              readOnly
            />
            <span>{user.name}</span>
          </div>
        )}
      />

      {/* Bulk actions */}
      <div className="bulk-actions">
        <button onClick={() => setSelected([])}>
          Clear All ({selected.length})
        </button>
      </div>
    </div>
  );
}

// For better UX with many selections:
// 1. Show count badge: "3 assignees selected"
// 2. Limit visible chips to 5, show "+10 more"
// 3. Add "Select all results" option (dangerous!)
// 4. Add "Select from saved group" feature
```
</details>

---

### Exercise 4: Real-Time Updates

**Q:** "Tickets are created/updated in real-time. How do you keep filters up-to-date?"

<details>
<summary>Click for answer</summary>

```javascript
// Use WebSocket for real-time updates
function useRealtimeTickets(filters) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket('wss://api.example.com/tickets');

    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);

      // Handle different update types
      switch (update.type) {
        case 'ticket_created':
          // Add to cache if matches filters
          if (matchesFilters(update.ticket, filters)) {
            queryClient.setQueryData(['tickets', filters], (old) => {
              return {
                ...old,
                data: [update.ticket, ...old.data]
              };
            });
          }
          break;

        case 'ticket_updated':
          // Update existing ticket
          queryClient.setQueryData(['tickets', filters], (old) => {
            return {
              ...old,
              data: old.data.map(ticket =>
                ticket.id === update.ticket.id
                  ? update.ticket
                  : ticket
              )
            };
          });
          break;

        case 'ticket_deleted':
          // Remove from cache
          queryClient.setQueryData(['tickets', filters], (old) => {
            return {
              ...old,
              data: old.data.filter(t => t.id !== update.ticketId)
            };
          });
          break;
      }
    };

    return () => ws.close();
  }, [filters, queryClient]);

  return useQuery({
    queryKey: ['tickets', filters],
    queryFn: () => fetchTickets(filters),
  });
}

// Show toast for new matching tickets
function RealtimeNotification() {
  const { data } = useRealtimeTickets();
  const [newCount, setNewCount] = useState(0);

  useEffect(() => {
    // Count increased
    if (data && prevData && data.total > prevData.total) {
      const diff = data.total - prevData.total;
      setNewCount(diff);

      toast.info(`${diff} new ticket(s) match your filters`, {
        action: {
          label: 'Refresh',
          onClick: () => refetch()
        }
      });
    }
  }, [data]);

  return newCount > 0 ? (
    <button onClick={refetch}>
      {newCount} new ticket(s) available
    </button>
  ) : null;
}

// Alternative: Polling (simpler but less efficient)
function usePollingTickets(filters) {
  return useQuery({
    queryKey: ['tickets', filters],
    queryFn: () => fetchTickets(filters),
    refetchInterval: 30000, // Poll every 30 seconds
    refetchIntervalInBackground: false, // Only when tab active
  });
}
```
</details>

---

## 📝 Summary Checklist

Before your interview, make sure you understand:

### Core Concepts
- [ ] Why 100K items can't be rendered normally (DOM/memory limits)
- [ ] How virtual scrolling works mathematically
- [ ] Client vs server-side filtering trade-offs
- [ ] Why 300ms debouncing is the sweet spot
- [ ] How AbortController cancels requests

### Implementation Details
- [ ] How to calculate visible items in virtual scroll
- [ ] How useDebounce hook works internally
- [ ] How React Query caches and deduplicates
- [ ] How to sync filters with URL
- [ ] How to implement optimistic updates

### Performance
- [ ] Memory comparison: 100K nodes vs virtual scrolling
- [ ] Network comparison: Full download vs server filtering
- [ ] Render time: With and without optimization
- [ ] Cache strategies: In-memory, session storage, service worker

### Interview Strategy
- [ ] Clarifying questions to ask
- [ ] How to draw architecture diagram
- [ ] Trade-offs to discuss
- [ ] Optimizations to mention
- [ ] How to handle follow-up questions

---

## 🎯 Interview Success Tips

### Do's ✅

1. **Start with clarifications**
   - "How many items total?"
   - "What's the expected response time?"
   - "Mobile support needed?"

2. **Think aloud**
   - "I'm considering virtual scrolling because..."
   - "The trade-off here is..."
   - "This could be optimized by..."

3. **Draw diagrams**
   - Component hierarchy
   - Data flow
   - User interaction flow

4. **Discuss trade-offs**
   - No solution is perfect
   - Explain why you chose this approach
   - Mention alternatives

5. **Mention real numbers**
   - "Reduces memory from 170 MB to 340 KB"
   - "Cuts API calls by 75%"
   - "300ms debounce is optimal"

### Don'ts ❌

1. ❌ Don't jump into code immediately
2. ❌ Don't ignore scale requirements
3. ❌ Don't forget about accessibility
4. ❌ Don't over-engineer for the sake of it
5. ❌ Don't argue with interviewer feedback

---

## 🚀 Final Interview Template

Use this structure:

```
1. Clarify (2 min)
   "Let me understand the requirements..."

2. Challenges (2 min)
   "The main challenges are..."

3. Solution (3 min)
   "I'd use virtual scrolling + server filtering + debouncing because..."

4. Architecture (2 min)
   [Draw diagram]

5. Implementation (5 min)
   "Here's how virtual scrolling works..."
   [Walk through code]

6. Optimizations (3 min)
   "We can improve this with caching, prefetching, memoization..."

7. Trade-offs (2 min)
   "I chose server-side because... Alternative would be..."

8. Monitoring (1 min)
   "We'd track search latency, API errors, user satisfaction..."

9. Questions? (2 min)
   "What would you like me to dive deeper into?"
```

---

**Total Time:** ~25 minutes (leaving 5 minutes for follow-ups)

Good luck with your Palo Alto Networks interview! 🎉

Remember: **The key is showing you can handle SCALE**. Anyone can build a dropdown for 50 items. Staff engineers design systems that work with 50,000 items! 🚀
