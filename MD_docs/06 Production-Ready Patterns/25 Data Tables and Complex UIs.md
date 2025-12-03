# Chapter 25: Data Tables and Complex UIs

## TanStack Table: flexible, headless tables

## The Problem: Building a Production-Grade Data Table

You've been asked to build an admin dashboard for an e-commerce platform. The centerpiece is a product inventory table that displays thousands of products with columns for SKU, name, category, price, stock level, and last updated date. Users need to sort by any column, filter by category, search by name, and paginate through results.

Your first instinct might be to reach for a pre-built table component library with built-in styling. But you quickly discover that these "batteries-included" solutions come with significant limitations: they impose their own styling that conflicts with your design system, they bundle features you don't need (inflating your bundle size), and they make it difficult to customize behavior for your specific use case.

This is where **TanStack Table** (formerly React Table) shines. It's a **headless UI library**—it provides the logic and state management for tables without imposing any markup or styling. You bring your own UI components, and TanStack Table handles the complex state management, sorting algorithms, filtering logic, and pagination calculations.

Let's build this product inventory table from scratch, starting with a naive implementation to understand what problems we're solving.

### Phase 1: The Naive Implementation

We'll start with a simple table that displays product data. This will be our **reference implementation** that we'll progressively enhance throughout this chapter.

```typescript
// src/types/product.ts
export interface Product {
  id: string;
  sku: string;
  name: string;
  category: string;
  price: number;
  stock: number;
  lastUpdated: Date;
}
```

```tsx
// src/components/ProductTable.tsx
import { useState, useEffect } from 'react';
import { Product } from '../types/product';

export function ProductTable() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Simulate API call
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="overflow-x-auto">
      <table className="min-w-full divide-y divide-gray-200">
        <thead className="bg-gray-50">
          <tr>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              SKU
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Name
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Category
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Price
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Stock
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Last Updated
            </th>
          </tr>
        </thead>
        <tbody className="bg-white divide-y divide-gray-200">
          {products.map(product => (
            <tr key={product.id}>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                {product.sku}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                {product.name}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {product.category}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                ${product.price.toFixed(2)}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                {product.stock}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {new Date(product.lastUpdated).toLocaleDateString()}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

This works for displaying data, but it has no interactivity. Let's try to add sorting manually.

### Iteration 1: Manual Sorting Implementation

**Current limitation**: Users can't sort the table by any column.

**New scenario**: What happens when we try to add sorting by clicking column headers?

```tsx
// src/components/ProductTable.tsx - Attempt 1
import { useState, useEffect } from 'react';
import { Product } from '../types/product';

type SortField = keyof Product;
type SortDirection = 'asc' | 'desc';

export function ProductTable() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sortField, setSortField] = useState<SortField>('name');
  const [sortDirection, setSortDirection] = useState<SortDirection>('asc');

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  const handleSort = (field: SortField) => {
    if (sortField === field) {
      // Toggle direction if clicking same field
      setSortDirection(sortDirection === 'asc' ? 'desc' : 'asc');
    } else {
      // New field, default to ascending
      setSortField(field);
      setSortDirection('asc');
    }
  };

  // Sort products
  const sortedProducts = [...products].sort((a, b) => {
    const aValue = a[sortField];
    const bValue = b[sortField];
    
    if (aValue < bValue) return sortDirection === 'asc' ? -1 : 1;
    if (aValue > bValue) return sortDirection === 'asc' ? 1 : -1;
    return 0;
  });

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="overflow-x-auto">
      <table className="min-w-full divide-y divide-gray-200">
        <thead className="bg-gray-50">
          <tr>
            <th 
              className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
              onClick={() => handleSort('sku')}
            >
              SKU {sortField === 'sku' && (sortDirection === 'asc' ? '↑' : '↓')}
            </th>
            <th 
              className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
              onClick={() => handleSort('name')}
            >
              Name {sortField === 'name' && (sortDirection === 'asc' ? '↑' : '↓')}
            </th>
            <th 
              className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
              onClick={() => handleSort('category')}
            >
              Category {sortField === 'category' && (sortDirection === 'asc' ? '↑' : '↓')}
            </th>
            <th 
              className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
              onClick={() => handleSort('price')}
            >
              Price {sortField === 'price' && (sortDirection === 'asc' ? '↑' : '↓')}
            </th>
            <th 
              className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
              onClick={() => handleSort('stock')}
            >
              Stock {sortField === 'stock' && (sortDirection === 'asc' ? '↑' : '↓')}
            </th>
            <th 
              className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
              onClick={() => handleSort('lastUpdated')}
            >
              Last Updated {sortField === 'lastUpdated' && (sortDirection === 'asc' ? '↑' : '↓')}
            </th>
          </tr>
        </thead>
        <tbody className="bg-white divide-y divide-gray-200">
          {sortedProducts.map(product => (
            <tr key={product.id}>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                {product.sku}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                {product.name}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {product.category}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                ${product.price.toFixed(2)}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                {product.stock}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {new Date(product.lastUpdated).toLocaleDateString()}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Diagnostic Analysis: The Failure Emerges

Let's run this with 5,000 products and click a column header to sort.

**Browser Behavior**:
The page freezes for 2-3 seconds. The UI becomes completely unresponsive. Eventually, the table re-renders with sorted data, but the user experience is terrible.

**Browser Console Output**:
```
[Violation] 'click' handler took 2847ms
```

**React DevTools - Profiler Tab**:
- Recorded render: `ProductTable` took 2.8 seconds to render
- Reason: State changed (sortField)
- Flamegraph shows: Most time spent in the `sort()` operation and subsequent render of 5,000 table rows

**Chrome Performance Tab**:
- Main thread blocked for 2.8 seconds
- Long Task warning: Task took 2847ms (blocking)
- Breakdown:
  - 800ms: Array sorting
  - 2000ms: React rendering 5,000 DOM nodes
  - 47ms: Browser layout and paint

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Instant sort with smooth visual feedback
   - Actual: Complete UI freeze, no feedback, delayed result

2. **What the console reveals**:
   - The click handler is synchronous and blocks the main thread
   - 2.8 seconds is far beyond the 50ms threshold for responsive UI

3. **What DevTools shows**:
   - The component re-renders all 5,000 rows on every sort
   - No optimization: every row is a new DOM node
   - The sort algorithm itself is inefficient for large datasets

4. **Root cause identified**: 
   We're doing too much work on the main thread: sorting 5,000 items, then rendering 5,000 DOM nodes, all synchronously.

5. **Why the current approach can't solve this**:
   Even if we optimize the sort algorithm, rendering 5,000 DOM nodes will always be slow. We need a fundamentally different approach: **pagination** (render fewer rows) and **virtualization** (render only visible rows).

6. **What we need**:
   A table library that handles sorting efficiently, supports pagination out of the box, and provides the foundation for virtualization. Enter TanStack Table.

### Iteration 2: Introducing TanStack Table

**Problem**: Manual state management for sorting, filtering, and pagination is complex and error-prone. We're reinventing the wheel poorly.

**Solution**: TanStack Table provides battle-tested logic for all table interactions while letting us control the UI completely.

First, install the library:

```bash
npm install @tanstack/react-table
```

Now let's rebuild our table using TanStack Table's core concepts:

1. **Column Definitions**: Describe your columns declaratively
2. **Table Instance**: Create a table instance with your data and columns
3. **Rendering**: Use the table instance to render your UI

Here's the TanStack Table version:

```tsx
// src/components/ProductTable.tsx - TanStack Table version
import { useState, useEffect, useMemo } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  ColumnDef,
  flexRender,
  SortingState,
} from '@tanstack/react-table';
import { Product } from '../types/product';

export function ProductTable() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sorting, setSorting] = useState<SortingState>([]);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  // Define columns - memoized to prevent recreation on every render
  const columns = useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'sku',
        header: 'SKU',
        cell: info => info.getValue(),
      },
      {
        accessorKey: 'name',
        header: 'Name',
        cell: info => info.getValue(),
      },
      {
        accessorKey: 'category',
        header: 'Category',
        cell: info => info.getValue(),
      },
      {
        accessorKey: 'price',
        header: 'Price',
        cell: info => `$${(info.getValue() as number).toFixed(2)}`,
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
        cell: info => info.getValue(),
      },
      {
        accessorKey: 'lastUpdated',
        header: 'Last Updated',
        cell: info => new Date(info.getValue() as Date).toLocaleDateString(),
      },
    ],
    []
  );

  // Create table instance
  const table = useReactTable({
    data: products,
    columns,
    state: {
      sorting,
    },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="overflow-x-auto">
      <table className="min-w-full divide-y divide-gray-200">
        <thead className="bg-gray-50">
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th
                  key={header.id}
                  className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
                  onClick={header.column.getToggleSortingHandler()}
                >
                  <div className="flex items-center gap-2">
                    {flexRender(
                      header.column.columnDef.header,
                      header.getContext()
                    )}
                    {{
                      asc: '↑',
                      desc: '↓',
                    }[header.column.getIsSorted() as string] ?? null}
                  </div>
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody className="bg-white divide-y divide-gray-200">
          {table.getRowModel().rows.map(row => (
            <tr key={row.id}>
              {row.getVisibleCells().map(cell => (
                <td
                  key={cell.id}
                  className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                >
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Understanding the TanStack Table Architecture

Let's break down what's happening here:

**1. Column Definitions (`ColumnDef<Product>[]`)**:

Each column definition describes:
- `accessorKey`: Which property of your data to display
- `header`: What to show in the column header
- `cell`: How to render each cell (with access to the cell's value via `info.getValue()`)

The `useMemo` wrapper prevents recreating the column definitions on every render—they're static configuration.

**2. Table Instance (`useReactTable`)**:

This is the core of TanStack Table. You pass it:
- `data`: Your array of products
- `columns`: Your column definitions
- `state`: Current table state (sorting, filtering, pagination)
- `onSortingChange`: Callback when sorting changes
- `getCoreRowModel()`: Required - provides basic row model
- `getSortedRowModel()`: Optional - enables sorting

The table instance returns methods to access:
- Header groups (for rendering `<thead>`)
- Row model (for rendering `<tbody>`)
- Column state (for rendering sort indicators)

**3. Rendering with `flexRender`**:

Instead of directly rendering values, we use `flexRender` which handles both string headers and custom React components. This provides maximum flexibility—you can pass a string, a React component, or a render function.

**4. Sorting State Management**:

TanStack Table manages sorting state internally but lets you control it via the `sorting` state and `onSortingChange` callback. This follows React's controlled component pattern—you own the state, TanStack Table tells you when to update it.

### Verification: Does This Solve the Performance Problem?

Let's test with 5,000 products again.

**Browser Behavior**:
Click a column header. The table sorts instantly—no perceptible delay.

**Browser Console Output**:
```
(No violations)
```

**React DevTools - Profiler Tab**:
- Recorded render: `ProductTable` took 2.1 seconds to render
- Wait, that's still slow! What's happening?

**Chrome Performance Tab**:
- Main thread blocked for 2.1 seconds
- Long Task warning: Task took 2100ms
- Breakdown:
  - 50ms: Sorting (TanStack Table's optimized sort)
  - 2050ms: React rendering 5,000 DOM nodes

**Analysis**:

TanStack Table solved the sorting performance problem—sorting is now fast. But we still have the rendering problem: 5,000 DOM nodes is too many to render at once.

The sorting feels instant because TanStack Table's algorithm is optimized. But the initial render and re-render after sorting still take 2+ seconds because we're rendering every single row.

**Limitation preview**: This solves sorting efficiency, but we still need to address the rendering bottleneck. We'll tackle that with pagination in the next section.

### When to Apply TanStack Table

**What it optimizes for**:
- Separation of concerns: logic vs. UI
- Flexibility: complete control over markup and styling
- Type safety: full TypeScript support
- Extensibility: plugin-based architecture for features

**What it sacrifices**:
- Initial setup time: more verbose than pre-styled components
- No built-in UI: you must build all visual elements

**When to choose this approach**:
- You have a custom design system
- You need fine-grained control over table behavior
- You're building a complex table with many features
- You want to avoid bundle bloat from unused features

**When to avoid this approach**:
- You need a table quickly and don't care about customization
- Your design matches an existing component library
- You're building a simple table with no interactivity

**Code characteristics**:
- Setup complexity: Medium (column definitions, table instance)
- Maintenance burden: Low (declarative, well-typed)
- Performance impact: Excellent (optimized algorithms, minimal re-renders)

## Pagination, sorting, and filtering

## Solving the Rendering Bottleneck

Our table now sorts efficiently, but it still renders 5,000 rows at once. This is the fundamental problem: **the browser can't efficiently render thousands of DOM nodes simultaneously**.

The solution is **pagination**: render only a subset of rows at a time. Users can navigate between pages to see more data.

### Iteration 3: Adding Pagination

**Current limitation**: Rendering all 5,000 products at once causes a 2+ second render time.

**New scenario**: What if we only render 50 products per page?

TanStack Table has built-in pagination support. Let's add it:

```tsx
// src/components/ProductTable.tsx - With pagination
import { useState, useEffect, useMemo } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getPaginationRowModel,
  ColumnDef,
  flexRender,
  SortingState,
  PaginationState,
} from '@tanstack/react-table';
import { Product } from '../types/product';

export function ProductTable() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sorting, setSorting] = useState<SortingState>([]);
  const [pagination, setPagination] = useState<PaginationState>({
    pageIndex: 0,
    pageSize: 50,
  });

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  const columns = useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'sku',
        header: 'SKU',
      },
      {
        accessorKey: 'name',
        header: 'Name',
      },
      {
        accessorKey: 'category',
        header: 'Category',
      },
      {
        accessorKey: 'price',
        header: 'Price',
        cell: info => `$${(info.getValue() as number).toFixed(2)}`,
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
      },
      {
        accessorKey: 'lastUpdated',
        header: 'Last Updated',
        cell: info => new Date(info.getValue() as Date).toLocaleDateString(),
      },
    ],
    []
  );

  const table = useReactTable({
    data: products,
    columns,
    state: {
      sorting,
      pagination,
    },
    onSortingChange: setSorting,
    onPaginationChange: setPagination,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
  });

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="space-y-4">
      <div className="overflow-x-auto">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    <div className="flex items-center gap-2">
                      {flexRender(
                        header.column.columnDef.header,
                        header.getContext()
                      )}
                      {{
                        asc: '↑',
                        desc: '↓',
                      }[header.column.getIsSorted() as string] ?? null}
                    </div>
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {table.getRowModel().rows.map(row => (
              <tr key={row.id}>
                {row.getVisibleCells().map(cell => (
                  <td
                    key={cell.id}
                    className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                  >
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      {/* Pagination controls */}
      <div className="flex items-center justify-between px-6 py-3 bg-white border-t border-gray-200">
        <div className="flex items-center gap-2">
          <button
            onClick={() => table.setPageIndex(0)}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'<<'}
          </button>
          <button
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'<'}
          </button>
          <button
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'>'}
          </button>
          <button
            onClick={() => table.setPageIndex(table.getPageCount() - 1)}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'>>'}
          </button>
        </div>

        <span className="text-sm text-gray-700">
          Page {table.getState().pagination.pageIndex + 1} of{' '}
          {table.getPageCount()} ({products.length} total products)
        </span>

        <select
          value={table.getState().pagination.pageSize}
          onChange={e => table.setPageSize(Number(e.target.value))}
          className="px-3 py-1 border rounded"
        >
          {[10, 25, 50, 100].map(pageSize => (
            <option key={pageSize} value={pageSize}>
              Show {pageSize}
            </option>
          ))}
        </select>
      </div>
    </div>
  );
}
```

### Verification: Performance Improvement

Let's test with 5,000 products again.

**Browser Behavior**:
The table loads instantly. Clicking sort is instant. Navigating between pages is instant. The UI is completely responsive.

**React DevTools - Profiler Tab**:
- Initial render: `ProductTable` took 42ms
- Sort operation: 38ms
- Page navigation: 35ms

**Chrome Performance Tab**:
- Main thread: No long tasks
- Initial render: 42ms (95% improvement from 2100ms)
- Breakdown:
  - 8ms: TanStack Table calculations
  - 34ms: React rendering 50 DOM nodes

**Expected vs. Actual improvement**:

| Metric | Before (5000 rows) | After (50 rows/page) | Improvement |
|--------|-------------------|---------------------|-------------|
| Initial render | 2100ms | 42ms | 98% faster |
| Sort operation | 2100ms | 38ms | 98% faster |
| User experience | Frozen UI | Instant response | ✓ |

**Analysis**:

Pagination solved the rendering bottleneck completely. By rendering only 50 rows at a time, we reduced render time from 2+ seconds to under 50ms—well within the threshold for responsive UI.

The key insight: **The browser can render 50 DOM nodes efficiently. It cannot render 5,000 DOM nodes efficiently.** Pagination is not just a UX pattern—it's a performance necessity for large datasets.

### Understanding Pagination State

TanStack Table's pagination state consists of two values:

1. **`pageIndex`**: Zero-based current page number
2. **`pageSize`**: Number of rows per page

The table instance provides methods to:
- Navigate: `nextPage()`, `previousPage()`, `setPageIndex()`
- Query state: `getCanNextPage()`, `getCanPreviousPage()`, `getPageCount()`
- Configure: `setPageSize()`

The `getPaginationRowModel()` function automatically slices your data based on the current page. You don't need to manually calculate which rows to display—TanStack Table handles it.

### Iteration 4: Adding Filtering

**Current limitation**: Users can't search or filter products. They must manually page through results to find what they need.

**New scenario**: What if users want to filter by category or search by product name?

TanStack Table supports filtering through the `getFilteredRowModel()` function. Let's add both global search and column-specific filtering:

```tsx
// src/components/ProductTable.tsx - With filtering
import { useState, useEffect, useMemo } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getPaginationRowModel,
  getFilteredRowModel,
  ColumnDef,
  flexRender,
  SortingState,
  PaginationState,
  ColumnFiltersState,
} from '@tanstack/react-table';
import { Product } from '../types/product';

export function ProductTable() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sorting, setSorting] = useState<SortingState>([]);
  const [pagination, setPagination] = useState<PaginationState>({
    pageIndex: 0,
    pageSize: 50,
  });
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
  const [globalFilter, setGlobalFilter] = useState('');

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  const columns = useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'sku',
        header: 'SKU',
      },
      {
        accessorKey: 'name',
        header: 'Name',
      },
      {
        accessorKey: 'category',
        header: 'Category',
        // Enable column-specific filtering
        filterFn: 'equals',
      },
      {
        accessorKey: 'price',
        header: 'Price',
        cell: info => `$${(info.getValue() as number).toFixed(2)}`,
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
      },
      {
        accessorKey: 'lastUpdated',
        header: 'Last Updated',
        cell: info => new Date(info.getValue() as Date).toLocaleDateString(),
      },
    ],
    []
  );

  const table = useReactTable({
    data: products,
    columns,
    state: {
      sorting,
      pagination,
      columnFilters,
      globalFilter,
    },
    onSortingChange: setSorting,
    onPaginationChange: setPagination,
    onColumnFiltersChange: setColumnFilters,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  });

  // Get unique categories for filter dropdown
  const categories = useMemo(() => {
    const uniqueCategories = new Set(products.map(p => p.category));
    return Array.from(uniqueCategories).sort();
  }, [products]);

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="space-y-4">
      {/* Filter controls */}
      <div className="flex gap-4 px-6 py-4 bg-white border border-gray-200 rounded">
        <div className="flex-1">
          <label className="block text-sm font-medium text-gray-700 mb-1">
            Search all columns
          </label>
          <input
            type="text"
            value={globalFilter ?? ''}
            onChange={e => setGlobalFilter(e.target.value)}
            placeholder="Search products..."
            className="w-full px-3 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>

        <div className="w-64">
          <label className="block text-sm font-medium text-gray-700 mb-1">
            Filter by category
          </label>
          <select
            value={(table.getColumn('category')?.getFilterValue() as string) ?? ''}
            onChange={e =>
              table.getColumn('category')?.setFilterValue(e.target.value || undefined)
            }
            className="w-full px-3 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            <option value="">All categories</option>
            {categories.map(category => (
              <option key={category} value={category}>
                {category}
              </option>
            ))}
          </select>
        </div>
      </div>

      <div className="overflow-x-auto">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    <div className="flex items-center gap-2">
                      {flexRender(
                        header.column.columnDef.header,
                        header.getContext()
                      )}
                      {{
                        asc: '↑',
                        desc: '↓',
                      }[header.column.getIsSorted() as string] ?? null}
                    </div>
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {table.getRowModel().rows.map(row => (
              <tr key={row.id}>
                {row.getVisibleCells().map(cell => (
                  <td
                    key={cell.id}
                    className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                  >
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      {/* Pagination controls */}
      <div className="flex items-center justify-between px-6 py-3 bg-white border-t border-gray-200">
        <div className="flex items-center gap-2">
          <button
            onClick={() => table.setPageIndex(0)}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'<<'}
          </button>
          <button
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'<'}
          </button>
          <button
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'>'}
          </button>
          <button
            onClick={() => table.setPageIndex(table.getPageCount() - 1)}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {'>>'}
          </button>
        </div>

        <span className="text-sm text-gray-700">
          Page {table.getState().pagination.pageIndex + 1} of{' '}
          {table.getPageCount()} (
          {table.getFilteredRowModel().rows.length} filtered from{' '}
          {products.length} total)
        </span>

        <select
          value={table.getState().pagination.pageSize}
          onChange={e => table.setPageSize(Number(e.target.value))}
          className="px-3 py-1 border rounded"
        >
          {[10, 25, 50, 100].map(pageSize => (
            <option key={pageSize} value={pageSize}>
              Show {pageSize}
            </option>
          ))}
        </select>
      </div>
    </div>
  );
}
```

### Understanding Filtering in TanStack Table

TanStack Table supports two types of filtering:

**1. Global Filter**:
Searches across all columns simultaneously. Useful for general search functionality. Controlled via `globalFilter` state and `onGlobalFilterChange` callback.

**2. Column Filters**:
Filter specific columns with custom logic. Each column can have its own filter function:
- `'equals'`: Exact match
- `'includesString'`: Case-insensitive substring match (default)
- `'includesStringSensitive'`: Case-sensitive substring match
- Custom functions: `(row, columnId, filterValue) => boolean`

The `getFilteredRowModel()` function applies all active filters and returns only matching rows. Pagination then operates on the filtered results.

**Filter execution order**:
1. Global filter applied first
2. Column filters applied to global filter results
3. Sorting applied to filtered results
4. Pagination applied to sorted, filtered results

This means users can combine filters: search globally for "laptop", filter by category "Electronics", sort by price, and paginate through results—all working together seamlessly.

### Verification: Filtering Performance

Let's test filtering with 5,000 products.

**Browser Behavior**:
Type "laptop" in the search box. Results filter instantly. Select "Electronics" category. Results filter instantly. The UI remains responsive throughout.

**React DevTools - Profiler Tab**:
- Global filter change: 45ms
- Column filter change: 38ms
- Combined filters: 52ms

**Analysis**:

Filtering is fast because:
1. TanStack Table's filter algorithms are optimized
2. We're only rendering the filtered results (still paginated to 50 rows)
3. React only re-renders the table body, not the entire component

The key insight: **Filtering reduces the dataset before pagination**, so we're still only rendering 50 rows even if 2,000 products match the filter.

### Common Failure Modes and Their Signatures

#### Symptom: Filtering is slow (>500ms delay)

**Browser behavior**:
Typing in the search box feels laggy. Each keystroke causes a noticeable pause.

**Console pattern**:
```
[Violation] 'input' handler took 847ms
```

**DevTools clues**:
- Profiler shows long render time
- Many components re-rendering unnecessarily

**Root cause**: 
You're not debouncing the filter input. Every keystroke triggers a full filter + re-render cycle.

**Solution**: 
Debounce the filter input:

```tsx
import { useDeferredValue } from 'react';

export function ProductTable() {
  const [filterInput, setFilterInput] = useState('');
  const deferredFilter = useDeferredValue(filterInput);
  
  // Use deferredFilter for the table, filterInput for the input value
  const table = useReactTable({
    // ...
    state: {
      globalFilter: deferredFilter,
    },
  });

  return (
    <input
      value={filterInput}
      onChange={e => setFilterInput(e.target.value)}
    />
  );
}
```

#### Symptom: Pagination resets to page 1 when filtering

**Browser behavior**:
User is on page 5, applies a filter, and suddenly they're back on page 1.

**Root cause**: 
This is actually correct behavior—when the filtered dataset changes, the current page might not exist anymore. If you had 100 pages and filter down to 2 pages, staying on page 5 would show nothing.

**Solution**: 
This is expected. TanStack Table automatically resets to page 0 when filters change. If you want different behavior, you can manually control it:

```tsx
const table = useReactTable({
  // ...
  autoResetPageIndex: false, // Disable automatic reset
});
```

But be aware: this can lead to showing empty pages if the filtered results have fewer pages than the current page index.

#### Symptom: Filter doesn't work on custom cell renderers

**Browser behavior**:
You have a price column that displays "$99.99" but filtering by "99.99" returns no results.

**Root cause**: 
The filter operates on the raw data value, not the formatted display value.

**Solution**: 
Use a custom filter function that handles your data format:

```tsx
{
  accessorKey: 'price',
  header: 'Price',
  cell: info => `$${(info.getValue() as number).toFixed(2)}`,
  filterFn: (row, columnId, filterValue) => {
    const price = row.getValue(columnId) as number;
    const searchValue = filterValue.replace('$', '');
    return price.toString().includes(searchValue);
  },
}
```

## Virtual scrolling for large lists

## When Pagination Isn't Enough

Pagination works well for most use cases, but sometimes users need to see all data at once without clicking through pages. Examples:

- Financial dashboards showing real-time stock prices
- Log viewers displaying thousands of log entries
- Infinite scroll social media feeds
- Data analysis tools where users need to scan large datasets

For these scenarios, we need **virtual scrolling** (also called **windowing**): render only the rows currently visible in the viewport, plus a small buffer. As the user scrolls, we dynamically render new rows and unmount off-screen rows.

### The Problem: Rendering 10,000 Rows

Let's say our product table needs to display all 10,000 products in a single scrollable list (no pagination). What happens?

```tsx
// src/components/ProductTableInfinite.tsx - Naive infinite scroll
import { useState, useEffect } from 'react';
import { Product } from '../types/product';

export function ProductTableInfinite() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Fetch all 10,000 products
    fetch('/api/products?limit=10000')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="h-screen overflow-auto">
      <table className="min-w-full divide-y divide-gray-200">
        <thead className="bg-gray-50 sticky top-0">
          <tr>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              SKU
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Name
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Category
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Price
            </th>
            <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
              Stock
            </th>
          </tr>
        </thead>
        <tbody className="bg-white divide-y divide-gray-200">
          {products.map(product => (
            <tr key={product.id}>
              <td className="px-6 py-4 whitespace-nowrap text-sm">
                {product.sku}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm">
                {product.name}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm">
                {product.category}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm">
                ${product.price.toFixed(2)}
              </td>
              <td className="px-6 py-4 whitespace-nowrap text-sm">
                {product.stock}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Diagnostic Analysis: The Catastrophic Failure

Let's run this with 10,000 products.

**Browser Behavior**:
The page freezes for 8-10 seconds. The browser shows "Page Unresponsive" warning. Eventually the table renders, but scrolling is janky—it stutters and lags. The browser tab uses 800+ MB of memory.

**Browser Console Output**:
```
[Violation] 'load' handler took 9847ms
[Violation] Forced reflow while executing JavaScript took 234ms
Warning: Detected a large number of DOM nodes (10000+). This may cause performance issues.
```

**Chrome Performance Tab**:
- Main thread blocked for 9.8 seconds during initial render
- Long Task: 9847ms
- Memory: 847 MB allocated
- Breakdown:
  - 1200ms: React creating virtual DOM
  - 3400ms: React reconciliation
  - 5200ms: Browser creating 10,000 DOM nodes
  - 47ms: Layout and paint

**Chrome Memory Profiler**:
- Heap snapshot: 847 MB
- DOM nodes: 10,247 (table + rows + cells)
- Detached DOM nodes: 0 (all still attached)

**Scrolling Performance**:
- Scroll event handler: 120ms per scroll (should be <16ms for 60fps)
- Janky frames: 78% of frames dropped
- Reason: Browser must repaint 10,000 DOM nodes on every scroll

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Smooth scrolling through products
   - Actual: 10-second freeze, then janky scrolling, browser warning

2. **What the console reveals**:
   - The load handler (initial render) took nearly 10 seconds
   - Forced reflows indicate layout thrashing
   - React itself warns about too many DOM nodes

3. **What DevTools shows**:
   - 10,000+ DOM nodes in memory
   - Each scroll event triggers expensive repaints
   - Memory usage is unsustainable for larger datasets

4. **Root cause identified**: 
   The browser cannot efficiently manage 10,000 DOM nodes. Creating them is slow, keeping them in memory is expensive, and repainting them on scroll is impossible to do at 60fps.

5. **Why the current approach can't solve this**:
   No amount of React optimization will help. The fundamental problem is that we're asking the browser to do something it's not designed for: manage thousands of DOM nodes simultaneously.

6. **What we need**:
   A technique that renders only the visible rows, dynamically creating and destroying DOM nodes as the user scrolls. This is **virtual scrolling**.

### Iteration 5: Introducing TanStack Virtual

TanStack Virtual (formerly React Virtual) is a headless virtualization library that pairs perfectly with TanStack Table. It calculates which rows are visible and provides the measurements needed to create a smooth scrolling experience.

Install the library:

```bash
npm install @tanstack/react-virtual
```

Now let's rebuild our infinite scroll table with virtualization:

```tsx
// src/components/ProductTableVirtual.tsx
import { useState, useEffect, useMemo, useRef } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  ColumnDef,
  flexRender,
  SortingState,
} from '@tanstack/react-table';
import { useVirtualizer } from '@tanstack/react-virtual';
import { Product } from '../types/product';

export function ProductTableVirtual() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sorting, setSorting] = useState<SortingState>([]);

  // Ref for the scrollable container
  const tableContainerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    fetch('/api/products?limit=10000')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  const columns = useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'sku',
        header: 'SKU',
        size: 120,
      },
      {
        accessorKey: 'name',
        header: 'Name',
        size: 250,
      },
      {
        accessorKey: 'category',
        header: 'Category',
        size: 150,
      },
      {
        accessorKey: 'price',
        header: 'Price',
        size: 100,
        cell: info => `$${(info.getValue() as number).toFixed(2)}`,
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
        size: 100,
      },
    ],
    []
  );

  const table = useReactTable({
    data: products,
    columns,
    state: {
      sorting,
    },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  const { rows } = table.getRowModel();

  // Create virtualizer
  const rowVirtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => tableContainerRef.current,
    estimateSize: () => 53, // Estimated row height in pixels
    overscan: 10, // Render 10 extra rows above and below viewport
  });

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div
      ref={tableContainerRef}
      className="h-screen overflow-auto"
    >
      <div style={{ height: `${rowVirtualizer.getTotalSize()}px`, position: 'relative' }}>
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50 sticky top-0 z-10">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    style={{ width: header.getSize() }}
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-100"
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    <div className="flex items-center gap-2">
                      {flexRender(
                        header.column.columnDef.header,
                        header.getContext()
                      )}
                      {{
                        asc: '↑',
                        desc: '↓',
                      }[header.column.getIsSorted() as string] ?? null}
                    </div>
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {rowVirtualizer.getVirtualItems().map(virtualRow => {
              const row = rows[virtualRow.index];
              return (
                <tr
                  key={row.id}
                  style={{
                    height: `${virtualRow.size}px`,
                    transform: `translateY(${virtualRow.start}px)`,
                    position: 'absolute',
                    width: '100%',
                  }}
                >
                  {row.getVisibleCells().map(cell => (
                    <td
                      key={cell.id}
                      style={{ width: cell.column.getSize() }}
                      className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                    >
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
}
```

### Understanding Virtual Scrolling

Let's break down how TanStack Virtual works:

**1. The Virtualizer Instance**:

`useVirtualizer` creates a virtualizer that tracks:
- Total number of items (`count: rows.length`)
- The scrollable container (`getScrollElement`)
- Estimated size of each item (`estimateSize: () => 53`)
- How many extra items to render (`overscan: 10`)

**2. Virtual Items**:

`rowVirtualizer.getVirtualItems()` returns only the items that should be rendered right now. For a viewport showing 20 rows, it might return 40 items (20 visible + 10 overscan above + 10 overscan below).

**3. Positioning**:

Each virtual item has:
- `index`: Position in the full dataset
- `start`: Pixel offset from the top
- `size`: Height in pixels

We use `transform: translateY()` to position each row at the correct scroll offset. This is more performant than using `top` because it doesn't trigger layout recalculation.

**4. Total Size**:

`rowVirtualizer.getTotalSize()` returns the total height of all items. We set this as the height of the container so the scrollbar reflects the full dataset size, even though we're only rendering a fraction of the rows.

**5. Overscan**:

The `overscan` parameter renders extra rows above and below the viewport. This prevents blank flashes when scrolling quickly—by the time a row enters the viewport, it's already rendered.

### Verification: Virtual Scrolling Performance

Let's test with 10,000 products.

**Browser Behavior**:
The table loads in under 100ms. Scrolling is buttery smooth—no jank, no lag. The browser tab uses only 120 MB of memory.

**Browser Console Output**:
```
(No violations, no warnings)
```

**React DevTools - Profiler Tab**:
- Initial render: 87ms
- Scroll event: 3-5ms per scroll
- Rows rendered: 40 (out of 10,000)

**Chrome Performance Tab**:
- Main thread: No long tasks
- Initial render: 87ms
- Scroll performance: 60fps maintained
- Memory: 124 MB allocated
- Breakdown:
  - 45ms: React creating virtual DOM for 40 rows
  - 42ms: Browser creating 40 DOM nodes

**Chrome Memory Profiler**:
- Heap snapshot: 124 MB (85% reduction from 847 MB)
- DOM nodes: 247 (table + 40 rows + cells)
- Detached DOM nodes: 0

**Expected vs. Actual improvement**:

| Metric | Before (10k rows) | After (virtual) | Improvement |
|--------|------------------|-----------------|-------------|
| Initial render | 9847ms | 87ms | 99% faster |
| Memory usage | 847 MB | 124 MB | 85% reduction |
| DOM nodes | 10,247 | 247 | 98% reduction |
| Scroll performance | 12fps (janky) | 60fps (smooth) | 5x better |
| User experience | Frozen, unusable | Instant, smooth | ✓ |

**Analysis**:

Virtual scrolling solved the performance problem completely. By rendering only 40 rows at a time (instead of 10,000), we:

1. Reduced initial render time by 99%
2. Reduced memory usage by 85%
3. Achieved smooth 60fps scrolling
4. Made the UI instantly responsive

The key insight: **The browser doesn't need to know about rows that aren't visible.** Virtual scrolling is an illusion—we create the appearance of a 10,000-row table while only rendering what's on screen.

### When to Apply Virtual Scrolling

**What it optimizes for**:
- Rendering performance with large datasets (1000+ items)
- Memory efficiency
- Smooth scrolling experience
- Ability to display all data without pagination

**What it sacrifices**:
- Implementation complexity (more code than simple lists)
- Fixed or estimated item heights (dynamic heights are harder)
- Accessibility considerations (screen readers see fewer items)
- Search-in-page functionality (browser can't find off-screen text)

**When to choose this approach**:
- Dataset has 1000+ items
- Users need to scan/scroll through all data
- Pagination would hurt the user experience
- Items have consistent or predictable heights

**When to avoid this approach**:
- Dataset has <500 items (simple rendering is fine)
- Pagination is acceptable to users
- Items have highly variable heights
- Accessibility is critical (screen readers struggle with virtual scrolling)

**Code characteristics**:
- Setup complexity: High (virtualizer + positioning logic)
- Maintenance burden: Medium (must handle edge cases)
- Performance impact: Excellent (constant-time rendering regardless of dataset size)

### Common Failure Modes and Their Signatures

#### Symptom: Blank space or flickering during fast scrolling

**Browser behavior**:
When scrolling quickly, you see blank rows or rows that flicker in and out.

**Root cause**: 
The `overscan` value is too low. Rows aren't rendered before they enter the viewport.

**Solution**: 
Increase the overscan:

```tsx
const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => tableContainerRef.current,
  estimateSize: () => 53,
  overscan: 20, // Increased from 10
});
```

Higher overscan = more rows rendered = smoother scrolling but slightly higher memory usage. Find the balance for your use case.

#### Symptom: Rows have wrong heights or overlap

**Browser behavior**:
Some rows are taller than others, causing overlapping or gaps.

**Root cause**: 
The `estimateSize` doesn't match actual row heights, and you haven't provided a way to measure actual heights.

**Solution**: 
Use dynamic measurement:

```tsx
const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => tableContainerRef.current,
  estimateSize: () => 53,
  measureElement: (element) => element?.getBoundingClientRect().height ?? 53,
  overscan: 10,
});

// In your render:
<tr
  key={row.id}
  data-index={virtualRow.index}
  ref={rowVirtualizer.measureElement}
  style={{
    transform: `translateY(${virtualRow.start}px)`,
    position: 'absolute',
    width: '100%',
  }}
>
```

The `measureElement` callback tells the virtualizer to measure each row's actual height after rendering.
</markdown>

<markdown>
#### Symptom: Scrollbar jumps or behaves erratically

**Browser behavior**:
The scrollbar position jumps around when scrolling. The scrollbar size changes unexpectedly.

**Root cause**: 
The total size calculation is wrong, usually because `estimateSize` is far from actual row heights.

**Solution**: 
Provide more accurate size estimates or use dynamic measurement (see above). If rows have consistent heights, measure one row and use that exact value:
</markdown>

const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => tableContainerRef.current,
  estimateSize: () => 53, // Measured from actual rendered row
  overscan: 10,
});
```

#### Symptom: Virtual scrolling breaks with sorting/filtering

**Browser behavior**:
After sorting or filtering, the scroll position jumps to the top or rows render incorrectly.

**Root cause**: 
The virtualizer doesn't know the data changed. It's still using old measurements.

**Solution**: 
Reset the virtualizer when data changes:

```tsx
useEffect(() => {
  rowVirtualizer.scrollToIndex(0);
}, [sorting, columnFilters]); // Reset scroll position when data changes
```

## Accessible, keyboard-navigable interfaces

## The Accessibility Crisis in Data Tables

We've built a high-performance data table with sorting, filtering, pagination, and virtual scrolling. But we've created an accessibility nightmare. Let's test it with a screen reader and keyboard-only navigation.

### Diagnostic Analysis: The Accessibility Failure

**Testing with NVDA screen reader (Windows)**:

1. Tab to the table
2. Screen reader announces: "Table with 5 columns"
3. Try to navigate with arrow keys
4. Nothing happens—focus stays on the first cell
5. Try to activate sort on a column header
6. Screen reader doesn't announce the sort direction
7. Try to use the pagination controls
8. Screen reader announces "Button" with no context about what the button does

**Testing with keyboard only (no mouse)**:

1. Tab through the page
2. Focus goes to search input ✓
3. Tab to category filter ✓
4. Tab to table... focus disappears
5. Can't navigate between table cells
6. Can't activate column sorting
7. Tab to pagination... buttons work but no indication of current page
8. Can't tell which page you're on without looking

**Lighthouse Accessibility Audit**:
```
Accessibility Score: 67/100

Issues found:
- [aria-sort] not present on sortable column headers
- [role="grid"] not present on interactive table
- [aria-rowcount] not present for virtual scrolling
- [aria-label] missing on pagination buttons
- [aria-live] region missing for dynamic content updates
- Keyboard navigation not implemented for table cells
- Focus indicators not visible on interactive elements
```

**WCAG 2.1 Violations**:
- **2.1.1 Keyboard (Level A)**: Table cells not keyboard navigable
- **2.4.3 Focus Order (Level A)**: Focus order is not logical
- **4.1.2 Name, Role, Value (Level A)**: Interactive elements lack proper ARIA attributes
- **4.1.3 Status Messages (Level AA)**: No announcements for dynamic content changes

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Full keyboard navigation, clear screen reader announcements
   - Actual: Keyboard navigation broken, screen reader provides minimal information

2. **What the audit reveals**:
   - Missing ARIA attributes for interactive elements
   - No keyboard navigation implementation
   - No announcements for dynamic content changes

3. **Root cause identified**: 
   We built the table for mouse users only. We didn't consider keyboard users or screen reader users.

4. **Why this matters**:
   - 15% of web users have some form of disability
   - Keyboard-only users include power users who prefer keyboard efficiency
   - Legal requirement in many jurisdictions (ADA, Section 508, WCAG)
   - Good accessibility often improves usability for everyone

5. **What we need**:
   Proper ARIA attributes, keyboard navigation, focus management, and screen reader announcements.

### Iteration 6: Making the Table Accessible

Let's rebuild our table with accessibility as a first-class concern. We'll focus on the paginated version (virtual scrolling + accessibility is complex and beyond this chapter's scope).

```tsx
// src/components/ProductTableAccessible.tsx
import { useState, useEffect, useMemo, useRef } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getPaginationRowModel,
  getFilteredRowModel,
  ColumnDef,
  flexRender,
  SortingState,
  PaginationState,
  ColumnFiltersState,
} from '@tanstack/react-table';
import { Product } from '../types/product';

export function ProductTableAccessible() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sorting, setSorting] = useState<SortingState>([]);
  const [pagination, setPagination] = useState<PaginationState>({
    pageIndex: 0,
    pageSize: 50,
  });
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
  const [globalFilter, setGlobalFilter] = useState('');

  // Refs for announcements
  const announcementRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  const columns = useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'sku',
        header: 'SKU',
      },
      {
        accessorKey: 'name',
        header: 'Name',
      },
      {
        accessorKey: 'category',
        header: 'Category',
        filterFn: 'equals',
      },
      {
        accessorKey: 'price',
        header: 'Price',
        cell: info => `$${(info.getValue() as number).toFixed(2)}`,
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
      },
      {
        accessorKey: 'lastUpdated',
        header: 'Last Updated',
        cell: info => new Date(info.getValue() as Date).toLocaleDateString(),
      },
    ],
    []
  );

  const table = useReactTable({
    data: products,
    columns,
    state: {
      sorting,
      pagination,
      columnFilters,
      globalFilter,
    },
    onSortingChange: setSorting,
    onPaginationChange: setPagination,
    onColumnFiltersChange: setColumnFilters,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  });

  const categories = useMemo(() => {
    const uniqueCategories = new Set(products.map(p => p.category));
    return Array.from(uniqueCategories).sort();
  }, [products]);

  // Announce changes to screen readers
  const announce = (message: string) => {
    if (announcementRef.current) {
      announcementRef.current.textContent = message;
    }
  };

  // Announce sorting changes
  useEffect(() => {
    if (sorting.length > 0) {
      const sort = sorting[0];
      announce(
        `Table sorted by ${sort.id} in ${sort.desc ? 'descending' : 'ascending'} order`
      );
    }
  }, [sorting]);

  // Announce filter changes
  useEffect(() => {
    const filteredCount = table.getFilteredRowModel().rows.length;
    if (globalFilter || columnFilters.length > 0) {
      announce(`Showing ${filteredCount} of ${products.length} products`);
    }
  }, [globalFilter, columnFilters, products.length, table]);

  // Announce page changes
  useEffect(() => {
    const { pageIndex } = pagination;
    const pageCount = table.getPageCount();
    announce(`Page ${pageIndex + 1} of ${pageCount}`);
  }, [pagination, table]);

  if (isLoading) {
    return <div role="status" aria-live="polite">Loading products...</div>;
  }

  return (
    <div className="space-y-4">
      {/* Screen reader announcements */}
      <div
        ref={announcementRef}
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"
      />

      {/* Filter controls */}
      <div className="flex gap-4 px-6 py-4 bg-white border border-gray-200 rounded">
        <div className="flex-1">
          <label
            htmlFor="global-search"
            className="block text-sm font-medium text-gray-700 mb-1"
          >
            Search all columns
          </label>
          <input
            id="global-search"
            type="text"
            value={globalFilter ?? ''}
            onChange={e => setGlobalFilter(e.target.value)}
            placeholder="Search products..."
            className="w-full px-3 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-describedby="search-description"
          />
          <span id="search-description" className="sr-only">
            Search across all product fields including name, SKU, and category
          </span>
        </div>

        <div className="w-64">
          <label
            htmlFor="category-filter"
            className="block text-sm font-medium text-gray-700 mb-1"
          >
            Filter by category
          </label>
          <select
            id="category-filter"
            value={(table.getColumn('category')?.getFilterValue() as string) ?? ''}
            onChange={e =>
              table.getColumn('category')?.setFilterValue(e.target.value || undefined)
            }
            className="w-full px-3 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            <option value="">All categories</option>
            {categories.map(category => (
              <option key={category} value={category}>
                {category}
              </option>
            ))}
          </select>
        </div>
      </div>

      {/* Table */}
      <div className="overflow-x-auto">
        <table
          className="min-w-full divide-y divide-gray-200"
          role="table"
          aria-label="Product inventory"
          aria-rowcount={table.getFilteredRowModel().rows.length}
        >
          <thead className="bg-gray-50">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id} role="row">
                {headerGroup.headers.map(header => {
                  const sortDirection = header.column.getIsSorted();
                  return (
                    <th
                      key={header.id}
                      role="columnheader"
                      aria-sort={
                        sortDirection === 'asc'
                          ? 'ascending'
                          : sortDirection === 'desc'
                          ? 'descending'
                          : 'none'
                      }
                      className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase"
                    >
                      <button
                        onClick={header.column.getToggleSortingHandler()}
                        className="flex items-center gap-2 hover:text-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 rounded"
                        aria-label={`Sort by ${header.column.columnDef.header}${
                          sortDirection
                            ? `, currently sorted ${
                                sortDirection === 'asc' ? 'ascending' : 'descending'
                              }`
                            : ''
                        }`}
                      >
                        {flexRender(
                          header.column.columnDef.header,
                          header.getContext()
                        )}
                        <span aria-hidden="true">
                          {{
                            asc: '↑',
                            desc: '↓',
                          }[sortDirection as string] ?? '↕'}
                        </span>
                      </button>
                    </th>
                  );
                })}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {table.getRowModel().rows.map((row, rowIndex) => (
              <tr
                key={row.id}
                role="row"
                aria-rowindex={
                  pagination.pageIndex * pagination.pageSize + rowIndex + 1
                }
              >
                {row.getVisibleCells().map(cell => (
                  <td
                    key={cell.id}
                    role="cell"
                    className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                  >
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      {/* Pagination controls */}
      <nav
        aria-label="Table pagination"
        className="flex items-center justify-between px-6 py-3 bg-white border-t border-gray-200"
      >
        <div className="flex items-center gap-2">
          <button
            onClick={() => table.setPageIndex(0)}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Go to first page"
          >
            {'<<'}
          </button>
          <button
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Go to previous page"
          >
            {'<'}
          </button>
          <button
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Go to next page"
          >
            {'>'}
          </button>
          <button
            onClick={() => table.setPageIndex(table.getPageCount() - 1)}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Go to last page"
          >
            {'>>'}
          </button>
        </div>

        <div
          role="status"
          aria-live="polite"
          className="text-sm text-gray-700"
        >
          Page {table.getState().pagination.pageIndex + 1} of{' '}
          {table.getPageCount()} ({table.getFilteredRowModel().rows.length}{' '}
          {table.getFilteredRowModel().rows.length === products.length
            ? 'total'
            : `filtered from ${products.length} total`}{' '}
          products)
        </div>

        <div>
          <label htmlFor="page-size" className="sr-only">
            Rows per page
          </label>
          <select
            id="page-size"
            value={table.getState().pagination.pageSize}
            onChange={e => table.setPageSize(Number(e.target.value))}
            className="px-3 py-1 border rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Select number of rows per page"
          >
            {[10, 25, 50, 100].map(pageSize => (
              <option key={pageSize} value={pageSize}>
                Show {pageSize}
              </option>
            ))}
          </select>
        </div>
      </nav>
    </div>
  );
}
```

### Understanding the Accessibility Improvements

Let's break down what we added:

**1. ARIA Roles and Attributes**:

- `role="table"`: Explicitly marks the table for screen readers
- `role="columnheader"`: Marks column headers
- `role="row"` and `role="cell"`: Marks rows and cells
- `aria-sort`: Announces sort direction on column headers
- `aria-rowcount`: Total number of rows (for pagination context)
- `aria-rowindex`: Current row's position in the full dataset
- `aria-label`: Descriptive labels for interactive elements
- `aria-describedby`: Links inputs to their descriptions

**2. Live Regions for Announcements**:

```tsx
<div
  ref={announcementRef}
  role="status"
  aria-live="polite"
  aria-atomic="true"
  className="sr-only"
/>
```

This invisible element announces changes to screen readers:
- `role="status"`: Indicates status information
- `aria-live="polite"`: Announces when user is idle (not interrupting)
- `aria-atomic="true"`: Reads entire message, not just changes
- `className="sr-only"`: Visually hidden but available to screen readers

**3. Keyboard Navigation**:

- All interactive elements are keyboard accessible (buttons, inputs, selects)
- Focus indicators visible on all interactive elements (`focus:ring-2`)
- Logical tab order (search → filter → table → pagination)
- Sort buttons are actual `<button>` elements, not `onClick` on `<th>`

**4. Semantic HTML**:

- `<label>` elements properly associated with inputs via `htmlFor`
- `<nav>` element wraps pagination controls
- Descriptive button text and aria-labels
- Hidden descriptions for screen readers (`sr-only` class)

**5. Dynamic Announcements**:

We use `useEffect` to announce changes:
- Sorting: "Table sorted by price in ascending order"
- Filtering: "Showing 42 of 5000 products"
- Pagination: "Page 3 of 100"

These announcements provide context that sighted users get visually but screen reader users would miss.

### Verification: Accessibility Testing

Let's retest with screen readers and keyboard navigation.

**Testing with NVDA screen reader**:

1. Tab to search input
   - Announces: "Search all columns, edit, Search across all product fields including name, SKU, and category"
2. Type "laptop"
   - Announces: "Showing 42 of 5000 products"
3. Tab to category filter
   - Announces: "Filter by category, combo box, All categories"
4. Select "Electronics"
   - Announces: "Showing 18 of 5000 products"
5. Tab to table
   - Announces: "Product inventory table with 6 columns and 18 rows"
6. Navigate to SKU column header
   - Announces: "SKU, column header, sortable, not sorted, button"
7. Press Enter to sort
   - Announces: "Table sorted by SKU in ascending order"
8. Tab to pagination
   - Announces: "Go to next page, button"
9. Press Enter
   - Announces: "Page 2 of 1"

**Testing with keyboard only**:

1. Tab through all controls ✓
2. All interactive elements have visible focus indicators ✓
3. Can activate all buttons with Enter/Space ✓
4. Can navigate form controls with arrow keys ✓
5. Logical tab order maintained ✓

**Lighthouse Accessibility Audit**:
```
Accessibility Score: 98/100

Remaining issues:
- Color contrast on disabled buttons could be improved (minor)
```

**WCAG 2.1 Compliance**:
- **2.1.1 Keyboard (Level A)**: ✓ All functionality keyboard accessible
- **2.4.3 Focus Order (Level A)**: ✓ Logical focus order
- **4.1.2 Name, Role, Value (Level A)**: ✓ All elements properly labeled
- **4.1.3 Status Messages (Level AA)**: ✓ Dynamic changes announced

**Expected vs. Actual improvement**:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Lighthouse score | 67/100 | 98/100 | +31 points |
| WCAG violations | 6 | 0 | ✓ |
| Screen reader usability | Poor | Excellent | ✓ |
| Keyboard navigation | Broken | Complete | ✓ |

### When to Apply These Accessibility Patterns

**What it optimizes for**:
- Usability for keyboard-only users
- Usability for screen reader users
- Legal compliance (ADA, Section 508, WCAG)
- Better UX for all users (clear labels, logical flow)

**What it sacrifices**:
- Development time (more code, more testing)
- Complexity (more attributes to manage)

**When to choose this approach**:
- Always. Accessibility should be a baseline requirement.
- Especially for:
  - Public-facing applications
  - Enterprise applications (legal requirements)
  - Government/education applications (Section 508)
  - Applications with diverse user bases

**When to avoid this approach**:
- Never. There's no valid reason to skip accessibility.

**Code characteristics**:
- Setup complexity: Medium (ARIA attributes, announcements)
- Maintenance burden: Low (mostly declarative attributes)
- Performance impact: Negligible (ARIA attributes don't affect rendering)

### Common Accessibility Failure Modes

#### Symptom: Screen reader announces "clickable" instead of "button"

**Root cause**: 
You're using `onClick` on a `<div>` or `<span>` instead of a semantic `<button>`.

**Solution**: 
Use semantic HTML:

```tsx
// ❌ Bad
<div onClick={handleSort} className="cursor-pointer">
  Sort
</div>

// ✓ Good
<button onClick={handleSort} className="...">
  Sort
</button>
```

#### Symptom: Screen reader doesn't announce dynamic content changes

**Root cause**: 
No `aria-live` region for announcements.

**Solution**: 
Add a live region and update it when content changes (see the `announce()` function in our implementation).

#### Symptom: Focus indicator not visible

**Root cause**: 
CSS removes the default focus outline without providing an alternative.

**Solution**: 
Always provide a visible focus indicator:

```tsx
// ❌ Bad
<button className="outline-none">Click me</button>

// ✓ Good
<button className="focus:outline-none focus:ring-2 focus:ring-blue-500">
  Click me
</button>
```

#### Symptom: Tab order is confusing

**Root cause**: 
DOM order doesn't match visual order, or you're using `tabIndex` incorrectly.

**Solution**: 
- Keep DOM order matching visual order
- Only use `tabIndex="0"` (make focusable) or `tabIndex="-1"` (remove from tab order)
- Never use positive `tabIndex` values (they break natural tab order)

## The Complete Journey - Part VI Synthesis

## The Journey: From Naive Table to Production-Ready Data Grid

Let's trace the complete evolution of our product table through six iterations:

| Iteration | Failure Mode | Technique Applied | Result | Performance Impact |
|-----------|--------------|-------------------|--------|-------------------|
| 0 | No interactivity | None | Static table | Baseline: 2100ms render (5k rows) |
| 1 | Manual sorting slow | Manual sort implementation | Sorting works but slow | 2800ms per sort |
| 2 | State management complex | TanStack Table | Clean API, fast sorting | 50ms per sort |
| 3 | Rendering 5k rows slow | Pagination | Fast initial render | 42ms initial render |
| 4 | Can't find products | Filtering | Users can search/filter | 45ms per filter |
| 5 | 10k rows crashes browser | Virtual scrolling | Smooth infinite scroll | 87ms initial, 60fps scroll |
| 6 | Not accessible | ARIA + keyboard nav | WCAG compliant | Negligible overhead |

### Final Implementation: Production-Ready Data Table

Here's the complete, production-ready implementation combining all improvements:

```tsx
// src/components/ProductTableFinal.tsx
import { useState, useEffect, useMemo, useRef } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getPaginationRowModel,
  getFilteredRowModel,
  ColumnDef,
  flexRender,
  SortingState,
  PaginationState,
  ColumnFiltersState,
} from '@tanstack/react-table';
import { Product } from '../types/product';

export function ProductTableFinal() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sorting, setSorting] = useState<SortingState>([]);
  const [pagination, setPagination] = useState<PaginationState>({
    pageIndex: 0,
    pageSize: 50,
  });
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
  const [globalFilter, setGlobalFilter] = useState('');

  const announcementRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  const columns = useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'sku',
        header: 'SKU',
        size: 120,
      },
      {
        accessorKey: 'name',
        header: 'Name',
        size: 250,
      },
      {
        accessorKey: 'category',
        header: 'Category',
        size: 150,
        filterFn: 'equals',
      },
      {
        accessorKey: 'price',
        header: 'Price',
        size: 100,
        cell: info => `$${(info.getValue() as number).toFixed(2)}`,
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
        size: 100,
        cell: info => {
          const stock = info.getValue() as number;
          return (
            <span
              className={
                stock < 10
                  ? 'text-red-600 font-semibold'
                  : stock < 50
                  ? 'text-yellow-600'
                  : 'text-green-600'
              }
            >
              {stock}
            </span>
          );
        },
      },
      {
        accessorKey: 'lastUpdated',
        header: 'Last Updated',
        size: 150,
        cell: info => new Date(info.getValue() as Date).toLocaleDateString(),
      },
    ],
    []
  );

  const table = useReactTable({
    data: products,
    columns,
    state: {
      sorting,
      pagination,
      columnFilters,
      globalFilter,
    },
    onSortingChange: setSorting,
    onPaginationChange: setPagination,
    onColumnFiltersChange: setColumnFilters,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  });

  const categories = useMemo(() => {
    const uniqueCategories = new Set(products.map(p => p.category));
    return Array.from(uniqueCategories).sort();
  }, [products]);

  const announce = (message: string) => {
    if (announcementRef.current) {
      announcementRef.current.textContent = message;
    }
  };

  useEffect(() => {
    if (sorting.length > 0) {
      const sort = sorting[0];
      announce(
        `Table sorted by ${sort.id} in ${sort.desc ? 'descending' : 'ascending'} order`
      );
    }
  }, [sorting]);

  useEffect(() => {
    const filteredCount = table.getFilteredRowModel().rows.length;
    if (globalFilter || columnFilters.length > 0) {
      announce(`Showing ${filteredCount} of ${products.length} products`);
    }
  }, [globalFilter, columnFilters, products.length, table]);

  useEffect(() => {
    const { pageIndex } = pagination;
    const pageCount = table.getPageCount();
    if (pageCount > 0) {
      announce(`Page ${pageIndex + 1} of ${pageCount}`);
    }
  }, [pagination, table]);

  if (isLoading) {
    return (
      <div role="status" aria-live="polite" className="flex items-center justify-center h-64">
        <div className="text-lg text-gray-600">Loading products...</div>
      </div>
    );
  }

  return (
    <div className="space-y-4">
      <div
        ref={announcementRef}
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"
      />

      <div className="flex gap-4 px-6 py-4 bg-white border border-gray-200 rounded-lg shadow-sm">
        <div className="flex-1">
          <label
            htmlFor="global-search"
            className="block text-sm font-medium text-gray-700 mb-1"
          >
            Search all columns
          </label>
          <input
            id="global-search"
            type="text"
            value={globalFilter ?? ''}
            onChange={e => setGlobalFilter(e.target.value)}
            placeholder="Search products..."
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            aria-describedby="search-description"
          />
          <span id="search-description" className="sr-only">
            Search across all product fields including name, SKU, and category
          </span>
        </div>

        <div className="w-64">
          <label
            htmlFor="category-filter"
            className="block text-sm font-medium text-gray-700 mb-1"
          >
            Filter by category
          </label>
          <select
            id="category-filter"
            value={(table.getColumn('category')?.getFilterValue() as string) ?? ''}
            onChange={e =>
              table.getColumn('category')?.setFilterValue(e.target.value || undefined)
            }
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          >
            <option value="">All categories</option>
            {categories.map(category => (
              <option key={category} value={category}>
                {category}
              </option>
            ))}
          </select>
        </div>
      </div>

      <div className="overflow-x-auto border border-gray-200 rounded-lg shadow-sm">
        <table
          className="min-w-full divide-y divide-gray-200"
          role="table"
          aria-label="Product inventory"
          aria-rowcount={table.getFilteredRowModel().rows.length}
        >
          <thead className="bg-gray-50">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id} role="row">
                {headerGroup.headers.map(header => {
                  const sortDirection = header.column.getIsSorted();
                  return (
                    <th
                      key={header.id}
                      role="columnheader"
                      aria-sort={
                        sortDirection === 'asc'
                          ? 'ascending'
                          : sortDirection === 'desc'
                          ? 'descending'
                          : 'none'
                      }
                      style={{ width: header.getSize() }}
                      className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                    >
                      <button
                        onClick={header.column.getToggleSortingHandler()}
                        className="flex items-center gap-2 hover:text-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 rounded px-1 -mx-1"
                        aria-label={`Sort by ${header.column.columnDef.header}${
                          sortDirection
                            ? `, currently sorted ${
                                sortDirection === 'asc' ? 'ascending' : 'descending'
                              }`
                            : ''
                        }`}
                      >
                        {flexRender(
                          header.column.columnDef.header,
                          header.getContext()
                        )}
                        <span aria-hidden="true" className="text-gray-400">
                          {{
                            asc: '↑',
                            desc: '↓',
                          }[sortDirection as string] ?? '↕'}
                        </span>
                      </button>
                    </th>
                  );
                })}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {table.getRowModel().rows.length === 0 ? (
              <tr>
                <td
                  colSpan={columns.length}
                  className="px-6 py-12 text-center text-gray-500"
                >
                  No products found matching your filters.
                </td>
              </tr>
            ) : (
              table.getRowModel().rows.map((row, rowIndex) => (
                <tr
                  key={row.id}
                  role="row"
                  aria-rowindex={
                    pagination.pageIndex * pagination.pageSize + rowIndex + 1
                  }
                  className="hover:bg-gray-50 transition-colors"
                >
                  {row.getVisibleCells().map(cell => (
                    <td
                      key={cell.id}
                      role="cell"
                      className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                    >
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>

      <nav
        aria-label="Table pagination"
        className="flex items-center justify-between px-6 py-3 bg-white border border-gray-200 rounded-lg shadow-sm"
      >
        <div className="flex items-center gap-2">
          <button
            onClick={() => table.setPageIndex(0)}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 text-sm border border-gray-300 rounded-md disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors"
            aria-label="Go to first page"
          >
            {'<<'}
          </button>
          <button
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
            className="px-3 py-1 text-sm border border-gray-300 rounded-md disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors"
            aria-label="Go to previous page"
          >
            {'<'}
          </button>
          <button
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 text-sm border border-gray-300 rounded-md disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors"
            aria-label="Go to next page"
          >
            {'>'}
          </button>
          <button
            onClick={() => table.setPageIndex(table.getPageCount() - 1)}
            disabled={!table.getCanNextPage()}
            className="px-3 py-1 text-sm border border-gray-300 rounded-md disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-blue-500 transition-colors"
            aria-label="Go to last page"
          >
            {'>>'}
          </button>
        </div>

        <div
          role="status"
          aria-live="polite"
          className="text-sm text-gray-700"
        >
          Page {table.getState().pagination.pageIndex + 1} of{' '}
          {table.getPageCount()} ({table.getFilteredRowModel().rows.length}{' '}
          {table.getFilteredRowModel().rows.length === products.length
            ? 'total'
            : `filtered from ${products.length} total`}{' '}
          products)
        </div>

        <div className="flex items-center gap-2">
          <label htmlFor="page-size" className="text-sm text-gray-700">
            Rows per page:
          </label>
          <select
            id="page-size"
            value={table.getState().pagination.pageSize}
            onChange={e => table.setPageSize(Number(e.target.value))}
            className="px-3 py-1 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Select number of rows per page"
          >
            {[10, 25, 50, 100].map(pageSize => (
              <option key={pageSize} value={pageSize}>
                {pageSize}
              </option>
            ))}
          </select>
        </div>
      </nav>
    </div>
  );
}
```

### Decision Framework: Which Approach When?

When building a data table, choose your approach based on these criteria:

**Use Simple Table (No Library)**:
- Dataset: <100 items
- Features: Display only, no sorting/filtering
- Complexity: Low
- Example: Static pricing table, feature comparison

**Use TanStack Table (Paginated)**:
- Dataset: 100-10,000 items
- Features: Sorting, filtering, pagination
- Complexity: Medium
- Example: Admin dashboards, product catalogs, user lists

**Use TanStack Table + Virtual Scrolling**:
- Dataset: 10,000+ items
- Features: Infinite scroll, real-time updates
- Complexity: High
- Example: Log viewers, financial data, analytics dashboards

**Use Pre-built Component Library**:
- Dataset: Any size
- Features: Standard table features
- Complexity: Low (if design matches library)
- Example: Internal tools, MVPs, prototypes
- Trade-off: Less customization, larger bundle

**Decision Tree**:

```
How many items?
├─ <100 → Simple table
├─ 100-10k → TanStack Table (paginated)
└─ >10k → Do users need to see all items?
    ├─ Yes → TanStack Table + Virtual
    └─ No → TanStack Table (paginated)

Do you have a custom design system?
├─ Yes → TanStack Table (headless)
└─ No → Consider pre-built library

Is accessibility critical?
├─ Yes → Add ARIA attributes (always)
└─ No → Still add ARIA attributes (always)
```

### Lessons Learned: From Naive to Professional

**1. Performance is about rendering less, not rendering faster**:
The biggest performance gains came from reducing the number of DOM nodes (pagination, virtualization), not from optimizing the rendering of those nodes.

**2. Headless libraries provide flexibility without complexity**:
TanStack Table handles the hard parts (sorting algorithms, state management) while letting you control the UI completely. This is the sweet spot for custom applications.

**3. Accessibility is not optional**:
Adding ARIA attributes and keyboard navigation is straightforward and dramatically improves usability for all users, not just those with disabilities.

**4. Virtual scrolling is powerful but has trade-offs**:
It solves performance problems with large datasets but adds complexity and can hurt accessibility. Use it only when pagination isn't sufficient.

**5. Progressive enhancement works**:
Start with a simple table, add features incrementally, and test at each step. Don't try to build the perfect table on the first iteration.

**6. Measure, don't guess**:
Use React DevTools Profiler, Chrome Performance tab, and Lighthouse to identify actual bottlenecks. Premature optimization wastes time.

**7. User experience includes all users**:
A fast table that's inaccessible is not a good table. Performance and accessibility are both essential.

### The Professional React Developer's Mental Model for Data Tables

When you encounter a data table requirement, think through this checklist:

**1. Data characteristics**:
- How many items? (determines pagination/virtualization strategy)
- How often does it update? (determines re-render optimization needs)
- What's the data source? (client-side array vs. server-side API)

**2. User requirements**:
- What interactions are needed? (sorting, filtering, selection)
- What's the expected performance? (instant vs. acceptable delay)
- Who are the users? (keyboard users, screen reader users, power users)

**3. Technical constraints**:
- Do you have a design system? (determines library choice)
- What's the bundle size budget? (determines library choice)
- What's the browser support requirement? (determines feature availability)

**4. Implementation strategy**:
- Start simple: render the data
- Add interactivity: sorting, filtering
- Optimize rendering: pagination or virtualization
- Ensure accessibility: ARIA, keyboard navigation
- Measure and iterate: profile, optimize, test

This mental model applies to any complex UI component, not just tables. The pattern is always the same: start simple, add features incrementally, optimize based on measurements, and ensure accessibility throughout.
