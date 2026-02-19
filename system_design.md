# Frontend System Design Interview Preparation

## Comprehensive Guide for Palo Alto Networks Staff UI Engineer

---

## 📋 Table of Contents

1. [RADIO Framework Overview](#radio-framework)
2. [System Design Interview Structure](#interview-structure)
3. [Key Frontend System Design Patterns](#design-patterns)
4. [Common System Design Questions](#common-questions)
5. [Palo Alto Specific: Jira Filter System](#palo-alto-specific)
6. [Performance & Scalability](#performance-scalability)
7. [Practice Problems](#practice-problems)
8. [Evaluation Criteria](#evaluation-criteria)
9. [Staff-Level Expectations](#staff-level-expectations)

---

## 🎯 RADIO Framework

**R**equirements → **A**rchitecture → **D**ata Model → **I**nterface → **O**ptimizations

### **R - Requirements Exploration (10-15 mins)**

**Goal:** Understand the problem scope and clarify ambiguities

**Questions to Ask:**

#### Functional Requirements:
- What are the core features? (Must-haves vs nice-to-haves)
- Who are the users? (Internal, external, technical level)
- What devices/browsers do we support?
- What's the expected user flow?
- Are there authentication/authorization requirements?
- What data do we need to display?

#### Non-Functional Requirements:
- **Scale:** How many users? (concurrent, daily active)
- **Performance:** Response time expectations? (< 100ms, < 1s)
- **Data Volume:** How much data? (100 items, 100K items, 1M items)
- **Accessibility:** WCAG compliance level?
- **Internationalization:** Multi-language support?
- **Offline Support:** Does it need to work offline?
- **SEO:** Is server-side rendering needed?

**Example for Dropdown with 100K Items:**
```
Clarifying Questions:
✅ How many items in the dropdown? (100K confirmed)
✅ Are items static or dynamic? (Dynamic, can change)
✅ Do we need search/filter? (Yes, essential)
✅ Multi-select or single-select? (Single-select)
✅ Keyboard navigation required? (Yes, a11y requirement)
✅ Expected response time? (< 100ms for interactions)
✅ Mobile support needed? (Yes, responsive design)
```

---

### **A - Architecture / High-Level Design (15-20 mins)**

**Goal:** Draw component hierarchy and data flow

#### Key Architectural Decisions:

**1. Component Structure:**
```
┌─────────────────────────────────────┐
│         Application                 │
│  ┌─────────────────────────────┐   │
│  │    Search & Filter Bar      │   │
│  │  ┌────────┐  ┌──────────┐  │   │
│  │  │ Search │  │ Filters  │  │   │
│  │  └────────┘  └──────────┘  │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │      Data Grid/Table        │   │
│  │  ┌──────────────────────┐  │   │
│  │  │  Virtual Scroll      │  │   │
│  │  │  Container           │  │   │
│  │  │  ┌────────────────┐ │  │   │
│  │  │  │ Visible Rows   │ │  │   │
│  │  │  └────────────────┘ │  │   │
│  │  └──────────────────────┘  │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │      Pagination             │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

**2. State Management Architecture:**
```
┌──────────────────────────────────────┐
│     Application State                │
├──────────────────────────────────────┤
│  Server State (React Query/SWR)     │
│  - Tickets data                      │
│  - Filter options (cached)           │
│  - User data                         │
├──────────────────────────────────────┤
│  UI State (useState/useReducer)      │
│  - Current filters                   │
│  - Search query                      │
│  - Visible rows                      │
│  - Selected items                    │
├──────────────────────────────────────┤
│  URL State (React Router)            │
│  - Query parameters                  │
│  - Shareable filter links            │
└──────────────────────────────────────┘
```

**3. Rendering Strategy:**
- **CSR (Client-Side Rendering):** For dynamic dashboards
- **SSR (Server-Side Rendering):** For SEO-critical pages
- **SSG (Static Site Generation):** For marketing pages
- **ISR (Incremental Static Regeneration):** For frequently updated content

**4. Architectural Patterns:**

**Single-Page Application (SPA):**
```javascript
// Pros: Fast navigation, rich interactions
// Cons: Initial load time, SEO challenges
// Use when: Dashboard, admin panels, internal tools

import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/tickets" element={<TicketList />} />
      </Routes>
    </BrowserRouter>
  );
}
```

**Micro-Frontends:**
```javascript
// Pros: Team autonomy, independent deployment
// Cons: Complexity, bundle duplication
// Use when: Large teams, different tech stacks

// Container app
import { MountRemoteApp } from '@module-federation/runtime';

function App() {
  return (
    <div>
      <Header />
      <MountRemoteApp name="ticketApp" />
      <MountRemoteApp name="analyticsApp" />
    </div>
  );
}
```

---

### **D - Data Model (10-15 mins)**

**Goal:** Define data structures and relationships

#### 1. Entity Definitions

**Ticket Entity:**
```typescript
interface Ticket {
  id: string;
  title: string;
  description: string;
  status: 'open' | 'in-progress' | 'resolved' | 'closed';
  priority: 'low' | 'medium' | 'high' | 'critical';
  assignee: User | null;
  reporter: User;
  tags: string[];
  createdAt: Date;
  updatedAt: Date;
  dueDate: Date | null;
  comments: Comment[];
  attachments: Attachment[];
}

interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
  role: 'admin' | 'user' | 'viewer';
}

interface Comment {
  id: string;
  author: User;
  content: string;
  createdAt: Date;
}

interface FilterState {
  search: string;
  status: string[];
  priority: string[];
  assignee: string[];
  tags: string[];
  dateRange: {
    start: Date | null;
    end: Date | null;
  };
}
```

#### 2. Data Flow Diagram

```
┌──────────┐
│  Server  │
└────┬─────┘
     │
     │ HTTP/GraphQL
     ▼
┌──────────────┐
│  API Layer   │ ← Fetching, caching, error handling
└──────┬───────┘
       │
       │ Normalized data
       ▼
┌───────────────┐
│  Cache Layer  │ ← React Query / SWR
└──────┬────────┘
       │
       │ Transformed data
       ▼
┌──────────────────┐
│  Component State │ ← useState / useReducer
└──────┬───────────┘
       │
       │ Props
       ▼
┌──────────────┐
│  UI Layer    │
└──────────────┘
```

#### 3. State Management Strategy

**Server State (React Query):**
```javascript
// Fetching and caching server data
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function useTickets(filters) {
  return useQuery({
    queryKey: ['tickets', filters],
    queryFn: () => fetchTickets(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 10 * 60 * 1000, // 10 minutes
  });
}

function useUpdateTicket() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (ticket) => updateTicket(ticket),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tickets'] });
    },
  });
}
```

**UI State (Reducer Pattern):**
```javascript
// Complex filter state management
function filterReducer(state, action) {
  switch (action.type) {
    case 'SET_SEARCH':
      return { ...state, search: action.payload };

    case 'TOGGLE_STATUS':
      return {
        ...state,
        status: state.status.includes(action.payload)
          ? state.status.filter(s => s !== action.payload)
          : [...state.status, action.payload]
      };

    case 'RESET_FILTERS':
      return initialFilterState;

    default:
      return state;
  }
}

function TicketFilters() {
  const [filters, dispatch] = useReducer(filterReducer, initialFilterState);

  // Sync with URL
  useEffect(() => {
    const params = new URLSearchParams(filters);
    window.history.pushState({}, '', `?${params}`);
  }, [filters]);

  return (/* Filter UI */);
}
```

---

### **I - Interface Definition (API) (10-15 mins)**

**Goal:** Define APIs between frontend and backend, and between components

#### 1. Backend API Design

**RESTful API:**
```javascript
// GET /api/tickets
// Query parameters for filtering, pagination, sorting
GET /api/tickets?
  search=bug&
  status=open,in-progress&
  assignee=user123&
  page=1&
  limit=50&
  sort=-createdAt

Response:
{
  "data": [
    {
      "id": "ticket-1",
      "title": "Fix login bug",
      "status": "open",
      // ... other fields
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 1250,
    "pages": 25
  }
}

// POST /api/tickets (Create ticket)
// PUT /api/tickets/:id (Update ticket)
// DELETE /api/tickets/:id (Delete ticket)
```

**GraphQL API (Alternative):**
```graphql
query GetTickets(
  $filters: TicketFilters
  $page: Int
  $limit: Int
) {
  tickets(filters: $filters, page: $page, limit: $limit) {
    edges {
      node {
        id
        title
        status
        priority
        assignee {
          id
          name
          avatar
        }
      }
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      totalCount
    }
  }
}
```

#### 2. Component API Design

**Dropdown Component Interface:**
```typescript
interface DropdownProps<T> {
  // Data
  options: T[];
  value: T | null;

  // Behavior
  onChange: (value: T) => void;
  onSearch?: (query: string) => void;

  // Display
  getOptionLabel: (option: T) => string;
  getOptionValue: (option: T) => string;
  placeholder?: string;

  // Features
  searchable?: boolean;
  clearable?: boolean;
  disabled?: boolean;
  loading?: boolean;

  // Performance (for large lists)
  virtualized?: boolean;
  itemHeight?: number;
  maxHeight?: number;

  // Accessibility
  'aria-label'?: string;
  'aria-describedby'?: string;
}

// Usage
<Dropdown
  options={users}
  value={selectedUser}
  onChange={setSelectedUser}
  getOptionLabel={(user) => user.name}
  getOptionValue={(user) => user.id}
  searchable
  virtualized
  itemHeight={48}
  maxHeight={400}
  aria-label="Select assignee"
/>
```

**Virtual List Component Interface:**
```typescript
interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  containerHeight: number;
  renderItem: (item: T, index: number) => ReactNode;
  overscan?: number; // Number of items to render outside viewport
  onEndReached?: () => void; // Infinite scroll
  threshold?: number; // Distance from end to trigger onEndReached
}
```

#### 3. API Interaction Patterns

**Debounced Search:**
```javascript
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

function SearchInput({ onSearch }) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      onSearch(debouncedQuery);
    }
  }, [debouncedQuery, onSearch]);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

**Request Cancellation:**
```javascript
function useTicketsSearch(query) {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;

    const controller = new AbortController();
    setLoading(true);

    fetch(`/api/tickets?search=${query}`, {
      signal: controller.signal
    })
      .then(res => res.json())
      .then(data => setData(data))
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error('Search failed:', err);
        }
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [query]);

  return { data, loading };
}
```

**Infinite Scroll with Chunked Loading:**
```javascript
function useInfiniteTickets(filters) {
  const [page, setPage] = useState(1);
  const [allData, setAllData] = useState([]);
  const [hasMore, setHasMore] = useState(true);

  const { data, isLoading } = useQuery({
    queryKey: ['tickets', filters, page],
    queryFn: () => fetchTickets({ ...filters, page, limit: 50 }),
    enabled: hasMore,
  });

  useEffect(() => {
    if (data) {
      setAllData(prev => [...prev, ...data.items]);
      setHasMore(data.hasMore);
    }
  }, [data]);

  const loadMore = useCallback(() => {
    if (!isLoading && hasMore) {
      setPage(prev => prev + 1);
    }
  }, [isLoading, hasMore]);

  return { data: allData, loadMore, hasMore, isLoading };
}
```

---

### **O - Optimizations (20-25 mins)**

**Goal:** Discuss performance, scalability, and UX improvements

This is typically **40% of the interview time** and where senior engineers demonstrate depth.

#### 1. Performance Optimizations

**Virtual Scrolling for Large Lists:**
```javascript
import { FixedSizeList } from 'react-window';

function VirtualizedDropdown({ items, onSelect }) {
  const Row = ({ index, style }) => (
    <div style={style} onClick={() => onSelect(items[index])}>
      {items[index].label}
    </div>
  );

  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={48}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// Custom Implementation (Simpler)
function SimpleVirtualList({ items, itemHeight, containerHeight }) {
  const [scrollTop, setScrollTop] = useState(0);

  const startIndex = Math.floor(scrollTop / itemHeight);
  const endIndex = Math.ceil((scrollTop + containerHeight) / itemHeight);
  const visibleItems = items.slice(startIndex, endIndex);

  const offsetY = startIndex * itemHeight;
  const totalHeight = items.length * itemHeight;

  return (
    <div
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.target.scrollTop)}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div style={{ transform: `translateY(${offsetY}px)` }}>
          {visibleItems.map((item, index) => (
            <div key={startIndex + index} style={{ height: itemHeight }}>
              {item.label}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

**Code Splitting and Lazy Loading:**
```javascript
import { lazy, Suspense } from 'react';

// Route-based splitting
const TicketList = lazy(() => import('./pages/TicketList'));
const Analytics = lazy(() => import('./pages/Analytics'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/tickets" element={<TicketList />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}

// Component-based splitting
const HeavyChart = lazy(() => import('./components/HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

**Memoization and React Performance:**
```javascript
// Memoize expensive computations
function TicketList({ tickets, filters }) {
  const filteredTickets = useMemo(() => {
    return tickets.filter(ticket =>
      ticket.status.includes(filters.status) &&
      ticket.title.toLowerCase().includes(filters.search.toLowerCase())
    );
  }, [tickets, filters]);

  return filteredTickets.map(ticket => (
    <TicketRow key={ticket.id} ticket={ticket} />
  ));
}

// Memoize callbacks to prevent re-renders
function TicketRow({ ticket }) {
  const handleClick = useCallback(() => {
    console.log('Clicked:', ticket.id);
  }, [ticket.id]);

  return <MemoizedButton onClick={handleClick} />;
}

// Memoize components
const TicketRow = memo(({ ticket, onSelect }) => {
  return (
    <div onClick={() => onSelect(ticket.id)}>
      {ticket.title}
    </div>
  );
}, (prevProps, nextProps) => {
  // Custom comparison
  return prevProps.ticket.id === nextProps.ticket.id;
});
```

#### 2. Caching Strategies

**Multi-Level Caching:**
```javascript
// Level 1: In-memory cache (React Query)
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000, // 10 minutes
    },
  },
});

// Level 2: Session storage (for dropdown options)
class DropdownCache {
  constructor() {
    this.cache = new Map();
  }

  get(key) {
    // Check memory first
    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    // Check session storage
    const stored = sessionStorage.getItem(key);
    if (stored) {
      const parsed = JSON.parse(stored);
      this.cache.set(key, parsed);
      return parsed;
    }

    return null;
  }

  set(key, value) {
    this.cache.set(key, value);
    sessionStorage.setItem(key, JSON.stringify(value));
  }
}

// Level 3: Service Worker (for offline support)
// sw.js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});
```

**Cache Invalidation:**
```javascript
// Time-based invalidation
function useCachedData(key, fetcher, ttl = 300000) {
  const [data, setData] = useState(null);

  useEffect(() => {
    const cached = localStorage.getItem(key);

    if (cached) {
      const { value, timestamp } = JSON.parse(cached);
      if (Date.now() - timestamp < ttl) {
        setData(value);
        return;
      }
    }

    fetcher().then(newData => {
      setData(newData);
      localStorage.setItem(key, JSON.stringify({
        value: newData,
        timestamp: Date.now()
      }));
    });
  }, [key, fetcher, ttl]);

  return data;
}

// Event-based invalidation
const queryClient = useQueryClient();

function useUpdateTicket() {
  return useMutation({
    mutationFn: updateTicket,
    onSuccess: (data) => {
      // Invalidate specific query
      queryClient.invalidateQueries({ queryKey: ['tickets', data.id] });

      // Invalidate all ticket lists
      queryClient.invalidateQueries({ queryKey: ['tickets'], exact: false });

      // Update cache directly for instant UI update
      queryClient.setQueryData(['tickets', data.id], data);
    },
  });
}
```

#### 3. Network Optimizations

**Request Batching:**
```javascript
class RequestBatcher {
  constructor(delay = 50) {
    this.delay = delay;
    this.queue = [];
    this.timer = null;
  }

  add(id) {
    this.queue.push(id);

    if (this.timer) clearTimeout(this.timer);

    this.timer = setTimeout(() => {
      this.flush();
    }, this.delay);
  }

  async flush() {
    const ids = [...new Set(this.queue)]; // Remove duplicates
    this.queue = [];

    const response = await fetch('/api/tickets/batch', {
      method: 'POST',
      body: JSON.stringify({ ids }),
    });

    return response.json();
  }
}

// Usage
const batcher = new RequestBatcher();

function TicketItem({ id }) {
  useEffect(() => {
    batcher.add(id); // Requests get batched automatically
  }, [id]);
}
```

**Request Deduplication:**
```javascript
// React Query automatically deduplicates requests
function TicketDashboard() {
  // Both components fetch same data, but only one request is made
  return (
    <>
      <TicketList />
      <TicketStats />
    </>
  );
}

function TicketList() {
  const { data } = useQuery({ queryKey: ['tickets'], queryFn: fetchTickets });
  return <div>{/* List UI */}</div>;
}

function TicketStats() {
  const { data } = useQuery({ queryKey: ['tickets'], queryFn: fetchTickets });
  return <div>{/* Stats UI */}</div>;
}
```

**Prefetching:**
```javascript
function TicketList() {
  const queryClient = useQueryClient();

  return tickets.map(ticket => (
    <TicketRow
      key={ticket.id}
      ticket={ticket}
      onMouseEnter={() => {
        // Prefetch ticket details on hover
        queryClient.prefetchQuery({
          queryKey: ['ticket', ticket.id],
          queryFn: () => fetchTicketDetails(ticket.id),
        });
      }}
    />
  ));
}
```

#### 4. User Experience Optimizations

**Optimistic Updates:**
```javascript
function useUpdateTicketStatus() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, status }) => updateTicketStatus(id, status),

    onMutate: async ({ id, status }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['tickets'] });

      // Snapshot previous value
      const previousTickets = queryClient.getQueryData(['tickets']);

      // Optimistically update
      queryClient.setQueryData(['tickets'], (old) =>
        old.map(ticket =>
          ticket.id === id ? { ...ticket, status } : ticket
        )
      );

      return { previousTickets };
    },

    onError: (err, variables, context) => {
      // Rollback on error
      queryClient.setQueryData(['tickets'], context.previousTickets);
    },

    onSettled: () => {
      // Refetch to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['tickets'] });
    },
  });
}
```

**Loading States and Skeletons:**
```javascript
function TicketList() {
  const { data, isLoading, isError } = useQuery({
    queryKey: ['tickets'],
    queryFn: fetchTickets,
  });

  if (isLoading) {
    return <TicketListSkeleton />;
  }

  if (isError) {
    return <ErrorMessage retry={() => refetch()} />;
  }

  return data.map(ticket => <TicketRow key={ticket.id} ticket={ticket} />);
}

function TicketListSkeleton() {
  return (
    <div>
      {Array.from({ length: 10 }).map((_, i) => (
        <div key={i} className="skeleton-row">
          <div className="skeleton-avatar" />
          <div className="skeleton-text" />
        </div>
      ))}
    </div>
  );
}
```

#### 5. Accessibility Optimizations

**Keyboard Navigation:**
```javascript
function AccessibleDropdown({ items, value, onChange }) {
  const [isOpen, setIsOpen] = useState(false);
  const [focusedIndex, setFocusedIndex] = useState(-1);
  const listRef = useRef(null);

  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setFocusedIndex(prev =>
          Math.min(prev + 1, items.length - 1)
        );
        break;

      case 'ArrowUp':
        e.preventDefault();
        setFocusedIndex(prev => Math.max(prev - 1, 0));
        break;

      case 'Enter':
        if (focusedIndex >= 0) {
          onChange(items[focusedIndex]);
          setIsOpen(false);
        }
        break;

      case 'Escape':
        setIsOpen(false);
        break;
    }
  };

  // Scroll focused item into view
  useEffect(() => {
    if (focusedIndex >= 0 && listRef.current) {
      const item = listRef.current.children[focusedIndex];
      item?.scrollIntoView({ block: 'nearest' });
    }
  }, [focusedIndex]);

  return (
    <div role="combobox" aria-expanded={isOpen}>
      <button
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleKeyDown}
        aria-haspopup="listbox"
      >
        {value || 'Select...'}
      </button>
      {isOpen && (
        <ul ref={listRef} role="listbox">
          {items.map((item, index) => (
            <li
              key={item.id}
              role="option"
              aria-selected={focusedIndex === index}
            >
              {item.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Screen Reader Announcements:**
```javascript
function LiveRegion({ message }) {
  return (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only"
    >
      {message}
    </div>
  );
}

function TicketList() {
  const { data, isLoading } = useQuery(['tickets'], fetchTickets);
  const [announcement, setAnnouncement] = useState('');

  useEffect(() => {
    if (isLoading) {
      setAnnouncement('Loading tickets...');
    } else if (data) {
      setAnnouncement(`${data.length} tickets loaded`);
    }
  }, [isLoading, data]);

  return (
    <>
      <LiveRegion message={announcement} />
      {/* Ticket list UI */}
    </>
  );
}
```

#### 6. Monitoring and Observability

**Performance Monitoring:**
```javascript
// Web Vitals tracking
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics({ name, delta, id }) {
  fetch('/analytics', {
    method: 'POST',
    body: JSON.stringify({ name, delta, id }),
  });
}

getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);

// Custom performance marks
performance.mark('filter-start');
// ... apply filters
performance.mark('filter-end');
performance.measure('filter-duration', 'filter-start', 'filter-end');

const measure = performance.getEntriesByName('filter-duration')[0];
console.log(`Filter took ${measure.duration}ms`);
```

**Error Boundary with Logging:**
```javascript
class ErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // Log to error reporting service
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <TicketList />
</ErrorBoundary>
```

---

## 🎯 Palo Alto Specific: Jira Filter System

### Problem Statement (EXACT Interview Question)

> **"Design the search and filter section of a ticket management system like Jira or ClickUp. The system receives data in CHUNKS, and dropdowns may contain HUNDREDS OF THOUSANDS of items."**

This was asked in recent 2024-2025 interviews for Senior Staff UI Engineer role.

---

### Step 1: Requirements (RADIO - R)

**Functional Requirements:**
- Search tickets by text (title, description)
- Filter by: status, priority, assignee, tags, date range
- Display results in paginated grid
- Filters should be combinable (AND logic)
- Clear all filters
- Save filter presets (optional)

**Non-Functional Requirements:**
- **Scale:** 100K tickets, 50K users, 10K tags
- **Performance:** < 100ms for filter changes, < 500ms for search
- **Dropdowns:** Support 100K items efficiently
- **Concurrent Users:** 1000+
- **Browser Support:** Modern browsers (Chrome, Firefox, Safari, Edge)
- **Accessibility:** WCAG 2.1 AA compliance
- **Mobile:** Responsive design

---

### Step 2: Architecture (RADIO - A)

```
┌─────────────────────────────────────────────────────┐
│                  Application                        │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │         Search & Filter Bar                   │ │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────┐  │ │
│  │  │  Search  │  │ Status   │  │  Priority │  │ │
│  │  │  Input   │  │ Dropdown │  │  Dropdown │  │ │
│  │  └──────────┘  └──────────┘  └───────────┘  │ │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────┐  │ │
│  │  │ Assignee │  │   Tags   │  │ Date Range│  │ │
│  │  │ Dropdown │  │Multi-Sel │  │   Picker  │  │ │
│  │  └──────────┘  └──────────┘  └───────────┘  │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │             Active Filters Chips              │ │
│  │  [Status: Open X]  [Priority: High X]        │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │           Ticket Grid/Table                   │ │
│  │  ┌─────────────────────────────────────────┐ │ │
│  │  │  Virtual Scroll Container               │ │ │
│  │  │  ┌───────────────────────────────────┐ │ │ │
│  │  │  │ Visible Rows (50 out of 100K)    │ │ │ │
│  │  │  │  • Ticket Row 1                   │ │ │ │
│  │  │  │  • Ticket Row 2                   │ │ │ │
│  │  │  │  • ...                            │ │ │ │
│  │  │  └───────────────────────────────────┘ │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │         Pagination / Load More                │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Component Hierarchy:**
```
App
├── FilterBar
│   ├── SearchInput (debounced)
│   ├── StatusDropdown (virtualized)
│   ├── PriorityDropdown (virtualized)
│   ├── AssigneeDropdown (virtualized, searchable, server-side)
│   ├── TagsMultiSelect (virtualized, searchable, server-side)
│   └── DateRangePicker
├── ActiveFiltersChips
└── TicketGrid
    ├── VirtualScrollContainer
    └── TicketRow (memoized)
```

---

### Step 3: Data Model (RADIO - D)

```typescript
// Filter State
interface FilterState {
  search: string;
  status: string[];
  priority: string[];
  assignee: string[];
  tags: string[];
  dateRange: {
    start: Date | null;
    end: Date | null;
  };
  page: number;
  limit: number;
}

// Ticket Entity
interface Ticket {
  id: string;
  title: string;
  description: string;
  status: TicketStatus;
  priority: TicketPriority;
  assignee: User | null;
  tags: string[];
  createdAt: Date;
  updatedAt: Date;
}

// Dropdown Options
interface DropdownOption {
  value: string;
  label: string;
  count?: number; // For showing "Open (45)"
}

// API Response for Chunked Data
interface TicketChunkResponse {
  data: Ticket[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    hasMore: boolean;
  };
  filters: {
    // Available filter options with counts
    statuses: DropdownOption[];
    priorities: DropdownOption[];
    // For large datasets, only return top N or search results
    topAssignees: DropdownOption[];
    topTags: DropdownOption[];
  };
}
```

**State Management:**
```javascript
// Using useReducer for complex filter state
const initialFilterState = {
  search: '',
  status: [],
  priority: [],
  assignee: [],
  tags: [],
  dateRange: { start: null, end: null },
  page: 1,
  limit: 50
};

function filterReducer(state, action) {
  switch (action.type) {
    case 'SET_SEARCH':
      return { ...state, search: action.payload, page: 1 };

    case 'TOGGLE_STATUS':
      const newStatuses = state.status.includes(action.payload)
        ? state.status.filter(s => s !== action.payload)
        : [...state.status, action.payload];
      return { ...state, status: newStatuses, page: 1 };

    case 'SET_ASSIGNEE':
      return { ...state, assignee: action.payload, page: 1 };

    case 'SET_DATE_RANGE':
      return { ...state, dateRange: action.payload, page: 1 };

    case 'RESET_FILTERS':
      return initialFilterState;

    case 'LOAD_MORE':
      return { ...state, page: state.page + 1 };

    default:
      return state;
  }
}
```

---

### Step 4: Interface (API) (RADIO - I)

**Backend API:**
```javascript
// GET /api/tickets
// Server-side filtering, pagination, search
GET /api/tickets?
  search=bug&
  status[]=open&
  status[]=in-progress&
  priority[]=high&
  assignee[]=user-123&
  tags[]=frontend&
  dateStart=2024-01-01&
  dateEnd=2024-12-31&
  page=1&
  limit=50&
  sort=-createdAt

Response:
{
  "data": [...tickets],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 1250,
    "hasMore": true
  }
}

// GET /api/filters/assignees?search=john
// For dropdown with search (server-side filtering)
GET /api/filters/assignees?search=john&limit=50

Response:
{
  "data": [
    { "value": "user-1", "label": "John Doe", "count": 45 },
    { "value": "user-2", "label": "John Smith", "count": 23 }
  ],
  "total": 245
}

// Similar endpoint for tags
GET /api/filters/tags?search=front&limit=50
```

**Component API:**
```typescript
// Virtualized Dropdown Component
<VirtualizedDropdown
  options={assignees}
  value={selectedAssignee}
  onChange={handleAssigneeChange}
  onSearch={handleAssigneeSearch}
  loading={isLoadingAssignees}
  placeholder="Select assignee..."
  searchable
  virtualized
  itemHeight={48}
  maxHeight={400}
/>

// Filter Bar Component
<FilterBar
  filters={filters}
  onFilterChange={handleFilterChange}
  onReset={handleResetFilters}
/>

// Ticket Grid Component
<TicketGrid
  tickets={tickets}
  loading={isLoading}
  onLoadMore={loadMore}
  hasMore={hasMore}
/>
```

---

### Step 5: Optimizations (RADIO - O)

#### 1. Virtual Scrolling for Dropdown (100K Items)

```javascript
import { FixedSizeList } from 'react-window';

function VirtualizedDropdown({ options, value, onChange, onSearch }) {
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');

  // Debounced search - critical for large datasets
  const debouncedSearch = useDebounce(searchQuery, 300);

  useEffect(() => {
    if (debouncedSearch) {
      onSearch(debouncedSearch);
    }
  }, [debouncedSearch, onSearch]);

  const Row = ({ index, style }) => {
    const option = options[index];
    const isSelected = value === option.value;

    return (
      <div
        style={style}
        className={`dropdown-item ${isSelected ? 'selected' : ''}`}
        onClick={() => {
          onChange(option.value);
          setIsOpen(false);
        }}
        role="option"
        aria-selected={isSelected}
      >
        <span>{option.label}</span>
        {option.count && <span className="count">({option.count})</span>}
      </div>
    );
  };

  return (
    <div className="virtualized-dropdown">
      <button
        onClick={() => setIsOpen(!isOpen)}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
      >
        {value || 'Select...'}
      </button>

      {isOpen && (
        <div className="dropdown-menu" role="listbox">
          <input
            type="text"
            placeholder="Search..."
            value={searchQuery}
            onChange={(e) => setSearchQuery(e.target.value)}
            autoFocus
          />

          <FixedSizeList
            height={400}
            itemCount={options.length}
            itemSize={48}
            width="100%"
          >
            {Row}
          </FixedSizeList>
        </div>
      )}
    </div>
  );
}
```

#### 2. Server-Side Filtering (Critical for 100K Items)

```javascript
// For 100K items, client-side filtering is NOT feasible
// Use server-side filtering with search

function useAssigneeSearch(query) {
  return useQuery({
    queryKey: ['assignees', query],
    queryFn: () =>
      fetch(`/api/filters/assignees?search=${query}&limit=50`)
        .then(r => r.json()),
    enabled: query.length > 0,
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
  });
}

function AssigneeDropdown({ value, onChange }) {
  const [searchQuery, setSearchQuery] = useState('');
  const debouncedQuery = useDebounce(searchQuery, 300);

  const { data, isLoading } = useAssigneeSearch(debouncedQuery);

  return (
    <VirtualizedDropdown
      options={data?.data || []}
      value={value}
      onChange={onChange}
      loading={isLoading}
      onSearch={setSearchQuery}
      placeholder="Search assignees..."
    />
  );
}
```

#### 3. Debouncing and Request Cancellation

```javascript
function useTicketsWithFilters(filters) {
  // Debounce filter changes to avoid excessive API calls
  const debouncedFilters = useDebounce(filters, 300);

  return useQuery({
    queryKey: ['tickets', debouncedFilters],
    queryFn: ({ signal }) =>
      fetch('/api/tickets?' + new URLSearchParams(debouncedFilters), {
        signal // AbortController for cancellation
      }).then(r => r.json()),
  });
}

// Custom debounce hook
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

#### 4. URL State Synchronization (Shareable Links)

```javascript
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

    if (newFilters.search) params.set('search', newFilters.search);
    newFilters.status.forEach(s => params.append('status', s));
    newFilters.priority.forEach(p => params.append('priority', p));
    newFilters.assignee.forEach(a => params.append('assignee', a));

    setSearchParams(params);
  }, [setSearchParams]);

  return [filters, updateFilters];
}

// Usage - filters are now shareable via URL!
function FilterBar() {
  const [filters, setFilters] = useFilterState();
  // User can share URL with filters applied
}
```

#### 5. Caching Filter Options

```javascript
// Cache dropdown options to avoid repeated fetching
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 10 * 60 * 1000, // 10 minutes for filter options
      gcTime: 30 * 60 * 1000, // 30 minutes garbage collection
    },
  },
});

// Prefetch common filters on mount
function useFilterPrefetch() {
  const queryClient = useQueryClient();

  useEffect(() => {
    // Prefetch status and priority (small, static lists)
    queryClient.prefetchQuery({
      queryKey: ['filters', 'status'],
      queryFn: fetchStatusOptions,
    });
    queryClient.prefetchQuery({
      queryKey: ['filters', 'priority'],
      queryFn: fetchPriorityOptions,
    });
  }, [queryClient]);
}
```

#### 6. Chunked Data Loading (Infinite Scroll)

```javascript
function useInfiniteTickets(filters) {
  return useInfiniteQuery({
    queryKey: ['tickets', filters],
    queryFn: ({ pageParam = 1 }) =>
      fetchTickets({ ...filters, page: pageParam, limit: 50 }),
    getNextPageParam: (lastPage) =>
      lastPage.pagination.hasMore
        ? lastPage.pagination.page + 1
        : undefined,
    staleTime: 2 * 60 * 1000, // 2 minutes
  });
}

function TicketGrid() {
  const [filters] = useFilterState();
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isLoading,
    isFetchingNextPage
  } = useInfiniteTickets(filters);

  // Flatten all pages into single array
  const allTickets = data?.pages.flatMap(page => page.data) || [];

  return (
    <InfiniteScroll
      dataLength={allTickets.length}
      next={fetchNextPage}
      hasMore={hasNextPage}
      loader={<LoadingSpinner />}
    >
      {allTickets.map(ticket => (
        <TicketRow key={ticket.id} ticket={ticket} />
      ))}
    </InfiniteScroll>
  );
}
```

---

### Complete Implementation Example

```javascript
// FilterBar.jsx - Main component integrating everything
function FilterBar() {
  const [filters, setFilters] = useReducer(filterReducer, initialFilterState);
  const debouncedFilters = useDebounce(filters, 300);

  // Fetch tickets with filters
  const { data, isLoading, isError, refetch } = useQuery({
    queryKey: ['tickets', debouncedFilters],
    queryFn: ({ signal }) => fetchTickets(debouncedFilters, signal),
  });

  // Sync with URL for shareable links
  useEffect(() => {
    const params = new URLSearchParams();
    if (filters.search) params.set('search', filters.search);
    filters.status.forEach(s => params.append('status', s));
    filters.priority.forEach(p => params.append('priority', p));
    filters.assignee.forEach(a => params.append('assignee', a));

    window.history.pushState({}, '', `?${params}`);
  }, [filters]);

  return (
    <div className="filter-bar">
      {/* Search Input with debouncing */}
      <SearchInput
        value={filters.search}
        onChange={(value) => setFilters({ type: 'SET_SEARCH', payload: value })}
        placeholder="Search tickets..."
      />

      {/* Status Dropdown - small, static list */}
      <MultiSelectDropdown
        options={statusOptions}
        value={filters.status}
        onChange={(value) => setFilters({ type: 'SET_STATUS', payload: value })}
        label="Status"
      />

      {/* Priority Dropdown - small, static list */}
      <MultiSelectDropdown
        options={priorityOptions}
        value={filters.priority}
        onChange={(value) => setFilters({ type: 'SET_PRIORITY', payload: value })}
        label="Priority"
      />

      {/* Assignee Dropdown - LARGE, virtualized with server-side search */}
      <VirtualizedSearchDropdown
        fetchOptions={searchAssignees}
        value={filters.assignee}
        onChange={(value) => setFilters({ type: 'SET_ASSIGNEE', payload: value })}
        label="Assignee"
        placeholder="Search assignees..."
      />

      {/* Tags Multi-Select - LARGE, virtualized with server-side search */}
      <VirtualizedSearchDropdown
        fetchOptions={searchTags}
        value={filters.tags}
        onChange={(value) => setFilters({ type: 'SET_TAGS', payload: value })}
        label="Tags"
        placeholder="Search tags..."
        multiple
      />

      {/* Date Range Picker */}
      <DateRangePicker
        value={filters.dateRange}
        onChange={(value) => setFilters({ type: 'SET_DATE_RANGE', payload: value })}
      />

      {/* Reset Button */}
      <button onClick={() => setFilters({ type: 'RESET_FILTERS' })}>
        Clear All Filters
      </button>

      {/* Active Filters Display */}
      <ActiveFiltersChips filters={filters} onRemove={setFilters} />

      {/* Results */}
      <div className="results">
        {isLoading && <LoadingSpinner />}
        {isError && <ErrorMessage onRetry={refetch} />}
        {data && (
          <>
            <div className="results-count">
              {data.pagination.total} tickets found
            </div>
            <TicketGrid tickets={data.data} />
          </>
        )}
      </div>
    </div>
  );
}
```

---

## 🎯 Common System Design Questions

### 1. **Design a News Feed (Facebook, Twitter)**

**Requirements:**
- Display posts from followed users
- Real-time updates
- Infinite scroll
- Like, comment, share functionality
- Optimistic updates

**Key Discussion Points:**
- Pagination vs infinite scroll
- WebSocket vs polling for real-time updates
- Optimistic UI updates
- Image lazy loading
- Virtual scrolling for performance
- Content moderation and filtering

---

### 2. **Design an Autocomplete / Typeahead**

**Requirements:**
- Search suggestions as user types
- Handle large datasets
- Keyboard navigation
- Highlight matching text
- Recent searches

**Key Discussion Points:**
- Debouncing vs throttling
- Client-side vs server-side filtering
- Trie data structure for prefix matching
- Caching strategies
- Request cancellation
- Accessibility (ARIA live regions)

---

### 3. **Design Image Gallery / Carousel**

**Requirements:**
- Display images in grid/carousel
- Lazy loading
- Lightbox view
- Touch gestures
- Responsive design

**Key Discussion Points:**
- Intersection Observer for lazy loading
- Image optimization (WebP, srcset)
- Touch event handling
- Keyboard navigation
- Virtual scrolling for large galleries
- Prefetching adjacent images

---

### 4. **Design a Chat Application**

**Requirements:**
- Real-time messaging
- Message history
- Online status
- Typing indicators
- Read receipts

**Key Discussion Points:**
- WebSocket vs Server-Sent Events
- Message pagination and virtualization
- Optimistic message sending
- Connection state management
- Local message storage (IndexedDB)
- Retry logic for failed messages

---

### 5. **Design a Video Streaming Platform**

**Requirements:**
- Video player with controls
- Adaptive streaming (quality adjustment)
- Subtitles/captions
- Playlist functionality
- Recommendations

**Key Discussion Points:**
- HLS/DASH protocols
- Buffering strategies
- Quality switching algorithms
- Thumbnail previews
- Analytics and tracking
- CDN integration

---

## 📊 System Design Evaluation Criteria

Interviewers evaluate based on:

### 1. **Requirements Gathering (20%)**
- ✅ Asks clarifying questions
- ✅ Identifies functional vs non-functional requirements
- ✅ Understands scale and constraints
- ✅ Prioritizes requirements (MVP vs nice-to-have)

### 2. **Architecture & Design (25%)**
- ✅ Clear component hierarchy
- ✅ Proper separation of concerns
- ✅ Scalable architecture
- ✅ Considers edge cases
- ✅ Justifies design decisions

### 3. **Data Modeling (15%)**
- ✅ Well-defined data structures
- ✅ Appropriate data flow
- ✅ State management strategy
- ✅ Handles data transformations

### 4. **API Design (15%)**
- ✅ Clean interface definitions
- ✅ RESTful principles or GraphQL best practices
- ✅ Error handling
- ✅ Pagination and filtering

### 5. **Optimizations (25%)**
- ✅ Performance considerations
- ✅ Caching strategies
- ✅ Network optimizations
- ✅ Accessibility
- ✅ Monitoring and debugging

### 6. **Communication**
- ✅ Explains thought process clearly
- ✅ Discusses trade-offs
- ✅ Responds well to feedback
- ✅ Manages time effectively

---

## 🚀 Staff-Level Expectations

### What Makes Staff-Level Different?

**Junior/Mid-Level Focus:**
- Implement given designs
- Solve well-defined problems
- Focus on code quality
- Limited scope

**Staff-Level Focus:**
- Design systems from scratch
- Handle ambiguous requirements
- Consider trade-offs at scale
- Mentor and influence team
- Bridge business and technical needs

### Staff-Level System Design Requirements:

**1. High-Level Design (HLD) Mastery**
- Design systems serving millions of users
- Understand distributed systems
- Handle failure scenarios
- Plan for scalability

**2. Low-Level Implementation**
- Bridge architecture with implementation
- Understand performance implications
- Debug complex issues
- Optimize critical paths

**3. Trade-Off Analysis**
```
Example Trade-Off Discussion:

Q: Should we use client-side or server-side filtering for 100K items?

Staff-Level Answer:
"It depends on several factors:

Client-Side Filtering:
✅ Pros: Instant filtering, no network latency, works offline
❌ Cons: Initial load time, memory constraints, stale data

Server-Side Filtering:
✅ Pros: Handles millions of items, fresh data, lower memory
❌ Cons: Network latency, server load, complexity

For 100K items, I'd recommend:
- Server-side filtering with debounced search (300ms)
- Client-side caching with 5-minute TTL
- Virtual scrolling to render only visible items
- Prefetch top 100 options for instant initial display

This balances performance, user experience, and scalability."
```

**4. Scalability Considerations**
- How does solution scale to 10x, 100x users?
- Database query optimization
- CDN and caching strategies
- Load balancing
- Monitoring and alerting

**5. Cross-Functional Impact**
- How does design affect other teams?
- API versioning and backward compatibility
- Migration strategy for breaking changes
- Documentation and knowledge sharing

---

## 🎯 Practice Problems

### Problem 1: Design a Rate Limiter
**Focus:** Distributed systems, algorithms, caching

**Requirements:**
- Limit API requests per user
- Different tiers (free, premium)
- Handle distributed servers
- Low latency

**Key Concepts:**
- Token bucket algorithm
- Sliding window
- Redis for distributed state
- Rate limit headers

---

### Problem 2: Design a Code Editor (Monaco, CodeMirror)
**Focus:** Performance, syntax highlighting, collaboration

**Requirements:**
- Syntax highlighting
- Auto-completion
- Multiple language support
- Real-time collaboration
- Large file handling

**Key Concepts:**
- Virtual scrolling for large files
- Web Workers for syntax parsing
- CRDT for conflict resolution
- Debounced auto-save

---

### Problem 3: Design a Dashboard with Real-Time Metrics
**Focus:** WebSocket, data visualization, performance

**Requirements:**
- Real-time metric updates
- Multiple chart types
- Historical data
- Custom date ranges
- Export functionality

**Key Concepts:**
- WebSocket vs Server-Sent Events
- Canvas vs SVG for charts
- Data aggregation
- Time-series optimization

---

### Problem 4: Design a File Upload System
**Focus:** Progress tracking, chunking, error handling

**Requirements:**
- Multiple file uploads
- Progress indicators
- Resume capability
- File validation
- Size limits

**Key Concepts:**
- Chunked upload
- FormData API
- Upload queue
- Retry logic
- Presigned URLs (S3)

---

### Problem 5: Design a Notification System
**Focus:** Real-time updates, state management

**Requirements:**
- Toast notifications
- Notification center
- Unread count
- Mark as read
- Categories

**Key Concepts:**
- WebSocket for real-time
- Notification queue
- Auto-dismiss logic
- Persistence (IndexedDB)
- Push notifications

---

## 📚 Top Resources for Staff-Level Prep

### **Courses (Highly Recommended)**

1. **ByteByteGo** by Alex Xu
   - Author of "System Design Interview" (Amazon bestseller)
   - Visual diagrams and explanations
   - Focus on scalability

2. **Grokking Modern System Design Interview**
   - By ex-FAANG engineers
   - Structured approach with RADIO-like framework
   - Modern architectures (microservices, serverless)

3. **System Design in a Hurry**
   - By ex-Meta and Amazon engineers
   - Quick preparation guide
   - Core concepts and patterns

### **Books**

1. **"System Design Interview" Vol 1 & 2** - Alex Xu
   - Comprehensive system design guide
   - Visual approach
   - Real interview questions

2. **"Designing Data-Intensive Applications"** - Martin Kleppmann
   - Deep dive into distributed systems
   - Database internals
   - Scalability patterns

### **Free Resources**

1. **Tech Interview Handbook**
   - By Yangshun (ex-Meta Staff Engineer)
   - System design guide
   - Frontend-specific content

2. **GreatFrontEnd**
   - RADIO framework guide
   - Frontend system design questions
   - React interview prep

3. **Frontend Interview Handbook**
   - User interface components
   - Machine coding rounds
   - Best practices

### **Practice Platforms**

1. **LeetCode System Design**
   - Discussion forums with real experiences
   - Company-specific questions

2. **Exponent**
   - Mock interviews
   - System design questions by company

3. **HelloInterview**
   - Peer practice
   - Feedback from experienced engineers

---

## 🎯 Interview Day Tips

### Time Management (60-minute interview)

```
0-10 min:  Requirements gathering (R)
10-25 min: Architecture & Data Model (A, D)
25-40 min: Interface & APIs (I)
40-60 min: Optimizations & Discussion (O) ← 40% of time!
```

### Do's ✅

1. **Start with clarifying questions** - Don't jump into design
2. **Think aloud** - Explain your reasoning process
3. **Draw diagrams** - Visual aids improve communication
4. **Discuss trade-offs** - No single right answer exists
5. **Be opinionated** - Have strong technical opinions with reasoning
6. **Scale matters** - Always discuss how solution scales
7. **User experience** - Consider loading, errors, accessibility
8. **Ask for feedback** - "Does this approach make sense?"

### Don'ts ❌

1. ❌ Don't start coding immediately
2. ❌ Don't ignore edge cases
3. ❌ Don't over-engineer the solution
4. ❌ Don't stay silent - communicate constantly
5. ❌ Don't argue with interviewer feedback
6. ❌ Don't skip the Optimization phase (40% of time!)
7. ❌ Don't forget about accessibility and error handling

### Staff-Level Differentiators

**What Separates Staff from Senior:**

1. **Ambiguity Handling**
   - Staff: Asks the right questions to clarify requirements
   - Senior: Waits for clear requirements

2. **Scope of Thinking**
   - Staff: Considers team impact, migration strategy, future extensibility
   - Senior: Focuses on immediate problem

3. **Trade-Off Analysis**
   - Staff: Discusses 3-4 alternatives with pros/cons
   - Senior: Presents one solution

4. **Production Thinking**
   - Staff: Monitoring, alerting, rollback strategy, A/B testing
   - Senior: Working solution

5. **Communication**
   - Staff: Teaches concepts, influences decisions
   - Senior: Explains solution

---

## 📝 Sample Interview Dialogue (Staff-Level)

**Interviewer:** "Design a dropdown that needs to handle 100,000 items."

**Staff Engineer Response:**

"Great question. Let me start by clarifying some requirements:

1. **Data characteristics**: Are these items static or dynamic? Do they change frequently?
2. **User behavior**: Do users typically scroll through all items, or do they know what they're looking for?
3. **Search**: Should we provide search functionality?
4. **Selection**: Single or multi-select?
5. **Performance expectations**: What's acceptable response time?
6. **Accessibility**: Do we need keyboard navigation and screen reader support?

[Interviewer provides answers]

Based on your answers, here's my approach:

**High-Level Strategy:**
For 100K items, rendering all at once is not feasible due to:
- DOM performance (browser struggles with 100K nodes)
- Memory constraints
- Initial render time

I'd use a combination of:
1. Virtual scrolling - render only visible items
2. Server-side filtering - search backend, not client
3. Smart defaults - show top 50 most-used items initially

**Architecture:**
[Draws component diagram]

**Trade-offs:**
- Client-side: Fast but memory-intensive
- Server-side: Scalable but has latency
- Hybrid (my recommendation): Best of both worlds

**Implementation details:**
[Discusses react-window, debouncing, caching]

**Optimizations:**
- Debounce search (300ms)
- Cache results (5-minute TTL)
- Prefetch on dropdown open
- AbortController for request cancellation

**Monitoring:**
- Track search-to-select ratio
- Measure dropdown open-to-select time
- Alert on slow search queries (>500ms)

What would you like me to dive deeper into?"

---

## ✅ Final Checklist

### Before Interview:
- [ ] Review RADIO framework
- [ ] Practice 5+ system design problems
- [ ] Understand Palo Alto specific question (Jira filters)
- [ ] Review virtual scrolling implementation
- [ ] Study React Query / SWR patterns
- [ ] Prepare behavioral stories (STAR method)
- [ ] Review networking basics (OSI model, TCP/IP)

### During Interview:
- [ ] Clarify requirements (10 minutes)
- [ ] Draw architecture diagram
- [ ] Define data models
- [ ] Design APIs
- [ ] Discuss optimizations (40% of time!)
- [ ] Consider accessibility
- [ ] Mention monitoring/observability
- [ ] Discuss trade-offs throughout

### Staff-Level Bonus:
- [ ] Discuss migration strategy
- [ ] Consider cross-team impact
- [ ] Mention A/B testing approach
- [ ] Talk about rollback plan
- [ ] Discuss documentation needs
- [ ] Consider future extensibility

---

Good luck with your Palo Alto Networks Staff UI Engineer System Design interview! 🚀

**Remember:** At Staff level, they're evaluating not just your technical skills, but your ability to:
- Handle ambiguity
- Think at scale
- Consider trade-offs
- Influence technical direction
- Bridge business and technical needs

Show them you can design systems that serve millions of users while maintaining code quality, team velocity, and user experience!
