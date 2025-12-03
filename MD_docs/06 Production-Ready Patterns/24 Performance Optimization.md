# Chapter 24: Performance Optimization

## React.memo and useMemo: when to use them

## The Anchor: A Product Dashboard That Works... Slowly

We're building a product analytics dashboard for an e-commerce platform. It displays:

- A list of products with sales metrics
- A chart showing sales trends
- Filter controls (category, date range, sort order)
- A summary statistics panel

The dashboard works correctly. Users can filter products, change date ranges, and see updated charts. But there's a problem: every interaction feels sluggish. Typing in the search box has noticeable lag. Changing a filter causes a visible freeze.

Let's build the naive implementation first, then diagnose why it's slow.

```tsx
// src/components/ProductDashboard.tsx
import { useState } from 'react';

interface Product {
  id: string;
  name: string;
  category: string;
  sales: number;
  revenue: number;
}

interface SalesData {
  date: string;
  amount: number;
}

// Simulated expensive calculation
function calculateTrend(data: SalesData[]): number {
  console.log('🔄 Calculating trend...');
  // Simulate complex statistical analysis
  let sum = 0;
  for (let i = 0; i < 1000000; i++) {
    sum += Math.random();
  }
  
  const trend = data.reduce((acc, curr, idx, arr) => {
    if (idx === 0) return 0;
    return acc + (curr.amount - arr[idx - 1].amount);
  }, 0);
  
  return trend / data.length;
}

// Simulated expensive filtering
function filterProducts(
  products: Product[],
  category: string,
  searchTerm: string
): Product[] {
  console.log('🔄 Filtering products...');
  // Simulate expensive operation
  let sum = 0;
  for (let i = 0; i < 500000; i++) {
    sum += Math.random();
  }
  
  return products.filter(p => {
    const matchesCategory = category === 'all' || p.category === category;
    const matchesSearch = p.name.toLowerCase().includes(searchTerm.toLowerCase());
    return matchesCategory && matchesSearch;
  });
}

function ProductList({ products }: { products: Product[] }) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      <h3>Products ({products.length})</h3>
      {products.map(product => (
        <div key={product.id} className="product-card">
          <h4>{product.name}</h4>
          <p>Category: {product.category}</p>
          <p>Sales: {product.sales} units</p>
          <p>Revenue: ${product.revenue.toLocaleString()}</p>
        </div>
      ))}
    </div>
  );
}

function SalesChart({ data }: { data: SalesData[] }) {
  console.log('🎨 Rendering SalesChart');
  const trend = calculateTrend(data);
  
  return (
    <div className="sales-chart">
      <h3>Sales Trend</h3>
      <p>Trend: {trend > 0 ? '📈' : '📉'} {trend.toFixed(2)}</p>
      <div className="chart-placeholder">
        {data.map((point, idx) => (
          <div key={idx} className="chart-bar" style={{ height: `${point.amount / 10}px` }}>
            {point.amount}
          </div>
        ))}
      </div>
    </div>
  );
}

function StatsSummary({ products }: { products: Product[] }) {
  console.log('🎨 Rendering StatsSummary');
  const totalSales = products.reduce((sum, p) => sum + p.sales, 0);
  const totalRevenue = products.reduce((sum, p) => sum + p.revenue, 0);
  
  return (
    <div className="stats-summary">
      <h3>Summary</h3>
      <p>Total Products: {products.length}</p>
      <p>Total Sales: {totalSales} units</p>
      <p>Total Revenue: ${totalRevenue.toLocaleString()}</p>
    </div>
  );
}

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  
  // Mock data
  const allProducts: Product[] = [
    { id: '1', name: 'Laptop Pro', category: 'electronics', sales: 150, revenue: 225000 },
    { id: '2', name: 'Desk Chair', category: 'furniture', sales: 300, revenue: 45000 },
    { id: '3', name: 'Coffee Maker', category: 'appliances', sales: 500, revenue: 50000 },
    { id: '4', name: 'Monitor 4K', category: 'electronics', sales: 200, revenue: 100000 },
    { id: '5', name: 'Standing Desk', category: 'furniture', sales: 100, revenue: 50000 },
  ];
  
  const salesData: SalesData[] = [
    { date: '2024-01', amount: 1000 },
    { date: '2024-02', amount: 1200 },
    { date: '2024-03', amount: 1100 },
    { date: '2024-04', amount: 1400 },
    { date: '2024-05', amount: 1600 },
  ];
  
  const filteredProducts = filterProducts(allProducts, category, searchTerm);
  
  const sortedProducts = [...filteredProducts].sort((a, b) => {
    return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
  });
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className="dashboard">
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList products={sortedProducts} />
        <SalesChart data={salesData} />
        <StatsSummary products={sortedProducts} />
      </div>
    </div>
  );
}
```

### The Failure: Every Keystroke Triggers Everything

Let's interact with this dashboard and observe what happens.

**User Action**: Type "Laptop" in the search box, one character at a time.

**Browser Behavior**:
- Noticeable lag between keystrokes
- UI feels unresponsive
- Each character typed causes a visible freeze (~200-300ms)

```bash
# Browser Console Output (typing "L", then "a", then "p"):

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering SalesChart
🔄 Calculating trend...
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering SalesChart
🔄 Calculating trend...
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering SalesChart
🔄 Calculating trend...
🎨 Rendering StatsSummary
```

### Diagnostic Analysis: Reading the Performance Failure

**React DevTools - Profiler Tab**:
- Recorded interaction: Typing "Laptop" (6 characters)
- Total renders: 6 (one per keystroke)
- Each render took ~250ms
- Total time: 1.5 seconds for 6 characters
- Components that rendered each time:
  - `ProductDashboard` ✓ (expected - state changed)
  - `ProductList` ✓ (expected - filtered products changed)
  - `SalesChart` ✗ (unexpected - sales data never changed)
  - `StatsSummary` ✓ (expected - filtered products changed)

**React DevTools - Components Tab**:
- `SalesChart` props: `{ data: Array(5) }`
- Props comparison: `data` array reference changes every render
- Reason for re-render: "Props changed"
- But the actual data inside the array is identical

**Chrome Performance Tab**:
- Main thread blocked for ~250ms per keystroke
- Breakdown per render:
  - `filterProducts`: ~100ms (expensive filtering)
  - `calculateTrend`: ~120ms (expensive calculation)
  - React reconciliation: ~30ms
- Total: ~250ms of blocked main thread

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Instant feedback when typing
   - Actual: 250ms lag per keystroke, feels sluggish

2. **What the console reveals**:
   - Every state change triggers a full re-render of all components
   - `calculateTrend` runs on every keystroke, even though sales data never changes
   - `filterProducts` runs on every keystroke (expected, but could be optimized)

3. **What DevTools shows**:
   - `SalesChart` re-renders unnecessarily
   - The `data` prop is a new array reference each render, even though contents are identical
   - Expensive calculations run repeatedly with the same inputs

4. **Root cause identified**: 
   - React re-renders all child components when parent state changes
   - Expensive calculations run on every render, even with unchanged inputs
   - New object/array references created each render trigger unnecessary child re-renders

5. **Why the current approach can't solve this**:
   - React's default behavior is to re-render everything when state changes
   - Without optimization, every component recalculates everything every time
   - We need to tell React: "Skip this work if the inputs haven't changed"

6. **What we need**:
   - A way to skip re-rendering components when their props haven't changed
   - A way to cache expensive calculation results
   - A way to preserve object/array references across renders

## React.memo: Preventing Unnecessary Component Re-renders

`React.memo` is a higher-order component that tells React: "Only re-render this component if its props have actually changed."

### How React.memo Works

Without `React.memo`, React's default behavior:
1. Parent state changes
2. Parent re-renders
3. All child components re-render, regardless of whether their props changed

With `React.memo`:
1. Parent state changes
2. Parent re-renders
3. React compares child's new props with previous props (shallow comparison)
4. If props are identical, skip re-rendering the child
5. If props changed, re-render the child

### Iteration 1: Memoizing SalesChart

The `SalesChart` component receives `data` that never changes. Let's prevent it from re-rendering unnecessarily.

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo } from 'react';

// ... (Product, SalesData interfaces unchanged)
// ... (calculateTrend, filterProducts functions unchanged)

// Wrap SalesChart with React.memo
const SalesChart = memo(function SalesChart({ data }: { data: SalesData[] }) {
  console.log('🎨 Rendering SalesChart');
  const trend = calculateTrend(data);
  
  return (
    <div className="sales-chart">
      <h3>Sales Trend</h3>
      <p>Trend: {trend > 0 ? '📈' : '📉'} {trend.toFixed(2)}</p>
      <div className="chart-placeholder">
        {data.map((point, idx) => (
          <div key={idx} className="chart-bar" style={{ height: `${point.amount / 10}px` }}>
            {point.amount}
          </div>
        ))}
      </div>
    </div>
  );
});

// ... (ProductList, StatsSummary unchanged)

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  
  const allProducts: Product[] = [
    { id: '1', name: 'Laptop Pro', category: 'electronics', sales: 150, revenue: 225000 },
    { id: '2', name: 'Desk Chair', category: 'furniture', sales: 300, revenue: 45000 },
    { id: '3', name: 'Coffee Maker', category: 'appliances', sales: 500, revenue: 50000 },
    { id: '4', name: 'Monitor 4K', category: 'electronics', sales: 200, revenue: 100000 },
    { id: '5', name: 'Standing Desk', category: 'furniture', sales: 100, revenue: 50000 },
  ];
  
  // Move salesData outside component or use useMemo to maintain reference
  const salesData: SalesData[] = [
    { date: '2024-01', amount: 1000 },
    { date: '2024-02', amount: 1200 },
    { date: '2024-03', amount: 1100 },
    { date: '2024-04', amount: 1400 },
    { date: '2024-05', amount: 1600 },
  ];
  
  const filteredProducts = filterProducts(allProducts, category, searchTerm);
  const sortedProducts = [...filteredProducts].sort((a, b) => {
    return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
  });
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className="dashboard">
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList products={sortedProducts} />
        <SalesChart data={salesData} />
        <StatsSummary products={sortedProducts} />
      </div>
    </div>
  );
}
```

**User Action**: Type "Laptop" in the search box again.

```bash
# Browser Console Output (typing "L", then "a", then "p"):

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering SalesChart
🔄 Calculating trend...
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary
```

### The Failure: React.memo Didn't Work!

**Browser Behavior**:
- `SalesChart` still renders on the first keystroke
- But then stops re-rendering on subsequent keystrokes
- Performance improved slightly, but still sluggish

**React DevTools - Profiler**:
- First render: ~250ms (SalesChart rendered)
- Subsequent renders: ~130ms (SalesChart skipped)
- Improvement: ~48% faster after first render

**Why did SalesChart render on the first keystroke?**

The problem is that `salesData` is defined inside the component function. Every time `ProductDashboard` re-renders, a new `salesData` array is created with a new reference. Even though the contents are identical, React's shallow comparison sees different references and considers the props "changed."

### Understanding React.memo's Comparison

`React.memo` uses **shallow comparison** by default:
- Primitive values (string, number, boolean): Compares by value
- Objects and arrays: Compares by reference

```typescript
// These are different references, even with identical contents
const arr1 = [1, 2, 3];
const arr2 = [1, 2, 3];
console.log(arr1 === arr2); // false

// These are the same reference
const arr3 = arr1;
console.log(arr1 === arr3); // true
```

Every render creates a new `salesData` array, so `React.memo` sees it as a prop change.

### Solution 1: Move Static Data Outside Component

If data doesn't depend on props or state, define it outside the component.

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo } from 'react';

// ... (interfaces unchanged)

// Move static data outside component
const SALES_DATA: SalesData[] = [
  { date: '2024-01', amount: 1000 },
  { date: '2024-02', amount: 1200 },
  { date: '2024-03', amount: 1100 },
  { date: '2024-04', amount: 1400 },
  { date: '2024-05', amount: 1600 },
];

const ALL_PRODUCTS: Product[] = [
  { id: '1', name: 'Laptop Pro', category: 'electronics', sales: 150, revenue: 225000 },
  { id: '2', name: 'Desk Chair', category: 'furniture', sales: 300, revenue: 45000 },
  { id: '3', name: 'Coffee Maker', category: 'appliances', sales: 500, revenue: 50000 },
  { id: '4', name: 'Monitor 4K', category: 'electronics', sales: 200, revenue: 100000 },
  { id: '5', name: 'Standing Desk', category: 'furniture', sales: 100, revenue: 50000 },
];

// ... (calculateTrend, filterProducts unchanged)

const SalesChart = memo(function SalesChart({ data }: { data: SalesData[] }) {
  console.log('🎨 Rendering SalesChart');
  const trend = calculateTrend(data);
  
  return (
    <div className="sales-chart">
      <h3>Sales Trend</h3>
      <p>Trend: {trend > 0 ? '📈' : '📉'} {trend.toFixed(2)}</p>
      <div className="chart-placeholder">
        {data.map((point, idx) => (
          <div key={idx} className="chart-bar" style={{ height: `${point.amount / 10}px` }}>
            {point.amount}
          </div>
        ))}
      </div>
    </div>
  );
});

// ... (ProductList, StatsSummary unchanged)

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  
  const filteredProducts = filterProducts(ALL_PRODUCTS, category, searchTerm);
  const sortedProducts = [...filteredProducts].sort((a, b) => {
    return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
  });
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className="dashboard">
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList products={sortedProducts} />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
    </div>
  );
}
```

**User Action**: Type "Laptop" in the search box.

```bash
# Browser Console Output (typing "L", then "a", then "p"):

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary
```

**Expected vs. Actual Improvement**:
- `SalesChart` no longer renders on any keystroke
- `calculateTrend` no longer runs (saved ~120ms per keystroke)
- Performance: ~130ms per keystroke (down from 250ms)
- Improvement: 48% faster

**React DevTools - Profiler**:
- Each render now takes ~130ms (vs. 250ms before)
- `SalesChart` shows "Did not render" for all typing interactions
- Main thread blocked time reduced by ~120ms per keystroke

**Limitation preview**: We still have `filterProducts` running on every keystroke, taking ~100ms. And we're still creating new arrays for `sortedProducts` on every render. Let's address those next.

## useMemo: Caching Expensive Calculations

`useMemo` tells React: "Only recalculate this value if its dependencies change."

### How useMemo Works

```typescript
const memoizedValue = useMemo(() => {
  // Expensive calculation
  return expensiveFunction(dependency1, dependency2);
}, [dependency1, dependency2]);
```

React will:
1. Run the calculation on first render
2. Cache the result
3. On subsequent renders, check if dependencies changed
4. If dependencies unchanged, return cached result
5. If dependencies changed, recalculate and cache new result

### Iteration 2: Memoizing Filtered Products

The `filterProducts` function runs on every render, even when `category` and `searchTerm` haven't changed (e.g., when clicking the sort button).

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, static data, helper functions unchanged)

const SalesChart = memo(function SalesChart({ data }: { data: SalesData[] }) {
  console.log('🎨 Rendering SalesChart');
  
  // Memoize the expensive trend calculation
  const trend = useMemo(() => {
    console.log('🔄 Calculating trend...');
    return calculateTrend(data);
  }, [data]);
  
  return (
    <div className="sales-chart">
      <h3>Sales Trend</h3>
      <p>Trend: {trend > 0 ? '📈' : '📉'} {trend.toFixed(2)}</p>
      <div className="chart-placeholder">
        {data.map((point, idx) => (
          <div key={idx} className="chart-bar" style={{ height: `${point.amount / 10}px` }}>
            {point.amount}
          </div>
        ))}
      </div>
    </div>
  );
});

// ... (ProductList, StatsSummary unchanged)

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  
  // Memoize filtered products - only recalculate when category or searchTerm changes
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  // Memoize sorted products - only recalculate when filteredProducts or sortOrder changes
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className="dashboard">
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList products={sortedProducts} />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
    </div>
  );
}
```

**User Action 1**: Type "Laptop" in the search box.

```bash
# Browser Console Output (typing "L", then "a", then "p"):

🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary

🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary
```

**User Action 2**: Click the sort button (without changing search or category).

```bash
# Browser Console Output:

🎨 Rendering ProductDashboard
🔄 Sorting products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary
```

**Expected vs. Actual Improvement**:

**Typing in search**:
- Still runs filtering and sorting (expected - dependencies changed)
- Performance: ~130ms per keystroke (unchanged from before)
- But now we have the foundation for further optimization

**Clicking sort button**:
- Before: Ran filtering (~100ms) + sorting (~10ms) = ~110ms
- After: Only runs sorting (~10ms)
- Improvement: 91% faster for sort-only operations

**React DevTools - Profiler**:
- Sort button click: 10ms (vs. 110ms before)
- Filtering only runs when `category` or `searchTerm` changes
- Sorting only runs when `filteredProducts` or `sortOrder` changes

### The Subtle Bug: useMemo Dependency Arrays

Let's introduce a common mistake to understand how dependency arrays work.

```tsx
// ❌ WRONG: Missing dependency
const filteredProducts = useMemo(() => {
  return filterProducts(ALL_PRODUCTS, category, searchTerm);
}, [category]); // Missing searchTerm!

// What happens:
// - Changing category triggers recalculation ✓
// - Changing searchTerm does NOT trigger recalculation ✗
// - filteredProducts becomes stale when searchTerm changes
```

**User Action**: Type "Laptop" in search box.

**Browser Behavior**:
- Products don't filter as you type
- Console shows no "Filtering products..." message
- UI shows all products regardless of search term

**React DevTools - Components Tab**:
- `searchTerm` state updates correctly
- `filteredProducts` value doesn't change
- Reason: useMemo dependencies don't include `searchTerm`

**The Rule**: Include ALL values from component scope that the memoized function uses. React's ESLint plugin (`eslint-plugin-react-hooks`) will warn you about missing dependencies.

### Iteration 3: Memoizing Child Components

Now let's memoize `ProductList` and `StatsSummary` to prevent unnecessary re-renders when their props haven't changed.

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, static data, helper functions unchanged)

// Memoize ProductList
const ProductList = memo(function ProductList({ products }: { products: Product[] }) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      <h3>Products ({products.length})</h3>
      {products.map(product => (
        <div key={product.id} className="product-card">
          <h4>{product.name}</h4>
          <p>Category: {product.category}</p>
          <p>Sales: {product.sales} units</p>
          <p>Revenue: ${product.revenue.toLocaleString()}</p>
        </div>
      ))}
    </div>
  );
});

const SalesChart = memo(function SalesChart({ data }: { data: SalesData[] }) {
  console.log('🎨 Rendering SalesChart');
  
  const trend = useMemo(() => {
    console.log('🔄 Calculating trend...');
    return calculateTrend(data);
  }, [data]);
  
  return (
    <div className="sales-chart">
      <h3>Sales Trend</h3>
      <p>Trend: {trend > 0 ? '📈' : '📉'} {trend.toFixed(2)}</p>
      <div className="chart-placeholder">
        {data.map((point, idx) => (
          <div key={idx} className="chart-bar" style={{ height: `${point.amount / 10}px` }}>
            {point.amount}
          </div>
        ))}
      </div>
    </div>
  );
});

// Memoize StatsSummary
const StatsSummary = memo(function StatsSummary({ products }: { products: Product[] }) {
  console.log('🎨 Rendering StatsSummary');
  
  // Memoize expensive calculations inside the component
  const totalSales = useMemo(() => {
    console.log('🔄 Calculating total sales...');
    return products.reduce((sum, p) => sum + p.sales, 0);
  }, [products]);
  
  const totalRevenue = useMemo(() => {
    console.log('🔄 Calculating total revenue...');
    return products.reduce((sum, p) => sum + p.revenue, 0);
  }, [products]);
  
  return (
    <div className="stats-summary">
      <h3>Summary</h3>
      <p>Total Products: {products.length}</p>
      <p>Total Sales: {totalSales} units</p>
      <p>Total Revenue: ${totalRevenue.toLocaleString()}</p>
    </div>
  );
});

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className="dashboard">
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList products={sortedProducts} />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
    </div>
  );
}
```

**User Action**: Click the sort button.

```bash
# Browser Console Output:

🎨 Rendering ProductDashboard
🔄 Sorting products...
🎨 Rendering ProductList
🎨 Rendering StatsSummary
🔄 Calculating total sales...
🔄 Calculating total revenue...
```

**Wait, that's wrong!** `ProductList` and `StatsSummary` still re-rendered, even though we wrapped them with `React.memo`.

### The Failure: React.memo Doesn't Work with New Array References

**Diagnostic Analysis**:

**React DevTools - Components Tab**:
- `ProductList` props: `{ products: Array(5) }`
- Props comparison: `products` array reference changed
- Reason: `sortedProducts` is a new array every time we sort

**The Problem**: Even though `useMemo` caches the sorted array, when we click the sort button, `sortOrder` changes, so `useMemo` recalculates and returns a **new array reference**. `React.memo` sees a different reference and re-renders the component.

**This is actually correct behavior!** The sorted array contents DID change (the order is different), so the components SHOULD re-render.

Let's verify this is working correctly by testing a scenario where the array contents truly don't change.

```tsx
// Add a new state that doesn't affect products
export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  const [theme, setTheme] = useState<'light' | 'dark'>('light'); // ← New state
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className={`dashboard theme-${theme}`}>
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
        
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Theme: {theme}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList products={sortedProducts} />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
    </div>
  );
}
```

**User Action**: Click the theme button.

```bash
# Browser Console Output:

🎨 Rendering ProductDashboard
```

**Expected vs. Actual Improvement**:
- Parent re-renders (expected - state changed)
- No child components re-render (expected - their props didn't change)
- No expensive calculations run (expected - dependencies didn't change)
- Performance: ~5ms (just React reconciliation)

**React DevTools - Profiler**:
- `ProductDashboard` rendered
- `ProductList`, `SalesChart`, `StatsSummary` all show "Did not render"
- Total render time: ~5ms

**This is the power of React.memo + useMemo working together**:
1. `useMemo` preserves array references when dependencies don't change
2. `React.memo` skips re-rendering when props (array references) don't change
3. Result: Changing unrelated state doesn't trigger expensive child re-renders

### When to Apply React.memo and useMemo

**React.memo**:

**Use when**:
- Component is expensive to render (complex UI, many elements)
- Component receives the same props frequently
- Component is rendered often due to parent re-renders
- Props are primitive values or stable references

**Don't use when**:
- Component is cheap to render (simple UI)
- Props change on every render anyway
- Component rarely re-renders
- You're wrapping every component "just in case" (premature optimization)

**useMemo**:

**Use when**:
- Calculation is expensive (loops, complex math, data transformations)
- Result is used as a prop for a memoized child component
- Result is used in a dependency array of another hook
- You've measured and confirmed it's a bottleneck

**Don't use when**:
- Calculation is cheap (simple arithmetic, single array access)
- Result changes on every render anyway
- You're memoizing everything "just in case" (premature optimization)
- The memoization overhead exceeds the calculation cost

### Common Failure Modes and Their Signatures

#### Symptom: React.memo doesn't prevent re-renders

**Browser behavior**:
Memoized component still re-renders on every parent render

**Console pattern**:
```bash
🎨 Rendering MemoizedComponent
🎨 Rendering MemoizedComponent
🎨 Rendering MemoizedComponent
```

**DevTools clues**:
- Component wrapped with `memo` in Components tab
- Props show different object/array references each render
- Reason: "Props changed"

**Root cause**: Props contain new object/array references on each render

**Solution**: 
- Move static data outside component
- Use `useMemo` to preserve references
- Use `useCallback` for function props (next section)

#### Symptom: useMemo dependencies cause infinite loops

**Browser behavior**:
Browser freezes, tab becomes unresponsive

**Console pattern**:
```bash
🔄 Calculating value...
🔄 Calculating value...
🔄 Calculating value...
[Repeats infinitely]
```

**DevTools clues**:
- React DevTools shows component rendered 1000+ times
- Profiler shows continuous rendering
- Main thread blocked

**Root cause**: Memoized value is used in its own dependency array, or dependency is an object that changes every render

**Solution**:
```tsx
// ❌ WRONG: Creates infinite loop
const value = useMemo(() => {
  return { data: expensiveCalc() };
}, [value]); // value depends on itself!

// ✓ CORRECT: Stable dependencies
const value = useMemo(() => {
  return { data: expensiveCalc(input) };
}, [input]);
```

#### Symptom: Stale values in memoized calculations

**Browser behavior**:
UI shows outdated data, doesn't update when it should

**Console pattern**:
```bash
🎨 Rendering Component
[No "Calculating..." message when expected]
```

**DevTools clues**:
- State/props show updated values
- Memoized value shows old value
- Dependency array missing required dependencies

**Root cause**: Missing dependencies in `useMemo` array

**Solution**: Include ALL values from component scope that the calculation uses. Enable ESLint rule `react-hooks/exhaustive-deps`.

### Performance Impact Summary

**Before optimization**:
- Typing in search: 250ms per keystroke
- Clicking sort: 110ms
- Changing theme: 250ms

**After optimization**:
- Typing in search: 130ms per keystroke (48% faster)
- Clicking sort: 10ms (91% faster)
- Changing theme: 5ms (98% faster)

**Trade-offs**:
- Code complexity: Moderate increase (memoization logic)
- Memory usage: Slight increase (cached values)
- Maintenance: Must keep dependency arrays correct
- Bundle size: No change (React.memo and useMemo are built-in)

**Limitation preview**: We've optimized component re-renders and expensive calculations, but we're still creating new function references on every render. When we pass functions as props to memoized components, they'll re-render unnecessarily. Let's address that next with `useCallback`.

## useCallback: probably not as often as you think

## The Failure: Memoized Components Re-render Due to Function Props

Let's add interactive features to our dashboard: users can click products to view details, and delete products from the list.

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, static data unchanged)

interface ProductListProps {
  products: Product[];
  onProductClick: (productId: string) => void;
  onProductDelete: (productId: string) => void;
}

const ProductList = memo(function ProductList({ 
  products, 
  onProductClick,
  onProductDelete 
}: ProductListProps) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      <h3>Products ({products.length})</h3>
      {products.map(product => (
        <div 
          key={product.id} 
          className="product-card"
          onClick={() => onProductClick(product.id)}
        >
          <h4>{product.name}</h4>
          <p>Category: {product.category}</p>
          <p>Sales: {product.sales} units</p>
          <p>Revenue: ${product.revenue.toLocaleString()}</p>
          <button 
            onClick={(e) => {
              e.stopPropagation();
              onProductDelete(product.id);
            }}
          >
            Delete
          </button>
        </div>
      ))}
    </div>
  );
});

// ... (SalesChart, StatsSummary unchanged)

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [selectedProductId, setSelectedProductId] = useState<string | null>(null);
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  // Event handlers defined inline
  const handleProductClick = (productId: string) => {
    console.log('📍 Product clicked:', productId);
    setSelectedProductId(productId);
  };
  
  const handleProductDelete = (productId: string) => {
    console.log('🗑️ Product deleted:', productId);
    // In real app, would call API to delete
  };
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className={`dashboard theme-${theme}`}>
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
        
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Theme: {theme}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList 
          products={sortedProducts}
          onProductClick={handleProductClick}
          onProductDelete={handleProductDelete}
        />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
      
      {selectedProductId && (
        <div className="product-detail">
          Selected: {selectedProductId}
        </div>
      )}
    </div>
  );
}
```

**User Action**: Click the theme button (which doesn't affect products at all).

```bash
# Browser Console Output:

🎨 Rendering ProductDashboard
🎨 Rendering ProductList
```

### Diagnostic Analysis: Function References Break Memoization

**Browser Behavior**:
- Clicking theme button causes `ProductList` to re-render
- Even though `products` array didn't change
- Performance degraded back to pre-optimization levels

**React DevTools - Components Tab**:
- `ProductList` props:
  - `products`: Array(5) [same reference as before]
  - `onProductClick`: function [different reference]
  - `onProductDelete`: function [different reference]
- Reason for re-render: "Props changed"
- Specifically: `onProductClick` and `onProductDelete` changed

**React DevTools - Profiler**:
- Theme button click took ~80ms (vs. 5ms before adding handlers)
- `ProductList` rendered unnecessarily
- All product cards re-rendered

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Instant theme change (5ms)
   - Actual: Noticeable lag (80ms)

2. **What the console reveals**:
   - `ProductList` renders even though products didn't change
   - The memoization we added earlier is now broken

3. **What DevTools shows**:
   - `products` prop has the same reference (good)
   - `onProductClick` and `onProductDelete` have different references (bad)
   - React.memo sees different function references and considers props "changed"

4. **Root cause identified**:
   - Every render creates new function instances for `handleProductClick` and `handleProductDelete`
   - Even though the function logic is identical, JavaScript creates new function objects
   - `React.memo` compares by reference, sees different functions, triggers re-render

5. **Why the current approach can't solve this**:
   - Functions defined in component body are recreated on every render
   - This is JavaScript's default behavior, not a React issue
   - We need to preserve function references across renders

6. **What we need**:
   - A way to create a function once and reuse the same reference
   - Only create a new function when its dependencies actually change
   - This is what `useCallback` provides

## useCallback: Memoizing Function References

`useCallback` is like `useMemo`, but specifically for functions. It tells React: "Only create a new function if dependencies change."

### How useCallback Works

```typescript
const memoizedCallback = useCallback(
  (arg) => {
    // Function logic
    doSomething(arg, dependency);
  },
  [dependency]
);
```

React will:
1. Create the function on first render
2. Cache the function reference
3. On subsequent renders, check if dependencies changed
4. If dependencies unchanged, return the same function reference
5. If dependencies changed, create a new function and cache it

### useCallback vs. useMemo for Functions

These are equivalent:

```typescript
// Using useCallback
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);

// Using useMemo (more verbose)
const handleClick = useMemo(() => {
  return () => {
    console.log('clicked');
  };
}, []);
```

`useCallback` is syntactic sugar for `useMemo` that returns a function.

### Iteration 1: Memoizing Event Handlers

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo, useCallback } from 'react';

// ... (interfaces, static data, child components unchanged)

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [selectedProductId, setSelectedProductId] = useState<string | null>(null);
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  // Memoize event handlers
  const handleProductClick = useCallback((productId: string) => {
    console.log('📍 Product clicked:', productId);
    setSelectedProductId(productId);
  }, []); // No dependencies - function logic doesn't depend on any props/state
  
  const handleProductDelete = useCallback((productId: string) => {
    console.log('🗑️ Product deleted:', productId);
    // In real app, would call API to delete
  }, []); // No dependencies
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className={`dashboard theme-${theme}`}>
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
        
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Theme: {theme}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList 
          products={sortedProducts}
          onProductClick={handleProductClick}
          onProductDelete={handleProductDelete}
        />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
      
      {selectedProductId && (
        <div className="product-detail">
          Selected: {selectedProductId}
        </div>
      )}
    </div>
  );
}
```

**User Action**: Click the theme button.

```bash
# Browser Console Output:

🎨 Rendering ProductDashboard
```

**Expected vs. Actual Improvement**:
- Parent re-renders (expected - state changed)
- `ProductList` does NOT re-render (expected - props unchanged)
- Performance: ~5ms (back to optimized level)

**React DevTools - Components Tab**:
- `ProductList` props:
  - `products`: Array(5) [same reference]
  - `onProductClick`: function [same reference]
  - `onProductDelete`: function [same reference]
- Result: "Did not render"

**React DevTools - Profiler**:
- Theme button click: 5ms (vs. 80ms before useCallback)
- `ProductList` shows "Did not render"
- Improvement: 94% faster

### The Failure: useCallback with Stale Closures

Let's add a feature: when deleting a product, show a confirmation with the product name.

```tsx
// src/components/ProductDashboard.tsx (updated)
export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [selectedProductId, setSelectedProductId] = useState<string | null>(null);
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  const handleProductClick = useCallback((productId: string) => {
    console.log('📍 Product clicked:', productId);
    setSelectedProductId(productId);
  }, []);
  
  // ❌ WRONG: Missing dependency
  const handleProductDelete = useCallback((productId: string) => {
    const product = sortedProducts.find(p => p.id === productId);
    if (product && window.confirm(`Delete ${product.name}?`)) {
      console.log('🗑️ Product deleted:', productId);
    }
  }, []); // Missing sortedProducts dependency!
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className={`dashboard theme-${theme}`}>
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
        
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Theme: {theme}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList 
          products={sortedProducts}
          onProductClick={handleProductClick}
          onProductDelete={handleProductDelete}
        />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
      
      {selectedProductId && (
        <div className="product-detail">
          Selected: {selectedProductId}
        </div>
      )}
    </div>
  );
}
```

**User Action**: 
1. Load the dashboard (shows all products)
2. Type "Laptop" in search (filters to one product)
3. Click delete on the Laptop product

**Browser Behavior**:
- Confirmation dialog shows: "Delete undefined?"
- Product name is missing from the confirmation

**Browser Console**:

```bash
🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...

[User types "Laptop"]
🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...

[User clicks delete]
🗑️ Product deleted: 1
```

### Diagnostic Analysis: Stale Closure in useCallback

**React DevTools - Components Tab**:
- `handleProductDelete` function created on first render
- At that time, `sortedProducts` contained all 5 products
- Function "closed over" that initial array
- When user filters to "Laptop", `sortedProducts` changes
- But `handleProductDelete` still references the old array (empty dependency array)
- When function runs, it searches the old array, finds nothing

**The Problem**: This is the classic "stale closure" issue. The function captured `sortedProducts` from the first render and never updated.

**Why it happens**:
```typescript
// First render: sortedProducts = [all 5 products]
const handleProductDelete = useCallback((productId: string) => {
  const product = sortedProducts.find(p => p.id === productId);
  // sortedProducts here refers to the array from first render
}, []); // Empty array means "never recreate this function"

// Second render: sortedProducts = [only Laptop]
// But handleProductDelete still uses the old sortedProducts!
```

### Iteration 2: Fixing the Stale Closure

```tsx
// src/components/ProductDashboard.tsx (updated)
export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [selectedProductId, setSelectedProductId] = useState<string | null>(null);
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  const handleProductClick = useCallback((productId: string) => {
    console.log('📍 Product clicked:', productId);
    setSelectedProductId(productId);
  }, []);
  
  // ✓ CORRECT: Include sortedProducts in dependencies
  const handleProductDelete = useCallback((productId: string) => {
    const product = sortedProducts.find(p => p.id === productId);
    if (product && window.confirm(`Delete ${product.name}?`)) {
      console.log('🗑️ Product deleted:', productId);
    }
  }, [sortedProducts]); // Now function updates when sortedProducts changes
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className={`dashboard theme-${theme}`}>
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
        
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Theme: {theme}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList 
          products={sortedProducts}
          onProductClick={handleProductClick}
          onProductDelete={handleProductDelete}
        />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
      
      {selectedProductId && (
        <div className="product-detail">
          Selected: {selectedProductId}
        </div>
      )}
    </div>
  );
}
```

**User Action**: Same as before - filter to "Laptop", then click delete.

**Browser Behavior**:
- Confirmation dialog shows: "Delete Laptop Pro?"
- Product name appears correctly

**But wait...** Now we have a new problem!

```bash
# Browser Console Output (typing "Laptop"):

🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...
🎨 Rendering ProductList

🎨 Rendering ProductDashboard
🔄 Filtering products...
🔄 Sorting products...
🎨 Rendering ProductList
```

### The Trade-off: Correctness vs. Performance

**React DevTools - Profiler**:
- Typing in search: Each keystroke causes `ProductList` to re-render
- Reason: `handleProductDelete` gets a new reference when `sortedProducts` changes
- Performance: Back to ~130ms per keystroke

**The Dilemma**:
- Without `sortedProducts` in dependencies: Function has stale data (bug)
- With `sortedProducts` in dependencies: Function recreates on every filter/sort, breaking memoization

**This is the fundamental trade-off with useCallback**: You must choose between correctness and performance. Correctness always wins.

### When useCallback Actually Helps

`useCallback` is only beneficial when:
1. The function is passed to a memoized child component
2. The function's dependencies are stable (don't change often)
3. The child component is expensive to render

In our case:
- ✓ Function passed to memoized `ProductList`
- ✗ Dependencies change frequently (`sortedProducts` changes on every filter/sort)
- ✓ `ProductList` is moderately expensive

**Result**: `useCallback` helps for theme changes, but not for filter/sort operations.

### Iteration 3: Optimizing with Stable References

One solution: Pass the product ID only, and let the child component handle the lookup.

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo, useCallback } from 'react';

// ... (interfaces, static data unchanged)

interface ProductListProps {
  products: Product[];
  onProductClick: (productId: string) => void;
  onProductDelete: (productId: string, productName: string) => void; // ← Changed
}

const ProductList = memo(function ProductList({ 
  products, 
  onProductClick,
  onProductDelete 
}: ProductListProps) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      <h3>Products ({products.length})</h3>
      {products.map(product => (
        <div 
          key={product.id} 
          className="product-card"
          onClick={() => onProductClick(product.id)}
        >
          <h4>{product.name}</h4>
          <p>Category: {product.category}</p>
          <p>Sales: {product.sales} units</p>
          <p>Revenue: ${product.revenue.toLocaleString()}</p>
          <button 
            onClick={(e) => {
              e.stopPropagation();
              onProductDelete(product.id, product.name); // ← Pass name directly
            }}
          >
            Delete
          </button>
        </div>
      ))}
    </div>
  );
});

// ... (SalesChart, StatsSummary unchanged)

export function ProductDashboard() {
  const [category, setCategory] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc');
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [selectedProductId, setSelectedProductId] = useState<string | null>(null);
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(ALL_PRODUCTS, category, searchTerm);
  }, [category, searchTerm]);
  
  const sortedProducts = useMemo(() => {
    console.log('🔄 Sorting products...');
    return [...filteredProducts].sort((a, b) => {
      return sortOrder === 'desc' ? b.revenue - a.revenue : a.revenue - b.revenue;
    });
  }, [filteredProducts, sortOrder]);
  
  const handleProductClick = useCallback((productId: string) => {
    console.log('📍 Product clicked:', productId);
    setSelectedProductId(productId);
  }, []);
  
  // Now function doesn't depend on sortedProducts
  const handleProductDelete = useCallback((productId: string, productName: string) => {
    if (window.confirm(`Delete ${productName}?`)) {
      console.log('🗑️ Product deleted:', productId);
    }
  }, []); // No dependencies - stable reference
  
  console.log('🎨 Rendering ProductDashboard');
  
  return (
    <div className={`dashboard theme-${theme}`}>
      <h1>Product Analytics Dashboard</h1>
      
      <div className="controls">
        <input
          type="text"
          placeholder="Search products..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
        />
        
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="furniture">Furniture</option>
          <option value="appliances">Appliances</option>
        </select>
        
        <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
          Sort: {sortOrder === 'asc' ? '↑' : '↓'}
        </button>
        
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Theme: {theme}
        </button>
      </div>
      
      <div className="dashboard-grid">
        <ProductList 
          products={sortedProducts}
          onProductClick={handleProductClick}
          onProductDelete={handleProductDelete}
        />
        <SalesChart data={SALES_DATA} />
        <StatsSummary products={sortedProducts} />
      </div>
      
      {selectedProductId && (
        <div className="product-detail">
          Selected: {selectedProductId}
        </div>
      )}
    </div>
  );
}
```

**User Action**: Click theme button.

```bash
# Browser Console Output:

🎨 Rendering ProductDashboard
```

**Expected vs. Actual Improvement**:
- Theme change: 5ms (ProductList doesn't re-render)
- Filter/sort: ProductList re-renders (expected - products changed)
- Delete confirmation: Shows correct product name
- No stale closure issues

**This is the pattern**: When possible, design your component APIs to avoid dependencies in callbacks.

## When NOT to Use useCallback

### Anti-pattern 1: Wrapping Every Function

```tsx
// ❌ WRONG: Unnecessary useCallback
function MyComponent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  // This button is not in a memoized child component
  return <button onClick={handleClick}>Click me</button>;
}

// ✓ CORRECT: No useCallback needed
function MyComponent() {
  const handleClick = () => {
    console.log('clicked');
  };
  
  return <button onClick={handleClick}>Click me</button>;
}
```

**Why**: Creating a new function on each render is cheap. `useCallback` adds overhead (dependency comparison, cache lookup). Only use it when passing to memoized children.

### Anti-pattern 2: useCallback with Unstable Dependencies

```tsx
// ❌ WRONG: Dependencies change every render anyway
function MyComponent({ items }: { items: Item[] }) {
  const handleClick = useCallback(() => {
    console.log(items.length);
  }, [items]); // items is a new array every render
  
  return <MemoizedChild onClick={handleClick} />;
}

// ✓ BETTER: Accept that memoization won't help here
function MyComponent({ items }: { items: Item[] }) {
  const handleClick = () => {
    console.log(items.length);
  };
  
  return <MemoizedChild onClick={handleClick} />;
}

// ✓ BEST: If items is expensive to render, memoize items instead
function MyComponent({ items }: { items: Item[] }) {
  const memoizedItems = useMemo(() => items, [items]);
  
  const handleClick = useCallback(() => {
    console.log(memoizedItems.length);
  }, [memoizedItems]);
  
  return <MemoizedChild onClick={handleClick} />;
}
```

### Anti-pattern 3: Premature Optimization

```tsx
// ❌ WRONG: Optimizing before measuring
function MyComponent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  const handleChange = useCallback((e) => {
    console.log(e.target.value);
  }, []);
  
  const handleSubmit = useCallback(() => {
    console.log('submitted');
  }, []);
  
  // None of these are passed to memoized children!
  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
}

// ✓ CORRECT: No optimization until proven necessary
function MyComponent() {
  const handleClick = () => console.log('clicked');
  const handleChange = (e) => console.log(e.target.value);
  const handleSubmit = () => console.log('submitted');
  
  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
}
```

### When to Apply useCallback

**Use when**:
- Function is passed as a prop to a `React.memo` component
- Function is used in a dependency array of another hook
- Function is expensive to create (rare)
- You've measured and confirmed it's a bottleneck

**Don't use when**:
- Function is only used in JSX event handlers
- Function dependencies change frequently anyway
- Component is not memoized
- You're "just being safe" without measuring

### Common Failure Modes and Their Signatures

#### Symptom: useCallback doesn't prevent re-renders

**Browser behavior**:
Memoized child component still re-renders when parent renders

**Console pattern**:
```bash
🎨 Rendering Parent
🎨 Rendering MemoizedChild
🎨 Rendering Parent
🎨 Rendering MemoizedChild
```

**DevTools clues**:
- Child component wrapped with `memo`
- Function prop wrapped with `useCallback`
- But child still re-renders
- Reason: "Props changed"

**Root cause**: Function dependencies change frequently, creating new function references

**Solution**: 
- Redesign component API to avoid dependencies
- Accept that memoization won't help in this case
- Consider if the child component is actually expensive enough to warrant optimization

#### Symptom: Stale values in callback

**Browser behavior**:
Function uses outdated state/props values

**Console pattern**:
```bash
📍 Expected value: 5
📍 Actual value: 0
```

**DevTools clues**:
- State/props show current values
- Function behavior uses old values
- Empty or incomplete dependency array

**Root cause**: Missing dependencies in `useCallback` array

**Solution**: Include ALL values from component scope that the function uses. Enable ESLint rule `react-hooks/exhaustive-deps`.

#### Symptom: Infinite loop with useCallback

**Browser behavior**:
Browser freezes, tab becomes unresponsive

**Console pattern**:
```bash
🎨 Rendering Component
🎨 Rendering Component
🎨 Rendering Component
[Repeats infinitely]
```

**DevTools clues**:
- Component rendered 1000+ times
- Profiler shows continuous rendering
- Callback in dependency array of useEffect

**Root cause**: Callback recreates on every render, triggering effect, which updates state, causing re-render

**Solution**:
```tsx
// ❌ WRONG: Creates infinite loop
useEffect(() => {
  callback();
}, [callback]); // callback recreates every render

// ✓ CORRECT: Stable callback
const callback = useCallback(() => {
  // logic
}, []); // Empty dependencies if possible

useEffect(() => {
  callback();
}, [callback]);
```

### Performance Impact Summary

**useCallback benefits**:
- Prevents re-renders of memoized children when function props don't change
- Enables stable references for hook dependencies
- Minimal memory overhead (one cached function reference)

**useCallback costs**:
- Code complexity (dependency arrays to maintain)
- Slight performance overhead (dependency comparison on each render)
- Risk of stale closures if dependencies incorrect

**When it matters**:
- Large lists with memoized items
- Expensive child components
- Functions passed through multiple component layers

**When it doesn't matter**:
- Simple event handlers
- Functions not passed to memoized components
- Functions with frequently changing dependencies

**Limitation preview**: We've optimized component re-renders and function references, but our entire application bundle loads upfront. Users download code for features they might never use. Let's address that with code splitting.

## Code splitting strategies

## The Failure: Everything Loads Upfront

Our dashboard has grown. We've added:
- A complex data visualization library (Recharts)
- A rich text editor for product descriptions
- A PDF export feature
- An admin panel for advanced settings

Most users never use these features, but they download all the code anyway.

Let's measure the impact.

```bash
# Build the application
npm run build

# Output:
dist/assets/index-a1b2c3d4.js    847.23 kB │ gzip: 284.15 kB
dist/assets/vendor-e5f6g7h8.js   1,234.56 kB │ gzip: 412.34 kB

Total bundle size: 2,081.79 kB (696.49 kB gzipped)
```

### Diagnostic Analysis: Bundle Size Impact

**Network Tab**:
- Initial page load downloads 2.08 MB of JavaScript
- On 3G connection: ~8 seconds to download
- On 4G connection: ~2 seconds to download
- Parse and execute time: ~1.5 seconds on mid-range device

**Lighthouse Report**:
- Performance score: 62/100
- Time to Interactive (TTI): 4.2s
- Total Blocking Time (TBT): 890ms
- First Contentful Paint (FCP): 1.8s

**Bundle Analyzer** (using `rollup-plugin-visualizer`):

```bash
# Install bundle analyzer
npm install --save-dev rollup-plugin-visualizer

# Add to vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true })
  ]
});

# Build and analyze
npm run build
```

**Bundle Analyzer Output**:
- `recharts`: 412 kB (20% of bundle)
- `react-quill` (rich text editor): 287 kB (14% of bundle)
- `jspdf` (PDF export): 198 kB (10% of bundle)
- Admin panel components: 156 kB (8% of bundle)
- Core dashboard: 1,028 kB (48% of bundle)

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Fast initial load
   - Actual: 4+ second wait before interactive

2. **What the metrics reveal**:
   - 52% of the bundle is features most users never use
   - Users pay the cost (download, parse, execute) upfront
   - Mobile users on slow connections suffer most

3. **What the bundle analyzer shows**:
   - Large third-party libraries loaded immediately
   - Admin features loaded for all users (even non-admins)
   - Visualization library loaded even if user never views charts

4. **Root cause identified**:
   - All imports are static (`import X from 'y'`)
   - Webpack/Vite bundles everything into initial chunk
   - No code splitting strategy

5. **Why the current approach can't solve this**:
   - Static imports are resolved at build time
   - All imported code goes into the main bundle
   - No way to defer loading until needed

6. **What we need**:
   - Load core features immediately
   - Defer optional features until user needs them
   - Split code into smaller chunks that load on demand

## React.lazy and Suspense: Route-Based Code Splitting

The most effective code splitting strategy: split by route. Users only download code for the pages they visit.

### How React.lazy Works

```typescript
// Static import - loads immediately
import AdminPanel from './AdminPanel';

// Dynamic import - loads on demand
const AdminPanel = React.lazy(() => import('./AdminPanel'));
```

`React.lazy` takes a function that returns a dynamic `import()`. This creates a separate bundle chunk that loads only when the component is rendered.

### Iteration 1: Splitting the Admin Panel

```tsx
// src/App.tsx (before)
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';
import { ProductDashboard } from './components/ProductDashboard';
import { AdminPanel } from './components/AdminPanel';
import { Analytics } from './components/Analytics';

export function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Dashboard</Link>
        <Link to="/analytics">Analytics</Link>
        <Link to="/admin">Admin</Link>
      </nav>
      
      <Routes>
        <Route path="/" element={<ProductDashboard />} />
        <Route path="/analytics" element={<Analytics />} />
        <Route path="/admin" element={<AdminPanel />} />
      </Routes>
    </BrowserRouter>
  );
}
```

```tsx
// src/App.tsx (after - with code splitting)
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';
import { ProductDashboard } from './components/ProductDashboard';

// Lazy load heavy components
const Analytics = lazy(() => import('./components/Analytics'));
const AdminPanel = lazy(() => import('./components/AdminPanel'));

function LoadingFallback() {
  return (
    <div className="loading-container">
      <div className="spinner" />
      <p>Loading...</p>
    </div>
  );
}

export function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Dashboard</Link>
        <Link to="/analytics">Analytics</Link>
        <Link to="/admin">Admin</Link>
      </nav>
      
      <Suspense fallback={<LoadingFallback />}>
        <Routes>
          <Route path="/" element={<ProductDashboard />} />
          <Route path="/analytics" element={<Analytics />} />
          <Route path="/admin" element={<AdminPanel />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**Important**: Components loaded with `React.lazy` must be default exports.

```tsx
// src/components/AdminPanel.tsx
// ❌ WRONG: Named export
export function AdminPanel() {
  return <div>Admin Panel</div>;
}

// ✓ CORRECT: Default export
export default function AdminPanel() {
  return <div>Admin Panel</div>;
}

// ✓ ALSO CORRECT: Named export with default
export function AdminPanel() {
  return <div>Admin Panel</div>;
}
export default AdminPanel;
```

**Build the application again**:

```bash
# Build with code splitting
npm run build

# Output:
dist/assets/index-a1b2c3d4.js           487.23 kB │ gzip: 162.15 kB
dist/assets/Analytics-b2c3d4e5.js       412.45 kB │ gzip: 138.22 kB
dist/assets/AdminPanel-c3d4e5f6.js      156.78 kB │ gzip: 52.34 kB
dist/assets/vendor-e5f6g7h8.js          1,025.33 kB │ gzip: 342.12 kB

Initial bundle: 1,512.56 kB (504.27 kB gzipped)
Lazy chunks: 569.23 kB (190.56 kB gzipped)
```

**Expected vs. Actual Improvement**:

**Initial load**:
- Before: 2,081.79 kB (696.49 kB gzipped)
- After: 1,512.56 kB (504.27 kB gzipped)
- Improvement: 27% smaller initial bundle

**Network Tab**:
- Initial page load: 1.51 MB (vs. 2.08 MB before)
- On 3G: ~6 seconds (vs. 8 seconds before)
- On 4G: ~1.5 seconds (vs. 2 seconds before)

**Lighthouse Report**:
- Performance score: 78/100 (vs. 62/100 before)
- TTI: 2.8s (vs. 4.2s before)
- TBT: 420ms (vs. 890ms before)
- FCP: 1.2s (vs. 1.8s before)

**User Experience**:
1. User loads homepage: Downloads 1.51 MB
2. User clicks "Analytics": Downloads additional 412 kB
3. User clicks "Admin": Downloads additional 157 kB

**React DevTools - Network Tab**:
- Initial load: `index.js`, `vendor.js`
- Navigate to /analytics: `Analytics.js` loads
- Navigate to /admin: `AdminPanel.js` loads
- Navigate back to /: No additional downloads (already cached)

### Suspense: Handling Loading States

`Suspense` is React's way of handling asynchronous component loading. It shows a fallback UI while the lazy component loads.

**How Suspense Works**:
1. React starts rendering the lazy component
2. Component is not loaded yet (dynamic import in progress)
3. React "suspends" rendering and shows the fallback
4. When import completes, React resumes rendering the actual component

**Suspense Boundaries**:
You can have multiple Suspense boundaries at different levels:

```tsx
// Coarse-grained: One boundary for entire route
<Suspense fallback={<PageLoader />}>
  <Routes>
    <Route path="/analytics" element={<Analytics />} />
    <Route path="/admin" element={<AdminPanel />} />
  </Routes>
</Suspense>

// Fine-grained: Separate boundaries for each route
<Routes>
  <Route 
    path="/analytics" 
    element={
      <Suspense fallback={<AnalyticsLoader />}>
        <Analytics />
      </Suspense>
    } 
  />
  <Route 
    path="/admin" 
    element={
      <Suspense fallback={<AdminLoader />}>
        <AdminPanel />
      </Suspense>
    } 
  />
</Routes>
```

### Iteration 2: Component-Level Code Splitting

Not just routes—split heavy components within a page.

```tsx
// src/components/ProductDashboard.tsx (before)
import { AdvancedChart } from './AdvancedChart'; // Heavy charting library
import { RichTextEditor } from './RichTextEditor'; // Heavy editor
import { PDFExporter } from './PDFExporter'; // Heavy PDF library

export function ProductDashboard() {
  const [showChart, setShowChart] = useState(false);
  const [showEditor, setShowEditor] = useState(false);
  
  return (
    <div>
      <h1>Dashboard</h1>
      
      <button onClick={() => setShowChart(true)}>
        Show Advanced Chart
      </button>
      
      {showChart && <AdvancedChart data={data} />}
      
      <button onClick={() => setShowEditor(true)}>
        Edit Description
      </button>
      
      {showEditor && <RichTextEditor />}
      
      <PDFExporter data={data} />
    </div>
  );
}
```

```tsx
// src/components/ProductDashboard.tsx (after - with code splitting)
import { lazy, Suspense, useState } from 'react';

// Lazy load heavy components
const AdvancedChart = lazy(() => import('./AdvancedChart'));
const RichTextEditor = lazy(() => import('./RichTextEditor'));
const PDFExporter = lazy(() => import('./PDFExporter'));

export function ProductDashboard() {
  const [showChart, setShowChart] = useState(false);
  const [showEditor, setShowEditor] = useState(false);
  
  return (
    <div>
      <h1>Dashboard</h1>
      
      <button onClick={() => setShowChart(true)}>
        Show Advanced Chart
      </button>
      
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <AdvancedChart data={data} />
        </Suspense>
      )}
      
      <button onClick={() => setShowEditor(true)}>
        Edit Description
      </button>
      
      {showEditor && (
        <Suspense fallback={<div>Loading editor...</div>}>
          <RichTextEditor />
        </Suspense>
      )}
      
      <Suspense fallback={<div>Loading PDF tools...</div>}>
        <PDFExporter data={data} />
      </Suspense>
    </div>
  );
}
```

**User Experience**:
1. User loads dashboard: Core UI appears immediately
2. User clicks "Show Advanced Chart": Brief loading indicator, then chart appears
3. Chart code (412 kB) only downloads when needed

### Iteration 3: Prefetching for Better UX

The problem with lazy loading: users see loading spinners. Solution: prefetch likely-needed code.

```tsx
// src/components/ProductDashboard.tsx (with prefetching)
import { lazy, Suspense, useState, useEffect } from 'react';

const AdvancedChart = lazy(() => import('./AdvancedChart'));
const RichTextEditor = lazy(() => import('./RichTextEditor'));

export function ProductDashboard() {
  const [showChart, setShowChart] = useState(false);
  const [showEditor, setShowEditor] = useState(false);
  
  // Prefetch chart when user hovers over button
  const prefetchChart = () => {
    import('./AdvancedChart');
  };
  
  // Prefetch editor when user focuses on button
  const prefetchEditor = () => {
    import('./RichTextEditor');
  };
  
  // Prefetch on idle (when browser has free time)
  useEffect(() => {
    if ('requestIdleCallback' in window) {
      requestIdleCallback(() => {
        import('./AdvancedChart');
        import('./RichTextEditor');
      });
    }
  }, []);
  
  return (
    <div>
      <h1>Dashboard</h1>
      
      <button 
        onClick={() => setShowChart(true)}
        onMouseEnter={prefetchChart}
        onFocus={prefetchChart}
      >
        Show Advanced Chart
      </button>
      
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <AdvancedChart data={data} />
        </Suspense>
      )}
      
      <button 
        onClick={() => setShowEditor(true)}
        onMouseEnter={prefetchEditor}
        onFocus={prefetchEditor}
      >
        Edit Description
      </button>
      
      {showEditor && (
        <Suspense fallback={<div>Loading editor...</div>}>
          <RichTextEditor />
        </Suspense>
      )}
    </div>
  );
}
```

**Prefetching Strategies**:

1. **On hover/focus**: Start loading when user shows intent
2. **On idle**: Load during browser idle time
3. **On route change**: Prefetch next likely route
4. **On viewport**: Load when element enters viewport

### Iteration 4: Vendor Code Splitting

Split large third-party libraries into separate chunks.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Split React and React DOM into separate chunk
          'react-vendor': ['react', 'react-dom'],
          
          // Split React Router into separate chunk
          'router': ['react-router-dom'],
          
          // Split large UI libraries
          'ui-vendor': ['recharts', 'react-quill'],
          
          // Split utility libraries
          'utils': ['date-fns', 'lodash-es'],
        },
      },
    },
  },
});
```

**Build output with vendor splitting**:

```bash
npm run build

# Output:
dist/assets/index-a1b2c3d4.js           287.23 kB │ gzip: 95.15 kB
dist/assets/react-vendor-b2c3d4e5.js    142.45 kB │ gzip: 47.22 kB
dist/assets/router-c3d4e5f6.js          45.78 kB │ gzip: 15.34 kB
dist/assets/ui-vendor-d4e5f6g7.js       699.12 kB │ gzip: 233.45 kB
dist/assets/utils-e5f6g7h8.js           87.34 kB │ gzip: 29.12 kB
dist/assets/Analytics-f6g7h8i9.js       412.45 kB │ gzip: 138.22 kB
dist/assets/AdminPanel-g7h8i9j0.js      156.78 kB │ gzip: 52.34 kB
```

**Benefits of vendor splitting**:
- React vendor chunk (142 kB) cached across all pages
- UI vendor chunk (699 kB) only loads when needed
- When you update your code, users don't re-download React
- Better long-term caching

### The Failure: Over-Splitting

Let's split EVERYTHING into tiny chunks.

```typescript
// vite.config.ts (over-aggressive splitting)
export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // Split every node_module into its own chunk
          if (id.includes('node_modules')) {
            return id.split('node_modules/')[1].split('/')[0];
          }
          
          // Split every component into its own chunk
          if (id.includes('src/components')) {
            return id.split('components/')[1].split('.')[0];
          }
        },
      },
    },
  },
});
```

```bash
npm run build

# Output: 147 separate chunk files!
dist/assets/Button-a1b2c3d4.js          2.34 kB
dist/assets/Input-b2c3d4e5.js           3.12 kB
dist/assets/Modal-c3d4e5f6.js           4.56 kB
dist/assets/Card-d4e5f6g7.js            2.89 kB
[... 143 more files ...]
```

**Network Tab**:
- Initial page load: 47 separate HTTP requests
- Each request has overhead (DNS, TCP, TLS handshake)
- Total download time: 3.2 seconds (vs. 1.5 seconds with reasonable splitting)
- HTTP/2 helps, but too many requests still hurts

**The Problem**: Over-splitting creates more overhead than it saves.

**The Rule**: Aim for 5-10 chunks for most applications. Each chunk should be at least 20-30 kB.

### When to Apply Code Splitting

**Route-based splitting** (always do this):
- Split each major route into its own chunk
- Users only download code for pages they visit
- Biggest performance win for least effort

**Component-based splitting** (selective):
- Split heavy components (charts, editors, maps)
- Split components behind user actions (modals, tabs)
- Split components below the fold

**Vendor splitting** (for production):
- Split React/React DOM (changes rarely)
- Split large UI libraries (recharts, material-ui)
- Split utility libraries (lodash, date-fns)

**Don't split**:
- Small components (<10 kB)
- Components used on every page
- Components above the fold
- Core application logic

### Common Failure Modes and Their Signatures

#### Symptom: Suspense fallback flashes briefly

**Browser behavior**:
Loading spinner appears for <100ms, then content appears

**Console pattern**:
```bash
[Suspense] Fallback shown
[Suspense] Content rendered
[Time elapsed: 45ms]
```

**DevTools clues**:
- Network tab shows chunk loaded from cache
- Very fast load time (<100ms)

**Root cause**: Component already cached, but Suspense still shows fallback

**Solution**: Use `startTransition` to avoid showing fallback for fast loads:

```tsx
import { lazy, Suspense, useState, useTransition } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  const [show, setShow] = useState(false);
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    startTransition(() => {
      setShow(true);
    });
  };
  
  return (
    <div>
      <button onClick={handleClick} disabled={isPending}>
        {isPending ? 'Loading...' : 'Show Component'}
      </button>
      
      {show && (
        <Suspense fallback={<div>Loading...</div>}>
          <HeavyComponent />
        </Suspense>
      )}
    </div>
  );
}
```

#### Symptom: Lazy component fails to load

**Browser behavior**:
Error boundary catches error, shows error UI

**Console pattern**:
```bash
ChunkLoadError: Loading chunk 5 failed.
(error: https://example.com/assets/Analytics-abc123.js)
```

**DevTools clues**:
- Network tab shows 404 or network error for chunk
- Chunk file missing or wrong path

**Root cause**: 
- Build output path doesn't match runtime path
- CDN cache issue
- Network failure

**Solution**: Add error boundary with retry logic:

```tsx
import { lazy, Suspense, Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class LazyLoadErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  handleRetry = () => {
    this.setState({ hasError: false, error: null });
  };
  
  render() {
    if (this.state.hasError) {
      return (
        <div className="error-container">
          <h2>Failed to load component</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={this.handleRetry}>Retry</button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <LazyLoadErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <LazyComponent />
      </Suspense>
    </LazyLoadErrorBoundary>
  );
}
```

#### Symptom: Preloading doesn't work

**Browser behavior**:
User clicks button, sees loading spinner despite prefetch attempt

**Console pattern**:
```bash
[Prefetch] Starting import
[User clicks button]
[Suspense] Fallback shown
[Suspense] Content rendered
```

**DevTools clues**:
- Network tab shows chunk loading AFTER button click
- Prefetch import() call not visible in network tab

**Root cause**: Prefetch import() not actually executed (conditional logic, timing issue)

**Solution**: Ensure prefetch runs unconditionally:

```tsx
// ❌ WRONG: Prefetch inside conditional
function App() {
  const [show, setShow] = useState(false);
  
  if (show) {
    // This runs AFTER user clicks, too late!
    import('./HeavyComponent');
  }
  
  return <button onClick={() => setShow(true)}>Show</button>;
}

// ✓ CORRECT: Prefetch on hover/focus
function App() {
  const [show, setShow] = useState(false);
  
  const prefetch = () => {
    import('./HeavyComponent');
  };
  
  return (
    <button 
      onClick={() => setShow(true)}
      onMouseEnter={prefetch}
      onFocus={prefetch}
    >
      Show
    </button>
  );
}
```

### Performance Impact Summary

**Before code splitting**:
- Initial bundle: 2,081 kB (696 kB gzipped)
- TTI: 4.2s
- Lighthouse score: 62/100

**After route-based splitting**:
- Initial bundle: 1,512 kB (504 kB gzipped)
- TTI: 2.8s
- Lighthouse score: 78/100
- Improvement: 27% smaller, 33% faster TTI

**After component-based splitting**:
- Initial bundle: 1,156 kB (385 kB gzipped)
- TTI: 2.1s
- Lighthouse score: 85/100
- Improvement: 44% smaller, 50% faster TTI

**After vendor splitting**:
- Initial bundle: 1,156 kB (385 kB gzipped)
- Better caching (React vendor chunk cached separately)
- Faster subsequent page loads

**Trade-offs**:
- Code complexity: Moderate increase (lazy imports, Suspense boundaries)
- Network requests: More requests, but smaller total size
- User experience: Brief loading states, but faster initial load
- Caching: Better long-term caching with vendor splitting

**Limitation preview**: We've optimized bundle size and loading, but we haven't systematically identified what's actually slow. Let's learn how to profile and fix real performance bottlenecks.

## Analyzing and fixing performance bottlenecks

## The Failure: "It Feels Slow" Without Metrics

Our dashboard is live. Users complain: "It feels slow." But what's actually slow? Where should we optimize?

Without measurement, optimization is guesswork. Let's learn to diagnose performance issues systematically.

## The Performance Profiling Toolkit

### Tool 1: React DevTools Profiler

**What it measures**:
- Which components rendered
- How long each render took
- Why each component rendered
- Render count and timing

**How to use it**:
1. Open React DevTools
2. Click "Profiler" tab
3. Click record button (⏺)
4. Interact with your app
5. Click stop button (⏹)
6. Analyze the flame graph

### Tool 2: Chrome Performance Tab

**What it measures**:
- JavaScript execution time
- Layout and paint operations
- Network activity
- Main thread blocking

**How to use it**:
1. Open Chrome DevTools
2. Click "Performance" tab
3. Click record button (⏺)
4. Interact with your app
5. Click stop button (⏹)
6. Analyze the timeline

### Tool 3: Lighthouse

**What it measures**:
- Core Web Vitals (LCP, FID, CLS)
- Time to Interactive (TTI)
- Total Blocking Time (TBT)
- First Contentful Paint (FCP)

**How to use it**:
1. Open Chrome DevTools
2. Click "Lighthouse" tab
3. Select "Performance"
4. Click "Analyze page load"

## Iteration 1: Profiling a Slow Interaction

Let's profile our dashboard's filter interaction.

```tsx
// src/components/ProductDashboard.tsx
import { useState, memo, useMemo } from 'react';

interface Product {
  id: string;
  name: string;
  category: string;
  sales: number;
  revenue: number;
  description: string;
}

// Simulate expensive filtering
function filterProducts(products: Product[], searchTerm: string): Product[] {
  console.log('🔄 Filtering products...');
  
  // Simulate expensive operation
  let sum = 0;
  for (let i = 0; i < 1000000; i++) {
    sum += Math.random();
  }
  
  return products.filter(p => 
    p.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
    p.description.toLowerCase().includes(searchTerm.toLowerCase())
  );
}

// Expensive component that renders product details
function ProductCard({ product }: { product: Product }) {
  console.log('🎨 Rendering ProductCard:', product.name);
  
  // Simulate expensive rendering
  let sum = 0;
  for (let i = 0; i < 100000; i++) {
    sum += Math.random();
  }
  
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>{product.category}</p>
      <p>Sales: {product.sales}</p>
      <p>Revenue: ${product.revenue.toLocaleString()}</p>
      <p>{product.description}</p>
    </div>
  );
}

function ProductList({ products }: { products: Product[] }) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

export function ProductDashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  
  // Mock data - 50 products
  const allProducts: Product[] = Array.from({ length: 50 }, (_, i) => ({
    id: `${i}`,
    name: `Product ${i}`,
    category: ['electronics', 'furniture', 'appliances'][i % 3],
    sales: Math.floor(Math.random() * 1000),
    revenue: Math.floor(Math.random() * 100000),
    description: `Description for product ${i}. This is a great product with many features.`,
  }));
  
  const filteredProducts = filterProducts(allProducts, searchTerm);
  
  return (
    <div className="dashboard">
      <h1>Product Dashboard</h1>
      
      <input
        type="text"
        placeholder="Search products..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      
      <ProductList products={filteredProducts} />
    </div>
  );
}
```

**User Action**: Type "Product 1" in the search box.

**Browser Behavior**:
- Noticeable lag between keystrokes
- UI freezes briefly on each keystroke
- Takes ~2 seconds to type 9 characters

### Diagnostic Analysis: React DevTools Profiler

**Step 1: Record the interaction**
1. Open React DevTools → Profiler tab
2. Click record (⏺)
3. Type "Product 1" in search box
4. Click stop (⏹)

**Profiler Output**:

**Flame Graph View**:
```
ProductDashboard (1,247ms)
├─ ProductList (1,156ms)
│  ├─ ProductCard (23ms) × 50 components
│  ├─ ProductCard (24ms)
│  ├─ ProductCard (22ms)
│  └─ ... (47 more)
└─ input (2ms)
```

**Ranked View** (sorted by render time):
1. `ProductList`: 1,156ms (93% of total time)
2. `ProductCard` (Product 0): 23ms
3. `ProductCard` (Product 1): 24ms
4. `ProductCard` (Product 2): 22ms
... (50 total ProductCard renders)

**Interactions Timeline**:
- Keystroke 1 ("P"): 1,247ms
- Keystroke 2 ("r"): 1,198ms
- Keystroke 3 ("o"): 1,223ms
... (9 total keystrokes)

**Why Each Component Rendered**:
- `ProductDashboard`: State changed (searchTerm)
- `ProductList`: Props changed (products array)
- `ProductCard` (all 50): Parent re-rendered

### Diagnostic Analysis: Chrome Performance Tab

**Step 1: Record the interaction**
1. Open Chrome DevTools → Performance tab
2. Click record (⏺)
3. Type "Product 1" in search box
4. Click stop (⏹)

**Performance Timeline**:

**Main Thread Activity**:
```
[Keystroke 1]
├─ Event: input (2ms)
├─ React: Render phase (1,156ms)
│  ├─ filterProducts (120ms)
│  └─ ProductCard × 50 (1,036ms)
├─ React: Commit phase (45ms)
├─ Layout (23ms)
└─ Paint (12ms)

Total: 1,358ms of blocked main thread
```

**Breakdown**:
- JavaScript execution: 1,201ms (88%)
- Rendering (layout + paint): 35ms (3%)
- Idle: 122ms (9%)

**Long Tasks** (>50ms):
- Task 1: 1,201ms (React render)
- Task 2: 45ms (React commit)

### Let's Parse This Evidence

**What the user experiences**:
- Expected: Instant feedback (<100ms)
- Actual: 1.2+ second lag per keystroke

**What React DevTools reveals**:
- `ProductList` takes 93% of render time
- All 50 `ProductCard` components re-render on every keystroke
- Each `ProductCard` takes ~23ms to render
- Total: 50 × 23ms = 1,150ms

**What Chrome Performance shows**:
- Main thread blocked for 1.2+ seconds
- JavaScript execution dominates (88% of time)
- `filterProducts` takes 120ms
- `ProductCard` rendering takes 1,036ms

**Root causes identified**:
1. **Expensive filtering**: `filterProducts` runs on every keystroke (120ms)
2. **Unnecessary re-renders**: All 50 `ProductCard` components re-render, even if product data unchanged
3. **Expensive rendering**: Each `ProductCard` has expensive logic (23ms × 50 = 1,150ms)

**Optimization priorities** (by impact):
1. Memoize `ProductCard` to prevent unnecessary re-renders (saves ~1,150ms)
2. Memoize `filterProducts` result (saves ~120ms)
3. Optimize `ProductCard` rendering logic (saves ~23ms per card)

### Iteration 2: Memoizing ProductCard

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, filterProducts unchanged)

// Memoize ProductCard
const ProductCard = memo(function ProductCard({ product }: { product: Product }) {
  console.log('🎨 Rendering ProductCard:', product.name);
  
  // Simulate expensive rendering
  let sum = 0;
  for (let i = 0; i < 100000; i++) {
    sum += Math.random();
  }
  
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>{product.category}</p>
      <p>Sales: {product.sales}</p>
      <p>Revenue: ${product.revenue.toLocaleString()}</p>
      <p>{product.description}</p>
    </div>
  );
});

// ... (ProductList, ProductDashboard unchanged)
```

**User Action**: Type "Product 1" in the search box.

```bash
# Browser Console Output (typing "P", then "r"):

🎨 Rendering ProductCard: Product 0
🎨 Rendering ProductCard: Product 1
🎨 Rendering ProductCard: Product 2
... (50 cards render on first keystroke)

🎨 Rendering ProductCard: Product 0
🎨 Rendering ProductCard: Product 1
🎨 Rendering ProductCard: Product 2
... (50 cards render on second keystroke too!)
```

### The Failure: React.memo Doesn't Help

**React DevTools Profiler**:
- Still 1,247ms per keystroke
- All 50 `ProductCard` components still re-render
- No improvement

**Why?** The `products` array is a new reference on every render. Even though the product objects inside are the same, the array itself is new.

### Iteration 3: Memoizing the Filtered Products Array

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, filterProducts, ProductCard unchanged)

function ProductList({ products }: { products: Product[] }) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

export function ProductDashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  
  const allProducts: Product[] = Array.from({ length: 50 }, (_, i) => ({
    id: `${i}`,
    name: `Product ${i}`,
    category: ['electronics', 'furniture', 'appliances'][i % 3],
    sales: Math.floor(Math.random() * 1000),
    revenue: Math.floor(Math.random() * 100000),
    description: `Description for product ${i}. This is a great product with many features.`,
  }));
  
  // Memoize filtered products
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(allProducts, searchTerm);
  }, [searchTerm]); // Only re-filter when searchTerm changes
  
  return (
    <div className="dashboard">
      <h1>Product Dashboard</h1>
      
      <input
        type="text"
        placeholder="Search products..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      
      <ProductList products={filteredProducts} />
    </div>
  );
}
```

**User Action**: Type "Product 1" in the search box.

```bash
# Browser Console Output (typing "P", then "r"):

🔄 Filtering products...
🎨 Rendering ProductCard: Product 0
🎨 Rendering ProductCard: Product 1
... (50 cards render on first keystroke)

🔄 Filtering products...
🎨 Rendering ProductCard: Product 0
🎨 Rendering ProductCard: Product 1
... (50 cards still render on second keystroke!)
```

### The Failure: Still Re-rendering Everything

**Why?** Even though we memoized `filteredProducts`, the array contents change on every keystroke (different products match the search term). So the array reference changes, and all `ProductCard` components re-render.

**This is actually correct behavior!** When the filtered results change, we SHOULD re-render the cards.

But we can still optimize: only re-render cards that actually changed.

### Iteration 4: Stable Product References

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, filterProducts unchanged)

const ProductCard = memo(function ProductCard({ product }: { product: Product }) {
  console.log('🎨 Rendering ProductCard:', product.name);
  
  // Simulate expensive rendering
  let sum = 0;
  for (let i = 0; i < 100000; i++) {
    sum += Math.random();
  }
  
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>{product.category}</p>
      <p>Sales: {product.sales}</p>
      <p>Revenue: ${product.revenue.toLocaleString()}</p>
      <p>{product.description}</p>
    </div>
  );
});

function ProductList({ products }: { products: Product[] }) {
  console.log('🎨 Rendering ProductList');
  return (
    <div className="product-list">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

export function ProductDashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  
  // Move allProducts outside component to maintain stable references
  const allProducts = useMemo(() => 
    Array.from({ length: 50 }, (_, i) => ({
      id: `${i}`,
      name: `Product ${i}`,
      category: ['electronics', 'furniture', 'appliances'][i % 3],
      sales: Math.floor(Math.random() * 1000),
      revenue: Math.floor(Math.random() * 100000),
      description: `Description for product ${i}. This is a great product with many features.`,
    }))
  , []); // Empty deps - create once
  
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(allProducts, searchTerm);
  }, [allProducts, searchTerm]);
  
  return (
    <div className="dashboard">
      <h1>Product Dashboard</h1>
      
      <input
        type="text"
        placeholder="Search products..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      
      <ProductList products={filteredProducts} />
    </div>
  );
}
```

**User Action**: Type "Product 1" in the search box.

```bash
# Browser Console Output (typing "P", then "r", then "o"):

🔄 Filtering products...
🎨 Rendering ProductCard: Product 0
🎨 Rendering ProductCard: Product 1
... (50 cards render on first keystroke)

🔄 Filtering products...
🎨 Rendering ProductCard: Product 10
🎨 Rendering ProductCard: Product 11
... (only 10 cards render - those matching "Pr")

🔄 Filtering products...
🎨 Rendering ProductCard: Product 10
... (only 1 card renders - "Product 10" matches "Pro")
```

**Expected vs. Actual Improvement**:

**React DevTools Profiler**:
- First keystroke ("P"): 1,247ms (all 50 cards render)
- Second keystroke ("r"): 287ms (only 10 cards render)
- Third keystroke ("o"): 45ms (only 1 card renders)
- Improvement: 77-96% faster after first keystroke

**Chrome Performance Tab**:
- Main thread blocked: 45ms (vs. 1,247ms before)
- JavaScript execution: 32ms (vs. 1,201ms before)
- Improvement: 96% reduction in main thread blocking

**Why it works now**:
1. `allProducts` array created once, stable references
2. Product objects inside array have stable references
3. When filtering, we return the SAME product objects
4. `React.memo` compares product objects by reference
5. Only cards with different product objects re-render

### Iteration 5: Optimizing the Expensive Rendering

Even with memoization, each `ProductCard` takes 23ms to render. Let's optimize the rendering logic.

```tsx
// src/components/ProductDashboard.tsx (updated)
import { useState, memo, useMemo } from 'react';

// ... (interfaces, filterProducts unchanged)

const ProductCard = memo(function ProductCard({ product }: { product: Product }) {
  console.log('🎨 Rendering ProductCard:', product.name);
  
  // ❌ REMOVED: Expensive simulation
  // let sum = 0;
  // for (let i = 0; i < 100000; i++) {
  //   sum += Math.random();
  // }
  
  // ✓ OPTIMIZED: Memoize expensive calculations
  const formattedRevenue = useMemo(() => {
    // If this were actually expensive...
    return product.revenue.toLocaleString();
  }, [product.revenue]);
  
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>{product.category}</p>
      <p>Sales: {product.sales}</p>
      <p>Revenue: ${formattedRevenue}</p>
      <p>{product.description}</p>
    </div>
  );
});

// ... (ProductList, ProductDashboard unchanged)
```

**Expected vs. Actual Improvement**:

**React DevTools Profiler**:
- First keystroke: 156ms (vs. 1,247ms before optimization)
- Subsequent keystrokes: 12-45ms
- Improvement: 87-99% faster

**Chrome Performance Tab**:
- Main thread blocked: 12ms (vs. 1,247ms before)
- JavaScript execution: 8ms (vs. 1,201ms before)
- Improvement: 99% reduction

**Lighthouse Score**:
- Before: 62/100
- After: 94/100
- TTI: 0.8s (vs. 4.2s before)
- TBT: 45ms (vs. 890ms before)

## Real-World Performance Patterns

### Pattern 1: Virtualization for Long Lists

When rendering 1000+ items, even optimized components are slow. Solution: only render visible items.

```tsx
// Install react-window
// npm install react-window

import { FixedSizeList } from 'react-window';

function VirtualizedProductList({ products }: { products: Product[] }) {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <ProductCard product={products[index]} />
    </div>
  );
  
  return (
    <FixedSizeList
      height={600}
      itemCount={products.length}
      itemSize={120}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

**Performance Impact**:
- Before: Render 1000 items = 23 seconds
- After: Render ~10 visible items = 230ms
- Improvement: 99% faster

### Pattern 2: Debouncing Expensive Operations

Don't run expensive operations on every keystroke. Wait for user to stop typing.

```tsx
import { useState, useMemo, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}

export function ProductDashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 300);
  
  const allProducts = useMemo(() => 
    Array.from({ length: 50 }, (_, i) => ({
      id: `${i}`,
      name: `Product ${i}`,
      category: ['electronics', 'furniture', 'appliances'][i % 3],
      sales: Math.floor(Math.random() * 1000),
      revenue: Math.floor(Math.random() * 100000),
      description: `Description for product ${i}. This is a great product with many features.`,
    }))
  , []);
  
  // Filter using debounced value
  const filteredProducts = useMemo(() => {
    console.log('🔄 Filtering products...');
    return filterProducts(allProducts, debouncedSearchTerm);
  }, [allProducts, debouncedSearchTerm]);
  
  return (
    <div className="dashboard">
      <h1>Product Dashboard</h1>
      
      <input
        type="text"
        placeholder="Search products..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      
      <p>Searching for: {debouncedSearchTerm || '(all products)'}</p>
      
      <ProductList products={filteredProducts} />
    </div>
  );
}
```

**Performance Impact**:
- Before: Filter on every keystroke (9 filters for "Product 1")
- After: Filter once after user stops typing (1 filter)
- Improvement: 89% fewer expensive operations

**User Experience**:
- Input updates immediately (instant feedback)
- Results update after 300ms pause
- Feels responsive while reducing work

### Pattern 3: Web Workers for Heavy Computation

Move expensive calculations off the main thread.

```typescript
// src/workers/filter.worker.ts
import { Product } from '../types';

self.onmessage = (e: MessageEvent) => {
  const { products, searchTerm } = e.data;
  
  // Expensive filtering logic runs in worker thread
  const filtered = products.filter((p: Product) => 
    p.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
    p.description.toLowerCase().includes(searchTerm.toLowerCase())
  );
  
  self.postMessage(filtered);
};
```

```tsx
// src/components/ProductDashboard.tsx
import { useState, useEffect, useMemo } from 'react';

export function ProductDashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  const [filteredProducts, setFilteredProducts] = useState<Product[]>([]);
  
  const allProducts = useMemo(() => 
    Array.from({ length: 50 }, (_, i) => ({
      id: `${i}`,
      name: `Product ${i}`,
      category: ['electronics', 'furniture', 'appliances'][i % 3],
      sales: Math.floor(Math.random() * 1000),
      revenue: Math.floor(Math.random() * 100000),
      description: `Description for product ${i}. This is a great product with many features.`,
    }))
  , []);
  
  useEffect(() => {
    const worker = new Worker(
      new URL('../workers/filter.worker.ts', import.meta.url),
      { type: 'module' }
    );
    
    worker.postMessage({ products: allProducts, searchTerm });
    
    worker.onmessage = (e: MessageEvent) => {
      setFilteredProducts(e.data);
    };
    
    return () => worker.terminate();
  }, [allProducts, searchTerm]);
  
  return (
    <div className="dashboard">
      <h1>Product Dashboard</h1>
      
      <input
        type="text"
        placeholder="Search products..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      
      <ProductList products={filteredProducts} />
    </div>
  );
}
```

**Performance Impact**:
- Main thread: No longer blocked by filtering
- UI remains responsive during expensive operations
- Filtering happens in parallel with rendering

**Trade-offs**:
- Setup complexity (worker files, message passing)
- Communication overhead (serializing data)
- Only worth it for truly expensive operations (>50ms)

## Common Performance Anti-Patterns

### Anti-pattern 1: Premature Optimization

```tsx
// ❌ WRONG: Optimizing before measuring
function MyComponent() {
  const value1 = useMemo(() => a + b, [a, b]);
  const value2 = useMemo(() => c * d, [c, d]);
  const value3 = useMemo(() => e - f, [e, f]);
  
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  return <div onClick={handleClick}>{value1 + value2 + value3}</div>;
}

// ✓ CORRECT: Optimize only proven bottlenecks
function MyComponent() {
  const value1 = a + b;
  const value2 = c * d;
  const value3 = e - f;
  
  const handleClick = () => {
    console.log('clicked');
  };
  
  return <div onClick={handleClick}>{value1 + value2 + value3}</div>;
}
```

### Anti-pattern 2: Optimizing the Wrong Thing

```tsx
// ❌ WRONG: Optimizing cheap operations
const sortedItems = useMemo(() => {
  return items.sort((a, b) => a.id - b.id);
}, [items]); // Sorting 5 items is cheap, useMemo overhead not worth it

// ✓ CORRECT: Optimize expensive operations
const sortedItems = useMemo(() => {
  return items.sort((a, b) => {
    // Complex sorting logic with multiple comparisons
    return expensiveCompare(a, b);
  });
}, [items]); // Expensive comparison, useMemo helps
```

### Anti-pattern 3: Creating New Objects in Render

```tsx
// ❌ WRONG: New object on every render
function MyComponent() {
  return <ChildComponent config={{ theme: 'dark', size: 'large' }} />;
}

// ✓ CORRECT: Stable object reference
const CONFIG = { theme: 'dark', size: 'large' };

function MyComponent() {
  return <ChildComponent config={CONFIG} />;
}

// ✓ ALSO CORRECT: Memoize if config depends on props/state
function MyComponent({ theme, size }) {
  const config = useMemo(() => ({ theme, size }), [theme, size]);
  return <ChildComponent config={config} />;
}
```

## Performance Optimization Checklist

**Before optimizing**:
- [ ] Profile with React DevTools Profiler
- [ ] Profile with Chrome Performance tab
- [ ] Run Lighthouse audit
- [ ] Identify the actual bottleneck (don't guess)
- [ ] Measure baseline performance

**Optimization strategies** (in order of impact):
1. [ ] Code splitting (route-based, component-based)
2. [ ] Memoize expensive calculations (useMemo)
3. [ ] Memoize expensive components (React.memo)
4. [ ] Virtualize long lists (react-window)
5. [ ] Debounce expensive operations
6. [ ] Optimize expensive rendering logic
7. [ ] Use Web Workers for heavy computation

**After optimizing**:
- [ ] Profile again to measure improvement
- [ ] Verify user experience improved
- [ ] Check for regressions (new bugs)
- [ ] Document why optimization was needed

**Limitation preview**: We've learned to identify and fix performance bottlenecks, but we need a systematic decision framework. When should we optimize? What technique should we use? Let's create a flowchart.

## The performance optimization flowchart

## The Performance Optimization Decision Framework

Performance optimization is not about applying every technique everywhere. It's about making informed decisions based on measurement and trade-offs.

### Step 1: Should You Optimize?

**Start here**: Is there a performance problem?

**Measure first**:
- Run Lighthouse audit
- Profile with React DevTools
- Profile with Chrome Performance tab
- Get real user metrics (if in production)

**Decision criteria**:

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| Lighthouse Score | 90-100 | 50-89 | 0-49 |
| Time to Interactive (TTI) | <3.8s | 3.8-7.3s | >7.3s |
| Total Blocking Time (TBT) | <200ms | 200-600ms | >600ms |
| First Contentful Paint (FCP) | <1.8s | 1.8-3.0s | >3.0s |
| Largest Contentful Paint (LCP) | <2.5s | 2.5-4.0s | >4.0s |

**If metrics are "Good"**: Don't optimize. Focus on features.

**If metrics are "Needs Improvement" or "Poor"**: Continue to Step 2.

### Step 2: What's the Bottleneck?

Use profiling tools to identify the actual problem.

**React DevTools Profiler** reveals:
- Which components are slow to render
- Which components render unnecessarily
- Why components rendered (props changed, state changed, parent rendered)

**Chrome Performance Tab** reveals:
- JavaScript execution time
- Layout and paint operations
- Network activity
- Main thread blocking

**Bundle Analyzer** reveals:
- Large dependencies
- Duplicate code
- Unused code

**Common bottlenecks**:

| Symptom | Likely Cause | Tool to Confirm |
|---------|--------------|-----------------|
| Slow initial load | Large bundle size | Bundle analyzer, Network tab |
| Slow page transitions | No code splitting | Bundle analyzer, Network tab |
| Laggy interactions | Expensive renders | React Profiler, Performance tab |
| Unresponsive UI | Main thread blocked | Performance tab |
| Slow data fetching | Network waterfall | Network tab |
| Memory leaks | Unmounted components | Memory profiler |

### Step 3: Choose the Right Optimization

Based on the bottleneck, choose the appropriate technique.

## The Optimization Flowchart

```
┌─────────────────────────────────────┐
│ Is there a performance problem?     │
│ (Measure with Lighthouse/Profiler)  │
└─────────────┬───────────────────────┘
              │
              ├─ No → Don't optimize
              │
              └─ Yes
                 │
┌────────────────▼────────────────────┐
│ What's the bottleneck?              │
│ (Profile to identify)               │
└─────────────┬───────────────────────┘
              │
              ├─ Large bundle size
              │  │
              │  └─ Is it route-specific code?
              │     ├─ Yes → Route-based code splitting
              │     └─ No → Is it a heavy component?
              │        ├─ Yes → Component-based code splitting
              │        └─ No → Vendor code splitting
              │
              ├─ Slow component renders
              │  │
              │  └─ Are components rendering unnecessarily?
              │     ├─ Yes → Are props changing?
              │     │  ├─ Yes → useMemo to stabilize props
              │     │  └─ No → React.memo to skip renders
              │     └─ No → Is rendering logic expensive?
              │        ├─ Yes → useMemo for calculations
              │        └─ No → Optimize rendering logic
              │
              ├─ Long lists (1000+ items)
              │  │
              │  └─ Virtualization (react-window)
              │
              ├─ Expensive operations on every keystroke
              │  │
              │  └─ Debouncing
              │
              ├─ Heavy computation blocking UI
              │  │
              │  └─ Web Workers
              │
              └─ Slow data fetching
                 │
                 └─ See Chapter 13 (React Query)
```

## Decision Matrix: Which Technique When?

### Bundle Size Optimization

| Scenario | Technique | When to Use | Impact | Complexity |
|----------|-----------|-------------|--------|------------|
| Large initial bundle | Route-based splitting | Always | High | Low |
| Heavy component | Component-based splitting | Component >50 kB | Medium | Low |
| Large dependencies | Vendor splitting | Dependencies >100 kB | Medium | Low |
| Unused code | Tree shaking | Always (automatic) | Medium | None |

### Render Optimization

| Scenario | Technique | When to Use | Impact | Complexity |
|----------|-----------|-------------|--------|------------|
| Unnecessary re-renders | React.memo | Expensive component, stable props | High | Low |
| Expensive calculations | useMemo | Calculation >10ms | High | Low |
| Function props | useCallback | Passed to memoized child | Medium | Low |
| Long lists | Virtualization | 1000+ items | Very High | Medium |
| Expensive operations | Debouncing | User input triggers expensive work | High | Low |
| Heavy computation | Web Workers | Computation >50ms | High | High |

### Trade-off Analysis

**React.memo**:
- ✅ Prevents unnecessary renders
- ✅ Low complexity
- ❌ Adds memory overhead
- ❌ Requires stable props
- **Use when**: Component is expensive AND props are stable

**useMemo**:
- ✅ Caches expensive calculations
- ✅ Stabilizes references
- ❌ Adds memory overhead
- ❌ Requires correct dependencies
- **Use when**: Calculation is expensive OR result used in dependencies

**useCallback**:
- ✅ Stabilizes function references
- ✅ Enables React.memo optimization
- ❌ Adds memory overhead
- ❌ Requires correct dependencies
- **Use when**: Function passed to memoized child

**Code splitting**:
- ✅ Reduces initial bundle
- ✅ Improves TTI
- ❌ Adds loading states
- ❌ More network requests
- **Use when**: Code not needed immediately

**Virtualization**:
- ✅ Handles massive lists
- ✅ Constant performance
- ❌ Complex setup
- ❌ Accessibility challenges
- **Use when**: Rendering 1000+ items

**Debouncing**:
- ✅ Reduces expensive operations
- ✅ Simple to implement
- ❌ Delayed feedback
- ❌ Requires tuning delay
- **Use when**: Expensive operation triggered by user input

**Web Workers**:
- ✅ Doesn't block main thread
- ✅ Parallel computation
- ❌ Complex setup
- ❌ Communication overhead
- **Use when**: Computation >50ms AND can be parallelized

## The Complete Journey: From Slow to Fast

Let's trace our dashboard's evolution through all optimizations.

### Iteration 0: Naive Implementation

**Characteristics**:
- No memoization
- No code splitting
- Expensive rendering logic
- All code in main bundle

**Performance**:
- Bundle size: 2,081 kB
- TTI: 4.2s
- Keystroke lag: 1,247ms
- Lighthouse score: 62/100

### Iteration 1: Route-Based Code Splitting

**Changes**:
- Split admin panel and analytics into separate chunks
- Lazy load with React.lazy and Suspense

**Performance**:
- Bundle size: 1,512 kB (27% smaller)
- TTI: 2.8s (33% faster)
- Keystroke lag: 1,247ms (unchanged)
- Lighthouse score: 78/100

**Lesson**: Code splitting improves initial load, not runtime performance.

### Iteration 2: Component Memoization

**Changes**:
- Wrapped ProductCard with React.memo
- Memoized filteredProducts with useMemo
- Stabilized product object references

**Performance**:
- Bundle size: 1,512 kB (unchanged)
- TTI: 2.8s (unchanged)
- Keystroke lag: 45ms (96% faster)
- Lighthouse score: 85/100

**Lesson**: Memoization improves runtime performance, not initial load.

### Iteration 3: Optimized Rendering Logic

**Changes**:
- Removed expensive simulation code
- Memoized expensive calculations within components

**Performance**:
- Bundle size: 1,512 kB (unchanged)
- TTI: 2.8s (unchanged)
- Keystroke lag: 12ms (99% faster)
- Lighthouse score: 94/100

**Lesson**: Optimizing the actual work is more effective than caching.

### Iteration 4: Debouncing

**Changes**:
- Debounced search input (300ms delay)
- Reduced number of filter operations

**Performance**:
- Bundle size: 1,512 kB (unchanged)
- TTI: 2.8s (unchanged)
- Keystroke lag: 0ms (instant input feedback)
- Filter operations: 1 (vs. 9 before)
- Lighthouse score: 94/100

**Lesson**: Debouncing improves perceived performance and reduces work.

### Final Implementation: Production-Ready

**All optimizations applied**:
- Route-based code splitting
- Component-based code splitting
- React.memo for expensive components
- useMemo for expensive calculations
- Stable object references
- Optimized rendering logic
- Debounced expensive operations

**Final Performance**:
- Bundle size: 1,512 kB (27% smaller than baseline)
- TTI: 2.8s (33% faster than baseline)
- Keystroke lag: 0ms (instant feedback)
- Filter operations: 89% fewer
- Lighthouse score: 94/100 (vs. 62/100 baseline)

**Performance Improvement Summary**:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Bundle size | 2,081 kB | 1,512 kB | 27% smaller |
| TTI | 4.2s | 2.8s | 33% faster |
| Keystroke lag | 1,247ms | 0ms | 100% faster |
| Lighthouse | 62/100 | 94/100 | 52% better |

## When NOT to Optimize

**Don't optimize if**:
- Metrics are already good (Lighthouse >90)
- Component renders <10ms
- Bundle size <500 kB
- Users aren't complaining
- You haven't measured the problem

**Don't optimize by**:
- Wrapping every component with React.memo
- Memoizing every calculation with useMemo
- Using useCallback for every function
- Splitting every component into separate chunks
- Applying every technique "just in case"

**The Rule**: Measure first, optimize second, measure again.

## The Professional React Developer's Performance Mindset

**1. Performance is a feature**: Users notice slow apps. Fast apps feel better.

**2. Measure, don't guess**: Profiling reveals the truth. Intuition often misleads.

**3. Optimize the right thing**: 80% of slowness comes from 20% of code. Find that 20%.

**4. Trade-offs exist**: Every optimization has costs. Choose wisely.

**5. Premature optimization is evil**: Don't optimize until you have a problem.

**6. User experience matters most**: Perceived performance > actual performance.

**7. Maintenance matters**: Complex optimizations must be worth the maintenance burden.

## The Complete Performance Optimization Checklist

**Phase 1: Measurement**
- [ ] Run Lighthouse audit
- [ ] Profile with React DevTools Profiler
- [ ] Profile with Chrome Performance tab
- [ ] Analyze bundle with bundle analyzer
- [ ] Identify the actual bottleneck
- [ ] Document baseline metrics

**Phase 2: Bundle Optimization**
- [ ] Implement route-based code splitting
- [ ] Implement component-based code splitting for heavy components
- [ ] Configure vendor code splitting
- [ ] Verify tree shaking is working
- [ ] Measure bundle size improvement

**Phase 3: Render Optimization**
- [ ] Identify components that render unnecessarily
- [ ] Apply React.memo to expensive components
- [ ] Stabilize props with useMemo
- [ ] Stabilize callbacks with useCallback
- [ ] Optimize expensive rendering logic
- [ ] Measure render time improvement

**Phase 4: Interaction Optimization**
- [ ] Identify expensive operations triggered by user input
- [ ] Implement debouncing for expensive operations
- [ ] Consider virtualization for long lists
- [ ] Consider Web Workers for heavy computation
- [ ] Measure interaction responsiveness

**Phase 5: Verification**
- [ ] Re-run Lighthouse audit
- [ ] Re-profile with React DevTools
- [ ] Re-profile with Chrome Performance tab
- [ ] Verify user experience improved
- [ ] Check for regressions
- [ ] Document final metrics

**Phase 6: Maintenance**
- [ ] Document why optimizations were needed
- [ ] Document trade-offs made
- [ ] Set up performance budgets
- [ ] Monitor performance in production
- [ ] Review optimizations periodically

## Lessons Learned: The Performance Journey

**From naive to professional**:

1. **Start simple**: Don't optimize prematurely. Build features first.

2. **Measure everything**: You can't improve what you don't measure.

3. **Optimize strategically**: Focus on the biggest bottlenecks first.

4. **Understand trade-offs**: Every optimization has costs. Choose wisely.

5. **Maintain balance**: Code complexity vs. performance gain.

6. **Think about users**: Perceived performance matters more than benchmarks.

7. **Keep learning**: Performance optimization is a continuous journey.

**The ultimate lesson**: Performance optimization is not about applying every technique everywhere. It's about making informed decisions based on measurement, understanding trade-offs, and prioritizing user experience.
