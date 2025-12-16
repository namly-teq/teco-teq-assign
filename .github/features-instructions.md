---
description: 'Guidelines for feature-specific smart components that compose UI with business logic'
applyTo:  'src/components/features/**'
---


## 🎯 What Are Feature Components?

Feature components are **smart components** that bridge UI and business logic.  
Think:  `LoginForm`, `UserProfileCard`, `ProductCheckout`, `CommentList`.

### Core Principles:
- ✅ **Compose UI components** - Build from `components/ui/`
- ✅ **Handle business logic** - Data fetching, validation, workflows
- ✅ **Manage feature state** - Local state specific to this feature
- ✅ **Connect to services** - API calls, global state, side effects
- ✅ **Reusable within domain** - Can be used across related pages

---

## ✅ What Feature Components CAN Do

### 1. Use Data Fetching Hooks
```tsx
// ✅ GOOD - Fetch and display user data
// components/features/UserProfileCard.tsx
import { useUser } from '@/hooks/useUser';
import { Card } from '@/components/ui/Card';
import { Avatar } from '@/components/ui/Avatar';
import { Badge } from '@/components/ui/Badge';
import { Skeleton } from '@/components/ui/Skeleton';

interface UserProfileCardProps {
  userId: string;
}

export const UserProfileCard = ({ userId }: UserProfileCardProps) => {
  const { data:  user, isLoading, error } = useUser(userId);

  if (isLoading) {
    return (
      <Card>
        <Card.Content>
          <Skeleton className="h-20 w-20 rounded-full" />
          <Skeleton className="h-4 w-32" />
        </Card.Content>
      </Card>
    );
  }

  if (error) {
    return (
      <Card>
        <Card.Content>
          <p className="text-destructive">Failed to load user profile</p>
        </Card. Content>
      </Card>
    );
  }

  if (!user) return null;

  return (
    <Card>
      <Card. Header>
        <Avatar src={user.avatar} alt={user. name} size="lg" />
        <div>
          <h3 className="text-lg font-semibold">{user.name}</h3>
          <p className="text-sm text-muted-foreground">{user. email}</p>
          {user.isVerified && <Badge variant="success">Verified</Badge>}
        </div>
      </Card.Header>
      <Card.Content>
        <p>{user.bio}</p>
      </Card.Content>
    </Card>
  );
};
```

### 2. Access Global State (Zustand, Context)
```tsx
// ✅ GOOD - Use authentication state
// components/features/UserMenu.tsx
import { useAuthStore } from '@/stores/auth-store';
import { Button } from '@/components/ui/Button';
import { Avatar } from '@/components/ui/Avatar';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/DropdownMenu';

export const UserMenu = () => {
  const { user, logout, isAuthenticated } = useAuthStore();

  if (! isAuthenticated || !user) {
    return <Button variant="ghost">Login</Button>;
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" className="relative h-10 w-10 rounded-full">
          <Avatar src={user.avatar} alt={user.name} size="sm" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem>Profile</DropdownMenuItem>
        <DropdownMenuItem>Settings</DropdownMenuItem>
        <DropdownMenuItem onClick={logout}>Logout</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
};
```

### 3. Implement Business Logic & Workflows
```tsx
// ✅ GOOD - Handle complex checkout flow
// components/features/CheckoutButton.tsx
import { useState } from 'react';
import { usePayment } from '@/hooks/usePayment';
import { useCartStore } from '@/stores/cart-store';
import { Button } from '@/components/ui/Button';
import { toast } from '@/lib/toast';
import { trackEvent } from '@/lib/analytics';

interface CheckoutButtonProps {
  onSuccess?:  () => void;
}

export const CheckoutButton = ({ onSuccess }: CheckoutButtonProps) => {
  const { total, items, clearCart } = useCartStore();
  const { processPayment, isProcessing } = usePayment();
  const [error, setError] = useState<string | null>(null);

  const handleCheckout = async () => {
    // Validation
    if (items.length === 0) {
      toast.error('Your cart is empty');
      return;
    }

    if (total <= 0) {
      toast.error('Invalid cart total');
      return;
    }

    setError(null);

    try {
      // Business logic
      const result = await processPayment({
        amount: total,
        items:  items.map((item) => ({
          id:  item.id,
          quantity: item.quantity,
        })),
      });

      // Analytics
      trackEvent('purchase_completed', {
        total,
        itemCount: items.length,
        orderId: result.orderId,
      });

      // Success workflow
      clearCart();
      toast.success(`Order ${result.orderId} placed successfully!`);
      onSuccess?.();
    } catch (err) {
      const message = err instanceof Error ? err. message : 'Payment failed';
      setError(message);
      toast.error(message);
      
      trackEvent('purchase_failed', { error: message });
    }
  };

  return (
    <div className="space-y-2">
      <Button
        onClick={handleCheckout}
        isLoading={isProcessing}
        disabled={items.length === 0 || total <= 0}
        className="w-full"
      >
        {isProcessing ? 'Processing...' : `Pay $${total. toFixed(2)}`}
      </Button>
      {error && <p className="text-sm text-destructive">{error}</p>}
    </div>
  );
};
```

### 4. Handle Form State & Validation
```tsx
// ✅ GOOD - Form with validation
// components/features/LoginForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useAuthStore } from '@/stores/auth-store';
import { Button } from '@/components/ui/Button';
import { Input } from '@/components/ui/Input';
import { Label } from '@/components/ui/Label';
import { toast } from '@/lib/toast';

const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type LoginFormData = z.infer<typeof loginSchema>;

interface LoginFormProps {
  onSuccess?: () => void;
}

export const LoginForm = ({ onSuccess }: LoginFormProps) => {
  const { login } = useAuthStore();
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (data: LoginFormData) => {
    try {
      await login(data. email, data.password);
      toast.success('Logged in successfully');
      onSuccess?.();
    } catch (error) {
      toast.error('Invalid email or password');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="email">Email</Label>
        <Input
          id="email"
          type="email"
          placeholder="you@example.com"
          {... register('email')}
          aria-invalid={!! errors.email}
        />
        {errors.email && (
          <p className="text-sm text-destructive">{errors.email.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="password">Password</Label>
        <Input
          id="password"
          type="password"
          {... register('password')}
          aria-invalid={!!errors.password}
        />
        {errors. password && (
          <p className="text-sm text-destructive">{errors.password.message}</p>
        )}
      </div>

      <Button type="submit" isLoading={isSubmitting} className="w-full">
        {isSubmitting ? 'Logging in...' : 'Login'}
      </Button>
    </form>
  );
};
```

### 5. Compose Multiple UI Components
```tsx
// ✅ GOOD - Complex composition
// components/features/ProductCard.tsx
import { type FC } from 'react';
import { Card } from '@/components/ui/Card';
import { Button } from '@/components/ui/Button';
import { Badge } from '@/components/ui/Badge';
import { AspectRatio } from '@/components/ui/AspectRatio';
import { useCartStore } from '@/stores/cart-store';
import { toast } from '@/lib/toast';
import type { Product } from '@/types';

interface ProductCardProps {
  product: Product;
  onViewDetails?: (product: Product) => void;
}

export const ProductCard: FC<ProductCardProps> = ({ 
  product, 
  onViewDetails 
}) => {
  const addItem = useCartStore((state) => state.addItem);

  const handleAddToCart = () => {
    addItem(product);
    toast.success(`${product.name} added to cart`);
  };

  return (
    <Card>
      <Card.Header className="p-0">
        <AspectRatio ratio={16 / 9}>
          <img
            src={product.image}
            alt={product.name}
            className="h-full w-full object-cover"
          />
        </AspectRatio>
        {product.isNew && (
          <Badge className="absolute right-2 top-2" variant="success">
            New
          </Badge>
        )}
      </Card.Header>
      <Card. Content className="space-y-2">
        <h3 className="font-semibold">{product.name}</h3>
        <p className="text-sm text-muted-foreground line-clamp-2">
          {product.description}
        </p>
        <div className="flex items-center justify-between">
          <span className="text-lg font-bold">${product.price}</span>
          {product.stock <= 5 && product.stock > 0 && (
            <Badge variant="warning">Only {product.stock} left</Badge>
          )}
          {product.stock === 0 && (
            <Badge variant="destructive">Out of stock</Badge>
          )}
        </div>
      </Card.Content>
      <Card.Footer className="flex gap-2">
        <Button
          variant="outline"
          className="flex-1"
          onClick={() => onViewDetails?.(product)}
        >
          Details
        </Button>
        <Button
          variant="primary"
          className="flex-1"
          onClick={handleAddToCart}
          disabled={product.stock === 0}
        >
          Add to Cart
        </Button>
      </Card.Footer>
    </Card>
  );
};
```

### 6. Handle Side Effects (useEffect)
```tsx
// ✅ GOOD - Sync with external services
// components/features/NotificationBell.tsx
import { useEffect, useState } from 'react';
import { Button } from '@/components/ui/Button';
import { Badge } from '@/components/ui/Badge';
import { BellIcon } from '@/components/ui/Icons';
import { useAuthStore } from '@/stores/auth-store';
import { notificationService } from '@/services/notification-service';

export const NotificationBell = () => {
  const { user } = useAuthStore();
  const [unreadCount, setUnreadCount] = useState(0);

  useEffect(() => {
    if (! user) return;

    // Subscribe to real-time notifications
    const unsubscribe = notificationService. subscribe(user.id, (count) => {
      setUnreadCount(count);
    });

    // Cleanup subscription
    return () => {
      unsubscribe();
    };
  }, [user]);

  if (!user) return null;

  return (
    <Button variant="ghost" className="relative">
      <BellIcon />
      {unreadCount > 0 && (
        <Badge
          variant="destructive"
          className="absolute -right-1 -top-1 h-5 w-5 p-0 text-xs"
        >
          {unreadCount > 9 ? '9+' : unreadCount}
        </Badge>
      )}
    </Button>
  );
};
```

---

## ❌ What Feature Components CANNOT Do

### 1. Import from Pages or Layouts
```tsx
// ❌ BAD - Importing from higher layers
// components/features/UserCard.tsx
import { DashboardPage } from '@/app/dashboard/page'; // ❌ NO! 
import { MainLayout } from '@/components/layouts/MainLayout'; // ❌ NO!

export const UserCard = () => {
  // This creates circular dependencies and tight coupling
};
```

**Why? ** Pages import features, features shouldn't import pages.  
**Dependency flow:** `pages → features → ui → hooks/utils`

**Fix:** If you need to navigate, accept a callback:
```tsx
// ✅ GOOD - Use callback props
interface UserCardProps {
  user: User;
  onNavigateToDashboard?: () => void;
}

export const UserCard = ({ user, onNavigateToDashboard }: UserCardProps) => {
  return (
    <Card>
      <Button onClick={onNavigateToDashboard}>Go to Dashboard</Button>
    </Card>
  );
};
```

### 2. Define Complex Business Rules (Extract to Services)
```tsx
// ❌ BAD - Complex business logic in component
export const PricingCard = ({ product }: { product: Product }) => {
  const calculatePrice = () => {
    let price = product.basePrice;
    
    // 50 lines of pricing logic
    if (user.isPremium) {
      price *= 0.9;
    }
    if (product.isOnSale) {
      price *= 0.8;
    }
    if (season === 'holiday') {
      price *= 0.85;
    }
    // ... more complex rules
    
    return price;
  };

  return <div>${calculatePrice()}</div>;
};
```

**Fix:** Extract to service:
```tsx
// ✅ GOOD - Business logic in service
// services/pricing-service.ts
export const pricingService = {
  calculate: (product: Product, user: User, context: PricingContext) => {
    let price = product.basePrice;
    // Complex pricing logic here
    return price;
  },
};

// components/features/PricingCard. tsx
export const PricingCard = ({ product }: { product: Product }) => {
  const { user } = useAuthStore();
  const price = pricingService.calculate(product, user, { season:  'holiday' });

  return <div>${price}</div>;
};
```

### 3. Contain Other Feature Components (Usually)
```tsx
// ⚠️ QUESTIONABLE - Feature nesting feature
// components/features/UserDashboard.tsx
import { UserProfileCard } from './UserProfileCard';
import { UserActivityFeed } from './UserActivityFeed';
import { UserSettings } from './UserSettings';

export const UserDashboard = () => {
  return (
    <div>
      <UserProfileCard />
      <UserActivityFeed />
      <UserSettings />
    </div>
  );
};
```

**Why questionable?** This is usually a **page-level concern**, not a feature. 

**Better:** Let pages compose features:
```tsx
// ✅ BETTER - Let page handle composition
// app/dashboard/page.tsx
import { UserProfileCard } from '@/components/features/UserProfileCard';
import { UserActivityFeed } from '@/components/features/UserActivityFeed';
import { UserSettings } from '@/components/features/UserSettings';

export default function DashboardPage() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <UserProfileCard userId={userId} />
      <UserActivityFeed userId={userId} />
      <UserSettings userId={userId} />
    </div>
  );
}
```

**Exception:** Container components that manage shared state for related features: 
```tsx
// ✅ OK - Container managing shared state
// components/features/ShoppingCart/ShoppingCartContainer.tsx
export const ShoppingCartContainer = () => {
  const [selectedShipping, setSelectedShipping] = useState<ShippingMethod>();

  return (
    <div>
      <CartItemList />
      <ShippingSelector onSelect={setSelectedShipping} />
      <OrderSummary shippingMethod={selectedShipping} />
      <CheckoutButton />
    </div>
  );
};
```

----

## 🎯 Naming Conventions

### Component Names Should Be Descriptive
```tsx
// ✅ GOOD - Clear purpose
LoginForm
UserProfileCard
ProductCheckoutFlow
CommentList
SearchBar
NotificationBell

// ❌ TOO VAGUE
Form           // Which form?
Card           // That's a UI component
List           // Too generic
Widget         // What does it do?
```

### File Structure for Complex Features
```
components/features/
├── UserProfile/
│   ├── UserProfileCard.tsx           # Main component
│   ├── UserProfileHeader.tsx         # Sub-component (only used here)
│   ├── UserProfileStats.tsx          # Sub-component
│   └── index.ts                      # Export { UserProfileCard }
├── ProductCheckout/
│   ├── ProductCheckoutForm.tsx
│   ├── ShippingStep.tsx
│   ├── PaymentStep.tsx
│   ├── ReviewStep.tsx
│   └── index.ts
└── CommentSection/
    ├── CommentList.tsx
    ├── CommentItem.tsx
    ├── CommentForm.tsx
    └── index.ts
```

## 🚫 Common Mistakes

### Mistake 1: Too Much Logic in Component
```tsx
// ❌ BAD - Business logic in component
export const PricingCalculator = ({ items }: { items: CartItem[] }) => {
  const calculateTotal = () => {
    // 100 lines of pricing logic with tax, discounts, shipping
    return total;
  };

  return <div>${calculateTotal()}</div>;
};

// ✅ GOOD - Logic in service, component stays thin
export const PricingCalculator = ({ items }: { items: CartItem[] }) => {
  const total = pricingService.calculateTotal(items);
  return <div>${total}</div>;
};
```

### Mistake 2: Not Handling Edge Cases
```tsx
// ❌ BAD - Assumes data is always present
export const UserCard = ({ userId }: { userId: string }) => {
  const { data:  user } = useUser(userId);

  return <div>{user.name}</div>; // ❌ Crashes if user is undefined! 
};

// ✅ GOOD - Handles all states
export const UserCard = ({ userId }: { userId: string }) => {
  const { data: user, isLoading, error } = useUser(userId);

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorCard message="Failed to load user" />;
  if (!user) return null;

  return <div>{user.name}</div>;
};
```

### Mistake 3: Direct Service Calls (Skip Hooks)
```tsx
// ❌ BAD - Direct service call
export const UserList = () => {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    userService.getAll().then(setUsers); // ❌ Manual data management
  }, []);

  return <div>{users.map(/* ... */)}</div>;
};

// ✅ GOOD - Use React Query hook
export const UserList = () => {
  const { data:  users, isLoading } = useUsers(); // ✅ Automatic caching, refetching

  if (isLoading) return <Skeleton />;

  return <div>{users.map(/* ... */)}</div>;
};
```

### Mistake 4: Mixing UI and Feature Logic
```tsx
// ❌ BAD - Styling and business logic mixed
// components/features/Button.tsx
export const SubmitButton = () => {
  const handleSubmit = async () => {
    // Business logic
  };

  return (
    <button className="bg-blue-500 px-4 py-2 rounded"> {/* ❌ Styling in feature */}
      Submit
    </button>
  );
};

// ✅ GOOD - Separate concerns
// components/features/CheckoutButton.tsx
export const CheckoutButton = () => {
  const handleCheckout = async () => {
    // Business logic
  };

  return <Button onClick={handleCheckout}>Checkout</Button>; // ✅ Use UI component
};
```

---

## 📚 Quick Reference

### When to Split a Feature Component

Split when:
- ✅ Component is > 200 lines
- ✅ Multiple distinct responsibilities
- ✅ Sub-components are only used within this feature (create folder)
- ✅ Logic can be extracted to custom hook

Don't split when:
- ❌ Component is already focused and < 150 lines
- ❌ Sub-components would be used elsewhere (make them UI components)
- ❌ Splitting adds complexity without clarity

---