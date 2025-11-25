# Chapter 25: Data Tables and Complex UIs

## TanStack Table: flexible, headless tables

## The Challenge of Complex Data Display

Displaying a simple list of items is straightforward. But as applications grow, so does the complexity of the data we need to present. Users expect to be able to sort, filter, and paginate through thousands of rows of data without a hitch. Building this from scratch is a monumental task, fraught with performance pitfalls and state management nightmares.

This chapter will guide you through building a high-performance, feature-rich data table, starting from a naive implementation and progressively layering on professional-grade solutions.

### Phase 1: Establish the Reference Implementation

Our anchor example for this chapter will be a `ProductDashboard`. It's a common UI pattern: a table that displays a list of products with various attributes like name, category, price, and stock level.

We'll start by building this with the most basic tools available: standard HTML `<table>` elements and `Array.prototype.map`. This will serve as our baseline and will quickly reveal the limitations of a manual approach.

**Project Structure**:
```
src/
└── app/
    └── products/
        ├── page.tsx          <- Our main page component
        ├── ProductDashboard.tsx  <- The table component we will build
        └── data.ts           <- Mock data for our products
```

First, let's define some mock data.

```typescript
// src/app/products/data.ts

export type Product = {
  id: number;
  name: string;
  category: 'Electronics' | 'Books' | 'Clothing' | 'Home Goods';
  price: number;
  stock: number;
};

export const mockData: Product[] = [
  { id: 1, name: "Laptop Pro", category: "Electronics", price: 1499.99, stock: 35 },
  { id: 2, name: "The Great Novel", category: "Books", price: 19.99, stock: 150 },
  { id: 3, name: "Classic T-Shirt", category: "Clothing", price: 29.99, stock: 200 },
  { id: 4, name: "Smart Thermostat", category: "Home Goods", price: 199.00, stock: 80 },
  { id: 5, name: "Wireless Headphones", category: "Electronics", price: 249.50, stock: 60 },
  // ...imagine 95 more entries for a total of 100
];

// Let's generate a larger dataset for later use
export const createLargeMockData = (count: number): Product[] => {
    const categories: Product['category'][] = ['Electronics', 'Books', 'Clothing', 'Home Goods'];
    const data: Product[] = [];
    for (let i = 0; i < count; i++) {
        data.push({
            id: i + 1,
            name: `Product ${i + 1}`,
            category: categories[i % categories.length],
            price: parseFloat((Math.random() * 1000).toFixed(2)),
            stock: Math.floor(Math.random() * 500),
        });
    }
    return data;
};

// For now, we'll use a smaller set
export const defaultProducts = createLargeMockData(100);
```

Now, let's create our initial, naive `ProductDashboard` component.

```tsx
// src/app/products/ProductDashboard.tsx

'use client';

import React from 'react';
import { Product, defaultProducts } from './data';

// Iteration 0: The Naive Table
export default function ProductDashboard() {
  const [products] = React.useState<Product[]>(() => defaultProducts);

  return (
    <div className="container">
      <h1>Product Dashboard</h1>
      <table>
        <thead>
          <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Category</th>
            <th>Price</th>
            <th>Stock</th>
          </tr>
        </thead>
        <tbody>
          {products.map((product) => (
            <tr key={product.id}>
              <td>{product.id}</td>
              <td>{product.name}</td>
              <td>{product.category}</td>
              <td>${product.price.toFixed(2)}</td>
              <td>{product.stock}</td>
            </tr>
          ))}
        </tbody>
      </table>
      <style jsx>{`
        .container { padding: 2rem; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
      `}</style>
    </div>
  );
}
```

And finally, we render it on our page.

```tsx
// src/app/products/page.tsx

import ProductDashboard from './ProductDashboard';

export default function ProductsPage() {
  return <ProductDashboard />;
}
```

This component works. It renders a table of 100 products. But it's a house of cards.

### Diagnostic Analysis: The Failure of Maintainability

This isn't a runtime error, but an architectural one. Let's analyze its weaknesses.

**User Experience**:
- The user sees a static table. They cannot sort, filter, or paginate. If there were 10,000 products, the page would be unusably long.

**Developer Experience**:
- **Rigid Columns**: The headers (`<th>`) and the data cells (`<td>`) are completely disconnected. If you want to reorder columns, you have to change the code in two separate places. If you forget one, the data will be misaligned with the headers.
- **No Reusability**: The rendering logic is tightly coupled to the `Product` data structure. You can't reuse this table for anything else.
- **Manual Logic**: How would you add sorting? You'd need to add state for the sort key and direction, write a click handler for each `<th>`, and implement a sorting function. What about filtering? More state, an input element, and a filtering function. This component would quickly become a monolithic mess of state management.

**Let's parse this evidence**:

1.  **What the user experiences**: A static, non-interactive data dump.
2.  **What the developer experiences**: A brittle component that is difficult and risky to change.
3.  **Root cause identified**: The component mixes data logic (what to display) with presentation logic (how to display it in a `<table>`) in a rigid, inseparable way.
4.  **Why the current approach can't solve this**: Every new feature (sorting, filtering, pagination) requires adding more complex, intertwined state and logic directly into the component, leading to an unmaintainable final product.
5.  **What we need**: A way to separate the *logic* of a table (state management for sorting, filtering, columns, data) from its *presentation* (the actual HTML tags).

### Iteration 1: Introducing TanStack Table

This is where a **headless UI** library comes in. TanStack Table is a prime example. It's "headless" because it provides no UI components out of the box. Instead, it gives you a powerful hook (`useReactTable`) that manages all the complex state and logic, and returns helper functions and data structures that you can use to render *your own* markup.

You control the UI, TanStack Table controls the state.

Let's refactor our `ProductDashboard` to use it. First, install the library:

```bash
npm install @tanstack/react-table
```

Now, let's see the refactored component.

**Before** (Iteration 0): The naive, manual table.

```tsx
// src/app/products/ProductDashboard.tsx (excerpt)
// ...
  return (
    <table>
      <thead>
        <tr>
          <th>ID</th>
          {/* ... other headers */}
        </tr>
      </thead>
      <tbody>
        {products.map((product) => (
          <tr key={product.id}>
            <td>{product.id}</td>
            {/* ... other cells */}
          </tr>
        ))}
      </tbody>
    </table>
  );
// ...
```

**After** (Iteration 1): Using `useReactTable`.

```tsx
// src/app/products/ProductDashboard.tsx (Version 1)

'use client';

import React from 'react';
import { Product, defaultProducts } from './data';
import {
  useReactTable,
  getCoreRowModel,
  ColumnDef,
  flexRender,
} from '@tanstack/react-table';

// Iteration 1: Refactoring with TanStack Table
export default function ProductDashboard() {
  const [data] = React.useState<Product[]>(() => defaultProducts);

  // Step 1: Define your columns
  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [
      {
        accessorKey: 'id',
        header: 'ID',
        cell: info => info.getValue(),
      },
      {
        accessorKey: 'name',
        header: 'Product Name',
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
        cell: info => `$${(info.getValue() as number).toFixed(2)}`, // Format the price
      },
      {
        accessorKey: 'stock',
        header: 'Stock',
        cell: info => info.getValue(),
      },
    ],
    []
  );

  // Step 2: Create the table instance
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
  });

  // Step 3: Render the table using the instance's helpers
  return (
    <div className="container">
      <h1>Product Dashboard</h1>
      <table>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th key={header.id}>
                  {header.isPlaceholder
                    ? null
                    : flexRender(
                        header.column.columnDef.header,
                        header.getContext()
                      )}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map(row => (
            <tr key={row.id}>
              {row.getVisibleCells().map(cell => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
      <style jsx>{`
        .container { padding: 2rem; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
      `}</style>
    </div>
  );
}
```

### Analysis of the Improvement

The UI looks identical, but the underlying architecture is vastly superior.

1.  **Decoupled Columns**: The `columns` array is now the single source of truth. It defines the data accessor (`accessorKey`), the header text (`header`), and the cell rendering logic (`cell`). To reorder columns, you just reorder the objects in this array. The table headers and body will update automatically.
2.  **Declarative Logic**: We tell `useReactTable` *what* we want (our data and columns) and *what capabilities* we need (`getCoreRowModel()`). The hook handles the complex logic of creating headers, rows, and cells.
3.  **Render Control**: We still have 100% control over the markup. The `flexRender` utility is a small helper that intelligently renders our `header` and `cell` definitions. We could replace it with custom JSX if we wanted.

**Limitation Preview**: Our table is now well-structured and maintainable, but it's still just a static display. We haven't added any of the user-facing features like sorting, filtering, or pagination. That's our next challenge.

## Pagination, sorting, and filtering

## Iteration 2: Adding Interactivity

Our `ProductDashboard` is built on a solid foundation, but a user can't do anything with it. In this section, we'll leverage TanStack Table's state management capabilities to add sorting, filtering, and pagination.

### Adding Sorting

**Current Limitation**: Users cannot reorder the data based on a column's value, such as sorting by price from lowest to highest.

**New Scenario**: A user clicks on the "Price" header, expecting the table to sort the products by price.

To enable sorting, we need to do three things:
1.  Add state to track the current sorting configuration.
2.  Tell the table instance how to process sorted rows using `getSortedRowModel`.
3.  Add `onClick` handlers to our column headers to toggle sorting.

Let's update the `ProductDashboard`.

```tsx
// src/app/products/ProductDashboard.tsx (Version 2 - Sorting)

'use client';

import React from 'react';
import { Product, defaultProducts } from './data';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel, // Import
  ColumnDef,
  flexRender,
  SortingState, // Import
} from '@tanstack/react-table';

export default function ProductDashboard() {
  const [data] = React.useState<Product[]>(() => defaultProducts);
  const [sorting, setSorting] = React.useState<SortingState>([]); // Add sorting state

  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [
      // ... (same column definitions as before)
      { accessorKey: 'id', header: 'ID' },
      { accessorKey: 'name', header: 'Product Name' },
      { accessorKey: 'category', header: 'Category' },
      { accessorKey: 'price', header: 'Price', cell: info => `$${(info.getValue() as number).toFixed(2)}` },
      { accessorKey: 'stock', header: 'Stock' },
    ],
    []
  );

  const table = useReactTable({
    data,
    columns,
    state: { // Pass state to the table
      sorting,
    },
    onSortingChange: setSorting, // Provide a state updater
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(), // Add the sorting row model
  });

  return (
    <div className="container">
      <h1>Product Dashboard</h1>
      <table>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th key={header.id} onClick={header.column.getToggleSortingHandler()}>
                  {flexRender(
                    header.column.columnDef.header,
                    header.getContext()
                  )}
                  {/* Add a sort indicator */}
                  {{
                    asc: ' 🔼',
                    desc: ' 🔽',
                  }[header.column.getIsSorted() as string] ?? null}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map(row => (
            <tr key={row.id}>
              {row.getVisibleCells().map(cell => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
      {/* ... styles */}
    </div>
  );
}
```

**Verification**: Now, when you click on a column header, the table re-renders, sorted by that column. Clicking again reverses the sort order.

**React DevTools Evidence**:
- **Components Tab**: Select `ProductDashboard`.
- **State**: You will see a `sorting` state array.
- **Behavior**: When you click the "Price" header, the state changes to `[{ id: 'price', desc: false }]`. Click again, and it becomes `[{ id: 'price', desc: true }]`. The table instance uses this state to re-order the rows provided by `getRowModel()`.

### Adding Global Filtering

**Current Limitation**: Users can't search for a specific product. They have to manually scan the entire table.

**New Scenario**: A user wants to find all products with "Laptop" in the name. They type "Laptop" into a search box, and the table updates to show only matching rows.

This requires a similar process: add state for the filter, provide the `getFilteredRowModel`, and connect an input element to update the filter state.

```tsx
// src/app/products/ProductDashboard.tsx (Version 3 - Filtering)
// ... imports
import {
  // ... other imports
  getFilteredRowModel, // Import
  // ... other types
  GlobalFilter,
} from '@tanstack/react-table';

export default function ProductDashboard() {
  const [data] = React.useState<Product[]>(() => defaultProducts);
  const [sorting, setSorting] = React.useState<SortingState>([]);
  const [globalFilter, setGlobalFilter] = React.useState(''); // Add filter state

  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [ /* ... same columns ... */ ],
    []
  );

  const table = useReactTable({
    data,
    columns,
    state: {
      sorting,
      globalFilter, // Pass filter state
    },
    onSortingChange: setSorting,
    onGlobalFilterChange: setGlobalFilter, // Provide updater
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(), // Add filter row model
  });

  return (
    <div className="container">
      <h1>Product Dashboard</h1>
      {/* Add a filter input */}
      <input
        value={globalFilter ?? ''}
        onChange={e => setGlobalFilter(e.target.value)}
        className="filter-input"
        placeholder="Search all columns..."
      />
      <table>
        {/* ... same table markup ... */}
      </table>
      <style jsx>{`
        /* ... other styles ... */
        .filter-input {
          margin-bottom: 1rem;
          padding: 0.5rem;
          font-size: 1rem;
        }
      `}</style>
    </div>
  );
}
```

**Verification**: As you type in the search box, the table rows dynamically update to show only the products that match your query across all columns.

### Adding Pagination

**Current Limitation**: The table shows all 100 items at once. For a dataset with thousands of items, this would be overwhelming and slow.

**New Scenario**: The user should see a manageable number of items per page (e.g., 10) and be able to navigate between pages.

This follows the same pattern: add pagination state, include the `getPaginationRowModel`, and render pagination controls.

```tsx
// src/app/products/ProductDashboard.tsx (Version 4 - Pagination)
// ... imports
import {
  // ... other imports
  getPaginationRowModel, // Import
} from '@tanstack/react-table';

export default function ProductDashboard() {
  // ... states for data, sorting, globalFilter
  const [data] = React.useState<Product[]>(() => defaultProducts);
  const [sorting, setSorting] = React.useState<SortingState>([]);
  const [globalFilter, setGlobalFilter] = React.useState('');

  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [ /* ... same columns ... */ ],
    []
  );

  const table = useReactTable({
    data,
    columns,
    state: {
      sorting,
      globalFilter,
    },
    onSortingChange: setSorting,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(), // Add pagination row model
  });

  return (
    <div className="container">
      <h1>Product Dashboard</h1>
      <input
        value={globalFilter ?? ''}
        onChange={e => setGlobalFilter(e.target.value)}
        className="filter-input"
        placeholder="Search all columns..."
      />
      <table>
        {/* ... same table markup ... */}
      </table>
      
      {/* Add Pagination Controls */}
      <div className="pagination">
        <button
          onClick={() => table.setPageIndex(0)}
          disabled={!table.getCanPreviousPage()}
        >
          {'<<'}
        </button>
        <button
          onClick={() => table.previousPage()}
          disabled={!table.getCanPreviousPage()}
        >
          {'<'}
        </button>
        <span>
          Page{' '}
          <strong>
            {table.getState().pagination.pageIndex + 1} of{' '}
            {table.getPageCount()}
          </strong>
        </span>
        <button
          onClick={() => table.nextPage()}
          disabled={!table.getCanNextPage()}
        >
          {'>'}
        </button>
        <button
          onClick={() => table.setPageIndex(table.getPageCount() - 1)}
          disabled={!table.getCanNextPage()}
        >
          {'>>'}
        </button>
      </div>
      <style jsx>{`
        /* ... other styles ... */
        .pagination { margin-top: 1rem; display: flex; gap: 0.5rem; align-items: center; }
        .pagination button { padding: 0.5rem; }
      `}</style>
    </div>
  );
}
```

**Verification**: The table now displays only 10 rows at a time (the default page size). The pagination controls allow you to navigate through all the data, and the buttons correctly disable themselves on the first and last pages.

**Limitation Preview**: Our table is now fully interactive for a dataset of 100 items. But what happens when we scale up to 10,000 items? Even with pagination, the browser still has to process the entire 10,000-item array in JavaScript for every sort or filter operation. More importantly, if we weren't using pagination, the DOM would be crushed. Let's explore that failure mode next.

## Virtual scrolling for large lists

## Iteration 3: Taming Large Datasets with Virtualization

Our `ProductDashboard` is feature-complete for small datasets. Now, we'll expose its critical performance flaw by feeding it a massive amount of data.

**Current Limitation**: The component's performance is directly tied to the total number of rows in the dataset. While pagination helps the UI, JavaScript operations like filtering and sorting still have to churn through the entire array. If we were to render all rows at once, the browser would freeze.

**New Scenario**: Our product catalog has grown from 100 items to 10,000. Let's see what happens.

First, let's modify our component to use a larger dataset and temporarily remove pagination to demonstrate the core problem of rendering too many DOM nodes.

```tsx
// src/app/products/ProductDashboard.tsx (Failing Version - 10k rows)

// ... imports
import { Product, createLargeMockData } from './data'; // Use the generator

export default function ProductDashboard() {
  // Use 10,000 items!
  const [data] = React.useState<Product[]>(() => createLargeMockData(10000));
  
  // ... states for sorting, filtering
  
  const table = useReactTable({
    data,
    columns,
    // ... state and updaters
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    // NO getPaginationRowModel for this test
  });

  return (
    <div className="container">
      {/* ... filter input ... */}
      <table>
        {/* ... thead ... */}
        <tbody>
          {/* This will now try to render 10,000 <tr> elements */}
          {table.getRowModel().rows.map(row => (
            <tr key={row.id}>
              {row.getVisibleCells().map(cell => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
      {/* ... no pagination controls ... */}
    </div>
  );
}
```

### Diagnostic Analysis: Reading the Failure

Running this code will cause a noticeable, and often severe, degradation in performance.

**Browser Behavior**:
- The page takes a very long time to load. The browser tab may become unresponsive or "hang".
- Scrolling is extremely laggy and "janky". The scrollbar may jump or not respond immediately.
- Typing in the filter input is slow. There's a visible delay between typing a character and seeing the table update.

**Browser DevTools Evidence (Performance Tab)**:
- **Flame Chart**: Recording a page load will show a very long "Scripting" task, potentially lasting several seconds. Inside, you'll see thousands of calls related to React rendering and DOM manipulation.
- **Main Thread Work**: The main thread will be blocked for a long duration, preventing any user interaction.

**Browser DevTools Evidence (Elements Tab)**:
- If you inspect the `<tbody>`, you will see 10,000 `<tr>` elements rendered in the DOM. This is the smoking gun.

**Let's parse this evidence**:

1.  **What the user experiences**: A slow, unresponsive, and unusable application.
2.  **What the console reveals**: No errors, just extreme sluggishness. This is a performance bug, not a logic error.
3.  **What DevTools shows**: The root cause is the sheer number of DOM nodes. The browser's rendering engine cannot efficiently handle painting, styling, and managing event listeners for tens of thousands of individual elements.
4.  **Root cause identified**: We are rendering every single row in our dataset to the DOM, even though the user can only see a handful at a time.
5.  **Why the current approach can't solve this**: React is fast, but the DOM is slow. No amount of React optimization can make rendering 10,000 complex table rows a fast operation.
6.  **What we need**: A way to render *only the rows currently visible in the viewport* and fake the existence of the rest. This technique is called **virtualization** or **windowing**.

### Iteration 4: Implementing Virtual Scrolling

We'll use another library from the TanStack ecosystem, `@tanstack/react-virtual`, which is designed to work seamlessly with TanStack Table.

First, install it:

```bash
npm install @tanstack/react-virtual
```

Now, let's integrate it into our `ProductDashboard`. The core idea is to:
1.  Wrap our table body in a container with a fixed height and `overflow: scroll`.
2.  Use the `useVirtualizer` hook to calculate which rows should be visible.
3.  Use CSS transforms to position the rendered rows correctly within the scrollable container, creating the illusion of a full list.

```tsx
// src/app/products/ProductDashboard.tsx (Version 5 - Virtualized)

'use client';

import React from 'react';
import { Product, createLargeMockData } from './data';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  ColumnDef,
  flexRender,
  SortingState,
} from '@tanstack/react-table';
import { useVirtualizer } from '@tanstack/react-virtual'; // Import

export default function ProductDashboard() {
  const [data] = React.useState<Product[]>(() => createLargeMockData(10000));
  const [sorting, setSorting] = React.useState<SortingState>([]);
  const [globalFilter, setGlobalFilter] = React.useState('');

  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [ /* ... same columns ... */ ],
    []
  );

  const table = useReactTable({
    data,
    columns,
    state: { sorting, globalFilter },
    onSortingChange: setSorting,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  });

  const tableContainerRef = React.useRef<HTMLDivElement>(null);
  const { rows } = table.getRowModel();

  const rowVirtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => tableContainerRef.current,
    estimateSize: () => 34, // Estimate row height for performance
    overscan: 5, // Render 5 extra items on each side
  });

  return (
    <div className="container">
      <h1>Product Dashboard (10,000 items)</h1>
      <input
        value={globalFilter ?? ''}
        onChange={e => setGlobalFilter(e.target.value)}
        className="filter-input"
        placeholder="Search all columns..."
      />
      
      {/* The scrollable container */}
      <div ref={tableContainerRef} className="table-container">
        <table style={{ width: '100%' }}>
          <thead>
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th key={header.id} onClick={header.column.getToggleSortingHandler()}>
                    {flexRender(header.column.columnDef.header, header.getContext())}
                    {{ asc: ' 🔼', desc: ' 🔽' }[header.column.getIsSorted() as string] ?? null}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          {/* The virtualized body */}
          <tbody style={{ height: `${rowVirtualizer.getTotalSize()}px`, position: 'relative' }}>
            {rowVirtualizer.getVirtualItems().map(virtualRow => {
              const row = rows[virtualRow.index];
              return (
                <tr
                  key={row.id}
                  style={{
                    position: 'absolute',
                    top: 0,
                    left: 0,
                    width: '100%',
                    height: `${virtualRow.size}px`,
                    transform: `translateY(${virtualRow.start}px)`,
                  }}
                >
                  {row.getVisibleCells().map(cell => (
                    <td key={cell.id}>
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
      <style jsx>{`
        /* ... other styles ... */
        .table-container {
          height: 600px; /* Crucial for virtualization */
          overflow: auto;
          border: 1px solid #ddd;
        }
        th {
          background-color: #f2f2f2;
          position: sticky; /* Keep headers visible */
          top: 0;
        }
      `}</style>
    </div>
  );
}
```

### Verification and Performance Improvement

**Expected vs. Actual Improvement**:

-   **Browser Behavior**: The page loads almost instantly. Scrolling is perfectly smooth, even with 10,000 items. Filtering is instantaneous.
-   **Performance Metrics (Profiler)**: The initial render time is now trivial, comparable to rendering just a dozen rows.
-   **DOM Inspector**: If you inspect the `<tbody>`, you will see only about 15-20 `<tr>` elements at any given time, no matter how far you scroll. The `transform: translateY(...)` style on each `<tr>` is rapidly changing as you scroll, moving the few rendered rows into the correct visual position.

We have successfully decoupled our application's performance from its dataset size.

**Limitation Preview**: Our table is now incredibly performant, but in the process of optimizing it with virtualization, we've potentially broken accessibility. Screen readers and keyboard navigation rely on a standard DOM structure, which our absolutely positioned `<tr>` elements disrupt. Our next and final task is to make our complex UI accessible.

## Accessible, keyboard-navigable interfaces

## Iteration 5: Ensuring Accessibility (A11y)

Performance is critical, but not at the expense of usability for all users. Our virtualized table is fast, but it presents challenges for assistive technologies like screen readers and for users who rely on keyboard navigation.

**Current Limitation**:
-   **Screen Readers**: A screen reader might not correctly interpret our `<tbody>` with absolutely positioned `<tr>` elements as a contiguous data table. It may not announce row/column counts or headers correctly.
-   **Keyboard Navigation**: Tabbing through the table might be unpredictable. Focus management within a virtualized grid is non-trivial.

**New Scenario**: A user with a visual impairment uses a screen reader to navigate the product data. Another user navigates the site using only their keyboard. Both should have a seamless experience.

### The Principles of Accessible Data Tables

To make our table accessible, we need to provide semantic information to the browser using ARIA (Accessible Rich Internet Applications) roles and attributes. For a data grid, the key roles are:

-   `role="grid"`: On the `<table>` element. This tells assistive tech that this is a data grid, not a simple layout table.
-   `role="row"`: On each `<tr>`.
-   `role="columnheader"`: On each `<th>`.
-   `role="gridcell"`: On each `<td>`.

TanStack Table's default markup with `<table>`, `<tr>`, `<th>`, and `<td>` already provides good semantics. Our main job is to enhance it, especially for sorting and within our virtualized context.

### Implementing ARIA Attributes

Let's add the necessary attributes to our `ProductDashboard`.

```tsx
// src/app/products/ProductDashboard.tsx (Version 6 - Accessible)

// ... (imports are the same as the virtualized version)

export default function ProductDashboard() {
  // ... (all state and hooks are the same)
  const [data] = React.useState<Product[]>(() => createLargeMockData(10000));
  const [sorting, setSorting] = React.useState<SortingState>([]);
  const [globalFilter, setGlobalFilter] = React.useState('');

  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [ /* ... same columns ... */ ],
    []
  );

  const table = useReactTable({ /* ... same options ... */ });

  const tableContainerRef = React.useRef<HTMLDivElement>(null);
  const { rows } = table.getRowModel();
  const rowVirtualizer = useVirtualizer({ /* ... same options ... */ });

  return (
    <div className="container">
      {/* ... h1 and input ... */}
      <div ref={tableContainerRef} className="table-container">
        <table role="grid"> {/* Add role="grid" */}
          <thead>
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id} role="row"> {/* Add role="row" */}
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    role="columnheader" /* Add role="columnheader" */
                    aria-sort={ // Announce sort direction
                      header.column.getIsSorted() === 'asc' ? 'ascending' :
                      header.column.getIsSorted() === 'desc' ? 'descending' :
                      'none'
                    }
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    {/* ... flexRender and sort indicator ... */}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody style={{ height: `${rowVirtualizer.getTotalSize()}px`, position: 'relative' }}>
            {rowVirtualizer.getVirtualItems().map(virtualRow => {
              const row = rows[virtualRow.index];
              return (
                <tr
                  key={row.id}
                  role="row" /* Add role="row" */
                  data-index={virtualRow.index} // Useful for focus management
                  ref={node => rowVirtualizer.measureElement(node)} // Helps with dynamic row heights
                  style={{
                    position: 'absolute',
                    top: 0,
                    left: 0,
                    width: '100%',
                    height: `${virtualRow.size}px`,
                    transform: `translateY(${virtualRow.start}px)`,
                  }}
                >
                  {row.getVisibleCells().map(cell => (
                    <td key={cell.id} role="gridcell"> {/* Add role="gridcell" */}
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
      {/* ... styles ... */}
    </div>
  );
}
```

### Verification with Accessibility Tools

1.  **Screen Reader**: Using a tool like NVDA, JAWS, or VoiceOver, you'll now hear the table announced as a "grid with X columns and 10,000 rows." When you navigate to a header, it will announce its name and sort state (e.g., "Price, column header, sorting ascending").
2.  **Browser DevTools (Accessibility Tab)**: Inspecting an element will now show its computed ARIA role. The `<table>` will have `role: grid`, and so on. This confirms the browser is parsing our semantic hints correctly.

### Keyboard Navigation Considerations

True grid-like keyboard navigation (using arrow keys to move between cells) is a complex topic that often requires a custom hook to manage focus. However, the most crucial aspect is ensuring that interactive elements are reachable.

-   **Headers**: Our `<th>` elements are clickable, so they should be focusable. We can add `tabIndex={0}` to them to ensure they are part of the tab order.
-   **Focus Management**: When data changes (e.g., after filtering), focus can be lost. A robust solution would involve programmatically setting focus back to a logical place, like the filter input or the table container.

For many applications, ensuring the primary interactive controls (filter, sort headers, pagination) are keyboard accessible provides the most value. Full cell-by-cell navigation is an advanced feature for spreadsheet-like applications.

### Common Failure Modes and Their Signatures

#### Symptom: Table doesn't update when parent component's state changes.

**Console pattern**: No errors. The UI is just stale.
**DevTools clues**: The `ProductDashboard` component re-renders, but the table content remains the same.
**Root cause**: The `data` and `columns` props passed to `useReactTable` are not stable. If you define them inside your component without `React.useMemo`, they get a new reference on every render, causing the table instance to reset internally in unexpected ways.
**Solution**: Always wrap your `columns` definition in `useMemo` and ensure the `data` array has a stable reference unless it has actually changed.

#### Symptom: Virtualized list has large blank gaps or jumps erratically when scrolling.

**Browser behavior**: Scrolling is not smooth; content appears to "pop in" late.
**DevTools clues**: The `transform: translateY(...)` values are changing in large, inconsistent increments.
**Root cause**: The `estimateSize` function in `useVirtualizer` is significantly different from the actual rendered height of the rows. This causes the virtualizer's calculations to be off.
**Solution**: Provide a more accurate `estimateSize`. If row heights are dynamic, implement a measurement strategy using the `measureElement` ref callback, as shown in the accessible example.

#### Symptom: Sorting or filtering doesn't work at all.

**Console pattern**: No errors. Clicking headers or typing in the filter does nothing.
**DevTools clues**: The `sorting` or `globalFilter` state in your component updates correctly, but the table doesn't re-render with the new data.
**Root cause**: You forgot to include the necessary row model in the `useReactTable` options. Forgetting `getSortedRowModel()` will disable sorting; forgetting `getFilteredRowModel()` will disable filtering.
**Solution**: Ensure the table instance is created with all the row models you need: `getCoreRowModel`, `getSortedRowModel`, `getFilteredRowModel`, `getPaginationRowModel`, etc.

### The Journey: From Problem to Solution

| Iteration | Failure Mode                               | Technique Applied          | Result                                       | Performance Impact                               |
| :-------- | :----------------------------------------- | :------------------------- | :------------------------------------------- | :----------------------------------------------- |
| 0         | Brittle, unmaintainable code               | Naive `<table>`            | Works, but is hard to change                 | Baseline                                         |
| 1         | Logic and presentation tightly coupled     | TanStack Table (Headless)  | Maintainable, decoupled column definitions   | Negligible overhead                              |
| 2         | No user interactivity                      | Table State Management     | Sorting, filtering, and pagination work      | JS computation on client for all data            |
| 3         | Browser freezes with large datasets        | Virtualization             | Renders 10,000+ rows with ease               | Drastic improvement: O(1) DOM nodes              |
| 4         | Poor experience for assistive technologies | ARIA Roles & Attributes    | Accessible to screen readers and keyboards   | No performance impact                            |

### Final Implementation

Here is the complete, production-ready `ProductDashboard` component that is performant, interactive, and accessible.

```tsx
// src/app/products/ProductDashboard.tsx (Final Version)

'use client';

import React from 'react';
import { Product, createLargeMockData } from './data';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  ColumnDef,
  flexRender,
  SortingState,
} from '@tanstack/react-table';
import { useVirtualizer } from '@tanstack/react-virtual';

export default function ProductDashboard() {
  const [data] = React.useState<Product[]>(() => createLargeMockData(10000));
  const [sorting, setSorting] = React.useState<SortingState>([]);
  const [globalFilter, setGlobalFilter] = React.useState('');

  const columns = React.useMemo<ColumnDef<Product>[]>(
    () => [
      { accessorKey: 'id', header: 'ID', size: 60 },
      { accessorKey: 'name', header: 'Product Name', size: 250 },
      { accessorKey: 'category', header: 'Category', size: 150 },
      { accessorKey: 'price', header: 'Price', size: 100, cell: info => `$${(info.getValue() as number).toFixed(2)}` },
      { accessorKey: 'stock', header: 'Stock', size: 100 },
    ],
    []
  );

  const table = useReactTable({
    data,
    columns,
    state: { sorting, globalFilter },
    onSortingChange: setSorting,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  });

  const tableContainerRef = React.useRef<HTMLDivElement>(null);
  const { rows } = table.getRowModel();

  const rowVirtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => tableContainerRef.current,
    estimateSize: () => 34,
    overscan: 10,
  });

  return (
    <div className="container">
      <h1>Product Dashboard (10,000 items)</h1>
      <input
        value={globalFilter ?? ''}
        onChange={e => setGlobalFilter(e.target.value)}
        className="filter-input"
        placeholder="Search all columns..."
      />
      
      <div ref={tableContainerRef} className="table-container">
        <table role="grid" style={{ width: table.getTotalSize() }}>
          <thead>
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id} role="row">
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    role="columnheader"
                    aria-sort={header.column.getIsSorted() === 'asc' ? 'ascending' : header.column.getIsSorted() === 'desc' ? 'descending' : 'none'}
                    onClick={header.column.getToggleSortingHandler()}
                    style={{ width: header.getSize() }}
                  >
                    {flexRender(header.column.columnDef.header, header.getContext())}
                    {{ asc: ' 🔼', desc: ' 🔽' }[header.column.getIsSorted() as string] ?? null}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody style={{ height: `${rowVirtualizer.getTotalSize()}px`, position: 'relative' }}>
            {rowVirtualizer.getVirtualItems().map(virtualRow => {
              const row = rows[virtualRow.index];
              return (
                <tr
                  key={row.id}
                  role="row"
                  data-index={virtualRow.index}
                  ref={node => rowVirtualizer.measureElement(node)}
                  style={{
                    position: 'absolute',
                    top: 0,
                    left: 0,
                    width: '100%',
                    height: `${virtualRow.size}px`,
                    transform: `translateY(${virtualRow.start}px)`,
                  }}
                >
                  {row.getVisibleCells().map(cell => (
                    <td key={cell.id} role="gridcell" style={{ width: cell.column.getSize() }}>
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
      <style jsx>{`
        .container { padding: 2rem; font-family: sans-serif; }
        .filter-input { margin-bottom: 1rem; padding: 0.5rem; font-size: 1rem; width: 100%; box-sizing: border-box; }
        .table-container { height: 600px; overflow: auto; border: 1px solid #ccc; }
        table { border-collapse: collapse; }
        th, td { border-bottom: 1px solid #ddd; border-right: 1px solid #ddd; padding: 8px; text-align: left; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
        th { background-color: #f2f2f2; position: sticky; top: 0; z-index: 1; cursor: pointer; user-select: none; }
      `}</style>
    </div>
  );
}
```

### Lessons Learned

Building complex UIs like data tables is a journey of layering concerns. We started with basic rendering, then added state management for interactivity, addressed performance bottlenecks with virtualization, and finally ensured the result was accessible. By using a headless library like TanStack Table, we retained full control over our markup and styling at every step, allowing us to solve each problem with the right tool without being locked into a specific UI framework.
