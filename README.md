# Performance Optimization in Modern React

This is a collection of the most important performance optimization techniques for Modern React applications. Made by [Cosden Solutions](https://cosden.solutions).

## 1. Leverage the React Compiler
The React compiler (previously known as React Forget) automatically memoizes components and values, eliminating the need for manual memoization with useMemo and useCallback.

```tsx
// ❌ Manual memoization (no longer needed)
function Component({ items }) {
  const sortedItems = useMemo(() => items.sort(), [items]);
  const handleClick = useCallback(() => {}, []);
  
  return <ChildComponent items={sortedItems} onClick={handleClick} />;
}

// ✅ Let the compiler handle it
function Component({ items }) {
  const sortedItems = items.sort(); // Automatically memoized
  const handleClick = () => {};     // Automatically memoized
  
  return <ChildComponent items={sortedItems} onClick={handleClick} />;
}
```
### Benefits

- Automatically determines when to memoize

- Handles edge cases better than humans

- Enforces Rules of Hooks compliance

- Reduces bundle size by removing manual memoization hooks

## 2. Move Logic Outside of React

Shift data fetching and side effects outside React components when possible, especially into routing libraries.

```tsx
// ❌ Data fetching inside component
function UserProfile() {
  const { data: user, isPending, isError } = useUserQuery();

  // ❌ Checking loading inside component
  if (isLoading) {
    return <div>Loading...</div>
  }

  // ❌ Checking error inside component
  if (isError) {
    return <div>Something went wrong</div>
  }

  return <Profile user={user} />;
}

// ✅ Using router loaders (TanStack)
// router.ts
const route = createFileRoute("/profile", {
  loader: async () => {
    const user = await fetch('/api/user').then(res => res.json());
    return { user };
  },
  pendingComponent: () => <div>Loading...</div>,
  errorComponent: () => <div>Something went wrong</div>,
  component: UserProfile,
});

function UserProfile() {
  const { user } = route.useLoaderData(); // ✅ Data available immediately
  return <Profile user={user} />;
}
```

### Benefits

- No loading states in components

- Data available before render

- No unnecessary re-renders

- Better caching capabilities

## 3. Use Server Components Strategically

Server Components allow you to move work to the server, reducing client-side JavaScript. Combine with client components for optimal performance.

```tsx
// ❌ Client component fetching data
'use client';
function BlogPost() {
  const [post, setPost] = useState(null);

  useEffect(() => {
    fetch('/api/post')
      .then(res => res.json())
      .then(setPost);
  }, []);

  if (!post) return <Loading />;
  return <Article post={post} />;
}

// ✅ Server component
async function BlogPost() {
  const post = await db.post.findUnique(); // Direct database access
  
  return <Article post={post} />; // No loading state needed
}
```

### Combine with Client Components for interactivity

```tsx
// BlogPost.tsx (Server Component)
async function BlogPost() {
  const post = await db.post.findUnique();
  return <Article post={post} comments={<Comments />} />;
}

// Comments.tsx (Client Component)
'use client';
function Comments() {
  const [comment, setComment] = useState('');
  // ... interactive logic
}
```

### When to use Server Components

- Data fetching

- Static content

- Accessing backend resources directly

- Large dependencies (render on server only)

## 4. Use Transitions for Non-Urgent Updates

Mark non-critical state updates as transitions to prevent UI blocking. Allow important updates to interrupt transitions.

```tsx
'use client';

function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e) {
    const value = e.target.value;
    setQuery(value); // Urgent update (input must feel responsive)
    
    startTransition(() => {
      // Non-urgent results can wait
      searchAPI(value).then(setResults);
    });
  }

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </div>
  );
}
```

### Benefits

- Keeps UI responsive during heavy updates

- Prioritizes user interactions

- Prevents janky input experiences

- Automatic interruption of stale updates

## 5. Use Partial Prerendering for Dynamic Content

Next.js 14.1+ introduced Partial Prerendering, combining static and dynamic content rendering for optimal performance.

```tsx
import { Suspense } from 'react';
import { UserActivity } from './user-activity';
import { StaticDashboardMetrics } from './static-metrics';

export default function Dashboard() {
  return (
    <div className="dashboard">
      {/* ✅ Static content: prerendered */}
      <StaticDashboardMetrics />

      {/* ✅ Dynamic content: rendered at request time */}
      <Suspense fallback={<p>Loading activity...</p>}>
        <UserActivity />
      </Suspense>
    </div>
  );
}
```

### Benefits

- Static content loads instantly

- Dynamic content streamed in when ready

- Pending state automatically handled through `Suspense`

## 6. Optimize Images and Media
Use modern techniques for handling media efficiently. Most frameworks offer tools/components to do this for you.

```tsx
// ❌ Regular img tag with unoptimized source
<img src="/large-image.jpg" alt="Example" />

// ✅ Next.js Image component
<Image
  src="/large-image.jpg"
  alt="Example"
  width={500}
  height={300}
  priority={false} // Lazy load by default
  quality={80}     // Adjust quality
  placeholder="blur" // Optional blur placeholder
/>
```

### Additional media optimizations

- Use modern formats (WebP, AVIF)

- Implement responsive images with srcset

- Lazy load offscreen media

- Consider CDN for media delivery

- Use video placeholders

## 7. Implement Smart Prefetching

Try to anticipate user needs and prefetch resources required before they are. This will significantly improve perceived performance.

```tsx
'use client';
// Using TanStack Router
const router = createFileRoute("/dashboard", {
    component: Dashboard,
    loader: fetchDashboardData,
    // ✅ Prefetch when link is hovered or focused
    preload: 'intent' 
  }
);

// Manual prefetching with React (concept)
function ProductLink({ id }) {
  const [prefetched, setPrefetched] = useState(false);

  const prefetch = useCallback(() => {
    if (!prefetched) {
      startTransition(() => {
        prefetchProductData(id);
        setPrefetched(true);
      });
    }
  }, [id, prefetched]);

  return (
    <Link 
      to={`/product/${id}`}
      onMouseEnter={prefetch}
      onFocus={prefetch}
    >
      Product
    </Link>
  );
}
```

## 8. Virtualize Long Lists

Render only visible items in large lists. Don't use up resources for what is not visible on the screen.

```tsx
'use client';
// ❌ Rendering all items
function List({ items }) {
  return (
    <ul>
      {items.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
    </ul>
  );
}

// ✅ Using virtualization
import { Virtuoso } from 'react-virtuoso';

function List({ items }) {
  return (
    <Virtuoso
      data={items}
      itemContent={(index, item) => <ListItem item={item} />}
      overscan={10} // Render slightly more items than visible
      style={{ height: '600px' }} // Required for calculation
    />
  );
}
```

### Alternative libraries

- react-window

- tanstack-virtual

## 9. Optimize Context Usage

Prevent unnecessary re-renders from context changes. Either split in multiple contexts, or use context selectors.

```tsx
'use client';
// ❌ Simple context causes all consumers to re-render, no matter what they consume
const UserContext = createContext();

function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  const [preferences, setPreferences] = useState({});
  
  return (
    <UserContext.Provider value={{ user, preferences, setUser, setPreferences }}>
      {children}
    </UserContext.Provider>
  );
}

// ✅ Split context or use selectors
const UserContext = createContext();
const UserDispatchContext = createContext();

function UserProvider({ children }) {
  const [state, dispatch] = useReducer(userReducer, initialState);
  
  return (
    <UserContext.Provider value={state}>
      <UserDispatchContext.Provider value={dispatch}>
        {children}
      </UserDispatchContext.Provider>
    </UserContext.Provider>
  );
}

// ✅ Even better with context selectors (use-context-selector)
function useTheme() {
  return useContextSelector(UserContext, state => state.preferences.theme);
}
```

## 10. Debounce and Throttle Events

Optimize frequent events like scrolling, resizing, or typing. Only run expensive code after a delay.

```tsx
'use client';
import { debounce } from 'lodash-es';

function Search() {
  const [results, setResults] = useState([]);
  
  // ❌ Calling API on every keystroke
  const handleChange = (e) => {
    fetchResults(e.target.value).then(setResults);
  };

  // ✅ Debounced API call
  const debouncedSearch = debounce((query) => {
    fetchResults(query).then(setResults);
  }, 300);

  const handleChange = (e) => {
    debouncedSearch(e.target.value);
  };

  return <input onChange={handleChange} />;
}
```
