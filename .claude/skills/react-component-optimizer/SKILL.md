---
name: react-component-optimizer
description: Analyze React component structure and suggest optimizations for performance, state management, code reuse, and bundle size. Use when reviewing or refactoring React components for efficiency.
---

# React Component Optimizer

A systematic approach to analyzing React (functional components with hooks) and identifying optimization opportunities across four dimensions: rendering performance, state management correctness, code reuse potential, and bundle size.

## How to Use This Skill

When analyzing a React component:

1. **Read the component thoroughly** - Understand its structure and intent
2. **Check each dimension below** - Walk through the analysis checklist
3. **Identify patterns** - Match observed code against the anti-patterns
4. **Propose fixes** - Use the refactoring examples provided
5. **Verify impact** - Ensure changes maintain functionality while improving efficiency

---

## 1. Rendering Performance

Goal: Minimize unnecessary re-renders, optimize effect dependencies, and prevent expensive operations in render.

### ✅ What to Look For

#### Unnecessary Re-renders (Anti-patterns)

**Problem: Function created in every render**
```jsx
// ❌ Bad: onClick handler recreated on every render
function OrderCard({ orderId }) {
  const handleClick = () => {
    console.log(`Clicked order ${orderId}`)
  }
  
  return <button onClick={handleClick}>View Order</button>
}

// ✅ Good: Memoized callback
function OrderCard({ orderId }) {
  const handleClick = useCallback(() => {
    console.log(`Clicked order ${orderId}`)
  }, [orderId])
  
  return <button onClick={handleClick}>View Order</button>
}
```

**Why**: New function on each render breaks referential equality. Child components re-render. useCallback memoizes the function.

**When to use useCallback**:
- Passing functions to memoized children (`React.memo`)
- Dependencies to other hooks (useEffect, useMemo)
- Event handlers that are compared for equality
- NOT every function (adds overhead, less readable)

---

**Problem: Expensive computations on every render**
```jsx
// ❌ Bad: Recalculates sorted array on every render
function OrderList({ orders }) {
  const sorted = orders.sort((a, b) => a.date - b.date)
  
  return sorted.map(order => <OrderItem key={order.id} order={order} />)
}

// ✅ Good: Memoized computation
function OrderList({ orders }) {
  const sorted = useMemo(() => {
    return [...orders].sort((a, b) => a.date - b.date)
  }, [orders])
  
  return sorted.map(order => <OrderItem key={order.id} order={order} />)
}
```

**Why**: Expensive operations (sort, filter, map chains) run on every render. `useMemo` runs only when dependencies change.

**Red flags for useMemo**:
- `.filter()`, `.map()`, `.sort()` operations
- Complex calculations (loops, recursion)
- Creating new objects/arrays
- NOT simple variable assignments

---

**Problem: Inline object/array literals as props**
```jsx
// ❌ Bad: New object on every render, breaks memoization
function OrderFilter() {
  return <FilterComponent options={{ warehouse: 'SF', category: 'Parts' }} />
}

// ✅ Good: Memoized or moved outside
const defaultOptions = { warehouse: 'SF', category: 'Parts' }

function OrderFilter() {
  return <FilterComponent options={defaultOptions} />
}

// OR use useMemo
function OrderFilter() {
  const options = useMemo(() => ({ warehouse: 'SF', category: 'Parts' }), [])
  return <FilterComponent options={options} />
}
```

**Why**: New object on each render breaks `React.memo`. Props comparison fails even if values are the same.

---

**Problem: useEffect without dependency array**
```jsx
// ❌ Bad: Effect runs on EVERY render
function OrderForm() {
  useEffect(() => {
    console.log('Effect ran')
  })
  
  return <form>...</form>
}

// ✅ Good: Effect runs once
function OrderForm() {
  useEffect(() => {
    console.log('Effect ran')
  }, [])
  
  return <form>...</form>
}

// ✅ Good: Effect runs when dependencies change
function OrderForm({ orderId }) {
  useEffect(() => {
    fetchOrder(orderId)
  }, [orderId])
  
  return <form>...</form>
}
```

**Why**: Missing dependency array = effect runs after every render (performance killer). Empty array = run once on mount.

---

**Problem: Missing dependencies in useEffect**
```jsx
// ❌ Bad: orderId used but not in dependency array
function OrderDetail({ orderId }) {
  useEffect(() => {
    fetchOrder(orderId)  // ❌ Stale orderId!
  }, [])
  
  return <div>...</div>
}

// ✅ Good: Include all accessed dependencies
function OrderDetail({ orderId }) {
  useEffect(() => {
    fetchOrder(orderId)
  }, [orderId])
  
  return <div>...</div>
}
```

**Why**: React can't track dependencies you don't list. Effect runs with stale values. ESLint `exhaustive-deps` catches this.

---

**Problem: Conditional dependency arrays**
```jsx
// ❌ Bad: Dependencies change conditionally (breaks rules of hooks)
function OrderForm({ orderId, shouldFetch }) {
  const deps = shouldFetch ? [orderId] : []  // ❌ Conditional array!
  
  useEffect(() => {
    if (shouldFetch) {
      fetchOrder(orderId)
    }
  }, deps)
  
  return <form>...</form>
}

// ✅ Good: Move condition inside effect
function OrderForm({ orderId, shouldFetch }) {
  useEffect(() => {
    if (shouldFetch) {
      fetchOrder(orderId)
    }
  }, [orderId, shouldFetch])
  
  return <form>...</form>
}
```

**Why**: Dependency array must be consistent. React needs to know the dependencies before the effect runs.

---

**Problem: Child re-renders because of new props**
```jsx
// ❌ Bad: Even though OrderItem hasn't changed, it re-renders
function OrderList({ orders }) {
  return orders.map(order => (
    <OrderItem key={order.id} order={order} onView={() => console.log(order.id)} />
  ))
}

// ✅ Good: Memoize child component
const OrderItem = React.memo(({ order, onView }) => {
  return <div onClick={onView}>{order.number}</div>
})

function OrderList({ orders }) {
  const handleView = useCallback((orderId) => {
    console.log(orderId)
  }, [])
  
  return orders.map(order => (
    <OrderItem key={order.id} order={order} onView={handleView} />
  ))
}
```

**Why**: `React.memo` prevents re-render if props haven't changed. Pair with `useCallback` for functions.

---

### 📋 Rendering Performance Checklist

- [ ] Functions passed as props → Wrap with `useCallback`
- [ ] Expensive computations (sort, filter, map chains) → Use `useMemo`
- [ ] Inline objects/arrays as props → Move outside or memoize
- [ ] useEffect without dependency array → Add empty `[]` or list dependencies
- [ ] Dependencies missing from useEffect → Run ESLint rule and add them
- [ ] Child components re-rendering unnecessarily → Wrap with `React.memo`
- [ ] Props being recreated every render → Use `useMemo` or move outside component

---

## 2. State Management & Hooks

Goal: Correct use of hooks, proper state initialization, and avoiding common hook mistakes.

### ✅ What to Look For

#### State vs Props Misuse

**Problem: Props being stored in state**
```jsx
// ❌ Bad: Redundant state
function OrderForm({ orderId }) {
  const [id, setId] = useState(orderId)  // Unnecessary!
  
  return <div>{id}</div>
}

// ✅ Good: Use prop directly
function OrderForm({ orderId }) {
  return <div>{orderId}</div>
}

// ✅ Good: If you need to track changes
function OrderForm({ orderId: initialId }) {
  const [id, setId] = useState(initialId)
  
  useEffect(() => {
    setId(initialId)  // Update when prop changes
  }, [initialId])
  
  return <div>{id}</div>
}
```

**Why**: State duplicate = two sources of truth. Causes bugs and unnecessary re-renders.

---

**Problem: useEffect with immediate state update**
```jsx
// ❌ Bad: Sets state immediately, causes extra render
function OrderDetail({ orderId }) {
  const [order, setOrder] = useState(null)
  
  useEffect(() => {
    const fetched = fetchOrderSync(orderId)  // Synchronous!
    setOrder(fetched)
  }, [orderId])
  
  return <div>{order?.number}</div>
}

// ✅ Good: Initialize state with data
function OrderDetail({ orderId }) {
  const [order] = useState(() => fetchOrderSync(orderId))
  return <div>{order?.number}</div>
}

// ✅ Good: For async data, use useEffect without extra update
function OrderDetail({ orderId }) {
  const [order, setOrder] = useState(null)
  
  useEffect(() => {
    fetchOrder(orderId).then(setOrder)
  }, [orderId])
  
  return <div>{order?.number}</div>
}
```

**Why**: Synchronous updates in effects cause extra renders. Use lazy initialization or handle async properly.

---

**Problem: Multiple useState calls for related state**
```jsx
// ❌ Bad: Multiple state calls for one concept
function OrderForm() {
  const [warehouseFilter, setWarehouseFilter] = useState('all')
  const [categoryFilter, setCategoryFilter] = useState('all')
  const [statusFilter, setStatusFilter] = useState('all')
  
  const handleFilterChange = (key, value) => {
    if (key === 'warehouse') setWarehouseFilter(value)
    if (key === 'category') setCategoryFilter(value)
    if (key === 'status') setStatusFilter(value)
  }
  
  return <div>...</div>
}

// ✅ Good: Combine related state
function OrderForm() {
  const [filters, setFilters] = useState({
    warehouse: 'all',
    category: 'all',
    status: 'all'
  })
  
  const handleFilterChange = (key, value) => {
    setFilters(prev => ({ ...prev, [key]: value }))
  }
  
  return <div>...</div>
}

// ✅ Good: For complex state, use useReducer
function OrderForm() {
  const [filters, dispatch] = useReducer(filterReducer, initialFilters)
  
  const handleFilterChange = (key, value) => {
    dispatch({ type: 'UPDATE_FILTER', key, value })
  }
  
  return <div>...</div>
}

function filterReducer(state, action) {
  switch (action.type) {
    case 'UPDATE_FILTER':
      return { ...state, [action.key]: action.value }
    case 'RESET':
      return initialFilters
    default:
      return state
  }
}
```

**Why**: Related state belongs together. `useReducer` is clearer for complex state machines.

---

**Problem: Stale closures in event handlers**
```jsx
// ❌ Bad: Handler captures old state value
function Counter() {
  const [count, setCount] = useState(0)
  
  const handleClick = () => {
    setTimeout(() => {
      console.log(`Count: ${count}`)  // ❌ Always logs 0!
    }, 1000)
  }
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={handleClick}>Log After 1s</button>
    </div>
  )
}

// ✅ Good: Use state directly in async
function Counter() {
  const [count, setCount] = useState(0)
  
  const handleClick = () => {
    setTimeout(() => {
      setCount(prev => {
        console.log(`Count: ${prev}`)  // ✅ Correct value
        return prev
      })
    }, 1000)
  }
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={handleClick}>Log After 1s</button>
    </div>
  )
}
```

**Why**: Closures capture current state. Async handlers often need the latest state, so use state updater function.

---

**Problem: useEffect cleaning up previous effect**
```jsx
// ❌ Bad: Cleanup runs before new effect
function UserProfile({ userId }) {
  useEffect(() => {
    let mounted = true
    
    fetchUser(userId).then(user => {
      if (mounted) setUser(user)  // ❌ May never set if cleanup runs first
    })
    
    return () => {
      mounted = false
    }
  }, [userId])
  
  return <div>...</div>
}

// ✅ Good: Use AbortController for cleaner cancellation
function UserProfile({ userId }) {
  useEffect(() => {
    const controller = new AbortController()
    
    fetchUser(userId, { signal: controller.signal })
      .then(user => setUser(user))
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err)
        }
      })
    
    return () => controller.abort()
  }, [userId])
  
  return <div>...</div>
}
```

**Why**: AbortController is the modern standard for cancelling in-flight requests.

---

### 📋 State Management Checklist

- [ ] Props being stored in state → Use prop directly
- [ ] Multiple useState calls for related state → Combine with useReducer
- [ ] useEffect with immediate setState → Use lazy initialization
- [ ] Dependencies missing from useEffect → Add to dependency array
- [ ] Stale closures in async handlers → Use state updater function
- [ ] No cleanup in effects that fetch data → Add AbortController or cleanup flag
- [ ] useState before conditional logic → Move conditionals inside effect

---

## 3. Code Reuse Patterns

Goal: Extract duplicated logic, identify custom hook candidates, and simplify component structure.

### ✅ What to Look For

#### Extractable Custom Hooks

**Pattern: Repeated state + logic across components**

If multiple components share the same state management and logic, extract a custom hook.

**Example: Filter state appears in 3 components**
```jsx
// ❌ Before: Repeated in each component
// OrdersView.jsx, InventoryView.jsx, RestockingView.jsx
function OrdersView() {
  const [filters, setFilters] = useState({
    warehouse: 'all',
    category: 'all',
    status: 'all'
  })
  
  const updateFilter = (key, value) => {
    setFilters(prev => ({ ...prev, [key]: value }))
  }
  
  const hasActiveFilters = filters.warehouse !== 'all' ||
                          filters.category !== 'all' ||
                          filters.status !== 'all'
  
  const resetFilters = () => {
    setFilters({ warehouse: 'all', category: 'all', status: 'all' })
  }
  
  return <div>...</div>
}

// ✅ After: Extract to custom hook
function useFilters(initialFilters = { warehouse: 'all', category: 'all', status: 'all' }) {
  const [filters, setFilters] = useState(initialFilters)
  
  const updateFilter = useCallback((key, value) => {
    setFilters(prev => ({ ...prev, [key]: value }))
  }, [])
  
  const hasActiveFilters = useMemo(() => {
    return Object.values(filters).some(v => v !== 'all')
  }, [filters])
  
  const resetFilters = useCallback(() => {
    setFilters(initialFilters)
  }, [initialFilters])
  
  return { filters, updateFilter, hasActiveFilters, resetFilters }
}

// Usage
function OrdersView() {
  const { filters, updateFilter, hasActiveFilters, resetFilters } = useFilters()
  return <div>...</div>
}
```

**Red flags for "extract a custom hook"**:
- Same `useState()` + `useEffect()` patterns in 2+ components
- Identical event handlers across components
- Duplicate API fetch logic
- Shared form validation across components

---

**Pattern: Prop drilling through multiple levels**

If a component passes props down 3+ levels or bubbles events up 3+ levels, consider:
1. A custom hook for shared state
2. Context API for deep nesting
3. Component composition with render props or children

```jsx
// ❌ Before: Prop drilling
<ParentComponent>
  <ChildComponent modal={modal} onClose={() => setModal(false)}>
    <GrandchildComponent modal={modal} onClose={() => onClose()} />
  </ChildComponent>
</ParentComponent>

// ✅ After: Context API
const ModalContext = createContext()

function ParentComponent() {
  const [isOpen, setIsOpen] = useState(false)
  
  return (
    <ModalContext.Provider value={{ isOpen, setIsOpen }}>
      <ChildComponent />
    </ModalContext.Provider>
  )
}

function GrandchildComponent() {
  const { isOpen, setIsOpen } = useContext(ModalContext)
  return <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
}
```

---

**Pattern: Duplicate JSX markup**

If the same HTML structure repeats (especially with minor variations), extract a sub-component.

```jsx
// ❌ Before: Copy-pasted structure
function Dashboard() {
  return (
    <div>
      <div className="card">
        <h3>{order.orderNumber}</h3>
        <p>{order.status}</p>
      </div>
      
      <div className="card">
        <h3>{invoice.invoiceNumber}</h3>
        <p>{invoice.status}</p>
      </div>
    </div>
  )
}

// ✅ After: Reusable component
function Card({ title, subtitle }) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <p>{subtitle}</p>
    </div>
  )
}

function Dashboard() {
  return (
    <div>
      <Card title={order.orderNumber} subtitle={order.status} />
      <Card title={invoice.invoiceNumber} subtitle={invoice.status} />
    </div>
  )
}
```

---

**Pattern: Wrapper components for prop transformation**

If a component wraps another just to transform props, merge them or use a higher-order component.

```jsx
// ❌ Before: Unnecessary wrapper
function UserWrapper({ userId }) {
  const [user, setUser] = useState(null)
  
  useEffect(() => {
    fetchUser(userId).then(setUser)
  }, [userId])
  
  return <UserCard user={user} />
}

// ✅ After: Move logic into UserCard or extract hook
function useUser(userId) {
  const [user, setUser] = useState(null)
  
  useEffect(() => {
    fetchUser(userId).then(setUser)
  }, [userId])
  
  return user
}

function UserCard({ userId }) {
  const user = useUser(userId)
  return <div>{user?.name}</div>
}
```

---

### 📋 Code Reuse Checklist

- [ ] Same state/methods in 2+ components → Extract custom hook
- [ ] Prop drilling 3+ levels → Use Context API or custom hook
- [ ] Identical API fetch patterns → Create shared fetch hook
- [ ] Repeated JSX markup → Extract sub-component
- [ ] Complex form validation duplicated → Extract validation hook
- [ ] Wrapper components for prop transformation → Merge logic into child or extract hook

---

## 4. Bundle Size Optimization

Goal: Reduce shipped JavaScript, remove unused code, and optimize imports.

### ✅ What to Look For

#### Unused Imports

**Problem: Import entire libraries when only using one function**
```jsx
// ❌ Bad: Imports lodash entire library (~70KB)
import _ from 'lodash'

function OrderList({ orders }) {
  const sorted = _.sortBy(orders, 'date')
  return <div>...</div>
}

// ✅ Good: Import specific function (tree-shakeable)
import { sortBy } from 'lodash-es'

function OrderList({ orders }) {
  const sorted = sortBy(orders, 'date')
  return <div>...</div>
}

// ✅ Best: Use native JavaScript
function OrderList({ orders }) {
  const sorted = [...orders].sort((a, b) => a.date - b.date)
  return <div>...</div>
}
```

**Common heavy libraries to audit**:
- `lodash` - Use native Array/Object methods or `lodash-es`
- `moment.js` - Use `date-fns` or native `Date`
- `axios` - Use native `fetch()` or keep if many requests
- `numeral.js` - Use `Intl.NumberFormat` (built-in)
- `underscore` - Similar to lodash, use native APIs

---

**Problem: Importing unused components**
```jsx
// ❌ Bad: Components imported but never used
import ModalComponent from './Modal'  // Never rendered
import FormComponent from './Form'     // Used

export default function App() {
  return <FormComponent />
}

// ✅ Good: Remove unused imports
import FormComponent from './Form'

export default function App() {
  return <FormComponent />
}
```

**How to find**:
- Search for component name in the file - if 0 results, it's unused
- Build tools with `--analyze` flag show unused code

---

**Problem: Large computed values stored in state**
```jsx
// ❌ Bad: Stores derived data (duplicates source)
function OrderList({ items }) {
  const [sortedItems, setSortedItems] = useState([])
  
  useEffect(() => {
    setSortedItems([...items].sort((a, b) => a.date - b.date))
  }, [items])
  
  return sortedItems.map(item => <Item key={item.id} item={item} />)
}

// ✅ Good: Compute on demand
function OrderList({ items }) {
  const sortedItems = useMemo(() => {
    return [...items].sort((a, b) => a.date - b.date)
  }, [items])
  
  return sortedItems.map(item => <Item key={item.id} item={item} />)
}
```

**Why**: Storing sorted/filtered data duplicates the source data in memory. `useMemo` calculates on-demand.

---

**Problem: No code splitting for large routes**
```jsx
// ❌ Bad: All routes bundled together
import OrdersView from './views/Orders'
import InventoryView from './views/Inventory'
import RestockingView from './views/Restocking'

const routes = [
  { path: '/orders', element: <OrdersView /> },
  { path: '/inventory', element: <InventoryView /> },
  { path: '/restocking', element: <RestockingView /> }
]

// ✅ Good: Lazy load with code splitting
const OrdersView = lazy(() => import('./views/Orders'))
const InventoryView = lazy(() => import('./views/Inventory'))
const RestockingView = lazy(() => import('./views/Restocking'))

const routes = [
  { path: '/orders', element: <Suspense fallback={<Loading />}><OrdersView /></Suspense> },
  { path: '/inventory', element: <Suspense fallback={<Loading />}><InventoryView /></Suspense> },
  { path: '/restocking', element: <Suspense fallback={<Loading />}><RestockingView /></Suspense> }
]
```

**Why**: Large routes are split into separate bundles and loaded only when needed.

---

**Problem: Inline functions as effect dependencies**
```jsx
// ❌ Bad: Function creates new dependency on every render
function OrderDetail({ orderId }) {
  const getOrder = () => fetchOrder(orderId)  // New function each render!
  
  useEffect(() => {
    getOrder()
  }, [getOrder])  // Dependency changes every render!
  
  return <div>...</div>
}

// ✅ Good: Use useCallback to stabilize
function OrderDetail({ orderId }) {
  const getOrder = useCallback(() => {
    return fetchOrder(orderId)
  }, [orderId])
  
  useEffect(() => {
    getOrder()
  }, [getOrder])
  
  return <div>...</div>
}

// ✅ Better: Pass value directly
function OrderDetail({ orderId }) {
  useEffect(() => {
    fetchOrder(orderId)
  }, [orderId])
  
  return <div>...</div>
}
```

**Why**: Functions as dependencies cause unnecessary effect re-runs, defeating memoization benefits.

---

### 📋 Bundle Size Checklist

- [ ] Heavy library imports → Check if native APIs suffice or switch to lighter alternatives
- [ ] Unused component/function imports → Remove
- [ ] Derived data stored in state → Convert to `useMemo`
- [ ] Direct library imports (not tree-shakeable) → Switch to named imports or `-es` variants
- [ ] Routes imported directly → Use lazy + Suspense for code splitting
- [ ] Large utility files (>5KB) → Split into smaller modules
- [ ] Multiple props/libraries for one task → Consolidate

---

## Analysis Workflow

When assigned to optimize a component:

### Step 1: Understand the Component
- Read the entire component top to bottom
- Identify all `useState()` calls and their purposes
- List all `useEffect()` hooks and their dependencies
- Scan the JSX for performance patterns
- Note any prop-passing chains

### Step 2: Walk Through Dimensions
Work through each section above:
1. **Rendering Performance** - Any re-render anti-patterns?
2. **State Management** - Are hooks used correctly?
3. **Code Reuse** - Can logic be extracted?
4. **Bundle Size** - Any unnecessary imports?

### Step 3: Prioritize Findings
Group by impact:
- **High**: Fixes that noticeably improve performance or fix correctness bugs
- **Medium**: Optimizations that prevent future issues
- **Low**: Code style improvements

### Step 4: Propose Changes
For each finding:
1. Show the current code
2. Explain why it's suboptimal
3. Provide the refactored version
4. Estimate impact (e.g., "Reduces re-renders by 50%")

### Step 5: Apply Interactively
- Show one change at a time
- Ask for user confirmation
- Explain the reasoning
- Provide context for any behavior changes

---

## Common Component Anti-Patterns

| Pattern | Problem | Solution |
|---------|---------|----------|
| Every callback wrapped in useCallback | Overhead without benefit | Only for memoized children or dependencies |
| useMemo for simple values | Premature optimization | Only for expensive computations |
| Props stored in state | Duplicate source of truth | Use props or sync with useEffect |
| useEffect without deps | Runs on every render | Add empty `[]` or list dependencies |
| Missing dependencies in effect | Stale values, hard to debug | Run ESLint rule, add all deps |
| Multiple unrelated useState | Scattered state logic | Combine with useReducer |
| Prop drilling 3+ levels | Component coupling | Use Context or custom hook |
| Inline object/array props | Breaks memoization | Move outside or memoize |
| No key or index key in list | Lost component state | Use stable ID |

---

## Quick Reference: Refactoring Snippets

### Extract Custom Hook
```javascript
function useMyFeature(initialValue) {
  const [state, setState] = useState(initialValue)
  
  const updateState = useCallback((value) => {
    setState(value)
  }, [])
  
  useEffect(() => {
    // Side effects here
  }, [state])
  
  return { state, updateState }
}

// Usage
function MyComponent() {
  const { state, updateState } = useMyFeature(0)
  return <div onClick={() => updateState(state + 1)}>{state}</div>
}
```

### Memoize Expensive Computation
```javascript
const MemoizedChild = React.memo(({ data, onAction }) => {
  return <div onClick={onAction}>{data.name}</div>
})

function ParentComponent({ items }) {
  const sorted = useMemo(() => {
    return [...items].sort((a, b) => a.date - b.date)
  }, [items])
  
  const handleAction = useCallback((itemId) => {
    console.log(`Clicked ${itemId}`)
  }, [])
  
  return sorted.map(item => (
    <MemoizedChild key={item.id} data={item} onAction={() => handleAction(item.id)} />
  ))
}
```

### Context for Prop Drilling
```javascript
const MyContext = createContext()

function Provider({ children }) {
  const [state, setState] = useState(null)
  
  return (
    <MyContext.Provider value={{ state, setState }}>
      {children}
    </MyContext.Provider>
  )
}

function useMyContext() {
  const context = useContext(MyContext)
  if (!context) {
    throw new Error('useMyContext must be used within Provider')
  }
  return context
}

// Usage
function ChildComponent() {
  const { state, setState } = useMyContext()
  return <div onClick={() => setState(newValue)}>{state}</div>
}
```

### Lazy Load Routes
```javascript
import { lazy, Suspense } from 'react'

const OrdersView = lazy(() => import('./views/Orders'))
const InventoryView = lazy(() => import('./views/Inventory'))

function App() {
  return (
    <Routes>
      <Route path="/orders" element={
        <Suspense fallback={<LoadingSpinner />}>
          <OrdersView />
        </Suspense>
      } />
      <Route path="/inventory" element={
        <Suspense fallback={<LoadingSpinner />}>
          <InventoryView />
        </Suspense>
      } />
    </Routes>
  )
}
```

---

## Resources

- [React Documentation: Hooks](https://react.dev/reference/react)
- [React Performance Optimization](https://react.dev/learn/render-and-commit)
- [React Patterns: Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [React Profiler Performance Analysis](https://react.dev/learn/react-developer-tools#react-profiler)
- [Web Vitals & Performance](https://web.dev/vitals/)
