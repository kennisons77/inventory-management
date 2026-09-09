---
name: vue-component-optimizer
description: Analyze Vue component structure and suggest optimizations for performance, reactivity, code reuse, and bundle size. Use when reviewing or refactoring Vue components for efficiency.
---

# Vue Component Optimizer

A systematic approach to analyzing Vue 3 (Composition API) components and identifying optimization opportunities across four dimensions: rendering performance, reactivity correctness, code reuse potential, and bundle size.

## How to Use This Skill

When analyzing a Vue component:

1. **Read the component thoroughly** - Understand its structure and intent
2. **Check each dimension below** - Walk through the analysis checklist
3. **Identify patterns** - Match observed code against the anti-patterns
4. **Propose fixes** - Use the refactoring examples provided
5. **Verify impact** - Ensure changes maintain functionality while improving efficiency

---

## 1. Rendering Performance

Goal: Minimize re-renders, unnecessary watchers, and expensive computed properties.

### ✅ What to Look For

#### Unnecessary Re-renders (Anti-patterns)

**Problem: v-if on frequently toggled elements**
```vue
<!-- ❌ Bad: DOM recreated on every toggle -->
<div v-if="isVisible">{{ expensiveComputation() }}</div>

<!-- ✅ Good: Display toggled, DOM reused -->
<div :style="{ display: isVisible ? 'block' : 'none' }">{{ result }}</div>
```

**Why**: `v-if` unmounts/remounts DOM nodes and runs all lifecycle hooks. `v-show` uses CSS display, reusing the DOM.

**When to use which**:
- `v-if`: Rarely toggled, heavy children (modals, sidebars)
- `v-show`: Frequently toggled (dropdown states, hover effects)

---

**Problem: Reactive data in v-for without stable keys**
```vue
<!-- ❌ Bad: Vue can't track which items changed -->
<div v-for="item in items" :key="index">{{ item.name }}</div>

<!-- ✅ Good: Stable identity for each item -->
<div v-for="item in items" :key="item.id">{{ item.name }}</div>
```

**Why**: Vue uses keys to map DOM to data. Changing keys (like index) breaks state and causes re-renders.

**What keys to use**:
- `item.id` or `item.sku` - Unique, stable identifiers
- `item.month` + `item.category` - Composite keys when single ID unavailable
- Never use `index` (breaks when array reorders)

---

**Problem: Expensive computations in templates**
```vue
<!-- ❌ Bad: Recalculates on every render -->
<div>{{ items.filter(x => x.active).map(x => x.value).reduce((a,b) => a+b, 0) }}</div>

<!-- ✅ Good: Computed once, memoized -->
<script setup>
const activeTotal = computed(() => 
  items.value.filter(x => x.active).map(x => x.value).reduce((a,b) => a+b, 0)
)
</script>

<div>{{ activeTotal }}</div>
```

**Why**: Templates re-run on every render. Computeds cache results (run only when dependencies change).

**Red flags in templates**:
- `.filter()`, `.map()`, `.sort()` - Move to computed
- Function calls (except simple getters/formatters)
- Chained operations

---

**Problem: Watcher on computed property**
```vue
<!-- ❌ Bad: Computed already reactive, watcher is redundant -->
<script setup>
const filtered = computed(() => items.value.filter(x => x.active))

watch(filtered, (newVal) => {
  console.log('Filtered changed:', newVal)
})
</script>

<!-- ✅ Good: Watch the source directly -->
<script setup>
watch(
  () => items.value,
  () => {
    const filtered = items.value.filter(x => x.active)
    console.log('Filtered changed:', filtered)
  }
)
</script>
```

**Why**: Watching a computed still triggers every time its dependencies change, but adds an extra listener.

---

**Problem: watch() with immediate + reactive dependency**
```vue
<!-- ❌ Bad: Runs twice on mount (immediate + dependency change) -->
<script setup>
watch(() => items.value, async () => {
  await fetchData()
}, { immediate: true })
</script>

<!-- ✅ Good: Use watchEffect (runs once) or restructure -->
<script setup>
watchEffect(async () => {
  await fetchData(items.value)
})
</script>
```

**Why**: `immediate: true` runs the watcher synchronously, then dependencies change, triggering it again.

---

### 📋 Rendering Performance Checklist

- [ ] `v-if` on frequently toggled elements → Replace with `v-show` or `:style`
- [ ] `v-for` with `index` key → Use stable ID (`.id`, `.sku`)
- [ ] Filters/maps/sorts in template → Extract to computed
- [ ] Function calls in templates → Move to methods or computed
- [ ] Watcher on computed property → Watch source or restructure
- [ ] `watch()` with `immediate: true` on reactive deps → Use `watchEffect`
- [ ] Large computed without explicit dependencies → Verify dependencies are correct

---

## 2. Reactivity Issues

Goal: Correct use of ref/reactive, dependency tracking, and avoiding stale closures.

### ✅ What to Look For

#### Ref vs Reactive Misuse

**Problem: Reactive for single values**
```vue
<!-- ❌ Bad: Overkill, loses type safety -->
<script setup>
const count = reactive({ value: 0 })
const increment = () => count.value++
</script>

<!-- ✅ Good: Simpler, better type inference -->
<script setup>
const count = ref(0)
const increment = () => count.value++
</script>
```

**When to use which**:
- `ref()`: Single values, primitives, any type (safer)
- `reactive()`: Objects with multiple properties that get destructured

---

**Problem: Destructuring reactive objects**
```vue
<!-- ❌ Bad: Loses reactivity when destructured -->
<script setup>
const state = reactive({ count: 0, message: 'Hello' })
const { count, message } = state  // Now static!

const increment = () => count++  // Doesn't update state.count
</script>

<!-- ✅ Good: Use toRefs to maintain reactivity -->
<script setup>
const state = reactive({ count: 0, message: 'Hello' })
const { count, message } = toRefs(state)

const increment = () => count.value++  // Updates state.count
</script>
```

**Why**: Destructuring breaks the proxy chain. `toRefs` wraps each property in a ref.

---

**Problem: Stale closures in event handlers**
```vue
<!-- ❌ Bad: Handler captures old value of 'id' -->
<script setup>
let id = ref(1)

const handleClick = () => {
  setTimeout(() => {
    console.log('ID:', id.value)  // ❌ May be stale if id changed
  }, 1000)
}

watch(id, (newId) => {
  id.value = newId  // Mutation confuses timing
})
</script>

<!-- ✅ Good: Capture value at call time -->
<script setup>
const id = ref(1)

const handleClick = () => {
  const capturedId = id.value
  setTimeout(() => {
    console.log('ID:', capturedId)  // Guaranteed current
  }, 1000)
}
</script>
```

**Why**: Refs update, but closures can capture old versions. Explicit capture prevents confusion.

---

**Problem: Missing dependencies in computed/watcher**
```vue
<!-- ❌ Bad: Changes to 'multiplier' don't trigger recompute -->
<script setup>
const count = ref(0)
const multiplier = ref(2)

const doubled = computed(() => {
  return count.value * 2  // ❌ Depends on multiplier but not tracked
})
</script>

<!-- ✅ Good: Include all dependencies -->
<script setup>
const doubled = computed(() => {
  return count.value * multiplier.value  // ✅ Both tracked
})
</script>
```

**Why**: Computed/watch only re-run when accessed properties change. Missing a property = missed updates.

---

### 📋 Reactivity Checklist

- [ ] Single-value objects using `reactive()` → Switch to `ref()`
- [ ] Destructured `reactive()` without `toRefs()` → Add `toRefs()` or access via `.value`
- [ ] Stale closure suspicions (async handlers) → Capture value at call time
- [ ] Computed/watch missing dependencies → Add all accessed reactive properties
- [ ] Computed depending on non-reactive data → Verify data is wrapped in ref/reactive

---

## 3. Code Reuse Patterns

Goal: Extract duplicated logic, identify composable candidates, and simplify component structure.

### ✅ What to Look For

#### Extractable Composables

**Pattern: Repeated state + logic across components**

If multiple components share the same state management and logic, extract a composable.

**Example: Filter state appears in 3 components**
```vue
<!-- ❌ Before: Repeated in each component -->
// OrdersView.vue, InventoryView.vue, RestockingView.vue
<script setup>
const filters = ref({
  warehouse: 'all',
  category: 'all',
  status: 'all'
})

const updateFilter = (key, value) => {
  filters.value[key] = value
}

const hasActiveFilters = computed(() => {
  return Object.values(filters.value).some(v => v !== 'all')
})

const resetFilters = () => {
  filters.value = { warehouse: 'all', category: 'all', status: 'all' }
}
</script>

<!-- ✅ After: Extract to composable -->
// composables/useFilters.js
export function useFilters() {
  const filters = ref({
    warehouse: 'all',
    category: 'all',
    status: 'all'
  })

  const updateFilter = (key, value) => {
    filters.value[key] = value
  }

  const hasActiveFilters = computed(() => {
    return Object.values(filters.value).some(v => v !== 'all')
  })

  const resetFilters = () => {
    filters.value = { warehouse: 'all', category: 'all', status: 'all' }
  }

  return { filters, updateFilter, hasActiveFilters, resetFilters }
}

// Usage in OrdersView.vue
<script setup>
const { filters, updateFilter, hasActiveFilters, resetFilters } = useFilters()
</script>
```

**Red flags for "extract a composable"**:
- Same `ref()` + `computed()` patterns in 2+ components
- Identical `watch()` or `onMounted()` hooks
- Duplicate API fetch logic
- Shared form validation across components

---

**Pattern: Complex prop drilling / event bubbling**

If a component passes props down 3+ levels or bubbles events up 3+ levels, consider:
1. A composable for shared state
2. Provide/inject for deep nesting
3. A slot-based approach for configuration

```vue
<!-- ❌ Before: Prop drilling -->
<ParentComponent :visible="modal" @close="modal = false" />
  <ChildComponent :visible="visible" @close="$emit('close')" />
    <GrandchildComponent :visible="visible" @close="$emit('close')" />

<!-- ✅ After: Provide/inject -->
// ParentComponent.vue
<script setup>
const isOpen = ref(false)
provide('modal', { isOpen })
</script>

// GrandchildComponent.vue
<script setup>
const { isOpen } = inject('modal')
</script>
```

---

**Pattern: Duplicate template markup**

If the same HTML structure repeats (especially with minor variations), extract a sub-component.

```vue
<!-- ❌ Before: Copy-pasted structure -->
<div class="card">
  <h3>{{ order.orderNumber }}</h3>
  <p>{{ order.status }}</p>
</div>

<div class="card">
  <h3>{{ invoice.invoiceNumber }}</h3>
  <p>{{ invoice.status }}</p>
</div>

<!-- ✅ After: Reusable component -->
<OrderCard :order="order" />
<InvoiceCard :invoice="invoice" />
```

---

### 📋 Code Reuse Checklist

- [ ] Same state/methods in 2+ components → Extract composable
- [ ] Prop drilling 3+ levels → Use provide/inject or composable
- [ ] Identical API fetch patterns → Create shared fetch composable
- [ ] Repeated template markup → Extract sub-component
- [ ] Complex form validation duplicated → Extract validation composable
- [ ] Shared computed property logic → Move to utility function or composable

---

## 4. Bundle Size Optimization

Goal: Reduce shipped JavaScript, remove unused code, and optimize imports.

### ✅ What to Look For

#### Unused Imports

**Problem: Import entire libraries when only using one function**
```vue
<!-- ❌ Bad: Imports lodash entire library (~70KB) -->
<script setup>
import _ from 'lodash'
const sorted = _.sortBy(items, 'name')
</script>

<!-- ✅ Good: Import specific function (tree-shakeable) -->
<script setup>
import { sortBy } from 'lodash-es'
const sorted = sortBy(items, 'name')
</script>
```

**Or use native JavaScript:**
```vue
<!-- ✅ Best: No import, native API (smallest) -->
<script setup>
const sorted = items.sort((a, b) => a.name.localeCompare(b.name))
</script>
```

**Common heavy libraries to audit**:
- `lodash` - Use native Array/Object methods or `lodash-es` for tree-shaking
- `moment.js` - Use `date-fns` or native `Date`
- `axios` - Use native `fetch()` or keep if many requests
- `numeral.js` - Use `Intl.NumberFormat` (built-in)

---

**Problem: Importing unused components**
```vue
<!-- ❌ Bad: Component imported but never used -->
<script setup>
import ModalComponent from './Modal.vue'  // Never rendered
import FormComponent from './Form.vue'     // Used
</script>

<template>
  <FormComponent />
</template>

<!-- ✅ Good: Remove unused imports -->
<script setup>
import FormComponent from './Form.vue'
</script>
```

**How to find**:
- Search for component name in template - if 0 results, it's unused
- Build tools (Vite, Webpack) can warn about unused exports with `--analyze`

---

**Problem: Large computed properties stored in state**
```vue
<!-- ❌ Bad: Stores derived data (duplicates source) -->
<script setup>
const items = ref([...])
const sortedItems = ref([])

onMounted(async () => {
  sortedItems.value = await fetchAndSort()  // Extra memory
})
</script>

<!-- ✅ Good: Compute on demand -->
<script setup>
const items = ref([...])
const sortedItems = computed(() => {
  return [...items.value].sort((a, b) => a.name.localeCompare(b.name))
})
</script>
```

**Why**: Storing sorted/filtered data duplicates the source data in memory. Computeds calculate on-demand.

---

**Problem: No code splitting for large views**
```vue
<!-- ❌ Bad: All views bundled together -->
import OrdersView from './views/Orders.vue'
import InventoryView from './views/Inventory.vue'
import RestockingView from './views/Restocking.vue'

<!-- ✅ Good: Lazy load with route splitting -->
const OrdersView = () => import('./views/Orders.vue')
const InventoryView = () => import('./views/Inventory.vue')
const RestockingView = () => import('./views/Restocking.vue')
```

**Why**: Large views are split into separate files and loaded only when needed.

---

### 📋 Bundle Size Checklist

- [ ] Heavy library imports → Check if native APIs suffice or switch to lighter alternatives
- [ ] Unused component/function imports → Remove
- [ ] Derived data stored in state → Convert to computed
- [ ] Direct library imports (not tree-shakeable) → Switch to named imports or `-es` variants
- [ ] Views imported directly in router → Use dynamic imports (`() => import()`)
- [ ] Large utility files (>5KB) → Split into smaller modules

---

## Analysis Workflow

When assigned to optimize a component:

### Step 1: Understand the Component
- Read the `<script setup>` section fully
- Identify all reactive state (refs, reactive)
- List all computed properties, watchers, methods
- Scan the template for loops, conditionals, event handlers

### Step 2: Walk Through Dimensions
Work through each section above:
1. **Rendering Performance** - Any anti-patterns?
2. **Reactivity Issues** - Are refs/reactive used correctly?
3. **Code Reuse** - Can logic be extracted?
4. **Bundle Size** - Any unnecessary imports?

### Step 3: Prioritize Findings
Group by impact:
- **High**: Fixes that improve performance noticeably or remove significant code
- **Medium**: Correctness fixes or minor optimizations
- **Low**: Code style or minor refactors

### Step 4: Propose Changes
For each finding:
1. Show the current code
2. Explain why it's suboptimal
3. Provide the refactored version
4. Estimate impact (e.g., "Removes 1 unnecessary watcher")

### Step 5: Apply Interactively
- Show one change at a time
- Ask for user confirmation
- Explain the reasoning
- Provide context for any behavior changes

---

## Common Component Anti-Patterns

| Pattern | Problem | Solution |
|---------|---------|----------|
| `v-if` on frequent toggle | Expensive DOM operations | Use `v-show` or `:style` |
| `v-for` with index key | Breaks Vue's tracking | Use stable ID (`.id`, `.sku`) |
| Computed without deps tracked | Stale values | List all accessed reactives |
| `reactive()` for single value | Type safety loss | Use `ref()` instead |
| Destructured `reactive()` | Loses reactivity | Use `toRefs()` wrapper |
| Watcher on computed | Redundant listener | Watch source directly |
| Repeated logic in components | Code duplication | Extract to composable |
| Heavy library import | Large bundle | Use native API or lite alternative |
| View imported statically | No code splitting | Use `() => import()` |

---

## Quick Reference: Refactoring Snippets

### Convert to Composable
```javascript
// composables/useMyFeature.js
import { ref, computed } from 'vue'

export function useMyFeature() {
  const state = ref(initialValue)
  const derived = computed(() => transform(state.value))
  
  const action = () => {
    state.value = newValue
  }
  
  return { state, derived, action }
}
```

### Safe Destructuring
```javascript
import { toRefs } from 'vue'

const myObject = reactive({ a: 1, b: 2 })
const { a, b } = toRefs(myObject)  // Stays reactive
```

### Template Optimization
```vue
<!-- Before: Computation in template -->
<div>{{ items.filter(x => x.active).length }}</div>

<!-- After: Computed property -->
<script setup>
const activeCount = computed(() => items.value.filter(x => x.active).length)
</script>
<div>{{ activeCount }}</div>
```

---

## Resources

- [Vue 3 Performance Guide](https://vuejs.org/guide/best-practices/performance.html)
- [Composition API Docs](https://vuejs.org/guide/extras/composition-api-faq.html)
- [Vue 3 Migration from Options API](https://vuejs.org/guide/extras/composition-api-faq.html)
