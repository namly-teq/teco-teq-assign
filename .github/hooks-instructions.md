---
description: 'Guidelines for custom React hooks that encapsulate reusable stateful logic'
applyTo: 'src/hooks/**'
---

## 🎯 What Are Custom Hooks?

Custom hooks are **reusable functions that encapsulate stateful logic**.   
Think:  `useAuth`, `useDebounce`, `useLocalStorage`, `useMediaQuery`, `useFetch`.

### Core Principles: 
- ✅ **Start with "use"** - Required by React (linting rules)
- ✅ **Reusable logic** - Used across multiple components
- ✅ **One responsibility** - Each hook does one thing well
- ✅ **Compose other hooks** - Built from React hooks or other custom hooks
- ✅ **Return consistent shape** - Predictable return values

---

## 🎨 Hook Categories

### 1. Data Fetching Hooks (React Query Wrappers)
```tsx
// hooks/useUser.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { userService } from '@/services/user-service';
import type { User, UpdateUserDto } from '@/types';

/**
 * Fetch a single user by ID
 */
export const useUser = (id: string) => {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => userService.getById(id),
    staleTime: 5 * 60 * 1000, // 5 minutes
    enabled:  !!id, // Don't fetch if no ID provided
  });
};
```

### 2. UI State Hooks
```tsx
// hooks/useDisclosure.ts
import { useState, useCallback } from 'react';

interface UseDisclosureReturn {
  isOpen: boolean;
  open: () => void;
  close: () => void;
  toggle: () => void;
}

/**
 * Manage open/close state for modals, dropdowns, etc. 
 * 
 * @param initialState - Initial open state (default: false)
 * @returns Object with isOpen state and control functions
 * 
 * @example
 * const { isOpen, open, close, toggle } = useDisclosure();
 * 
 * <Dialog open={isOpen} onOpenChange={toggle}>
 *   <Button onClick={open}>Open Dialog</Button>
 * </Dialog>
 */
export const useDisclosure = (initialState = false): UseDisclosureReturn => {
  const [isOpen, setIsOpen] = useState(initialState);

  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen((prev) => !prev), []);

  return { isOpen, open, close, toggle };
};
```

### 3. Utility Hooks
```tsx
// hooks/useDebounce.ts
import { useEffect, useState } from 'react';

/**
 * Debounce a value - delays updating until user stops changing it
 * 
 * @param value - The value to debounce
 * @param delay - Delay in milliseconds (default: 500ms)
 * @returns Debounced value
 * 
 * @example
 * const [search, setSearch] = useState('');
 * const debouncedSearch = useDebounce(search, 500);
 * 
 * // Only triggers API call after user stops typing for 500ms
 * const { data } = useSearchResults(debouncedSearch);
 */
export const useDebounce = <T,>(value: T, delay = 500): T => {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
};
```

### 4. Browser API Hooks

```tsx
// hooks/useWindowSize.ts
import { useState, useEffect } from 'react';

interface WindowSize {
  width: number;
  height: number;
}

/**
 * Track window dimensions
 * 
 * @returns Object with current width and height
 * 
 * @example
 * const { width, height } = useWindowSize();
 * 
 * if (width < 768) {
 *   return <MobileView />;
 * }
 */
export const useWindowSize = (): WindowSize => {
  const [windowSize, setWindowSize] = useState<WindowSize>({
    width: typeof window !== 'undefined' ? window.innerWidth : 0,
    height: typeof window !== 'undefined' ? window.innerHeight :  0,
  });

  useEffect(() => {
    const handleResize = () => {
      setWindowSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };

    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);

  return windowSize;
};
```


### 5. Form Hooks
```tsx
// hooks/useFormField.ts
import { useState, useCallback, type ChangeEvent } from 'react';

interface UseFormFieldReturn<T> {
  value: T;
  onChange: (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => void;
  setValue: (value: T) => void;
  reset: () => void;
}

/**
 * Manage individual form field state
 * 
 * @param initialValue - Initial field value
 * @returns Object with value, onChange handler, setValue, and reset
 * 
 * @example
 * const email = useFormField('');
 * 
 * <input {... email} type="email" />
 */
export const useFormField = <T extends string = string>(
  initialValue: T
): UseFormFieldReturn<T> => {
  const [value, setValue] = useState<T>(initialValue);

  const onChange = useCallback(
    (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
      setValue(e.target.value as T);
    },
    []
  );

  const reset = useCallback(() => {
    setValue(initialValue);
  }, [initialValue]);

  return { value, onChange, setValue, reset };
};
```

### 6. Animation/Timing Hooks
```tsx
// hooks/useTimeout.ts
import { useEffect, useRef } from 'react';

/**
 * Execute callback after a delay
 * 
 * @param callback - Function to execute
 * @param delay - Delay in milliseconds (null to cancel)
 * 
 * @example
 * useTimeout(() => {
 *   setShowNotification(false);
 * }, 3000);
 */
export const useTimeout = (callback: () => void, delay: number | null) => {
  const savedCallback = useRef(callback);

  // Update callback ref when it changes
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) {
      return;
    }

    const timeoutId = setTimeout(() => {
      savedCallback.current();
    }, delay);

    return () => {
      clearTimeout(timeoutId);
    };
  }, [delay]);
};
```
---

## ✅ Hook Best Practices

### 1. Always Start with "use"
```tsx
// ✅ GOOD
export const useAuth = () => { /* ... */ };
export const useDebounce = () => { /* ... */ };

// ❌ BAD - Not a valid hook name
export const getAuth = () => { /* ... */ };
export const debounce = () => { /* ... */ };
```

### 2. Return Consistent Shapes
```tsx
// ✅ GOOD - Object with named properties
export const useAuth = () => {
  return {
    user,
    login,
    logout,
    isAuthenticated,
    isLoading,
  };
};

// ✅ ALSO GOOD - Tuple for simple cases
export const useToggle = (initial: boolean) => {
  const [value, setValue] = useState(initial);
  const toggle = () => setValue(!value);
  return [value, toggle] as const;
};

// ❌ BAD - Inconsistent return
export const useAuth = () => {
  if (isLoading) return null; // ❌ Different return type
  return { user, login, logout };
};
```

### 3. Document with JSDoc
```tsx
/**
 * Debounce a value - delays updating until user stops changing it
 * 
 * @param value - The value to debounce
 * @param delay - Delay in milliseconds (default: 500ms)
 * @returns Debounced value
 * 
 * @example
 * const debouncedSearch = useDebounce(search, 500);
 */
export const useDebounce = <T,>(value: T, delay = 500): T => {
  // Implementation
};
```

### 4. Use TypeScript Generics for Flexibility
```tsx
// ✅ GOOD - Generic hook works with any type
export const useLocalStorage = <T,>(key: string, initialValue: T) => {
  // Type-safe for any T
};

// Usage: 
const [user, setUser] = useLocalStorage<User | undefined>('user', undefined);
const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');
```

### 5. Cleanup Side Effects
```tsx
// ✅ GOOD - Always cleanup
export const useEventListener = (event: string, handler: () => void) => {
  useEffect(() => {
    window.addEventListener(event, handler);
    
    return () => {
      window.removeEventListener(event, handler); // ✅ Cleanup
    };
  }, [event, handler]);
};

// ❌ BAD - Memory leak! 
export const useEventListener = (event: string, handler: () => void) => {
  useEffect(() => {
    window.addEventListener(event, handler); // ❌ No cleanup
  }, [event, handler]);
};
```

### 6. Stabilize Callbacks with useCallback
```tsx
// ✅ GOOD - Memoized functions in dependencies
export const useClickOutside = (handler: () => void) => {
  const ref = useRef(null);

  // Stabilize handler to prevent effect re-running unnecessarily
  const stableHandler = useCallback(handler, []);

  useEffect(() => {
    const handleClick = (e: MouseEvent) => {
      if (ref. current && !ref.current.contains(e.target)) {
        stableHandler();
      }
    };

    document.addEventListener('mousedown', handleClick);
    return () => document.removeEventListener('mousedown', handleClick);
  }, [stableHandler]);

  return ref;
};
```

### 7. Handle SSR (Server-Side Rendering)
```tsx
// ✅ GOOD - SSR-safe
export const useWindowSize = () => {
  const [size, setSize] = useState({
    width: typeof window !== 'undefined' ? window.innerWidth : 0,
    height:  typeof window !== 'undefined' ?  window.innerHeight : 0,
  });

  useEffect(() => {
    // This only runs on client
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
};
```

---

## ❌ Common Mistakes

### Mistake 1: Not Following Rules of Hooks
```tsx
// ❌ BAD - Conditional hook call
export const useConditionalData = (shouldFetch: boolean) => {
  if (shouldFetch) {
    const { data } = useQuery(/* ... */); // ❌ Conditional hook! 
    return data;
  }
  return null;
};

// ✅ GOOD - Hook always called
export const useConditionalData = (shouldFetch: boolean) => {
  const { data } = useQuery({
    queryKey: ['data'],
    queryFn: fetchData,
    enabled: shouldFetch, // ✅ Use enabled option
  });
  
  return data;
};
```

### Mistake 2: Missing Dependencies
```tsx
// ❌ BAD - Missing dependency
export const useFetch = (url: string) => {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(url).then(res => res.json()).then(setData);
  }, []); // ❌ Missing 'url' dependency! 

  return data;
};

// ✅ GOOD - Complete dependencies
export const useFetch = (url: string) => {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(url).then(res => res.json()).then(setData);
  }, [url]); // ✅ Includes 'url'

  return data;
};
```

### Mistake 3: Not Memoizing Returned Functions
```tsx
// ❌ BAD - New function on every render
export const useCounter = () => {
  const [count, setCount] = useState(0);

  const increment = () => setCount(c => c + 1); // ❌ New function each time

  return { count, increment };
};

// ✅ GOOD - Memoized function
export const useCounter = () => {
  const [count, setCount] = useState(0);

  const increment = useCallback(() => setCount(c => c + 1), []); // ✅ Stable

  return { count, increment };
};
```

### Mistake 4: Doing Too Much in One Hook
```tsx
// ❌ BAD - Hook doing everything
export const useUserDashboard = (userId: string) => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [friends, setFriends] = useState([]);
  const [notifications, setNotifications] = useState([]);

  // 100 lines of logic... 

  return { user, posts, friends, notifications, /* ... */ };
};

// ✅ GOOD - Separate focused hooks
export const useUser = (userId: string) => { /* ... */ };
export const usePosts = (userId: string) => { /* ... */ };
export const useFriends = (userId: string) => { /* ... */ };
export const useNotifications = (userId: string) => { /* ... */ };

// Compose in component: 
const Dashboard = ({ userId }: { userId: string }) => {
  const { data: user } = useUser(userId);
  const { data: posts } = usePosts(userId);
  const { data: friends } = useFriends(userId);
  const { data: notifications } = useNotifications(userId);
  
  // ... 
};
```
