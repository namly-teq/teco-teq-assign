---
description: 'Global state management patterns using Zustand for app-wide state'
applyTo: 'src/stores/**'
---

## 🎯 What Are Zustand Stores?

Stores are **global state containers** that hold app-wide data accessible from any component.    
Think:   `authStore`, `themeStore`, `cartStore`, `notificationStore`.

### Core Principles:
- ✅ **Global app state only** - Not for local UI state
- ✅ **Minimal and focused** - Each store has one domain
- ✅ **Type-safe** - Full TypeScript types
- ✅ **Immutable updates** - Don't mutate state directly
- ✅ **Selective subscriptions** - Components only re-render when their data changes

---

## 🎯 When to Use Zustand

### ✅ Use Zustand For:
- **Authentication state** - Current user, tokens, auth status
- **App-wide settings** - Theme, locale, preferences
- **Shopping cart** - Items, totals, checkout state
- **UI state shared across many components** - Sidebar open/closed, modals
- **WebSocket/real-time data** - Live updates, notifications

### ❌ Don't Use Zustand For: 
- **Server data** - Use TanStack Query instead
- **Form state** - Use React Hook Form
- **Local UI state** - Use `useState` in components
- **URL state** - Use router (search params, path params)
- **Component-specific state** - Keep it local with `useState`

---

## 📊 Using Stores in Components

### Selective Subscription (Performance Optimization)

```tsx
// ✅ GOOD - Only subscribes to specific state
const UserProfile = () => {
  // Only re-renders when user changes
  const user = useAuthStore((state) => state.user);
  
  return <div>{user?.name}</div>;
};

// ❌ BAD - Subscribes to entire store
const UserProfile = () => {
  // Re-renders on ANY auth state change
  const { user, token, isLoading, login, logout } = useAuthStore();
  
  return <div>{user?.name}</div>;
};
```

### Multiple Selectors

```tsx
// ✅ GOOD - Multiple specific selectors
const CartSummary = () => {
  const itemCount = useCartStore((state) => state.itemCount);
  const total = useCartStore((state) => state.total);
  
  return (
    <div>
      <p>Items: {itemCount}</p>
      <p>Total:  ${total}</p>
    </div>
  );
};
```

### Actions Only

```tsx
// ✅ GOOD - Only grab actions (doesn't cause re-renders)
const LoginButton = () => {
  const login = useAuthStore((state) => state.login);
  
  const handleLogin = async () => {
    await login('user@example.com', 'password');
  };
  
  return <button onClick={handleLogin}>Login</button>;
};
```

### Using with Effects

```tsx
// Connect WebSocket on mount
const LiveFeed = () => {
  const connect = useLiveDataStore((state) => state.connect);
  const disconnect = useLiveDataStore((state) => state.disconnect);
  const updates = useLiveDataStore((state) => state.updates);

  useEffect(() => {
    connect('wss://api.example.com/live');
    
    return () => {
      disconnect();
    };
  }, [connect, disconnect]);

  return (
    <div>
      {updates.map((update) => (
        <div key={update.id}>{update.type}</div>
      ))}
    </div>
  );
};
```

---

## 🎨 Advanced Patterns

### Immer Middleware (Easier Mutations)

```tsx
// stores/complex-store.ts
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface ComplexState {
  nested: {
    deeply: {
      value: number;
    };
  };
  items: Array<{ id: string; name: string }>;
  
  updateNestedValue: (value: number) => void;
  addItem: (item: { id: string; name: string }) => void;
}

export const useComplexStore = create<ComplexState>()(
  immer((set) => ({
    nested: {
      deeply: {
        value: 0,
      },
    },
    items:  [],

    // With immer, you can "mutate" directly
    updateNestedValue: (value) => {
      set((state) => {
        state. nested.deeply.value = value; // Looks like mutation, actually immutable
      });
    },

    addItem: (item) => {
      set((state) => {
        state.items.push(item); // Looks like mutation, actually immutable
      });
    },
  }))
);
```

### Resetting Store

```tsx
// stores/resettable-store.ts
import { create } from 'zustand';

interface ResettableState {
  count: number;
  name: string;
  
  increment: () => void;
  setName: (name: string) => void;
  reset: () => void;
}

const initialState = {
  count: 0,
  name: '',
};

export const useResettableStore = create<ResettableState>((set) => ({
  ...initialState,

  increment: () => set((state) => ({ count: state.count + 1 })),
  setName: (name) => set({ name }),

  // Reset to initial state
  reset:  () => set(initialState),
}));
```

---

## ✅ Store Best Practices

### 1. One Store Per Domain
```tsx
// ✅ GOOD - Focused stores
useAuthStore()       // Authentication
useCartStore()       // Shopping cart
useThemeStore()      // Theme preferences
useUIStore()         // UI state

// ❌ BAD - One giant store
useAppStore()        // Everything mixed together
```

### 2. Separate State and Actions
```tsx
// ✅ GOOD - Clear separation
interface MyState {
  // State
  count: number;
  name: string;
  
  // Actions
  increment: () => void;
  setName: (name: string) => void;
}
```

### 3. Use Getters for Computed Values
```tsx
// ✅ GOOD - Getter for computed value
interface CartState {
  items: CartItem[];
  
  get total(): number; // Computed on access
}

// ❌ BAD - Storing computed value
interface CartState {
  items: CartItem[];
  total: number; // Must manually update
}
```

### 4. Persist Only What's Needed
```tsx
persist(
  (set, get) => ({ /* ... */ }),
  {
    name: 'auth-storage',
    partialize: (state) => ({
      token: state.token,     // ✅ Persist
      user: state.user,       // ✅ Persist
      // isLoading: false     // ❌ Don't persist transient state
    }),
  }
)
```

### 5. Clean Up Side Effects
```tsx
// ✅ GOOD - Cleanup WebSocket on disconnect
disconnect: () => {
  const ws = get().ws;
  if (ws) {
    ws.close();
    set({ isConnected: false, ws: null });
  }
}
```

---

## ❌ Common Mistakes

### Mistake 1: Using Store for Server Data
```tsx
// ❌ BAD - Storing fetched data in Zustand
const useUserStore = create((set) => ({
  users: [],
  fetchUsers: async () => {
    const data = await userService.getAll();
    set({ users: data });
  },
}));

// ✅ GOOD - Use React Query for server data
const useUsers = () => {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => userService.getAll(),
  });
};
```

### Mistake 2: Mutating State Directly
```tsx
// ❌ BAD - Direct mutation
addItem: (item) => {
  get().items.push(item); // ❌ Mutates state! 
}

// ✅ GOOD - Immutable update
addItem: (item) => {
  set((state) => ({
    items: [...state.items, item],
  }));
}

// ✅ ALSO GOOD - Use Immer middleware
addItem: (item) => {
  set((state) => {
    state.items.push(item); // ✅ Immer handles immutability
  });
}
```

### Mistake 3: Over-subscribing in Components
```tsx
// ❌ BAD - Entire store subscription
const MyComponent = () => {
  const store = useAuthStore(); // Re-renders on ANY change
  return <div>{store.user?.name}</div>;
};

// ✅ GOOD - Selective subscription
const MyComponent = () => {
  const user = useAuthStore((state) => state.user); // Only user changes
  return <div>{user?.name}</div>;
};
```

### Mistake 4: Storing Derived Data
```tsx
// ❌ BAD - Storing computed values
interface CartState {
  items: CartItem[];
  total: number; // Must update manually
  itemCount: number; // Must update manually
  
  addItem: (item) => {
    // Have to calculate and update total, itemCount... 
  }
}

// ✅ GOOD - Use getters
interface CartState {
  items: CartItem[];
  
  get total(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
  
  get itemCount(): number {
    return this.items.length;
  }
}
```

---

## 📐 Store Template

```tsx
// stores/my-store.ts
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';

interface MyState {
  // State
  value: string;
  count: number;
  isLoading: boolean;

  // Actions
  setValue: (value: string) => void;
  increment: () => void;
  reset: () => void;
}

const initialState = {
  value: '',
  count: 0,
  isLoading: false,
};

export const useMyStore = create<MyState>()(
  devtools(
    persist(
      (set, get) => ({
        ... initialState,

        setValue: (value) => set({ value }),

        increment: () => set((state) => ({ count: state. count + 1 })),

        reset: () => set(initialState),
      }),
      {
        name: 'my-storage',
        partialize: (state) => ({
          // Only persist what's needed
          value: state.value,
          count: state.count,
        }),
      }
    ),
    { name: 'MyStore' }
  )
);
```
