---
description: 'Standards for pure, reusable design system components in components/ui/'
applyTo: 'src/components/ui/**'
---

## 🎯 What Are UI Components?

UI components are **presentational, reusable building blocks** of your design system.   
Think:  buttons, inputs, cards, badges, modals, dropdowns. 

### Core Principles:
- ✅ **Container-Presentational Pattern** - Build presentational component only
- ✅ **Pure & Predictable:** Same props always produce same output
- ✅ **Reusable:** Can be used anywhere in the app
- ✅ **No Business Logic:** No API calls, no auth checks, no feature-specific code
- ✅ **Composable:** Can be combined to build complex interfaces

---

## ✅ What UI Components CAN Do

### 1. Accept Props for Customization
```tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  disabled?: boolean;
  onClick?: () => void;
  children:  ReactNode;
}
```

### 2. Render Visual Elements
```tsx
export const Button: FC<ButtonProps> = ({ 
  variant = 'primary',
  size = 'md', 
  children,
  ... props 
}) => {
  return (
    <button 
      className={buttonVariants({ variant, size })} 
      {...props}
    >
      {children}
    </button>
  );
};
```

### 3. Handle UI-Only State (Internal)
```tsx
// ✅ GOOD - Internal hover/focus state is OK
export const Accordion = ({ title, children }: AccordionProps) => {
  const [isOpen, setIsOpen] = useState(false); // ✅ UI-only state

  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>
        {title}
      </button>
      {isOpen && <div>{children}</div>}
    </div>
  );
};
```
### 4. Forward Refs
```tsx
import { forwardRef } from 'react';

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ className, ...props }, ref) => {
    return (
      <input
        ref={ref}
        className={cn('base-input-styles', className)}
        {...props}
      />
    );
  }
);

Input.displayName = 'Input';
```

---

## ❌ What UI Components CANNOT Do

### 1. Import from Feature or Page Layers
```tsx
// ❌ BAD - UI importing business logic
import { useAuth } from '@/features/auth/useAuth';
import { DashboardLayout } from '@/components/layouts/DashboardLayout';

export const Button = () => {
  const { user } = useAuth(); // ❌ NO!  Business logic in UI
  // ...
};
```

**Why? ** This couples your design system to specific features, making it non-reusable.

**Fix:** Pass data as props instead:
```tsx
// ✅ GOOD - Data passed from parent
interface ButtonProps {
  showUserBadge?: boolean;
  userName?: string;
}

export const Button = ({ showUserBadge, userName }: ButtonProps) => {
  return (
    <button>
      {showUserBadge && <span>{userName}</span>}
    </button>
  );
};
```

### 2. Make API Calls or Data Fetching
```tsx
// ❌ BAD - API call in UI component
export const UserAvatar = ({ userId }: { userId: string }) => {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`) // ❌ NO! Side effect in UI
      .then(res => res.json())
      .then(setUser);
  }, [userId]);

  return <img src={user?.avatar} />;
};
```

**Why?** UI components should be side-effect free and easy to test.

**Fix:** Pass data from parent:
```tsx
// ✅ GOOD - Data passed as prop
interface UserAvatarProps {
  src:  string;
  alt: string;
  size?: 'sm' | 'md' | 'lg';
}

export const UserAvatar:  FC<UserAvatarProps> = ({ src, alt, size = 'md' }) => {
  return (
    <img 
      src={src} 
      alt={alt} 
      className={avatarSizes[size]} 
    />
  );
};
```

### 3. Access Global State (Zustand, Redux, Context)
```tsx
// ❌ BAD - Accessing global state
import { useThemeStore } from '@/stores/theme-store';

export const Card = ({ children }: CardProps) => {
  const theme = useThemeStore((state) => state.theme); // ❌ NO! 
  
  return <div className={theme === 'dark' ? 'dark' : 'light'}>{children}</div>;
};
```

**Why?** Makes the component dependent on specific state structure.

**Fix:** Use props or CSS variables:
```tsx
// ✅ GOOD - Use props
export const Card = ({ variant, children }: CardProps) => {
  return <div className={cardVariants({ variant })}>{children}</div>;
};

// ✅ BETTER - Use CSS variables (theme-aware without JS)
export const Card = ({ children }: CardProps) => {
  return (
    <div className="bg-card text-card-foreground"> {/* Tailwind CSS vars */}
      {children}
    </div>
  );
};
```

### 4. Contain Feature-Specific Logic
```tsx
// ❌ BAD - Payment-specific logic in UI component
export const Button = ({ onClick }: ButtonProps) => {
  const handleClick = () => {
    // Validate payment form
    // Process payment
    // Track analytics
    onClick?.();
  };

  return <button onClick={handleClick}>Pay Now</button>;
};
```

**Fix:** Keep UI generic, move logic to feature component:
```tsx
// ✅ GOOD - Generic UI button
export const Button = ({ onClick, children }: ButtonProps) => {
  return <button onClick={onClick}>{children}</button>;
};

// Feature component handles business logic
// components/features/PaymentButton.tsx
export const PaymentButton = () => {
  const handlePayment = () => {
    validatePaymentForm();
    processPayment();
    trackAnalytics();
  };

  return <Button onClick={handlePayment}>Pay Now</Button>;
};
```

---

## 📐 UI Component Template

Use this structure for all UI components:

```tsx
// components/ui/ComponentName.tsx
import { type FC, type HTMLAttributes } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

// 1. Define variants using CVA
const componentVariants = cva(
  'base-classes-that-always-apply', // Base styles
  {
    variants: {
      variant: {
        default: 'default-variant-styles',
        primary: 'primary-variant-styles',
        secondary: 'secondary-variant-styles',
      },
      size: {
        sm: 'small-size-styles',
        md: 'medium-size-styles',
        lg: 'large-size-styles',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'md',
    },
  }
);

// 2. Define props interface
interface ComponentNameProps
  extends HTMLAttributes<HTMLDivElement>, // Extend native HTML props
    VariantProps<typeof componentVariants> { // Add variant props
  /**
   * Brief description of what this prop does
   */
  customProp?: string;
  /**
   * Boolean props use "is/has/should" prefix
   */
  isDisabled?: boolean;
}

// 3. Component implementation
export const ComponentName: FC<ComponentNameProps> = ({
  variant,
  size,
  customProp,
  isDisabled,
  className,
  children,
  ...props // Spread remaining HTML props
}) => {
  return (
    <div
      className={cn(
        componentVariants({ variant, size }), // Apply variants
        isDisabled && 'opacity-50 cursor-not-allowed', // Conditional classes
        className // Allow prop-based overrides
      )}
      aria-disabled={isDisabled}
      {...props}
    >
      {children}
    </div>
  );
};

ComponentName.displayName = 'ComponentName';
```

---

## 🎨 Styling Guidelines

### Use Class Variance Authority (CVA)
```tsx
import { cva } from 'class-variance-authority';

const buttonVariants = cva(
  // Base styles (always applied)
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover: bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
      },
      size:  {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant:  'default',
      size:  'default',
    },
  }
);
```

### Compound Variants (When Combinations Matter)
```tsx
const badgeVariants = cva('badge-base', {
  variants: {
    variant: { solid: 'badge-solid', outline: 'badge-outline' },
    color: { red: 'badge-red', blue: 'badge-blue' },
  },
  compoundVariants: [
    {
      variant: 'solid',
      color: 'red',
      className: 'bg-red-600 text-white', // Special styling for solid + red
    },
  ],
});
```

### Use `cn()` for Conditional Classes
```tsx
import { cn } from '@/lib/utils';

<div className={cn(
  'base-classes',
  isActive && 'active-classes',
  isDisabled && 'disabled-classes',
  size === 'lg' && 'large-classes',
  className // Always allow prop overrides last
)} />
```

## 🚫 Common Mistakes

### Mistake 1: Too Many Props (Prop Explosion)
```tsx
// ❌ BAD - Too many individual props
<Card
  headerText="Title"
  showHeader={true}
  headerColor="blue"
  bodyText="Content"
  showFooter={true}
  footerText="Footer"
  footerAlign="right"
/>

// ✅ GOOD - Composition
<Card>
  <Card.Header className="text-blue-600">Title</Card.Header>
  <Card.Content>Content</Card.Content>
  <Card.Footer className="text-right">Footer</Card.Footer>
</Card>
```

### Mistake 3: Inline Styles
```tsx
// ❌ BAD - Inline styles
<button style={{ backgroundColor: 'blue', padding: '10px' }}>

// ✅ GOOD - Tailwind classes or CSS modules
<button className="bg-blue-600 px-4 py-2">
```

---
