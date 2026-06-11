---
name: refactor
description: Identify code smells and refactor React code for better quality. Use when improving code or cleaning up.
allowed-tools: Read, Edit, Glob, Grep, Bash(git:*)
---

# Refactor Skill

Identifies and fixes code quality issues in React applications.

## Code Smell Detection

### 1. Component Issues

**Large Components (>300 lines)**
- Extract sub-components
- Move logic to custom hooks
- Separate concerns

**Prop Drilling (>3 levels)**
- Use React Context
- Use Zustand store
- Compose components differently

**God Components**
- Components doing too many things
- Split by responsibility

### 2. Hook Issues

**Missing Dependencies**
```jsx
// Bad
useEffect(() => {
  fetchData(id);
}, []); // Missing id dependency

// Good
useEffect(() => {
  fetchData(id);
}, [id]);
```

**Stale Closures**
```jsx
// Bad
const handleClick = () => {
  console.log(count); // Stale value
};

// Good
const handleClick = useCallback(() => {
  console.log(count);
}, [count]);
```

### 3. State Issues

**Derived State**
```jsx
// Bad - storing derived state
const [items, setItems] = useState([]);
const [filteredItems, setFilteredItems] = useState([]);

useEffect(() => {
  setFilteredItems(items.filter(i => i.active));
}, [items]);

// Good - compute during render
const [items, setItems] = useState([]);
const filteredItems = useMemo(
  () => items.filter(i => i.active),
  [items]
);
```

**Boolean Flag Explosion**
```jsx
// Bad
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);

// Good - use a status enum or React Query
const [status, setStatus] = useState('idle'); // 'idle' | 'loading' | 'error' | 'success'
```

### 4. Pattern Violations

**Direct DOM Manipulation**
```jsx
// Bad
document.getElementById('modal').style.display = 'block';

// Good
const [isOpen, setIsOpen] = useState(false);
return isOpen && <Modal />;
```

**Index as Key**
```jsx
// Bad (if list can reorder)
items.map((item, index) => <Item key={index} />)

// Good
items.map(item => <Item key={item.id} />)
```

## Refactoring Patterns

### Extract Custom Hook
```jsx
// Before
function Component() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, []);

  // ... component logic
}

// After
function useData() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, []);

  return { data, loading };
}

function Component() {
  const { data, loading } = useData();
  // ... simpler component logic
}
```

### Extract Sub-Component
```jsx
// Before - monolithic
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          <img src={user.avatar} />
          <span>{user.name}</span>
          <button onClick={() => deleteUser(user.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}

// After - composed
function UserListItem({ user, onDelete }) {
  return (
    <li>
      <img src={user.avatar} />
      <span>{user.name}</span>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </li>
  );
}

function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <UserListItem key={user.id} user={user} onDelete={deleteUser} />
      ))}
    </ul>
  );
}
```

### Replace useState with useReducer
```jsx
// Before - complex state logic
const [state, setState] = useState({ items: [], filter: '', sort: 'asc' });

const addItem = (item) => setState(s => ({ ...s, items: [...s.items, item] }));
const setFilter = (filter) => setState(s => ({ ...s, filter }));
const toggleSort = () => setState(s => ({ ...s, sort: s.sort === 'asc' ? 'desc' : 'asc' }));

// After - reducer pattern
function reducer(state, action) {
  switch (action.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.payload] };
    case 'SET_FILTER':
      return { ...state, filter: action.payload };
    case 'TOGGLE_SORT':
      return { ...state, sort: state.sort === 'asc' ? 'desc' : 'asc' };
    default:
      return state;
  }
}

const [state, dispatch] = useReducer(reducer, { items: [], filter: '', sort: 'asc' });
```

## Workflow

1. **Analyze** - Read the file and identify issues
2. **Prioritize** - Focus on high-impact changes
3. **Refactor** - Make incremental changes
4. **Test** - Verify behavior is unchanged
5. **Commit** - Use `refactor:` prefix

## Output Format

When analyzing code, report:

```
## Issues Found

| Severity | Issue | Location | Suggested Fix |
|----------|-------|----------|---------------|
| High | Missing useCallback | Line 45 | Wrap handler in useCallback |
| Medium | Prop drilling | Props passed 4 levels | Extract to context |
| Low | Magic number | Line 23 | Extract to constant |

## Recommended Actions

1. [High Priority] Extract data fetching to custom hook
2. [Medium Priority] Split component into smaller pieces
3. [Low Priority] Add missing TypeScript types
```
