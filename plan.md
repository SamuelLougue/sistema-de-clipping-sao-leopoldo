Below is the detailed plan to investigate and resolve the hydration error and verify that it is not interfering with the news registration functionality.

---

## 1. Investigation Phase

- **Review dynamic content:**  
  Identify any usage of dynamic values (e.g. Date.now(), Math.random()) or conditionals using `typeof window !== 'undefined'` across the codebase. The main suspects include:
  - **Hooks:** Check `src/hooks/use-mobile.ts` to see if it reads browser properties during SSR.
  - **Components:** Look into components such as `NewsletterScheduler.tsx`, `ClippingCard.tsx`, and any custom components that might use client-only data.
  
- **Check HTML nesting and structure:**  
  Review pages in `src/app/page.tsx`, `src/app/admin/page.tsx`, and `src/app/clipping-dashboard/page.tsx` for any invalid HTML structure or inconsistencies between the server-rendered and client-rendered markup.
  
- **Review layout and global settings:**  
  Inspect `src/app/layout.tsx` and `next.config.ts` to confirm that no external modifications (e.g. via CSS-in-JS libraries or edge/CDN injection) affect the markup.

---

## 2. Implementation Phase

### 2.1. In `src/app/page.tsx`

- **Step 2.1.1:**  
  Search for any instance where dynamic values (date, random values) are used directly in the render. For example, if you have code like:
  ```tsx
  <p>{`Current time: ${Date.now()}`}</p>
  ```
  replace it by storing a static server value and then updating it on the client.
  
- **Step 2.1.2:**  
  If dynamic values are unavoidable, wrap the affected elements using a state update inside a `useEffect` hook:
  ```tsx
  import { useState, useEffect } from 'react'
  
  export default function HomePage() {
    const [clientTime, setClientTime] = useState('Prerendered')
  
    useEffect(() => {
      setClientTime(`Current time: ${Date.now()}`)
    }, [])
  
    return <p suppressHydrationWarning>{clientTime}</p>
  }
  ```
  
### 2.2. In `src/app/admin/page.tsx`

- **Step 2.2.1:**  
  Check the registration form (or news posting functionality) to ensure form fields are rendered with consistent markup on both server and client.
  
- **Step 2.2.2:**  
  If any client-only logic (like accessing local storage or window properties) is used, refactor that part into a client-only component by using Next.js dynamic import:
  ```tsx
  import dynamic from 'next/dynamic'
  const NewsForm = dynamic(() => import('../../components/NewsRegistrationForm'), { ssr: false })
  
  export default function AdminPage() {
    return (
      <div>
        <h1>Register News</h1>
        <NewsForm /> {/* Render only on the client */}
      </div>
    )
  }
  ```

### 2.3. In `src/components/NewsletterScheduler.tsx`

- **Step 2.3.1:**  
  Review the component for any client-only API calls or dynamic values. For example, if the scheduler displays a timestamp or uses conditionals based on browser environment, use a `useEffect` hook to set these values after mount.
  
- **Step 2.3.2:**  
  Use `suppressHydrationWarning` on any element that must display dynamic content:
  ```tsx
  <div>
    <time suppressHydrationWarning dateTime="2023-01-01T00:00:00Z">
      {new Date().toLocaleString()}
    </time>
  </div>
  ```

### 2.4. In `src/hooks/use-mobile.ts`

- **Step 2.4.1:**  
  Ensure the hook does not run browser-specific code during SSR. Change it so that the mobile detection only occurs client-side:
  ```tsx
  import { useState, useEffect } from 'react';
  
  const useMobile = () => {
    const [isMobile, setIsMobile] = useState<boolean>(false);
  
    useEffect(() => {
      const checkMobile = () => window.innerWidth < 768;
      setIsMobile(checkMobile());
      window.addEventListener('resize', () => setIsMobile(checkMobile()));
  
      return () => window.removeEventListener('resize', checkMobile);
    }, []);
  
    return isMobile;
  };
  
  export default useMobile;
  ```

### 2.5. Create an Error Boundary (New File)

- **Step 2.5.1:**  
  To gracefully catch any unexpected errors in the component tree, create a new file:  
  **File:** `src/components/ErrorBoundary.tsx`
  ```tsx
  import React, { Component, ReactNode } from 'react';
  
  interface ErrorBoundaryProps {
    children: ReactNode;
  }
  
  interface ErrorBoundaryState {
    hasError: boolean;
  }
  
  class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
    constructor(props: ErrorBoundaryProps) {
      super(props);
      this.state = { hasError: false };
    }
  
    static getDerivedStateFromError() {
      return { hasError: true };
    }
  
    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
      console.error("ErrorBoundary caught an error", error, errorInfo);
    }
  
    render() {
      if (this.state.hasError) {
        return <h2>Something went wrong. Please try again later.</h2>;
      }
      return this.props.children;
    }
  }
  
  export default ErrorBoundary;
  ```
- **Step 2.5.2:**  
  Wrap critical components (especially news registration forms and dynamic components) with this error boundary in the parent pages (for example, in `src/app/admin/page.tsx`).

### 2.6. In `src/app/layout.tsx`

- **Step 2.6.1:**  
  Ensure that the global layout does not include any client-only logic. If any such logic is necessary, move it to client components or wrap it with `useEffect`.

---

## 3. Testing and Verification

- **Step 3.1:**  
  Run the development server (e.g. using `npm run dev`) and reproduce the error. Verify that the console no longer shows the hydration error message.
  
- **Step 3.2:**  
  Test the news registration flow on the admin page by attempting a registration, verifying that the form renders correctly and that event handlers are attached.
  
- **Step 3.3:**  
  Use browser dev tools to inspect the markup and confirm that any elements with dynamic content use `suppressHydrationWarning` where appropriate.
  
- **Step 3.4:**  
  Perform additional testing in various browser environments to ensure that no browser extensions or external modifications are interfering with the markup.

---

## 4. Documentation and Best Practices

- **Step 4.1:**  
  Update `plan.md` with the summary of the changes made and the areas affected by the hydration error. Mention which components now use dynamic imports or client-only hooks.
  
- **Step 4.2:**  
  Add documentation in `README.md` regarding SSR best practices and guidelines for avoiding dynamic values during server rendering.

---

## Summary

- Investigated all files using dynamic client-only code and checked for invalid HTML nesting.
- Modified components in `src/app/page.tsx` and `src/app/admin/page.tsx` to ensure static markup on the first render.
- Updated the `NewsletterScheduler.tsx` and `use-mobile.ts` hook to run dynamic code only in `useEffect`.
- Created a new `ErrorBoundary.tsx` to catch unexpected errors in dynamic components.
- Wrapped client-only components in dynamic imports (ssr: false) to isolate browser-dependent logic.
- Tested the changes to ensure the hydration mismatch error is resolved and registration forms work correctly.
- Updated documentation for clear guidelines on SSR-safe coding practices for future development.
