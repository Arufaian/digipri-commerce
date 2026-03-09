# Tutorial Lengkap: Multi-Actor Authentication dalam Laravel dengan Fortify & Svelte

## Daftar Isi

1. [Pengantar](#pengantar)
2. [Konsep & Arsitektur](#konsep--arsitektur)
3. [Auth Flow Original di Aplikasi Anda](#auth-flow-original-di-aplikasi-anda)
4. [Implementasi Multi-Actor (Step-by-Step)](#implementasi-multi-actor-step-by-step)
5. [Middleware & Route Redirect](#middleware--route-redirect)
6. [Route Structure untuk Multi-Actor](#route-structure-untuk-multi-actor)
7. [Frontend: Conditional Rendering Berdasarkan Role](#frontend-conditional-rendering-berdasarkan-role)
8. [Testing Multi-Actor Authentication](#testing-multi-actor-authentication)
9. [Best Practices & Security](#best-practices--security)
10. [Troubleshooting](#troubleshooting)

---

## Pengantar

### Apa yang Akan Kita Pelajari?

Dokumentasi ini menjelaskan bagaimana mengimplementasikan multi-actor authentication dalam aplikasi Laravel e-commerce Anda dengan pattern:

- **Admin**: Akses dashboard khusus untuk mengelola produk, pesanan, dan sistem
- **Customer**: Akses public catalog tanpa login, login untuk transaksi & profile management
- **Single Login Form** dengan middleware redirect logic berdasarkan role

Ini adalah pattern yang digunakan oleh website e-commerce profesional seperti Otten Coffee (referensi Anda).

### Use Case Aplikasi Anda

Saat ini aplikasi Anda memiliki:

- ✅ Single User model dengan Fortify authentication
- ✅ Email verification
- ✅ Two-factor authentication (TOTP)
- ✅ Svelte + Inertia.js frontend

Yang akan kita tambahkan:

- ✅ `role` column di tabel users (`'admin'` atau `'customer'`)
- ✅ Middleware redirect logic yang mengarahkan user ke dashboard mereka
- ✅ Public catalog pages (/products, /services) accessible tanpa login
- ✅ Admin dashboard (/admin/dashboard)
- ✅ Customer dashboard + profile management (/customer/dashboard, /customer/profile)

### Mengapa Pattern Ini?

**Keuntungan approach ini:**

1. ✅ **Sederhana** - Hanya tambah 1 kolom `role` di users table
2. ✅ **Secure** - Fortify handle password hashing dan validation
3. ✅ **Maintainable** - Single login form, middleware handle routing
4. ✅ **Scalable** - Mudah tambah actor baru (seller, vendor, dll)
5. ✅ **Fortify-native** - Tidak perlu override Fortify behavior
6. ✅ **Svelte-friendly** - Conditional rendering di frontend mudah

### Persyaratan

Sebelum memulai, pastikan Anda sudah punya:

- ✅ Laravel 12 dengan Fortify v1 (sudah ada di aplikasi Anda)
- ✅ Inertia.js v2 dan Svelte (sudah ada di aplikasi Anda)
- ✅ Database migration setup (sudah ada)
- ✅ Familiarity dengan Laravel routes dan middleware (basic level)

---

## Konsep & Arsitektur

### Apa itu "Actor" atau "Role"?

**Actor** adalah persona pengguna dalam sistem yang memiliki akses dan permissions berbeda:

```
SISTEM E-COMMERCE ANDA

PUBLIC AREA (Tanpa login)
├── /products - Browse products
├── /services - Browse services
└── /products/{id} - View product details

↓ User klik "Login" → /login

LOGIN FORM (Single entry point)
├── Email + Password
└── Fortify handle validation & auth

↓ Middleware check role

ADMIN AREA (/admin/*)
├── /admin/dashboard
├── /admin/products
├── /admin/orders
└── ...admin-only routes

CUSTOMER AREA (/customer/*)
├── /customer/dashboard
├── /customer/orders
├── /customer/profile
├── /customer/addresses
└── ...customer-specific routes
```

### Database Schema

**Tabel `users` sebelum (current):**

```
id | name | email | password | email_verified_at | created_at | updated_at
```

**Tabel `users` setelah (dengan multi-actor):**

```
id | name | email | password | role | email_verified_at | created_at | updated_at
                                ↑
                         KOLOM BARU: 'admin' atau 'customer'
```

Enum values untuk `role`:

- `'admin'` - Full access ke dashboard admin
- `'customer'` - Regular customer, akses public catalog dan customer area

### Terminology

**Guard**: Authentication mechanism (di Laravel ada `web` guard untuk session-based auth)  
**Role/Actor**: Type of user (admin, customer)  
**Middleware**: Code yang run sebelum route handler (bisa check role, redirect, dll)  
**Redirect**: Mengirim user ke URL berbeda setelah login based on role

---

## Auth Flow Original di Aplikasi Anda

### Current Architecture (Sebelum Multi-Actor)

Mari kita lihat bagaimana auth bekerja saat ini di aplikasi Anda:

```
USER FLOW:
1. User visit /  (Welcome page)
2. User click Login button
3. Form POST to /login
4. Fortify AuthenticatedSessionController handle POST
5. Check email + password
6. If valid: Create session, redirect to /dashboard
7. HandleInertiaRequests middleware inject user into props
```

**Files yang bertanggung jawab:**

| File                                            | Role             | Deskripsi                                           |
| ----------------------------------------------- | ---------------- | --------------------------------------------------- |
| `config/fortify.php`                            | Configuration    | Define Fortify features (login, register, 2FA, dll) |
| `config/auth.php`                               | Configuration    | Define guards dan auth providers                    |
| `app/Models/User.php`                           | Model            | User model dengan auth traits                       |
| `app/Providers/FortifyServiceProvider.php`      | Service Provider | Map views ke Inertia components, configure actions  |
| `routes/web.php`                                | Routes           | Define main routes, include settings routes         |
| `routes/settings.php`                           | Routes           | Settings routes untuk profile, password, 2FA        |
| `resources/js/pages/auth/Login.svelte`          | Frontend         | Login form component                                |
| `app/Http/Middleware/HandleInertiaRequests.php` | Middleware       | Inject shared props (termasuk auth user)            |

### Detailed Login Flow

Berikut step-by-step apa yang terjadi saat login:

**Step 1: User submits login form**

```svelte
<!-- resources/js/pages/auth/Login.svelte -->
<script>
    import { store } from '@/routes/login';

    async function handleSubmit(form) {
        await store.post(); // POST ke /login
    }
</script>
```

**Step 2: Fortify AuthenticatedSessionController@store handle POST**

Fortify's built-in controller (Anda tidak perlu edit)

- POST /login → AuthenticatedSessionController@store
- Validate email & password
- Find user by email
- Check password menggunakan Hash::check()
- If valid: Session::regenerate() untuk security
- Redirect ke 'home' (default /dashboard)

**Step 3: User redirected ke dashboard**

Saat ini redirect hardcoded ke `/dashboard`. Di step ini, Fortify tidak tahu tentang roles.

**Step 4: Middleware inject user ke props**

```php
// app/Http/Middleware/HandleInertiaRequests.php
public function share(Request $request): array
{
    return [
        'auth' => [
            'user' => $request->user(),
        ],
    ];
}
```

Setiap Inertia response akan include `auth.user` di frontend.

---

## Implementasi Multi-Actor (Step-by-Step)

Sekarang kita akan menambahkan multi-actor support dengan 5 langkah besar:

1. **Add `role` column ke users table (Migration)**
2. **Update User model untuk handle roles**
3. **Create middleware untuk redirect based on role**
4. **Create controller untuk manage admin/customer logic**
5. **Create routes untuk /admin/_ dan /customer/_**

### Step 1: Create Migration untuk Add Role Column

Pertama, kita perlu membuat migration untuk add `role` column ke users table.

**Command:**

```bash
sail artisan make:migration add_role_to_users_table
```

**File**: `database/migrations/[timestamp]_add_role_to_users_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->enum('role', ['admin', 'customer'])
                ->default('customer')
                ->after('email');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('role');
        });
    }
};
```

**Jalankan migration:**

```bash
sail artisan migrate
```

**Verifikasi di database:**

```bash
sail mysql
USE laravel;
DESCRIBE users;
# Seharusnya ada column 'role' dengan enum values 'admin' dan 'customer'
```

### Step 2: Update User Model

Edit `app/Models/User.php` untuk add role-related methods:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\TwoFactorAuthenticatable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    use HasFactory, Notifiable, TwoFactorAuthenticatable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role',  // ADD THIS
    ];

    protected $hidden = [
        'password',
        'remember_token',
        'two_factor_secret',
        'two_factor_recovery_codes',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
            'two_factor_confirmed_at' => 'datetime',
        ];
    }

    // ADD HELPER METHODS

    /**
     * Check if user is admin.
     */
    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }

    /**
     * Check if user is customer.
     */
    public function isCustomer(): bool
    {
        return $this->role === 'customer';
    }

    /**
     * Check if user has specific role.
     */
    public function hasRole(string $role): bool
    {
        return $this->role === $role;
    }
}
```

**Sekarang user model Anda punya methods:**

```php
$user = Auth::user();

$user->isAdmin();          // true jika role='admin'
$user->isCustomer();       // true jika role='customer'
$user->hasRole('admin');   // true jika role='admin'
```

### Step 3: Create Middleware untuk Redirect Based on Role

Sekarang kita create middleware yang akan redirect user ke dashboard mereka based on role setelah login.

**Command:**

```bash
sail artisan make:middleware AuthenticateByRole
```

**File**: `app/Http/Middleware/AuthenticateByRole.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AuthenticateByRole
{
    /**
     * Handle an incoming request.
     *
     * Middleware ini redirect user ke dashboard mereka berdasarkan role
     * setelah login.
     *
     * Flow:
     * 1. User login via /login
     * 2. Fortify redirect ke /dashboard (default)
     * 3. Middleware ini intercept request ke /dashboard
     * 4. Check user.role:
     *    - Jika 'admin' → redirect ke /admin/dashboard
     *    - Jika 'customer' → stay di /dashboard
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->user() && $request->path() === 'dashboard') {
            if ($request->user()->isAdmin()) {
                return redirect('/admin/dashboard');
            }
        }

        return $next($request);
    }
}
```

**Register middleware di bootstrap/app.php:**

Buka `bootstrap/app.php` dan cari section `withMiddleware()`, lalu tambahkan:

```php
use App\Http\Middleware\AuthenticateByRole;

->withMiddleware(function (Middleware $middleware) {
    // ... existing middleware ...

    $middleware->web(append: [
        AuthenticateByRole::class,
    ]);
})
```

**Bagaimana middleware ini bekerja:**

```
1. User POST /login dengan credentials
2. Fortify validate dan authenticate
3. Session created
4. Fortify redirect ke /dashboard (Fortify default home)
5. Middleware AuthenticateByRole intercept request
6. Check: user.role == 'admin'?
   - YES: redirect ke /admin/dashboard
   - NO: continue, show /dashboard
```

### Step 4: Create Admin & Customer Dashboard Controllers

Sekarang kita buat controllers untuk handle admin dan customer dashboards.

**Admin Dashboard Controller:**

```bash
sail artisan make:controller Admin/DashboardController
```

**File**: `app/Http/Controllers/Admin/DashboardController.php`

```php
<?php

namespace App\Http\Controllers\Admin;

use Inertia\Inertia;
use Inertia\Response;
use App\Http\Controllers\Controller;

class DashboardController extends Controller
{
    /**
     * Display admin dashboard.
     *
     * Hanya admin yang bisa akses (dijaga oleh middleware di route)
     */
    public function show(): Response
    {
        return Inertia::render('admin/Dashboard', [
            'stats' => [
                'total_orders' => 123,
                'total_revenue' => 50000,
                'total_products' => 45,
            ],
            'recent_orders' => [],
        ]);
    }
}
```

**Customer Dashboard Controller:**

```bash
sail artisan make:controller Customer/DashboardController
```

**File**: `app/Http/Controllers/Customer/DashboardController.php`

```php
<?php

namespace App\Http\Controllers\Customer;

use Inertia\Inertia;
use Inertia\Response;
use App\Http\Controllers\Controller;

class DashboardController extends Controller
{
    /**
     * Display customer dashboard.
     */
    public function show(): Response
    {
        return Inertia::render('customer/Dashboard', [
            'recent_orders' => auth()->user()->orders ?? [],
        ]);
    }
}
```

### Step 5: Create AdminOnly & CustomerOnly Middleware

Sekarang kita perlu 2 middleware tambahan untuk protect admin dan customer routes:

**Create Admin Middleware:**

```bash
sail artisan make:middleware AdminOnly
```

**File**: `app/Http/Middleware/AdminOnly.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AdminOnly
{
    /**
     * Handle an incoming request.
     *
     * Ensure user is authenticated AND has 'admin' role.
     * Jika tidak, abort dengan 403 (Forbidden).
     */
    public function handle(Request $request, Closure $next): Response
    {
        if (!$request->user() || !$request->user()->isAdmin()) {
            abort(403, 'Unauthorized. Admin access only.');
        }

        return $next($request);
    }
}
```

**Create Customer Middleware:**

```bash
sail artisan make:middleware CustomerOnly
```

**File**: `app/Http/Middleware/CustomerOnly.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CustomerOnly
{
    /**
     * Handle an incoming request.
     *
     * Ensure user is authenticated AND has 'customer' role.
     */
    public function handle(Request $request, Closure $next): Response
    {
        if (!$request->user() || !$request->user()->isCustomer()) {
            abort(403, 'Unauthorized. Customer access only.');
        }

        return $next($request);
    }
}
```

**Register middleware di bootstrap/app.php:**

```php
use App\Http\Middleware\AdminOnly;
use App\Http\Middleware\CustomerOnly;
use App\Http\Middleware\AuthenticateByRole;

->withMiddleware(function (Middleware $middleware) {
    // ... existing ...

    // Add named middleware untuk digunakan di routes
    $middleware->alias([
        'admin' => AdminOnly::class,
        'customer' => CustomerOnly::class,
    ]);

    // Append AuthenticateByRole ke web middleware stack
    $middleware->web(append: [
        AuthenticateByRole::class,
    ]);
})
```

### Step 6: Update Routes

Sekarang kita update routes untuk add `/admin/\*`` routes.

** dan `/customer/\*File**: `routes/web.php`

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Admin\DashboardController as AdminDashboardController;
use App\Http\Controllers\Customer\DashboardController as CustomerDashboardController;

// PUBLIC ROUTES (tanpa login)
Route::get('/', function () {
    return inertia('Welcome', [
        'canRegister' => Route::has('register'),
        'canResetPassword' => Route::has('password.request'),
    ]);
});

// Katalog produk dan services (public)
Route::get('/products', function () {
    return inertia('Products/Index');
})->name('products.index');

Route::get('/services', function () {
    return inertia('Services/Index');
})->name('services.index');

// AUTH ROUTES (Fortify handle these, no need to define explicitly)
// POST /login - AuthenticatedSessionController@store
// POST /register - RegisteredUserController@store
// POST /logout - AuthenticatedSessionController@destroy

// PROTECTED ROUTES (require auth)
Route::middleware(['auth', 'verified'])->group(function () {
    // Home/Dashboard (untuk customer)
    Route::get('/dashboard', function () {
        return inertia('Dashboard');
    })->name('dashboard');

    // Settings routes (shared for both admin & customer)
    require __DIR__ . '/settings.php';
});

// ADMIN ROUTES
Route::middleware(['auth', 'verified', 'admin'])->prefix('admin')->name('admin.')->group(function () {
    Route::get('/dashboard', [AdminDashboardController::class, 'show'])->name('dashboard');

    Route::get('/products', function () {
        return inertia('admin/Products/Index');
    })->name('products.index');

    Route::get('/orders', function () {
        return inertia('admin/Orders/Index');
    })->name('orders.index');
});

// CUSTOMER ROUTES
Route::middleware(['auth', 'verified', 'customer'])->prefix('customer')->name('customer.')->group(function () {
    Route::get('/dashboard', [CustomerDashboardController::class, 'show'])->name('dashboard');

    Route::get('/profile', function () {
        return inertia('customer/Profile');
    })->name('profile');

    Route::get('/orders', function () {
        return inertia('customer/Orders');
    })->name('orders');

    Route::get('/addresses', function () {
        return inertia('customer/Addresses');
    })->name('addresses');
});
```

---

## Middleware & Route Redirect

### Middleware Execution Flow

Setelah Anda implement langkah-langkah di atas, berikut adalah flow yang terjadi:

```
┌─────────────────────────────────────────────────────────────┐
│                     USER LOGIN                               │
│  1. User POST /login (email + password)                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              FORTIFY AUTHENTICATION                           │
│  1. AuthenticatedSessionController@store                     │
│  2. Validate email & password                                │
│  3. Session::regenerate()                                    │
│  4. Redirect to 'home' (/dashboard)                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              MIDDLEWARE AuthenticateByRole                   │
│  1. Check: path == 'dashboard' && authenticated?             │
│  2. Check: user->isAdmin()?                                  │
│     - YES: redirect('/admin/dashboard')                      │
│     - NO: continue (stay at /dashboard)                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│         DESTINATION: /dashboard OR /admin/dashboard          │
│                                                               │
│  If /dashboard:                                              │
│  - HandleInertiaRequests middleware inject user              │
│  - Render Dashboard.svelte component                         │
│  - Show customer-specific content                            │
│                                                               │
│  If /admin/dashboard:                                        │
│  - Protected by AdminOnly middleware                         │
│  - HandleInertiaRequests middleware inject user              │
│  - Render admin/Dashboard.svelte component                   │
│  - Show admin-specific content                               │
└─────────────────────────────────────────────────────────────┘
```

### Detailed Middleware Order

Di bootstrap/app.php, middleware execute dalam order ini:

```php
->withMiddleware(function (Middleware $middleware) {
    // 1. Global middleware (semua routes)
    //    - EncryptCookies
    //    - TrimStrings
    //    - ConvertEmptyStringsToNull

    // 2. Web middleware stack
    //    - VerifyCsrfToken
    //    - HandleInertiaRequests
    //    - AuthenticateByRole  ← Custom middleware

    // 3. Named middleware (Fortify auth)
    //    - auth (verify user authenticated)
    //    - verified (verify email verified)
    //    - admin (verify user is admin)
    //    - customer (verify user is customer)
});
```

### How Fortify Handles Redirect

Setelah login, Fortify redirect ke route named `'home'` by default. Anda bisa customize ini di `config/fortify.php`:

```php
'home' => '/dashboard',  // Default redirect setelah login
```

Dengan middleware approach, lebih clean karena:

- Admin otomatis di-redirect ke /admin/dashboard
- Customer stay di /dashboard

---

## Route Structure untuk Multi-Actor

### Complete Routes Setup

Berikut adalah struktur routes lengkap setelah multi-actor implementation:

```
PUBLIC ROUTES (No Auth Required)
├── GET  /                          → Welcome page
├── GET  /products                  → Product catalog
├── GET  /services                  → Services catalog
├── GET  /products/{id}             → Product detail
├── GET  /services/{id}              → Service detail
│
FORTIFY ROUTES (Auto-registered)
├── GET  /login                     → Login form (Login.svelte)
├── POST /login                     → Authenticate user
├── GET  /register                  → Register form (Register.svelte)
├── POST /register                  → Create new user
├── POST /logout                    → Logout
├── GET  /forgot-password           → Reset password form
├── POST /forgot-password           → Send reset email
├── POST /reset-password/{token}    → Confirm password reset
├── GET  /email/verify              → Email verification page
├── POST /email/verify              → Verify email
├── GET  /two-factor-challenge      → 2FA entry form
├── POST /two-factor-challenge      → Verify 2FA code
│
PROTECTED ROUTES (Auth + Verified)
├── GET  /dashboard                 → Customer dashboard (redirect admin to /admin/dashboard)
│
SETTINGS ROUTES (Auth + Verified)
├── GET  /settings/profile          → Edit profile
├── PATCH /settings/profile         → Update profile
├── DELETE /settings/profile        → Delete account
├── GET  /settings/password         → Change password
├── PUT  /settings/password         → Update password
├── GET  /settings/appearance       → Appearance settings
├── GET  /settings/two-factor      → 2FA settings
│
ADMIN ROUTES (Auth + Verified + Admin Role)
├── GET  /admin/dashboard           → Admin dashboard
├── GET  /admin/products            → Manage products
├── GET  /admin/orders              → Manage orders
│
CUSTOMER ROUTES (Auth + Verified + Customer Role)
├── GET  /customer/dashboard        → Customer dashboard
├── GET  /customer/profile          → Edit profile
├── GET  /customer/orders           → Order history
├── GET  /customer/addresses        → Manage addresses
```

### Route File Organization

Untuk keep code organized, Anda bisa split routes ke multiple files:

**File**: `routes/admin.php`

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Admin\DashboardController;

Route::get('/dashboard', [DashboardController::class, 'show'])->name('dashboard');

Route::get('/products', function () {
    return inertia('admin/Products/Index');
})->name('products.index');

Route::get('/orders', function () {
    return inertia('admin/Orders/Index');
})->name('orders.index');
```

**File**: `routes/customer.php`

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Customer\DashboardController;

Route::get('/dashboard', [DashboardController::class, 'show'])->name('dashboard');

Route::get('/profile', function () {
    return inertia('customer/Profile');
})->name('profile');

Route::get('/orders', function () {
    return inertia('customer/Orders');
})->name('orders');

Route::get('/addresses', function () {
    return inertia('customer/Addresses');
})->name('addresses');
```

**Usage di web.php:**

```php
// ADMIN ROUTES
Route::middleware(['auth', 'verified', 'admin'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        require __DIR__ . '/admin.php';
    });

// CUSTOMER ROUTES
Route::middleware(['auth', 'verified', 'customer'])
    ->prefix('customer')
    ->name('customer.')
    ->group(function () {
        require __DIR__ . '/customer.php';
    });
```

---

## Frontend: Conditional Rendering Berdasarkan Role

### Passing User Role ke Frontend

User data sudah di-inject oleh middleware. Setiap Inertia response otomatis include:

```svelte
$page.props.auth.user = {
  id: 1,
  name: "John Doe",
  email: "john@example.com",
  role: "admin",  // ← Ini role yang kita tambahkan
  email_verified_at: "2024-01-01T10:00:00",
  created_at: "2024-01-01T10:00:00",
  updated_at: "2024-01-01T10:00:00"
}
```

### Type Safety dengan TypeScript

Update type definitions untuk include role:

**File**: `resources/js/types/auth.ts`

```typescript
export type User = {
    id: number;
    name: string;
    email: string;
    role: 'admin' | 'customer'; // ← Add this
    avatar?: string;
    email_verified_at: string | null;
    created_at: string;
    updated_at: string;
};

export type Auth = {
    user: User | null;
};
```

### Component Examples: Conditional Rendering

**Example 1: Navigation Bar dengan Role-Specific Links**

```svelte
<!-- resources/js/components/Navigation.svelte -->
<script lang="ts">
    import type { User } from '@/types/auth';
    import { page } from '@inertiajs/svelte';

    $: user = $page.props.auth.user as User | null;
</script>

<nav class="bg-white shadow">
    <div class="container mx-auto px-4 py-4 flex justify-between">
        <div>
            <a href="/" class="font-bold text-xl">My Store</a>
        </div>

        <div class="flex gap-4 items-center">
            {#if !user}
                <a href="/login" class="btn btn-outline">Login</a>
                <a href="/register" class="btn btn-primary">Register</a>
            {:else if user.role === 'admin'}
                <a href="/admin/dashboard" class="nav-link">Dashboard</a>
                <a href="/admin/products" class="nav-link">Products</a>
                <a href="/admin/orders" class="nav-link">Orders</a>
                <form method="POST" action="/logout">
                    <button>Logout</button>
                </form>
            {:else if user.role === 'customer'}
                <a href="/products" class="nav-link">Products</a>
                <a href="/services" class="nav-link">Services</a>
                <a href="/customer/orders" class="nav-link">My Orders</a>
                <a href="/customer/profile" class="nav-link">Profile</a>
                <form method="POST" action="/logout">
                    <button>Logout</button>
                </form>
            {/if}
        </div>
    </div>
</nav>
```

**Example 2: Dashboard Component dengan Role-Specific Content**

```svelte
<!-- resources/js/pages/Dashboard.svelte -->
<script lang="ts">
    import { page } from '@inertiajs/svelte';
    import type { User } from '@/types/auth';

    $: user = $page.props.auth.user as User;
    $: isAdmin = user.role === 'admin';
    $: isCustomer = user.role === 'customer';
</script>

<div>
    <h1>Welcome, {user.name}!</h1>

    {#if isAdmin}
        <div class="admin-dashboard">
            <h2>Admin Dashboard</h2>

            <div class="stats-grid">
                <div class="stat-card">
                    <h3>Total Orders</h3>
                    <p class="text-2xl font-bold">
                        {$page.props.stats?.total_orders || 0}
                    </p>
                </div>
                <div class="stat-card">
                    <h3>Total Revenue</h3>
                    <p class="text-2xl font-bold">
                        ${$page.props.stats?.total_revenue || 0}
                    </p>
                </div>
                <div class="stat-card">
                    <h3>Total Products</h3>
                    <p class="text-2xl font-bold">
                        {$page.props.stats?.total_products || 0}
                    </p>
                </div>
            </div>

            <div class="admin-actions mt-8">
                <a href="/admin/products" class="btn btn-primary"
                    >Manage Products</a
                >
                <a href="/admin/orders" class="btn btn-primary">View Orders</a>
            </div>
        </div>
    {:else if isCustomer}
        <div class="customer-dashboard">
            <h2>Customer Dashboard</h2>

            <div class="customer-sections">
                <section>
                    <h3>My Orders</h3>
                    <a href="/customer/orders" class="link">View all orders →</a
                    >
                </section>

                <section>
                    <h3>Quick Links</h3>
                    <ul>
                        <li><a href="/products">Browse Products</a></li>
                        <li><a href="/services">View Services</a></li>
                        <li>
                            <a href="/customer/addresses">Manage Addresses</a>
                        </li>
                    </ul>
                </section>
            </div>
        </div>
    {/if}
</div>

<style>
    .stats-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
        gap: 2rem;
        margin: 2rem 0;
    }

    .stat-card {
        border: 1px solid #ddd;
        padding: 1.5rem;
        border-radius: 8px;
        background: #f9f9f9;
    }
</style>
```

**Example 3: Protected Component untuk Admin-Only Features**

```svelte
<!-- resources/js/components/AdminPanel.svelte -->
<script lang="ts">
    import { page } from '@inertiajs/svelte';
    import type { User } from '@/types/auth';

    $: user = $page.props.auth.user as User | null;

    function canViewAdminPanel() {
        return user?.role === 'admin';
    }
</script>

{#if canViewAdminPanel()}
    <div class="admin-panel">
        <h2>Admin Controls</h2>
        <p>This section is only visible to admins</p>

        <button on:click={handleAdminAction}> Perform Admin Action </button>
    </div>
{:else}
    <p class="text-red-500">Access Denied. Admin access required.</p>
{/if}
```

### Helper Utility Functions

Create helper functions untuk check roles:

**File**: `resources/js/lib/auth.ts`

```typescript
import type { User } from '@/types/auth';

export function isAdmin(user: User | null): boolean {
    return user?.role === 'admin' ?? false;
}

export function isCustomer(user: User | null): boolean {
    return user?.role === 'customer' ?? false;
}

export function hasRole(user: User | null, role: string): boolean {
    return user?.role === role ?? false;
}

export function can(user: User | null, action: string): boolean {
    if (isAdmin(user)) return true;

    if (action === 'view_products' && isCustomer(user)) return true;
    if (action === 'create_order' && isCustomer(user)) return true;

    return false;
}
```

**Use di components:**

```svelte
<script>
    import { isAdmin, can } from '@/lib/auth';
    import { page } from '@inertiajs/svelte';

    $: user = $page.props.auth.user;
    $: adminMode = isAdmin(user);
    $: canCreateOrder = can(user, 'create_order');
</script>
```

---

## Testing Multi-Actor Authentication

### Test Setup

Untuk test multi-actor auth, kita perlu create test data dengan berbagai roles.

**Update User Factory:**

**File**: `database/factories/UserFactory.php`

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class UserFactory extends Factory
{
    protected static ?string $password;

    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'role' => 'customer',  // Default to customer
            'email_verified_at' => now(),
            'password' => static::$password ??= Hash::make('password'),
            'remember_token' => Str::random(10),
        ];
    }

    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }

    // NEW: Admin user factory state
    public function admin(): static
    {
        return $this->state(fn (array $attributes) => [
            'role' => 'admin',
        ]);
    }

    // NEW: Customer user factory state
    public function customer(): static
    {
        return $this->state(fn (array $attributes) => [
            'role' => 'customer',
        ]);
    }

    public function withTwoFactor(): static
    {
        return $this->state(fn (array $attributes) => [
            'two_factor_secret' => encrypt('test-secret'),
            'two_factor_confirmed_at' => now(),
            'two_factor_recovery_codes' => encrypt(json_encode([
                'code1', 'code2', 'code3',
            ])),
        ]);
    }
}
```

### Feature Tests untuk Multi-Actor

**Test Admin Login & Redirect:**

**File**: `tests/Feature/Auth/AdminAuthenticationTest.php`

```php
<?php

namespace Tests\Feature\Auth;

use App\Models\User;
use Tests\TestCase;

class AdminAuthenticationTest extends TestCase
{
    public function test_admin_can_login(): void
    {
        $admin = User::factory()->admin()->create([
            'email' => 'admin@example.com',
            'password' => bcrypt('password'),
        ]);

        $response = $this->post('/login', [
            'email' => 'admin@example.com',
            'password' => 'password',
        ]);

        // Admin should be authenticated
        $this->assertAuthenticated();

        // Admin should be redirected to admin dashboard
        $response->assertRedirect('/admin/dashboard');
    }

    public function test_admin_can_access_admin_dashboard(): void
    {
        $admin = User::factory()->admin()->create();

        $response = $this->actingAs($admin)->get('/admin/dashboard');

        $response->assertStatus(200);
        $response->assertInertia(fn ($page) => $page
            ->component('admin/Dashboard')
        );
    }

    public function test_customer_cannot_access_admin_dashboard(): void
    {
        $customer = User::factory()->customer()->create();

        $response = $this->actingAs($customer)->get('/admin/dashboard');

        // Should be forbidden (403)
        $response->assertStatus(403);
    }

    public function test_unauthenticated_user_cannot_access_admin_dashboard(): void
    {
        $response = $this->get('/admin/dashboard');

        // Should redirect to login
        $response->assertRedirect('/login');
    }
}
```

**Test Customer Login & Access:**

**File**: `tests/Feature/Auth/CustomerAuthenticationTest.php`

```php
<?php

namespace Tests\Feature\Auth;

use App\Models\User;
use Tests\TestCase;

class CustomerAuthenticationTest extends TestCase
{
    public function test_customer_can_login(): void
    {
        $customer = User::factory()->customer()->create([
            'email' => 'customer@example.com',
            'password' => bcrypt('password'),
        ]);

        $response = $this->post('/login', [
            'email' => 'customer@example.com',
            'password' => 'password',
        ]);

        $this->assertAuthenticated();

        // Customer should stay on dashboard (not redirect)
        $response->assertRedirect('/dashboard');
    }

    public function test_customer_can_access_customer_routes(): void
    {
        $customer = User::factory()->customer()->create();

        $response = $this->actingAs($customer)->get('/customer/orders');

        $response->assertStatus(200);
    }

    public function test_admin_cannot_access_customer_routes(): void
    {
        $admin = User::factory()->admin()->create();

        $response = $this->actingAs($admin)->get('/customer/orders');

        $response->assertStatus(403);
    }

    public function test_customer_can_browse_public_catalog(): void
    {
        // Public routes should accessible tanpa login
        $response = $this->get('/products');

        $response->assertStatus(200);
        $response->assertInertia(fn ($page) => $page
            ->component('Products/Index')
        );
    }
}
```

**Test Middleware Redirect:**

**File**: `tests/Feature/Auth/AuthenticateByRoleTest.php`

```php
<?php

namespace Tests\Feature\Auth;

use App\Models\User;
use Tests\TestCase;

class AuthenticateByRoleTest extends TestCase
{
    public function test_admin_redirected_from_dashboard_to_admin_dashboard(): void
    {
        $admin = User::factory()->admin()->create();

        $response = $this->actingAs($admin)->get('/dashboard');

        $response->assertRedirect('/admin/dashboard');
    }

    public function test_customer_not_redirected_from_dashboard(): void
    {
        $customer = User::factory()->customer()->create();

        $response = $this->actingAs($customer)->get('/dashboard');

        $response->assertStatus(200);
        $response->assertInertia(fn ($page) => $page
            ->component('Dashboard')
        );
    }
}
```

### Run Tests

```bash
# Run all auth tests
sail artisan test --filter=Auth --compact

# Run specific test
sail artisan test tests/Feature/Auth/AdminAuthenticationTest.php --compact

# Run tests with coverage
sail artisan test --filter=Auth --coverage
```

---

## Best Practices & Security

### Security Considerations

#### 1. Role Enforcement di Backend

Selalu check role di backend, jangan hanya di frontend:

```php
// ✅ GOOD: Backend enforce
public function update(Request $request, Post $post)
{
    // User harus admin
    if (!$request->user()->isAdmin()) {
        abort(403);
    }

    // Update logic
}

// ❌ BAD: Hanya frontend check
// Frontend bisa di-bypass dengan network tools
```

#### 2. Middleware Protection

Selalu gunakan middleware untuk protect routes:

```php
// ✅ GOOD
Route::middleware(['auth', 'verified', 'admin'])->group(function () {
    Route::get('/admin/products', [ProductController::class, 'index']);
});

// ❌ BAD
Route::get('/admin/products', [ProductController::class, 'index'])
    ->middleware('auth'); // Missing 'verified' dan 'admin'
```

#### 3. Authorization di Controller

```php
// ✅ GOOD: Explicit authorization
public function delete(Request $request, User $user)
{
    if (!$request->user()->isAdmin()) {
        abort(403, 'Only admins can delete users');
    }

    $user->delete();
}

// Alternative: gunakan Laravel Gates & Policies
public function delete(Request $request, User $user)
{
    $this->authorize('delete-user', $user);
    $user->delete();
}
```

#### 4. Role-Based Logging

Log important actions dengan role info:

```php
\Log::info('Order created', [
    'order_id' => $order->id,
    'user_id' => auth()->id(),
    'user_role' => auth()->user()->role,  // Log the role
]);
```

### Role Management Best Practices

#### 1. Consistent Role Values

Gunakan enum atau constants untuk role values:

```php
// ✅ GOOD: Constants
class UserRoles {
    const ADMIN = 'admin';
    const CUSTOMER = 'customer';
}

// Use: $user->role === UserRoles::ADMIN

// ❌ BAD: Magic strings
if ($user->role === 'admin') { }  // Easy typo!
```

#### 2. Query Optimization

Jika ada banyak role checks dalam loop:

```php
// ❌ BAD: N+1 queries
foreach ($users as $user) {
    if ($user->isAdmin()) { }  // Query per user!
}

// ✅ GOOD: Eager load atau use role column directly
$admins = User::where('role', 'admin')->get();
```

---

## Troubleshooting

### Common Issues & Solutions

#### Issue 1: User redirected ke /admin/dashboard tapi dapat 403 Forbidden

**Symptom:**

```
User role='admin', redirect to /admin/dashboard, tapi dapat 403 Forbidden
```

**Cause:**
Middleware exec order salah atau user.role tidak tersimpan correctly.

**Solution:**

```bash
# 1. Check user.role di database
sail mysql
SELECT id, email, role FROM users;

# 2. Verify user object di tinker
sail artisan tinker
User::find(1)->role;  // Seharusnya 'admin'
User::find(1)->isAdmin();  // Seharusnya true
```

#### Issue 2: Middleware AuthenticateByRole tidak working

**Symptom:**
Admin login tapi tidak di-redirect ke /admin/dashboard.

**Cause:**
Middleware tidak register atau tidak exec.

**Solution:**

1. Check bootstrap/app.php:

```php
// Make sure AuthenticateByRole registered:
$middleware->web(append: [
    AuthenticateByRole::class,
]);
```

2. Check middleware exec order:

```bash
sail artisan route:list --middleware
# Seharusnya AuthenticateByRole ada di middleware list
```

#### Issue 3: Role column not existing

**Symptom:**

```
SQLSTATE[42S22]: Column not found
```

**Solution:**

```bash
# Pastikan migration dijalankan
sail artisan migrate

# Check tabel structure
sail mysql
DESCRIBE users;  # Seharusnya ada 'role' column
```

#### Issue 4: Frontend tidak terlihat user.role

**Symptom:**
$page.props.auth.user ada, tapi user.role undefined di Svelte.

**Cause:**
Migration belum run atau user tidak punya role.

**Solution:**

1. Check HandleInertiaRequests middleware:

```php
// app/Http/Middleware/HandleInertiaRequests.php
return [
    'auth' => [
        'user' => $request->user(),  // Should include role
    ],
];

// Verify user include role:
$user = $request->user();
dd($user->role);  // Should output 'admin' or 'customer'
```

2. Run migration:

```bash
sail artisan migrate
```

#### Issue 5: adminOnly middleware not working

**Symptom:**
Customer bisa akses /admin/dashboard (seharusnya forbidden).

**Cause:**
AdminOnly middleware tidak register dengan baik.

**Solution:**

1. Check bootstrap/app.php:

```php
$middleware->alias([
    'admin' => AdminOnly::class,
    'customer' => CustomerOnly::class,
]);
```

2. Check route:

```php
// Middleware harus di-specify:
Route::middleware(['auth', 'verified', 'admin'])->group(function () {
    Route::get('/admin/dashboard', ...);
});
```

### Debug Techniques

#### Debug 1: Check Middleware Execution

```php
// In your middleware, add logging:
public function handle(Request $request, Closure $next): Response
{
    \Log::info('AuthenticateByRole executing', [
        'user' => $request->user()?->id,
        'user_role' => $request->user()?->role,
        'path' => $request->path(),
    ]);

    return $next($request);
}

// Check logs
sail logs -f
```

#### Debug 2: Test Role Methods

```bash
sail artisan tinker

$user = User::find(1);
$user->role;           # Check role value
$user->isAdmin();      # Check method
$user->isCustomer();   # Check method

dd($user->toArray());  # See all user data
```

#### Debug 3: Check Middleware Registration

```bash
sail artisan middleware:list
# Seharusnya AuthenticateByRole, AdminOnly, CustomerOnly di sini
```

---

## Kesimpulan & Next Steps

Selamat! Anda sudah memahami cara implement multi-actor authentication dalam Laravel Anda!

### Apa yang sudah kita buat:

✅ Menambahkan kolom `role` ke tabel users  
✅ Membuat middleware redirect berdasarkan role  
✅ Membuat controllers untuk admin & customer dashboards  
✅ Membuat routes terpisah untuk /admin/_ dan /customer/_  
✅ Implementasi conditional rendering di frontend  
✅ Membuat tests untuk multi-actor flows

### Langkah berikutnya:

1. **Implementasi seluruh Admin Area**
    - Product management (CRUD)
    - Order management
    - Analytics dashboard

2. **Implementasi Customer Area**
    - Order history & tracking
    - Address management
    - Order checkout flow

3. **Enhanced Authorization**
    - Laravel Gates & Policies untuk granular permissions

4. **Frontend Improvements**
    - Responsive admin dashboard
    - Customer-friendly product pages

### File Summary

```
NEW/MODIFIED FILES:
├── database/migrations/
│   └── [timestamp]_add_role_to_users_table.php  (NEW)
├── app/Models/
│   └── User.php  (MODIFIED - add role column & methods)
├── app/Http/Middleware/
│   ├── AuthenticateByRole.php  (NEW)
│   ├── AdminOnly.php  (NEW)
│   └── CustomerOnly.php  (NEW)
├── app/Http/Controllers/
│   ├── Admin/DashboardController.php  (NEW)
│   └── Customer/DashboardController.php  (NEW)
├── routes/
│   ├── web.php  (MODIFIED)
│   ├── admin.php  (NEW)
│   └── customer.php  (NEW)
├── bootstrap/
│   └── app.php  (MODIFIED - register middleware)
├── database/factories/
│   └── UserFactory.php  (MODIFIED - add admin/customer states)
├── resources/js/
│   ├── types/auth.ts  (MODIFIED - add role to User type)
│   └── lib/auth.ts  (NEW - helper functions)
└── tests/Feature/Auth/
    ├── AdminAuthenticationTest.php  (NEW)
    ├── CustomerAuthenticationTest.php  (NEW)
    └── AuthenticateByRoleTest.php  (NEW)
```

### Quick Reference Commands

```bash
# Create migration
sail artisan make:migration add_role_to_users_table

# Create middleware
sail artisan make:middleware AuthenticateByRole
sail artisan make:middleware AdminOnly
sail artisan make:middleware CustomerOnly

# Create controllers
sail artisan make:controller Admin/DashboardController
sail artisan make:controller Customer/DashboardController

# Run migration
sail artisan migrate

# Create test users
sail artisan tinker
User::factory()->admin()->create(['email' => 'admin@test.com', 'password' => bcrypt('password')]);
User::factory()->customer()->create(['email' => 'customer@test.com', 'password' => bcrypt('password')]);

# Run tests
sail artisan test --filter=Auth --compact
```

---

**Terakhir diupdate:** 2026-03-09  
**Untuk:** Laravel 12 dengan Fortify v1 & Svelte  
**Database:** SQLite / MySQL  
**PHP:** 8.5.3

---

**Happy coding!** 🚀 Jika ada pertanyaan, refer kembali ke dokumentasi ini atau cek section Troubleshooting.
