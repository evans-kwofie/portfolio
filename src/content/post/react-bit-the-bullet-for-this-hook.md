---
title: React bit the bullet for this hook (React Series)
description: React 19’s 'use' API allows for conditional data fetching and surgical context subscriptions, finally breaking the most restrictive Rule of Hooks.
publishDate: December 24, 2025
tags: ["react", "web development", "javascript", "react-19"]
draft: false
---

Depending on how long you have been building with React, you've likely come across the strict limitations of the Hook system. For years, one rule in particular made development unnecessarily difficult and often forced re-render heavy architectures:

**"Don’t call Hooks inside loops, conditions, or nested functions."**

This meant that if you had a hook that should only fire based on a specific condition, you were out of luck. You had to execute it at the top level regardless, often passing "disabled" flags into hooks or creating extra wrapper components just to isolate the logic.

```jsx
// ❌ The "Old" Way: Subscribing at the top level regardless of need
function UserDashboard({ userId, isProfileVisible }) {
	// We HAVE to call this here to follow the "Top Level" rule
	// Even if isProfileVisible is false, this component is now
	// subscribed to all UserContext changes.
	const user = useContext(UserContext);

	// If we had a custom hook, we'd often pass a "skip" or "disabled" flag
	// internally just to prevent it from doing work, but it's still called!
	const analytics = useAnalytics(userId, { skip: !isProfileVisible });

	if (!isProfileVisible) {
		return <p>Dashboard is hidden</p>;
	}

	return (
		<div>
			<h1>Welcome, {user.name}</h1>
			<button onClick={() => analytics.track("View")}>Track View</button>
		</div>
	);
}
```

### The Problem with Top-Level Subscriptions

If you get an "ick" from passing around a bunch of props or creating "Wrapper Components" just to avoid a hook call, you aren't alone.

Previously, if you needed to access a Context (like `ThemeContext` or `UserPermissions`), you had to call `useContext` at the very top. This meant the component was **forced** to subscribe to that context and re-render every time it changed, even if the component wasn't currently using that data due to an `if` statement later in the code.

---

## Enter the `use` API

The React team introduced the `use` API, and while it looks like a hook, it is better understood as a resource unwrapper or getter. What this means is that it allows you to obtain the value from an action or operation, usually without having to peek behind the covers. The use API simply asks React for the value inside a resource and lets React decide when that value is available for use (no pun intended).

It has two superpowers, which we’ll dive into below:

### 1. Unwrapping Promises

The first superpower of `use` is that it allows you to unwrap an asynchronous resource (like a Promise) directly in your render logic. In the past, you needed `useEffect` and `useState` to fetch data as shown in the example below:

```tsx
function ItemList() {
	const [items, setItems] = useState<Item[]>([]);
	const [isLoading, setIsLoading] = useState(true);
	const [error, setError] = useState<string | null>(null);

	useEffect(() => {
		const fetchItems = async () => {
			setIsLoading(true);
			try {
				const data = await someQuery();
				setItems(data);
			} catch (err) {
				setItems([]);
				setError(err instanceof Error ? err.message : "Unknown error");
			} finally {
				setIsLoading(false);
			}
		};

		fetchItems();
	}, []);

	if (isLoading) return <p>Loading...</p>;
	if (error) return <p>Error: {error}</p>;

	return (
		<ul>
			{items.map((item) => (
				<li key={item.id}>{item.name}</li>
			))}
		</ul>
	);
}
```

Now, you can just pass your fetch promise to `use`. The component will "suspend" until the promise resolves, and React handles the rest.

```tsx
import { use, Suspense } from "react";

function ItemList({ itemsPromise }: { itemsPromise: Promise<Item[]> }) {
	// Component "suspends" here until the promise resolves
	const items = use(itemsPromise);

	return (
		<ul>
			{items.map((item) => (
				<li key={item.id}>{item.name}</li>
			))}
		</ul>
	);
}

// Parent component creates a stable promise and wraps with Suspense
function App() {
	const itemsPromise = fetchItems(); // Created outside render or cached

	return (
		<Suspense fallback={<p>Loading...</p>}>
			<ItemList itemsPromise={itemsPromise} />
		</Suspense>
	);
}
```

Notice how much cleaner this is. Instead of juggling three pieces of state (`items`, `isLoading`, `error`) and managing them inside a `useEffect`, we simply pass a promise to `use` and let React's Suspense architecture handle the loading state declaratively. The component reads the data as if it were synchronous: no conditionals, no loading checks, just the happy path.

For error handling, you wrap the component with an Error Boundary, keeping your component focused on rendering data rather than managing failure states.

The `useEffect` approach in the first example also has well-documented pitfalls: race conditions, stale closures, and the infamous "fetch on mount" pattern that leads to waterfalls. Though libraries like TanStack Query exist, with `use`, React provides a first-class primitive that sidesteps many of these problems entirely. This means you can rely on React alone for data fetching in smaller projects, and reach for TanStack Query or SWR when you need their extra features.

### 2. The Rule Breaker: Conditional Context

The second superpower, and the one that solves our "Rules of Hooks" headache, is the ability to read Context conditionally.

Because `use` is an operator and not a standard hook, you can call it inside if blocks or after early returns. This changes everything for performance and code cleanliness.

```tsx
function SettingsPanel({ isAdmin }) {
	if (!isAdmin) {
		return <p>Access Denied</p>;
	}

	// ✅ Valid React 19 code:
	// We only subscribe to the AdminContext if the condition is met.
	const adminSettings = use(AdminContext);

	return <div>{adminSettings.secretKey}</div>;
}
```

## Why This Matters

By allowing `use` to bypass the traditional top-level rule, React 19 provides several major benefits:

- **Surgical Subscriptions:** You only subscribe to Context when you actually need it. If your code doesn't reach the `use(Context)` line, the component won't re-render when that context changes.

- **Reduced Prop Drilling:** You don't have to pass data through three layers just to avoid calling a hook in a child component.

- **Dynamic Composition:** You can now compose your UI more naturally, reading data exactly where and when it becomes relevant.

### A Critical "Gotcha": Stable Promises

While `use` is powerful, it comes with a warning. If you pass a Promise to `use`, that promise must be stable. If you create the promise inside the component's render body, you will trigger an infinite loop:

1. Component renders and creates a new `Promise()`.
2. `use(promise)` suspends the component.
3. Promise resolves, triggering a re-render.
4. Component renders again, creating a brand new `Promise()`.
5. Loop repeats forever.

Always ensure your promises are created outside the component or memoized via a cache.

### A Note on the "Rules"

It is important to remember that the "Rules of Hooks" still apply to `useState`, `useEffect`, and your custom hooks. You still can't put a `useState` inside an `if` block. Think of `use` not as a hook that breaks the rules, but as a new category of operator that gives us the flexibility we've been asking for since 2018.

---

## Wrapping Up

The `use` API is a small addition with big implications. It finally gives React developers the escape hatch they've wanted for conditional logic without compromising the predictability that hooks were designed to provide. Whether you're unwrapping promises or surgically subscribing to context, `use` makes your components cleaner and more performant.

If you're migrating to React 19, this is one of the first features worth exploring.
