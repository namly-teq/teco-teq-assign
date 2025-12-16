---
description: 'Service layer patterns for API clients, external integrations, and data access logic'
applyTo: 'src/services/**'
---
## 🎯 What Are Services?

Services are **organized collections of API client functions** that handle external communication.   
Think:  `userService`, `authService`, `paymentService`, `notificationService`.

### Core Principles:
- ✅ **Separation of concerns** - Keep API logic out of components
- ✅ **Type-safe** - Full TypeScript types for requests and responses
- ✅ **Centralized error handling** - Consistent error patterns
- ✅ **Testable** - Easy to mock in tests
- ✅ **Reusable** - One service, many consumers
- ✅ **Single responsibility** - One service per domain/resource

---

## 🏗️ Service Layer Architecture

```
Component/Hook → Service → API Client → Backend API
```

### Example Flow:
```tsx
// 1. Component calls hook
const UserProfile = () => {
  const { data: user } = useUser('123');
  // ... 
};

// 2. Hook calls service
export const useUser = (id: string) => {
  return useQuery({
    queryKey: ['user', id],
    queryFn:  () => userService.getById(id), // ← Service call
  });
};

// 3. Service calls API client
export const userService = {
  getById: (id: string) => 
    apiClient.get<User>(`/users/${id}`).then(res => res.data),
};

// 4. API client makes HTTP request
export const apiClient = axios.create({
  baseURL:  import.meta.env.VITE_API_BASE_URL,
});
```

---

## ✅ Service Best Practices

### 1. One Service Per Domain
```tsx
// ✅ GOOD - Focused services
userService.ts        // User operations
authService.ts        // Authentication
paymentService.ts     // Payments
productService.ts     // Products

// ❌ BAD - Generic "api" service with everything
apiService.ts         // 1000 lines of mixed operations
```

### 2. Export as Const Object
```tsx
// ✅ GOOD - Const object (tree-shakeable)
export const userService = {
  getAll: () => { /* ... */ },
  getById: () => { /* ... */ },
} as const;

// ❌ BAD - Individual exports (harder to mock)
export function getUsers() { /* ... */ }
export function getUserById() { /* ... */ }
```

### 3. Always Type Request/Response
```tsx
// ✅ GOOD - Full type safety
getById: async (id: string): Promise<User> => {
  const response = await apiClient.get<User>(`/users/${id}`);
  return response.data;
}

// ❌ BAD - Untyped
getById: async (id) => {
  const response = await apiClient.get(`/users/${id}`);
  return response.data;
}
```

### 4. Extract Response Data
```tsx
// ✅ GOOD - Return just the data
getById: async (id: string): Promise<User> => {
  const response = await apiClient.get<User>(`/users/${id}`);
  return response.data; // ✅ Return data only
}

// ❌ BAD - Return entire Axios response
getById: async (id:  string) => {
  return await apiClient.get<User>(`/users/${id}`); // ❌ Returns AxiosResponse
}
```

### 5. Document Each Method
```tsx
/**
 * Get a user by their unique ID
 * 
 * @param id - User's unique identifier
 * @returns Promise resolving to User object
 * @throws {NotFoundError} If user doesn't exist
 */
getById: async (id: string): Promise<User> => {
  const response = await apiClient.get<User>(`/users/${id}`);
  return response.data;
}
```

---

## ❌ Common Mistakes

### Mistake 1: Business Logic in Services
```tsx
// ❌ BAD - Business logic in service
export const userService = {
  updateProfile: async (id: string, data: UpdateUserDto) => {
    // ❌ Validation logic here
    if (data.age < 18) {
      throw new Error('Must be 18+');
    }

    // ❌ Formatting logic here
    const formattedData = {
      ... data,
      name: data.name.trim().toLowerCase(),
    };

    return apiClient.patch(`/users/${id}`, formattedData);
  },
};

// ✅ GOOD - Service just makes API calls
export const userService = {
  updateProfile: async (id:  string, data: UpdateUserDto) => {
    const response = await apiClient.patch<User>(`/users/${id}`, data);
    return response.data;
  },
};

// Validation happens in form (Zod schema)
// Formatting happens in component/hook before calling service
```

### Mistake 2: Not Handling Errors
```tsx
// ❌ BAD - Swallow errors silently
getById: async (id: string) => {
  try {
    const response = await apiClient.get(`/users/${id}`);
    return response.data;
  } catch (error) {
    console.log(error); // ❌ Just logging
    return null; // ❌ Hiding the error
  }
}

// ✅ GOOD - Let errors propagate (handle in component/hook)
getById: async (id: string): Promise<User> => {
  const response = await apiClient.get<User>(`/users/${id}`);
  return response.data;
}
```

### Mistake 3: Hard-coding URLs
```tsx
// ❌ BAD - Hard-coded URLs
getById: async (id: string) => {
  return fetch(`https://api.example.com/users/${id}`); // ❌ Hard-coded
}

// ✅ GOOD - Use configured API client
getById: async (id:  string): Promise<User> => {
  const response = await apiClient.get<User>(`/users/${id}`);
  return response.data;
}
```

### Mistake 4: Mixing Service and State
```tsx
// ❌ BAD - State management in service
let cachedUser: User | null = null;

export const userService = {
  getById: async (id: string) => {
    if (cachedUser?. id === id) {
      return cachedUser; // ❌ Managing state in service
    }

    const response = await apiClient.get(`/users/${id}`);
    cachedUser = response.data;
    return cachedUser;
  },
};

// ✅ GOOD - State managed by React Query in hooks
export const userService = {
  getById: async (id: string): Promise<User> => {
    const response = await apiClient.get<User>(`/users/${id}`);
    return response.data;
  },
};

// React Query handles caching in hook layer
export const useUser = (id: string) => {
  return useQuery({
    queryKey: ['user', id],
    queryFn:  () => userService.getById(id),
    staleTime: 5 * 60 * 1000, // ✅ Cache config here
  });
};
```

---
