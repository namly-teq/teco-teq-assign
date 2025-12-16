---
description: 'Guidelines for layout components that structure pages and provide consistent shells'
applyTo: 'src/components/layouts/**'
---

## 🎯 What Are Layout Components?

Layout components are **structural wrappers** that define page organization and navigation.   
Think:  `AppShell`, `DashboardLayout`, `AuthLayout`, `MainLayout`, `Header`, `Sidebar`, `Footer`.

### Core Principles:
- ✅ **Provide page structure** - Define where content goes
- ✅ **Consistent navigation** - Headers, sidebars, footers
- ✅ **Reusable across pages** - Multiple pages share the same layout
- ✅ **Handle global UI** - Navigation, breadcrumbs, user menu
- ✅ **Responsive by default** - Mobile, tablet, desktop layouts

---

## ✅ What Layout Components CAN Do

### 1. Define Page Structure
```tsx
// ✅ GOOD - Basic app shell
// components/layouts/AppShell. tsx
import { type FC, type ReactNode } from 'react';
import { Header } from './Header';
import { Footer } from './Footer';

interface AppShellProps {
  children: ReactNode;
}

export const AppShell: FC<AppShellProps> = ({ children }) => {
  return (
    <div className="flex min-h-screen flex-col">
      <Header />
      <main className="flex-1 container mx-auto px-4 py-6">
        {children}
      </main>
      <Footer />
    </div>
  );
};
```

### 2. Compose Feature Components (Navigation)
```tsx
// ✅ GOOD - Dashboard layout with sidebar
// components/layouts/DashboardLayout.tsx
import { type FC, type ReactNode } from 'react';
import { Sidebar } from '@/components/features/Sidebar';
import { DashboardHeader } from '@/components/features/DashboardHeader';
import { Breadcrumbs } from '@/components/features/Breadcrumbs';

interface DashboardLayoutProps {
  children: ReactNode;
  breadcrumbs?:  Array<{ label: string; href: string }>;
}

export const DashboardLayout: FC<DashboardLayoutProps> = ({ 
  children,
  breadcrumbs 
}) => {
  return (
    <div className="flex h-screen overflow-hidden">
      {/* Sidebar */}
      <Sidebar />

      {/* Main content area */}
      <div className="flex flex-1 flex-col overflow-hidden">
        <DashboardHeader />
        
        {breadcrumbs && (
          <div className="border-b px-6 py-3">
            <Breadcrumbs items={breadcrumbs} />
          </div>
        )}

        <main className="flex-1 overflow-y-auto p-6">
          {children}
        </main>
      </div>
    </div>
  );
};
```

### 3. Handle Responsive Layouts
```tsx
// ✅ GOOD - Mobile-responsive layout
// components/layouts/MainLayout.tsx
import { type FC, type ReactNode, useState } from 'react';
import { Header } from './Header';
import { MobileNav } from '@/components/features/MobileNav';
import { DesktopSidebar } from '@/components/features/DesktopSidebar';
import { useMediaQuery } from '@/hooks/useMediaQuery';
import { Button } from '@/components/ui/Button';
import { MenuIcon } from '@/components/ui/Icons';

interface MainLayoutProps {
  children: ReactNode;
}

export const MainLayout: FC<MainLayoutProps> = ({ children }) => {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const [isMobileMenuOpen, setIsMobileMenuOpen] = useState(false);

  return (
    <div className="relative flex min-h-screen">
      {/* Desktop sidebar */}
      {! isMobile && <DesktopSidebar />}

      {/* Mobile menu */}
      {isMobile && (
        <MobileNav 
          isOpen={isMobileMenuOpen} 
          onClose={() => setIsMobileMenuOpen(false)} 
        />
      )}

      {/* Main content */}
      <div className="flex flex-1 flex-col">
        <Header 
          leftSlot={
            isMobile && (
              <Button 
                variant="ghost" 
                size="icon"
                onClick={() => setIsMobileMenuOpen(true)}
              >
                <MenuIcon />
              </Button>
            )
          }
        />
        <main className="flex-1 p-4 md:p-6">
          {children}
        </main>
      </div>
    </div>
  );
};
```

### 4. Provide Layout Context
```tsx
// ✅ GOOD - Layout with context for child components
// components/layouts/SettingsLayout.tsx
import { type FC, type ReactNode, createContext, useContext, useState } from 'react';
import { Tabs, TabsList, TabsTrigger } from '@/components/ui/Tabs';

interface SettingsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const SettingsContext = createContext<SettingsContextValue | null>(null);

export const useSettingsLayout = () => {
  const context = useContext(SettingsContext);
  if (!context) {
    throw new Error('useSettingsLayout must be used within SettingsLayout');
  }
  return context;
};

interface SettingsLayoutProps {
  children: ReactNode;
  defaultTab?: string;
}

export const SettingsLayout: FC<SettingsLayoutProps> = ({ 
  children,
  defaultTab = 'general' 
}) => {
  const [activeTab, setActiveTab] = useState(defaultTab);

  return (
    <SettingsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="container mx-auto max-w-6xl py-6">
        <h1 className="mb-6 text-3xl font-bold">Settings</h1>
        
        <div className="flex flex-col gap-6 md:flex-row">
          {/* Settings navigation */}
          <aside className="w-full md:w-48">
            <Tabs value={activeTab} onValueChange={setActiveTab} orientation="vertical">
              <TabsList className="flex flex-col items-stretch">
                <TabsTrigger value="general">General</TabsTrigger>
                <TabsTrigger value="security">Security</TabsTrigger>
                <TabsTrigger value="notifications">Notifications</TabsTrigger>
                <TabsTrigger value="billing">Billing</TabsTrigger>
              </TabsList>
            </Tabs>
          </aside>

          {/* Settings content */}
          <main className="flex-1">
            {children}
          </main>
        </div>
      </div>
    </SettingsContext.Provider>
  );
};
```

### 5. Access Global State (Auth, Theme)
```tsx
// ✅ GOOD - Protected layout with auth check
// components/layouts/ProtectedLayout.tsx
import { type FC, type ReactNode, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { useAuthStore } from '@/stores/auth-store';
import { Spinner } from '@/components/ui/Spinner';
import { DashboardLayout } from './DashboardLayout';

interface ProtectedLayoutProps {
  children: ReactNode;
  requiredRole?: 'admin' | 'user';
}

export const ProtectedLayout: FC<ProtectedLayoutProps> = ({ 
  children,
  requiredRole 
}) => {
  const router = useRouter();
  const { user, isAuthenticated, isLoading } = useAuthStore();

  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      router.push('/login');
    }
  }, [isAuthenticated, isLoading, router]);

  useEffect(() => {
    if (requiredRole && user && user.role !== requiredRole) {
      router.push('/unauthorized');
    }
  }, [user, requiredRole, router]);

  // Show loading while checking auth
  if (isLoading) {
    return (
      <div className="flex h-screen items-center justify-center">
        <Spinner size="lg" />
      </div>
    );
  }

  // Don't render until authenticated
  if (!isAuthenticated || ! user) {
    return null;
  }

  // Role check
  if (requiredRole && user.role !== requiredRole) {
    return null;
  }

  return <DashboardLayout>{children}</DashboardLayout>;
};
```

### 6. Handle Layout-Specific State
```tsx
// ✅ GOOD - Collapsible sidebar state
// components/layouts/AdminLayout.tsx
import { type FC, type ReactNode, useState } from 'react';
import { cn } from '@/lib/utils';
import { AdminSidebar } from '@/components/features/AdminSidebar';
import { AdminHeader } from '@/components/features/AdminHeader';
import { Button } from '@/components/ui/Button';
import { ChevronLeftIcon, ChevronRightIcon } from '@/components/ui/Icons';

interface AdminLayoutProps {
  children: ReactNode;
}

export const AdminLayout: FC<AdminLayoutProps> = ({ children }) => {
  const [isSidebarCollapsed, setIsSidebarCollapsed] = useState(false);

  return (
    <div className="flex h-screen">
      {/* Sidebar with collapse */}
      <aside
        className={cn(
          'relative border-r bg-muted/40 transition-all duration-300',
          isSidebarCollapsed ? 'w-16' : 'w-64'
        )}
      >
        <AdminSidebar isCollapsed={isSidebarCollapsed} />
        
        {/* Collapse toggle */}
        <Button
          variant="ghost"
          size="icon"
          className="absolute -right-3 top-6 z-10 h-6 w-6 rounded-full border bg-background"
          onClick={() => setIsSidebarCollapsed(!isSidebarCollapsed)}
        >
          {isSidebarCollapsed ? <ChevronRightIcon /> : <ChevronLeftIcon />}
        </Button>
      </aside>

      {/* Main content */}
      <div className="flex flex-1 flex-col overflow-hidden">
        <AdminHeader />
        <main className="flex-1 overflow-y-auto p-6">
          {children}
        </main>
      </div>
    </div>
  );
};
```

### 7. Integrate with Meta/SEO
```tsx
// ✅ GOOD - Layout with SEO defaults (Next.js example)
// components/layouts/MarketingLayout.tsx
import { type FC, type ReactNode } from 'react';
import Head from 'next/head';
import { MarketingHeader } from '@/components/features/MarketingHeader';
import { MarketingFooter } from '@/components/features/MarketingFooter';

interface MarketingLayoutProps {
  children: ReactNode;
  title?: string;
  description?:  string;
}

export const MarketingLayout:  FC<MarketingLayoutProps> = ({ 
  children,
  title = 'Your Product',
  description = 'Default description'
}) => {
  const fullTitle = title === 'Your Product' ? title : `${title} | Your Product`;

  return (
    <>
      <Head>
        <title>{fullTitle}</title>
        <meta name="description" content={description} />
        <meta property="og:title" content={fullTitle} />
        <meta property="og:description" content={description} />
      </Head>

      <div className="flex min-h-screen flex-col">
        <MarketingHeader />
        <main className="flex-1">
          {children}
        </main>
        <MarketingFooter />
      </div>
    </>
  );
};
```

---

## ❌ What Layout Components CANNOT Do

### 1.  Contain Business Logic
```tsx
// ❌ BAD - Processing payments in layout
export const CheckoutLayout = ({ children }: { children: ReactNode }) => {
  const processPayment = async () => {
    // 50 lines of payment logic ❌ NO!  
  };

  return (
    <div>
      <button onClick={processPayment}>Pay</button>
      {children}
    </div>
  );
};
```

**Why? ** Layouts should be structural, not functional.  
**Fix:** Move business logic to feature components used inside pages.

### 2. Fetch Data Directly
```tsx
// ❌ BAD - Data fetching in layout
export const DashboardLayout = ({ children }: { children: ReactNode }) => {
  const [stats, setStats] = useState(null);

  useEffect(() => {
    fetch('/api/stats').then(/* ... */); // ❌ NO!  
  }, []);

  return (
    <div>
      <div>Stats:  {stats}</div>
      {children}
    </div>
  );
};
```

**Why?** Data fetching should be page/feature-specific, not layout-wide.

**Exception:** Navigation data that's truly layout-level: 
```tsx
// ✅ OK - Fetch navigation menu (layout-level concern)
export const AppLayout = ({ children }: { children: ReactNode }) => {
  const { data:  menuItems } = useNavigationMenu(); // ✅ Navigation is layout concern

  return (
    <div>
      <Header menuItems={menuItems} />
      {children}
    </div>
  );
};
```

### 3. Be Too Specific to One Page
```tsx
// ❌ BAD - "Layout" that's really just a page component
// components/layouts/UserProfileLayout.tsx
export const UserProfileLayout = ({ userId }: { userId: string }) => {
  const { data: user } = useUser(userId); // ❌ Too specific!  

  return (
    <div>
      <h1>{user.name}'s Profile</h1>
      {/* User-specific content */}
    </div>
  );
};
```

**Why?** This is really a page component, not a reusable layout.  
**Fix:** If it's only used on one page, it's not a layout.

```tsx
// ✅ GOOD - Truly reusable profile layout
// components/layouts/ProfileLayout.tsx
export const ProfileLayout = ({ children }: { children: ReactNode }) => {
  return (
    <div className="container mx-auto max-w-4xl">
      <div className="grid grid-cols-1 gap-6 md:grid-cols-3">
        <aside className="md:col-span-1">
          {/* Profile sidebar - content provided by page */}
        </aside>
        <main className="md:col-span-2">
          {children}
        </main>
      </div>
    </div>
  );
};
```

---

## 🚫 Common Mistakes

### Mistake 1: Layouts That Are Really Pages
```tsx
// ❌ BAD - This is a page, not a layout
// components/layouts/UserDashboardLayout.tsx
export const UserDashboardLayout = () => {
  const { data: user } = useUser();
  const { data: stats } = useStats();

  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <StatsCards stats={stats} />
      <RecentActivity />
    </div>
  );
};
```

**Fix:** This should be a page component that uses a layout: 
```tsx
// ✅ GOOD - Separate layout and page
// app/dashboard/page.tsx
export default function DashboardPage() {
  const { data: user } = useUser();
  const { data: stats } = useStats();

  return (
    <DashboardLayout>
      <h1>Welcome, {user.name}</h1>
      <StatsCards stats={stats} />
      <RecentActivity />
    </DashboardLayout>
  );
}
```

### Mistake 2: Fixed Heights Breaking Content
```tsx
// ❌ BAD - Fixed height cuts off content
export const BadLayout = ({ children }: { children: ReactNode }) => {
  return (
    <div className="h-screen">
      <header className="h-16">Header</header>
      <main className="h-[calc(100vh-4rem)]"> {/* ❌ Doesn't account for footer */}
        {children}
      </main>
      <footer className="h-20">Footer</footer> {/* Gets cut off!  */}
    </div>
  );
};

// ✅ GOOD - Flexible layout
export const GoodLayout = ({ children }: { children: ReactNode }) => {
  return (
    <div className="flex min-h-screen flex-col">
      <header className="h-16 flex-shrink-0">Header</header>
      <main className="flex-1 overflow-y-auto">
        {children}
      </main>
      <footer className="flex-shrink-0">Footer</footer>
    </div>
  );
};
```

### Mistake 3: Non-Responsive Navigation
```tsx
// ❌ BAD - Doesn't adapt to mobile
export const BadLayout = ({ children }: { children:  ReactNode }) => {
  return (
    <div className="flex">
      <aside className="w-64"> {/* ❌ Takes up too much space on mobile */}
        <Nav />
      </aside>
      <main>{children}</main>
    </div>
  );
};

// ✅ GOOD - Responsive navigation
export const GoodLayout = ({ children }: { children:  ReactNode }) => {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="flex">
      {isMobile ?  (
        <Drawer isOpen={isOpen} onClose={() => setIsOpen(false)}>
          <Nav />
        </Drawer>
      ) : (
        <aside className="w-64">
          <Nav />
        </aside>
      )}
      <main className="flex-1">{children}</main>
    </div>
  );
};
```

### Mistake 4: Missing Semantic HTML
```tsx
// ❌ BAD - No semantic landmarks
export const BadLayout = ({ children }: { children: ReactNode }) => {
  return (
    <div>
      <div>Header</div> {/* ❌ Should be <header> */}
      <div>{children}</div> {/* ❌ Should be <main> */}
      <div>Footer</div> {/* ❌ Should be <footer> */}
    </div>
  );
};

// ✅ GOOD - Proper landmarks
export const GoodLayout = ({ children }: { children: ReactNode }) => {
  return (
    <div>
      <header>Header</header>
      <main>{children}</main>
      <footer>Footer</footer>
    </div>
  );
};
```
