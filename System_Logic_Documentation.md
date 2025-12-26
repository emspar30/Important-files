# 🧠 ORG MERCH - Complete System Logic Documentation

> **System**: University Organization Merchandise E-Commerce Platform  
> **Last Updated**: December 26, 2024  
> **Purpose**: Deep-dive documentation of all system logic with line-by-line explanations

---

## 📋 Table of Contents

1. [System Overview & User Flows](#1-system-overview--user-flows)
2. [Authentication System](#2-authentication-system)
3. [User Registration & Onboarding](#3-user-registration--onboarding)
4. [Membership System (Auto-Detection & Approval)](#4-membership-system-auto-detection--approval)
5. [Product & Variant System](#5-product--variant-system)
6. [Cart System Logic](#6-cart-system-logic)
7. [Checkout System (The Core)](#7-checkout-system-the-core)
8. [Order Lifecycle & Status Management](#8-order-lifecycle--status-management)
9. [Refund System](#9-refund-system)
10. [Admin Panel Logic](#10-admin-panel-logic)
11. [Stock Management Deep Dive](#11-stock-management-deep-dive)
12. [Pricing & Discount Engine](#12-pricing--discount-engine)

---

## 1. System Overview & User Flows

### 1.1 What is ORG MERCH?

ORG MERCH is a complete e-commerce platform designed for **university organizations** (like student clubs, societies) to sell merchandise to students. Think of it as "Shopee for University Orgs".

### 1.2 The Two User Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                      TWO DISTINCT USER TYPES                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────┐          ┌─────────────────────┐           │
│  │      STUDENT        │          │     ORG ADMIN       │           │
│  │    (Customer)       │          │   (Seller/Staff)    │           │
│  ├─────────────────────┤          ├─────────────────────┤           │
│  │ • Browse Products   │          │ • Manage Products   │           │
│  │ • Add to Cart       │          │ • Process Orders    │           │
│  │ • Place Orders      │          │ • Approve Members   │           │
│  │ • Request Refunds   │          │ • Handle Refunds    │           │
│  │ • Join Orgs         │          │ • View Analytics    │           │
│  └─────────────────────┘          └─────────────────────┘           │
│           │                                  │                       │
│           │         COMMUNICATION            │                       │
│           └─────────── Messages ─────────────┘                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 Complete Student Journey

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE STUDENT JOURNEY                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. ENTRY                                                                │
│     ├── Register (Email/Password) ──────────────┐                       │
│     │                                           ▼                       │
│     └── Google OAuth ─────────────────────▶ ONBOARDING                  │
│                                             (if new user)               │
│                                                 │                       │
│  2. ORGANIZATION MEMBERSHIP                     ▼                       │
│     User selects orgs ──▶ Status: PENDING ──▶ Admin Reviews             │
│                                                 │                       │
│                              ┌──────────────────┴──────────────────┐    │
│                              ▼                                     ▼    │
│                         APPROVED                              REJECTED  │
│                         (10% discount!)                     (no discount)│
│                              │                                          │
│  3. SHOPPING                 ▼                                          │
│     Browse ──▶ View Product ──▶ Select Variant ──▶ Add to Cart        │
│                                                          │              │
│  4. CHECKOUT                                             ▼              │
│     ├── Full Payment (Pay 100% now)                    CART             │
│     ├── Partial Payment (Pay 50%+ now, rest later)      │              │
│     ├── COD (Cash on Delivery/Pickup)                   ▼              │
│     └── Reservation (₱20/item to hold)             CHECKOUT             │
│                                                          │              │
│  5. ORDER LIFECYCLE                                      ▼              │
│     PENDING ──▶ PROCESSING ──▶ READY ──▶ COMPLETED                     │
│        │                         │                                      │
│        └── CANCEL (12hr window)  └── REQUEST REFUND                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Authentication System

### 2.1 Student Login Logic

**File**: `app/Http/Controllers/Api/AuthController.php` (Lines 77-103)

```php
public function login(Request $request)
{
    $data = $request->validate([
        'email' => 'required_without:username|email',
        'username' => 'required_without:email|string',
        'password' => 'required|string',
    ]);

    $user = null;
    if (!empty($data['email'])) {
        $user = User::where('email', $data['email'])->first();
    } else {
        $user = User::where('username', $data['username'])->first();
    }

    if (!$user || !Hash::check($data['password'], $user->password)) {
        return response()->json(['message' => 'invalid_credentials'], 401);
    }

    // Revoke all existing tokens for this user (single session only)
    $user->tokens()->delete();

    // Create new Sanctum token
    $token = $user->createToken('api_token')->plainTextToken;

    return response()->json(['token' => $token, 'user' => $user]);
}
```

**🔍 How This Works:**

| Line                                                  | What It Does                                                                              |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `'email' => 'required_without:username'`              | User can login with EITHER email OR username - not both required                          |
| `User::where('email', $data['email'])->first()`       | First tries to find user by email                                                         |
| `User::where('username', $data['username'])->first()` | If no email provided, tries username lookup                                               |
| `Hash::check($data['password'], $user->password)`     | Securely compares password using bcrypt hashing                                           |
| `$user->tokens()->delete()`                           | **Single Session Policy**: Revokes ALL existing tokens, logging user out of other devices |
| `$user->createToken('api_token')->plainTextToken`     | Creates new Laravel Sanctum bearer token for API authentication                           |

### 2.2 Admin Login (Separate Authentication)

**File**: `app/Http/Controllers/Api/AuthController.php` (Lines 193-223)

```php
public function adminLogin(Request $request)
{
    $data = $request->validate([
        'admin_username' => 'required|string',
        'admin_password' => 'required|string',
    ]);

    $admin = OrgAdmin::where('admin_username', $data['admin_username'])->first();

    if (!$admin) {
        return response()->json(['message' => 'invalid_credentials'], 401);
    }

    // Check admin_password or creator_password
    $passwordValid = ($admin->admin_password && Hash::check($data['admin_password'], $admin->admin_password))
        || ($admin->creator_password && Hash::check($data['admin_password'], $admin->creator_password));

    if (!$passwordValid) {
        return response()->json(['message' => 'invalid_credentials'], 401);
    }

    // Revoke all existing tokens for this admin
    $admin->tokens()->delete();

    // Create new Sanctum token for admin
    $token = $admin->createToken('admin_token')->plainTextToken;

    return response()->json([
        'token' => $token,
        'admin' => $admin->load('organization:id,name,slug')
    ]);
}
```

**🔍 How This Works:**

| Line                                                    | What It Does                                                                                 |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `OrgAdmin::where('admin_username', ...)`                | Admin accounts are in a **separate table** (`org_admins`), completely isolated from students |
| `$admin->admin_password` and `$admin->creator_password` | Admins can have TWO passwords - primary and creator (backup)                                 |
| `$admin->load('organization:id,name,slug')`             | Return includes org info - admin is ALWAYS tied to ONE organization                          |

**🔑 Key Design Decision**: Students and Admins use the **same Sanctum token system** but **different tables**. The `AdminMiddleware` checks if the authenticated user is an `OrgAdmin` instance.

---

## 3. User Registration & Onboarding

### 3.1 Regular Registration

**File**: `app/Http/Controllers/Api/AuthController.php` (Lines 15-75)

```php
public function register(Request $request)
{
    $data = $request->validate([
        'student_id' => ['required','regex:/^\d{3}-\d{5}$/','unique:users,student_id'],
        'first_name' => 'required|string|max:255',
        'last_name' => 'required|string|max:255',
        'username' => 'required|string|max:50|unique:users,username',
        'email' => 'required|email|unique:users,email',
        'password' => 'required|string|min:6',
        'organization_ids' => 'nullable|array',
        'organization_ids.*' => 'exists:organizations,id',
    ]);

    $user = User::create([
        'student_id' => $data['student_id'],
        'first_name' => $data['first_name'],
        'last_name' => $data['last_name'],
        'username' => $data['username'],
        'email' => $data['email'],
        'password' => Hash::make($data['password']),
    ]);

    // Attach selected organizations to user with 'pending' status
    if (!empty($data['organization_ids'])) {
        $attachData = [];
        foreach ($data['organization_ids'] as $orgId) {
            $attachData[$orgId] = ['status' => 'pending'];
        }
        $user->organizations()->attach($attachData);

        // Create notification for each organization's admins
        foreach ($data['organization_ids'] as $orgId) {
            $org = Organization::find($orgId);
            if ($org) {
                Notification::create([
                    'organization_id' => $orgId,
                    'user_id' => $user->id,
                    'title' => 'New Membership Request',
                    'body' => "{$user->first_name} {$user->last_name} ({$user->student_id}) has requested to join {$org->name}.",
                    'data' => json_encode([
                        'type' => 'membership_request',
                        'user_id' => $user->id,
                        'organization_id' => $orgId,
                    ]),
                ]);
            }
        }
    }

    // Don't auto-login on registration
    return response()->json([
        'message' => 'Registration successful. Please login to get your access token.',
        'user' => $user
    ], 201);
}
```

**🔍 Line-by-Line Explanation:**

| Line                                                | What It Does                                                           |
| --------------------------------------------------- | ---------------------------------------------------------------------- |
| `'regex:/^\d{3}-\d{5}$/'`                           | Student ID MUST match format `XXX-XXXXX` (e.g., `123-45678`)           |
| `'organization_ids.*' => 'exists:organizations,id'` | Validates each org ID actually exists in database                      |
| `$attachData[$orgId] = ['status' => 'pending']`     | Creates pivot table entry with **pending** status (not auto-approved!) |
| `$user->organizations()->attach($attachData)`       | Laravel's `attach()` inserts into `organization_user` pivot table      |
| `Notification::create(...)`                         | Immediately notifies org admins of the membership request              |
| **No token returned**                               | User must explicitly login after registration (security best practice) |

### 3.2 OAuth Onboarding (Google Login New Users)

**File**: `app/Http/Controllers/Api/OnboardingController.php` (Lines 42-126)

When a user signs up via Google for the first time, they have incomplete profile data. Onboarding collects the missing information:

```php
public function complete(Request $request)
{
    $user = Auth::user();

    $validated = $request->validate([
        'username' => [
            'required',
            'string',
            'min:3',
            'max:50',
            'regex:/^[a-zA-Z0-9_]+$/',
            Rule::unique('users', 'username')->ignore($user->id),
        ],
        'first_name' => 'required|string|max:100',
        'last_name' => 'required|string|max:100',
        'student_id' => [
            'required',
            'string',
            'max:50',
            Rule::unique('users', 'student_id')->ignore($user->id),
        ],
        'course' => 'required|string|max:100',
        'year_level' => 'required|string|max:20',
        'organization_ids' => 'nullable|array',
    ]);

    // Update user profile
    $user->update([
        'username' => $validated['username'],
        'first_name' => $validated['first_name'],
        'last_name' => $validated['last_name'],
        'student_id' => $validated['student_id'],
        'course' => $validated['course'],
        'year_level' => $validated['year_level'],
        'is_onboarded' => true,  // Mark as complete!
    ]);

    // Add organization membership requests
    if (!empty($validated['organization_ids'])) {
        foreach ($validated['organization_ids'] as $orgId) {
            $exists = DB::table('organization_user')
                ->where('user_id', $user->id)
                ->where('organization_id', $orgId)
                ->exists();

            if (!$exists) {
                DB::table('organization_user')->insert([
                    'user_id' => $user->id,
                    'organization_id' => $orgId,
                    'status' => 'pending',
                ]);
                // ... notification creation
            }
        }
    }
}
```

**🔍 Key Points:**

| Concept                                | Explanation                                                          |
| -------------------------------------- | -------------------------------------------------------------------- |
| `is_onboarded` flag                    | Frontend checks this flag - if `false`, redirects to onboarding page |
| `Rule::unique(...)->ignore($user->id)` | Allows user to keep their own existing value during update           |
| Membership check `$exists`             | Prevents duplicate membership requests                               |

---

## 4. Membership System (Auto-Detection & Approval)

### 4.1 The Membership Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MEMBERSHIP LIFECYCLE                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   REGISTRATION/ONBOARDING                                             │
│           │                                                           │
│           ▼                                                           │
│   ┌───────────────┐                                                   │
│   │    PENDING    │ ◄──── User selects org during registration        │
│   └───────┬───────┘                                                   │
│           │                                                           │
│           │ Admin reviews (sees notification)                         │
│           │                                                           │
│    ┌──────┴──────┐                                                    │
│    ▼             ▼                                                    │
│ ┌──────┐    ┌──────────┐                                              │
│ │APPROVE│    │ REJECT  │                                              │
│ └───┬──┘    └────┬─────┘                                              │
│     │            │                                                    │
│     ▼            ▼                                                    │
│ ┌────────────────────────────────────────────┐                        │
│ │ Status updated in organization_user table  │                        │
│ │ User notified automatically                │                        │
│ └────────────────────────────────────────────┘                        │
│     │                                                                 │
│     ▼                                                                 │
│ ┌────────────────────────────────────────────┐                        │
│ │   APPROVED MEMBERS GET 10% DISCOUNT        │                        │
│ │   on that organization's products!         │                        │
│ └────────────────────────────────────────────┘                        │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Admin Approving Membership

**File**: `app/Http/Controllers/Api/AdminController.php` (Lines 437-466)

```php
public function approveMembership(Request $request, $userId)
{
    $admin = $request->attributes->get('org_admin');
    $user = User::findOrFail($userId);

    // Update the pivot table
    $user->organizations()->updateExistingPivot($admin->organization_id, [
        'status' => 'approved',
        'reviewed_at' => now(),
        'reviewed_by' => $admin->id,
    ]);

    // Create notification for the user
    Notification::create([
        'organization_id' => $admin->organization_id,
        'user_id' => $userId,
        'title' => 'Membership Approved! 🎉',
        'body' => "Your membership request for {$admin->organization->name} has been approved. You can now enjoy member-exclusive discounts!",
        'data' => json_encode([
            'type' => 'membership_approved',
            'organization_id' => $admin->organization_id,
        ]),
    ]);
}
```

**🔍 How This Works:**

| Line                                     | What It Does                                                                  |
| ---------------------------------------- | ----------------------------------------------------------------------------- |
| `$request->attributes->get('org_admin')` | Admin was set by `AdminMiddleware` after verifying the user is an OrgAdmin    |
| `updateExistingPivot(...)`               | Updates the `organization_user` pivot table row for this user+org combination |
| `'reviewed_by' => $admin->id`            | Records which admin approved it (audit trail)                                 |
| Notification created                     | User sees "Membership Approved!" in their notification center                 |

### 4.3 Auto-Detecting Membership for Discounts

**File**: `app/Http/Controllers/Api/CartController.php` (Lines 115-131)

```php
// Get user's organization memberships for discount calculation
$userOrgIds = $user->organizations()->pluck('organizations.id')->toArray();

// Add member info to each item
$cart->items->each(function($item) use ($userOrgIds) {
    $orgId = $item->merchandise->organization_id ?? null;
    $item->is_member = $orgId && in_array($orgId, $userOrgIds);
});
```

**🔍 How This Works:**

| Step                                     | What Happens                                       |
| ---------------------------------------- | -------------------------------------------------- |
| 1. `$user->organizations()->pluck(...)`  | Gets all org IDs the user belongs to (any status!) |
| 2. `$item->merchandise->organization_id` | Gets which org sells this product                  |
| 3. `in_array($orgId, $userOrgIds)`       | Checks if user is member of that org               |
| 4. `$item->is_member = true/false`       | Frontend uses this to show "Member: 10% off" badge |

**⚠️ Note**: This checks membership existence but NOT status. The checkout controller specifically checks for `approved` status before applying the discount.

---

## 5. Product & Variant System

### 5.1 Product (Merchandise) Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MERCHANDISE DATA MODEL                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  merchandises table                                                  │
│  ├── id, organization_id, name, description                         │
│  ├── price: 599.00                                                   │
│  ├── discount: 20 (means 20% off)                                    │
│  ├── stock: 100 (TOTAL stock across all variants)                    │
│  ├── total_sales: 45                                                 │
│  ├── accept_preorder: true/false                                     │
│  └── expected_date: 2024-02-15 (for preorders)                       │
│                                                                      │
│  merchandise_variants table (one-to-many)                            │
│  ├── variant_type: "Size" or "Color"                                 │
│  ├── variant_options: ["S", "M", "L", "XL"]                          │
│  └── variant_stock: {"Red - S": 10, "Red - M": 15, ...}             │
│                                                                      │
│  merchandise_images table (one-to-many)                              │
│  ├── image_path, is_primary, sort_order                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Preorder Detection

**File**: `app/Http/Controllers/Api/MerchandiseController.php` (Lines 100-103)

```php
// Filter for preorder items
if ($request->has('accept_preorder') && $request->accept_preorder === 'true') {
    $query->where('accept_preorder', true);
}
```

**🔍 Preorder System Explained:**

| Field             | Purpose                                                             |
| ----------------- | ------------------------------------------------------------------- |
| `accept_preorder` | Boolean flag - if `true`, items can be preordered even if stock=0   |
| `expected_date`   | When the product is expected to arrive                              |
| Preorder checkout | User gets 10% discount, no stock deduction, marked as `is_preorder` |

### 5.3 On-Sale Detection (Discounted Products)

**File**: `app/Http/Controllers/Api/MerchandiseController.php` (Lines 95-98)

```php
// Filter for on-sale items (discount > 0)
if ($request->has('on_sale') && $request->on_sale === 'true') {
    $query->where('discount', '>', 0);
}
```

**🔍 How "On Sale" Works:**

1. Admin sets `discount` field (e.g., 20 = 20% off)
2. Frontend shows original price crossed out + discounted price
3. At checkout, discount is automatically calculated from this field

---

## 6. Cart System Logic

### 6.1 Adding to Cart with Stock Validation

**File**: `app/Http/Controllers/Api/CartController.php` (Lines 137-294)

```php
public function add(Request $request)
{
    $data = $request->validate([
        'merchandise_id' => 'required|exists:merchandises,id',
        'quantity' => 'required|integer|min:1',
        'variant' => 'nullable|string',
        'variant_options' => 'nullable|array',
    ]);

    $user = $request->user();
    $merch = Merchandise::findOrFail($data['merchandise_id']);

    // Get available stock (check variant stock if applicable)
    $availableStock = $merch->stock;

    // ... variant stock checking logic ...

    // Check existing quantity in cart for same item/variant
    $existingCart = OrderCart::where('user_id', $user->id)->where('type', 'cart')->first();
    $existingQty = 0;
    if ($existingCart) {
        $existingItem = OrderCartItem::where('orders_and_cart_id', $existingCart->id)
            ->where('merchandise_id', $merch->id)
            ->when($variantJson, fn($q) => $q->where('variant', $variantJson))
            ->when(!$variantJson, fn($q) => $q->whereNull('variant'))
            ->first();
        if ($existingItem) {
            $existingQty = $existingItem->quantity;
        }
    }

    // Validate total quantity against stock
    $totalRequestedQty = $existingQty + $data['quantity'];
    if ($totalRequestedQty > $availableStock) {
        $canAdd = max(0, $availableStock - $existingQty);
        return response()->json([
            'message' => $canAdd > 0
                ? "Only {$canAdd} more item(s) available. You already have {$existingQty} in your cart."
                : "Sorry, only {$availableStock} item(s) available in stock.",
            'available_stock' => $availableStock,
            'in_cart' => $existingQty,
            'can_add' => $canAdd
        ], 422);
    }

    // create or get cart
    $cart = OrderCart::firstOrCreate(
        ['user_id' => $user->id, 'type' => 'cart'],
        ['status' => 'pending', 'total_amount' => 0]
    );

    // add or update item
    $item = OrderCartItem::where('orders_and_cart_id', $cart->id)
        ->where('merchandise_id', $merch->id)
        ->when($variantJson, fn($q) => $q->where('variant', $variantJson))
        ->first();

    if ($item) {
        $item->quantity += $data['quantity'];
        $item->save();
    } else {
        OrderCartItem::create([
            'orders_and_cart_id' => $cart->id,
            'merchandise_id' => $merch->id,
            'quantity' => $data['quantity'],
            'price' => $merch->price,
            'variant' => $variantJson,
        ]);
    }
}
```

**🔍 Critical Logic Explained:**

| Step                                   | What Happens                                              | Why It Matters                                                                       |
| -------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Check existing cart quantity           | `$existingQty = $existingItem->quantity`                  | Prevents user from adding 10 items when they already have 5 in cart and only 8 exist |
| `$totalRequestedQty > $availableStock` | Calculates: existing in cart + new request vs available   | `cart(5) + add(5) = 10 > stock(8)` = ERROR                                           |
| `firstOrCreate` for cart               | Creates cart if doesn't exist                             | Users get ONE active cart at a time                                                  |
| Variant matching                       | Same product but different variant = different cart items | "T-Shirt Red S" and "T-Shirt Blue M" are separate lines                              |

---

## 7. Checkout System (The Core)

### 7.1 The Three Checkout Modes

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THREE CHECKOUT ENDPOINTS                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. checkout()  ─────────────────────────────────────────────────   │
│     • Converts ENTIRE cart to order                                  │
│     • Cart is emptied after checkout                                 │
│     • Used when: "Checkout All" button                               │
│                                                                      │
│  2. directCheckout()  ───────────────────────────────────────────   │
│     • "Buy Now" - bypasses cart entirely                             │
│     • Creates order from single product                              │
│     • Cart remains UNTOUCHED                                         │
│     • Used when: Product page "Buy Now" button                       │
│                                                                      │
│  3. checkoutSelected()  ─────────────────────────────────────────   │
│     • Partial checkout - only selected items                         │
│     • Selected items removed from cart                               │
│     • Unselected items REMAIN in cart                                │
│     • Used when: "Checkout (3)" button for selected                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Complete Checkout Flow

**File**: `app/Http/Controllers/Api/CheckoutController.php` (Lines 17-406)

```php
public function checkout(Request $request)
{
    $data = $request->validate([
        'payment_method' => 'required|string|in:physical,gcash',
        'payment_type' => 'required|string|in:full,partial,cod',
        'fulfillment_method' => 'required|string|in:pickup,delivery',
        'amount_paid' => 'required|numeric|min:0',
        'is_preorder' => 'boolean',
        'is_reservation' => 'boolean',
        // ... more fields
    ]);

    $user = $request->user();
    $cart = OrderCart::with('items.merchandise.organization')
        ->where('user_id', $user->id)
        ->where('type', 'cart')
        ->first();

    if (!$cart || $cart->items->isEmpty()) {
        return response()->json(['message' => 'Cart is empty'], 400);
    }

    // ═══════════════════════════════════════════════════════════
    // STEP 1: CALCULATE ORIGINAL SUBTOTAL
    // ═══════════════════════════════════════════════════════════
    $originalSubtotal = $cart->items->sum(function($item) {
        return $item->quantity * $item->price;
    });

    // ═══════════════════════════════════════════════════════════
    // STEP 2: CALCULATE PRODUCT DISCOUNTS (merchandise-level)
    // ═══════════════════════════════════════════════════════════
    $productDiscountAmount = 0;
    foreach ($cart->items as $item) {
        $merchandise = $item->merchandise;
        if ($merchandise && $merchandise->discount > 0) {
            $itemTotal = $item->quantity * $item->price;
            $productDiscountAmount += ($itemTotal * $merchandise->discount / 100);
        }
    }
    $subtotal = $originalSubtotal - $productDiscountAmount;

    // ═══════════════════════════════════════════════════════════
    // STEP 3: CALCULATE MEMBER/PREORDER DISCOUNTS
    // ═══════════════════════════════════════════════════════════
    $memberDiscountAmount = 0;
    $discountType = null;
    $userOrgIds = $user->organizations()->pluck('organizations.id')->toArray();

    $isPreorder = $data['is_preorder'] ?? false;

    if ($isPreorder) {
        // Preorders get flat 10% on subtotal
        $memberDiscountAmount = $subtotal * 0.10;
        $discountType = 'preorder_10%';
    } else {
        // Member discount: 10% on original price for items from orgs user belongs to
        foreach ($cart->items as $item) {
            $merchOrgId = $item->merchandise->organization_id;
            if (in_array($merchOrgId, $userOrgIds)) {
                $itemTotal = $item->quantity * $item->price;
                $memberDiscountAmount += ($itemTotal * 0.10);
                $discountType = 'member_10%';
            }
        }
    }

    $totalDiscountAmount = $productDiscountAmount + $memberDiscountAmount;

    // ═══════════════════════════════════════════════════════════
    // STEP 4: SHIPPING FEE
    // ═══════════════════════════════════════════════════════════
    $shippingFee = 0;
    if ($data['fulfillment_method'] === 'delivery') {
        $shippingFee = 50; // Flat ₱50 delivery fee
    }

    // ═══════════════════════════════════════════════════════════
    // STEP 5: CALCULATE FINAL TOTAL
    // ═══════════════════════════════════════════════════════════
    $totalAmount = $subtotal - $memberDiscountAmount + $shippingFee;

    // RESERVATION OVERRIDE: ₱20 per item regardless of price
    $isReservation = $data['is_reservation'] ?? false;
    if ($isReservation) {
        $itemCount = $cart->items->sum('quantity');
        $totalAmount = $itemCount * 20;
    }

    // ═══════════════════════════════════════════════════════════
    // STEP 6: VALIDATE PAYMENT RULES
    // ═══════════════════════════════════════════════════════════
    $this->validatePaymentRules($data, $totalAmount, $isReservation);

    // ═══════════════════════════════════════════════════════════
    // STEP 7: CALCULATE BALANCE DUE
    // ═══════════════════════════════════════════════════════════
    $balanceDue = max(0, $totalAmount - $data['amount_paid']);

    $paymentDueDate = null;
    if ($data['payment_type'] === 'partial' && $balanceDue > 0) {
        $paymentDueDate = Carbon::now()->addDays(3); // 3 days to pay balance
    }

    // ═══════════════════════════════════════════════════════════
    // STEP 8: CONVERT CART TO ORDER
    // ═══════════════════════════════════════════════════════════
    $cart->type = 'order';
    $cart->status = $isReservation ? 'completed' : 'pending';
    $cart->payment_method = $data['payment_method'];
    // ... save all calculated fields
    $cart->order_tracking_number = $isReservation
        ? 'RES-' . strtoupper(uniqid())
        : 'ORD-' . strtoupper(uniqid());
    $cart->save();

    // ═══════════════════════════════════════════════════════════
    // STEP 9: DEDUCT STOCK & UPDATE SALES
    // ═══════════════════════════════════════════════════════════
    foreach ($cart->items as $item) {
        $merchandise = Merchandise::find($item->merchandise_id);
        if ($merchandise) {
            $merchandise->increment('total_sales', $item->quantity);

            if ($item->variant) {
                // Deduct from variant stock + sync main stock
                $this->deductVariantStock(...);
                $merchandise->syncStockFromVariants();
            } else {
                $merchandise->stock = max(0, $merchandise->stock - $item->quantity);
                $merchandise->save();
            }
        }
    }

    // ═══════════════════════════════════════════════════════════
    // STEP 10: CREATE PAYMENT RECORD
    // ═══════════════════════════════════════════════════════════
    Payment::create([
        'orders_and_cart_id' => $cart->id,
        'payment_method' => $data['payment_method'],
        'transaction_id' => $this->generateTransactionId($data['payment_method']),
        'amount' => $data['amount_paid'],
        'status' => $balanceDue > 0 ? 'partial' : 'full',
    ]);

    // ═══════════════════════════════════════════════════════════
    // STEP 11: CREATE USER NOTIFICATION
    // ═══════════════════════════════════════════════════════════
    Notification::create([
        'user_id' => $user->id,
        'title' => 'Order Placed Successfully',
        'body' => 'Thank you for your order! Your order is being processed.',
    ]);

    // ═══════════════════════════════════════════════════════════
    // STEP 12: CREATE NEW EMPTY CART
    // ═══════════════════════════════════════════════════════════
    OrderCart::create([
        'user_id' => $user->id,
        'type' => 'cart',
        'status' => 'pending',
        'total_amount' => 0
    ]);
}
```

**🔍 Pricing Formula Visualization:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CHECKOUT PRICING FORMULA                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. ORIGINAL_SUBTOTAL = Σ(quantity × price) for all items            │
│     Example: 2 × ₱599 + 1 × ₱399 = ₱1,597                            │
│                                                                      │
│  2. PRODUCT_DISCOUNT = Σ(item_total × discount%) for discounted      │
│     Example: T-shirt has 20% off: ₱599 × 0.20 = ₱119.80             │
│                                                                      │
│  3. SUBTOTAL = ORIGINAL - PRODUCT_DISCOUNT                           │
│     Example: ₱1,597 - ₱119.80 = ₱1,477.20                           │
│                                                                      │
│  4. MEMBER_DISCOUNT (if member of selling org):                      │
│     = item_original × 10% (only for member items)                    │
│     OR if preorder: SUBTOTAL × 10%                                   │
│                                                                      │
│  5. SHIPPING = ₱50 if delivery, ₱0 if pickup                         │
│                                                                      │
│  6. TOTAL = SUBTOTAL - MEMBER_DISCOUNT + SHIPPING                    │
│                                                                      │
│  7. RESERVATION OVERRIDE: TOTAL = quantity × ₱20                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 Payment Validation Rules

**File**: `app/Http/Controllers/Api/CheckoutController.php` (Lines 411-457)

```php
private function validatePaymentRules($data, $totalAmount, $isReservation = false)
{
    // RESERVATION RULES
    if ($isReservation) {
        if ($data['payment_method'] !== 'gcash') {
            throw new \Exception('Reservations require GCash payment');
        }
        return; // No other validation for reservations
    }

    // PHYSICAL (CASH) PAYMENT RULES
    if ($data['payment_method'] === 'physical') {
        if ($data['payment_type'] === 'cod') {
            return; // COD = pay on delivery, no upfront payment needed
        }
        if ($data['payment_type'] === 'partial' && $data['amount_paid'] > 0) {
            throw new \Exception('For partial payments, please pay at the organization office');
        }
    }

    // GCASH PAYMENT RULES
    if ($data['payment_method'] === 'gcash') {
        if ($data['amount_paid'] <= 0) {
            throw new \Exception('GCash payment requires an amount greater than 0');
        }

        // PARTIAL: Must pay at least 50%
        if ($data['payment_type'] === 'partial') {
            $minimumPartial = $totalAmount * 0.5;
            if ($data['amount_paid'] < $minimumPartial) {
                throw new \Exception("Partial payment must be at least 50% (₱" . number_format($minimumPartial, 2) . ")");
            }
        }

        // FULL: Must pay entire amount
        if ($data['payment_type'] === 'full' && $data['amount_paid'] < $totalAmount) {
            throw new \Exception('Full payment amount must equal total: ₱' . number_format($totalAmount, 2));
        }
    }
}
```

**🔍 Payment Rules Summary:**

| Payment Method | Payment Type | Rule                                               |
| -------------- | ------------ | -------------------------------------------------- |
| GCash          | Full         | Must pay 100%                                      |
| GCash          | Partial      | Must pay ≥50%, balance due in 3 days               |
| Physical       | COD          | Pay ₱0 now, full amount on pickup/delivery         |
| Physical       | Partial      | Must go to org office (online partial not allowed) |
| Any            | Reservation  | Fixed ₱20/item, GCash only                         |

---

## 8. Order Lifecycle & Status Management

### 8.1 Order Status Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ORDER STATUS FLOW                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                      ┌──────────────┐                                    │
│  Order Created ────▶ │   PENDING    │ ◄─────── Can CANCEL within 12 hr  │
│                      └──────┬───────┘                                    │
│                             │ Admin confirms                             │
│                             ▼                                            │
│                      ┌──────────────┐                                    │
│                      │  CONFIRMED   │                                    │
│                      └──────┬───────┘                                    │
│                             │ Admin starts processing                    │
│                             ▼                                            │
│                      ┌──────────────┐                                    │
│                      │  PROCESSING  │                                    │
│                      └──────┬───────┘                                    │
│                             │ Order is ready                             │
│                             ▼                                            │
│                      ┌──────────────┐                                    │
│                      │    READY     │ ◄─────── User can REQUEST REFUND  │
│                      └──────┬───────┘                                    │
│                             │ User clicks "Confirm Received"             │
│                             ▼                                            │
│                      ┌──────────────┐                                    │
│                      │  COMPLETED   │                                    │
│                      └──────────────┘                                    │
│                                                                          │
│  ALTERNATIVE FLOWS:                                                      │
│  • PENDING ──▶ CANCELLED (by user within 12hr, or by admin anytime)     │
│  • READY ──▶ REFUND_PENDING ──▶ REFUNDED (if admin approves)            │
│  • RESERVATION: PENDING ──▶ COMPLETED (immediately after payment)       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 User Cancellation (12-Hour Window)

**File**: `app/Http/Controllers/Api/OrderController.php` (Lines 170-224)

```php
public function cancelOrder(Request $request, $id)
{
    $order = OrderCart::with('items.merchandise')
        ->where('user_id', $request->user()->id)
        ->where('type', 'order')
        ->where('id', $id)
        ->where('status', 'pending')  // ONLY pending orders can be cancelled
        ->firstOrFail();

    // Check if order is within 12-hour cancellation window
    $orderDate = $order->created_at;
    $now = now();
    $hoursSinceOrder = $orderDate->diffInHours($now);

    if ($hoursSinceOrder >= 12) {
        return response()->json([
            'message' => 'Cannot cancel order. Cancellation window (12 hours) has expired.'
        ], 422);
    }

    // RESTORE STOCK atomically
    DB::beginTransaction();
    try {
        foreach ($order->items as $item) {
            $this->restoreStockForItem($item);
        }

        $order->status = 'cancelled';
        $order->save();

        DB::commit();
    } catch (\Exception $e) {
        DB::rollBack();
        // ... error handling
    }
}
```

**🔍 Why 12-Hour Window?**

-   Gives users time to realize mistakes
-   But not so long that orgs can't plan inventory
-   After 12 hours, user must contact admin or request refund

### 8.3 Admin Order Status Update

**File**: `app/Http/Controllers/Api/AdminOrderController.php` (Lines 61-126)

```php
public function updateStatus(Request $request, $id)
{
    $admin = $request->attributes->get('org_admin');

    $order = OrderCart::with('items.merchandise')
        ->where('type', 'order')
        ->whereHas('items.merchandise', function($query) use ($admin) {
            $query->where('organization_id', $admin->organization_id);
        })
        ->findOrFail($id);

    $validated = $request->validate([
        'status' => 'required|string|in:pending,confirmed,processing,ready,completed,shipped,delivered,cancelled',
    ]);

    $previousStatus = $order->status;
    $newStatus = $validated['status'];

    DB::beginTransaction();
    try {
        // If changing to cancelled, RESTORE STOCK
        if ($newStatus === 'cancelled' && $previousStatus !== 'cancelled') {
            foreach ($order->items as $item) {
                $this->restoreStockForItem($item, $admin->organization_id);
            }
        }

        $order->update(['status' => $newStatus]);
        DB::commit();
    } catch (\Exception $e) {
        DB::rollBack();
    }
}
```

**🔍 Key Points:**

| Feature                                                  | Explanation                                               |
| -------------------------------------------------------- | --------------------------------------------------------- |
| `whereHas('items.merchandise', ...)`                     | Admin can ONLY see orders containing their org's products |
| `$admin->organization_id` check in `restoreStockForItem` | Only restores stock for THEIR products, not other orgs'   |
| Transaction `DB::beginTransaction()`                     | Ensures stock restoration is atomic (all or nothing)      |

---

## 9. Refund System

### 9.1 Refund Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         REFUND FLOW                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Order Status = READY                                                │
│         │                                                            │
│         │ User clicks "Request Refund"                               │
│         ▼                                                            │
│  ┌──────────────────┐                                                │
│  │  refund_status   │ = 'pending'                                    │
│  │  refund_reason   │ = "Item arrived damaged"                       │
│  │  refund_requested_at = now()                                      │
│  └────────┬─────────┘                                                │
│           │                                                          │
│           │ Admin sees in Refunds page                               │
│           │                                                          │
│    ┌──────┴──────┐                                                   │
│    ▼             ▼                                                   │
│ APPROVE        REJECT                                                │
│    │             │                                                   │
│    ▼             ▼                                                   │
│ refund_status  refund_status                                         │
│ = 'approved'   = 'rejected'                                          │
│    │                                                                 │
│    ▼                                                                 │
│ Order status = 'completed'                                           │
│ User notified                                                        │
│ (Manual refund processing by org)                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 User Requesting Refund

**File**: `app/Http/Controllers/Api/OrderController.php` (Lines 412-456)

```php
public function requestRefund(Request $request, $id)
{
    $order = OrderCart::where('user_id', $request->user()->id)
        ->where('type', 'order')
        ->where('id', $id)
        ->firstOrFail();

    // RULE 1: Only READY orders can be refunded
    if ($order->status !== 'ready') {
        return response()->json([
            'message' => 'Refund can only be requested for orders that are ready for pickup/delivery.'
        ], 422);
    }

    // RULE 2: No duplicate requests
    if ($order->refund_status && $order->refund_status !== 'none') {
        return response()->json([
            'message' => 'A refund has already been requested for this order.'
        ], 422);
    }

    // RULE 3: Reservations cannot be refunded
    if ($order->is_reservation) {
        return response()->json([
            'message' => 'Reservations are not eligible for refunds.'
        ], 422);
    }

    $order->refund_status = 'pending';
    $order->refund_reason = $request->input('reason');
    $order->refund_requested_at = now();
    $order->save();
}
```

### 9.3 Admin Approving Refund

**File**: `app/Http/Controllers/Api/AdminOrderController.php` (Lines 362-408)

```php
public function approveRefund(Request $request, $id)
{
    $admin = $request->attributes->get('org_admin');

    $order = OrderCart::where('type', 'order')
        ->where('id', $id)
        ->where('refund_status', 'pending')
        ->whereHas('items.merchandise', function($query) use ($admin) {
            $query->where('organization_id', $admin->organization_id);
        })
        ->firstOrFail();

    $order->refund_status = 'approved';
    $order->status = 'completed';  // Mark order as completed
    $order->save();

    // Notify user
    Notification::create([
        'organization_id' => $admin->organization_id,
        'user_id' => $order->user_id,
        'title' => 'Refund Approved',
        'body' => "Your refund request for Order #{$order->id} has been approved. The refund will be processed shortly.",
    ]);
}
```

**⚠️ Note**: Actual money refund is processed manually by the organization. The system only tracks the approval status.

---

## 10. Admin Panel Logic

### 10.1 Admin Middleware (Gate Keeping)

**File**: `app/Http/Middleware/AdminMiddleware.php`

```php
public function handle(Request $request, Closure $next)
{
    // First check if authenticated via Sanctum
    if (!Auth::guard('sanctum')->check()) {
        return response()->json([
            'message' => 'Unauthorized. Admin access required.'
        ], 403);
    }

    // Get the authenticated user
    $user = Auth::guard('sanctum')->user();

    // Check if the authenticated user is an OrgAdmin
    if (!($user instanceof OrgAdmin)) {
        return response()->json([
            'message' => 'Unauthorized. Admin access required.'
        ], 403);
    }

    // Set the admin in request for controllers to use
    $request->attributes->set('org_admin', $user);

    return $next($request);
}
```

**🔍 How It Works:**

| Step                                 | What Happens                                                |
| ------------------------------------ | ----------------------------------------------------------- |
| 1. `Auth::guard('sanctum')->check()` | Verifies there's a valid token                              |
| 2. `$user instanceof OrgAdmin`       | Checks the token belongs to `org_admins` table, not `users` |
| 3. `$request->attributes->set(...)`  | Makes admin object available to controllers                 |

### 10.2 Analytics Calculations

**File**: `app/Http/Controllers/Api/AdminController.php` (Lines 33-198)

```php
public function analytics(Request $request)
{
    $admin = $request->attributes->get('org_admin');
    $days = $request->get('days', 30);

    // Filter orders by admin's organization
    $orderQuery = Order::where('type', 'order')
        ->whereHas('items.merchandise', function($query) use ($admin) {
            $query->where('organization_id', $admin->organization_id);
        });

    // PERCENTAGE CHANGE FORMULA
    // (current - previous) / previous × 100
    $sales_change = $previous_sales > 0
        ? (($total_sales - $previous_sales) / $previous_sales) * 100
        : 0;

    // AVERAGE ORDER VALUE
    $avg_order_value = $total_orders > 0 ? $total_sales / $total_orders : 0;

    // CONVERSION RATE
    $conversion_rate = $all_time_orders > 0
        ? ($completed_orders / $all_time_orders) * 100
        : 0;

    // TOP SELLING PRODUCTS (by revenue)
    $top_products = DB::table('orders_and_cart_items')
        ->join('merchandises', ...)
        ->selectRaw('SUM(quantity) as units_sold, SUM(quantity * price) as revenue')
        ->groupBy('merchandises.id')
        ->orderByDesc('revenue')
        ->limit(5)
        ->get();

    // LOW STOCK ITEMS (stock < 10)
    $low_stock_items = DB::table('merchandises')
        ->where('organization_id', $admin->organization_id)
        ->where('stock', '<', 10)
        ->where('stock', '>', 0)
        ->count();
}
```

---

## 11. Stock Management Deep Dive

### 11.1 Stock Deduction with Variants

**File**: `app/Http/Controllers/Api/CheckoutController.php` (Lines 186-330)

```php
// If item has variant, deduct from variant stock
if ($item->variant) {
    $variantOptions = json_decode($item->variant, true);

    // Handle double-encoded variants
    if (isset($variantOptions['variant']) && is_string($variantOptions['variant'])) {
        $innerVariant = json_decode($variantOptions['variant'], true);
        if ($innerVariant) {
            $variantOptions = $innerVariant;
        }
    }

    // Find the variant record
    $merchandiseVariant = DB::table('merchandise_variants')
        ->where('merchandise_id', $merchandise->id)
        ->first();

    if ($merchandiseVariant) {
        $variantStock = json_decode($merchandiseVariant->variant_stock, true) ?: [];

        // BUILD POSSIBLE KEY COMBINATIONS
        // User selected: {"size": "S", "color": "Red"}
        // Stock might have: "Red - S" or "S - Red" or "red - s"
        $values = array_values($variantOptions);  // ["S", "Red"]
        $possibleKeys = [];

        // Try: "s - red" (lowercase)
        $possibleKeys[] = strtolower(implode(' - ', $values));
        // Try: "S - Red" (original case)
        $possibleKeys[] = implode(' - ', $values);
        // Try: reversed order
        $reversedValues = array_reverse($values);
        $possibleKeys[] = strtolower(implode(' - ', $reversedValues));

        // Find matching key and deduct
        foreach ($possibleKeys as $key) {
            if (isset($variantStock[$key])) {
                $variantStock[$key] = max(0, $variantStock[$key] - $item->quantity);

                // Update database
                DB::table('merchandise_variants')
                    ->where('id', $merchandiseVariant->id)
                    ->update(['variant_stock' => json_encode($variantStock)]);
                break;
            }
        }

        // SYNC MAIN STOCK
        $merchandise->syncStockFromVariants();
    }
} else {
    // No variant - simple deduction
    $merchandise->stock = max(0, $merchandise->stock - $item->quantity);
    $merchandise->save();
}
```

**🔍 Why So Complex?**

The variant key matching is complex because:

1. Admin might enter "Red - Small" but user selects {"color": "Red", "size": "Small"}
2. JSON key order isn't guaranteed
3. Case sensitivity varies

### 11.2 Stock Sync from Variants

**File**: `app/Models/Merchandise.php` (Lines 207-217)

```php
public function syncStockFromVariants()
{
    $variant = $this->variants()->first();
    if ($variant && is_array($variant->variant_stock)) {
        // Sum all variant stock values
        // {"Red - S": 10, "Red - M": 15, "Blue - S": 20} = 45
        $totalStock = array_sum($variant->variant_stock);
        $this->stock = $totalStock;
        $this->save();
        return $totalStock;
    }
    return $this->stock;
}
```

**🔍 Why Sync?**

-   Main `stock` field is used for quick availability checks
-   Variant stock is the source of truth for per-variant quantities
-   After any variant stock change, main stock must be recalculated

---

## 12. Pricing & Discount Engine

### 12.1 Discount Stacking

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DISCOUNT STACKING RULES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DISCOUNT TYPE 1: PRODUCT DISCOUNT                                   │
│  ├── Set by Admin on merchandise.discount field                      │
│  ├── Applied first to original price                                 │
│  └── Example: 20% off T-shirt (₱599 → ₱479.20)                       │
│                                                                      │
│  DISCOUNT TYPE 2: MEMBER/PREORDER DISCOUNT                           │
│  ├── 10% for approved organization members                           │
│  ├── OR 10% for preorders (not both)                                 │
│  └── Applied after product discount                                  │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────   │
│                                                                      │
│  EXAMPLE CALCULATION:                                                │
│  ─────────────────────                                               │
│  Original Price:         ₱599.00                                     │
│  Product Discount (20%): -₱119.80                                    │
│  Subtotal:               ₱479.20                                     │
│  Member Discount (10%):  -₱47.92                                     │
│  Shipping:               +₱50.00                                     │
│  ─────────────────────                                               │
│  TOTAL:                  ₱481.28                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.2 Reservation Special Pricing

```php
if ($isReservation) {
    $itemCount = $cart->items->sum('quantity');
    $totalAmount = $itemCount * 20;  // ₱20 × quantity
}
```

**Why Reservations?**

-   For out-of-stock items that will be restocked
-   User pays ₱20/item to "reserve" their spot
-   Full payment collected when item arrives
-   Acts as commitment fee (non-refundable)

---

## 13. Messaging System (Student ↔ Admin Chat)

### 13.1 What is the Messaging System?

Think of this like **Facebook Messenger** but built specifically for the store. Students can chat directly with organization admins to ask questions about products, orders, or any concerns. It's a two-way communication channel.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MESSAGING SYSTEM OVERVIEW                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   STUDENT SIDE                          ADMIN SIDE                   │
│   ────────────                          ──────────                   │
│   📱 Messages Page                      💼 Admin Messages Page       │
│      │                                      │                        │
│      ├── See all conversations              ├── See all customer     │
│      │   with different orgs                │   conversations        │
│      │                                      │                        │
│      ├── Send text messages                 ├── Reply to students    │
│      │                                      │                        │
│      ├── Send images (up to 10)             ├── Send images          │
│      │                                      │                        │
│      ├── See "sent" ✓ / "seen" ✓✓           ├── See unread badges    │
│      │                                      │                        │
│      └── Unsend messages                    └── Unsend own messages  │
│          Delete conversations                   Delete conversations │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.2 Message Status Tracking

**File**: `app/Http/Controllers/Api/MessageController.php` (Lines 15-37)

```php
public function getUnreadCount(Request $request)
{
    $user = $request->user();

    // Get total unread messages (messages from admins that haven't been seen)
    $totalUnread = Message::where('user_id', $user->id)
        ->where('is_from_admin', true)
        ->where('status', '!=', 'seen')
        ->count();

    // Get unread count per organization
    $unreadByOrg = Message::where('user_id', $user->id)
        ->where('is_from_admin', true)
        ->where('status', '!=', 'seen')
        ->selectRaw('organization_id, COUNT(*) as unread_count')
        ->groupBy('organization_id')
        ->pluck('unread_count', 'organization_id');

    return response()->json([
        'total_unread' => $totalUnread,
        'unread_by_organization' => $unreadByOrg
    ]);
}
```

**🧠 Beginner Explanation (Like Explaining to a 10-Year-Old):**

Imagine you have a mailbox. This code is like checking your mailbox to see how many letters you haven't read yet.

| What The Code Does              | Real-World Analogy                                      |
| ------------------------------- | ------------------------------------------------------- |
| `where('user_id', $user->id)`   | "Show me only MY letters, not someone else's"           |
| `where('is_from_admin', true)`  | "Only count letters FROM the store, not letters I sent" |
| `where('status', '!=', 'seen')` | "Only count letters I haven't opened yet"               |
| `->count()`                     | "How many are there?"                                   |
| `->groupBy('organization_id')`  | "Separate them into piles - one pile per store"         |

**Result**: The app shows a red badge with "3" on the chat icon, meaning you have 3 unread messages!

### 13.3 Sending a Message

**File**: `app/Http/Controllers/Api/MessageController.php` (Lines 89-113)

```php
public function send(Request $request)
{
    $data = $request->validate([
        'organization_id' => 'required|exists:organizations,id',
        'body' => 'nullable|string',
        'images' => 'nullable|array|max:10',
        'images.*' => 'string|max:500',
    ]);

    // Require at least body or images
    if (empty($data['body']) && empty($data['images'])) {
        return response()->json(['message' => 'Message body or images required'], 422);
    }

    $msg = Message::create([
        'user_id' => $request->user()->id,
        'organization_id' => $data['organization_id'],
        'body' => $data['body'] ?? '',
        'images' => $data['images'] ?? null,
        'status' => 'sent',
        'is_from_admin' => false,
    ]);

    return response()->json($msg, 201);
}
```

**🧠 Line-by-Line Breakdown:**

| Line                                                  | What It Does                                      | Why It Matters                                                    |
| ----------------------------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------- |
| `'images' => 'nullable\|array\|max:10'`               | User can attach up to 10 images                   | Prevents someone from sending 1000 images and crashing the system |
| `if (empty($data['body']) && empty($data['images']))` | Check: did they type anything OR attach pictures? | Can't send an empty message!                                      |
| `'status' => 'sent'`                                  | Message starts as "sent"                          | Later changes to "seen" when admin reads it                       |
| `'is_from_admin' => false`                            | This message is FROM a student                    | So the system knows who sent it                                   |

### 13.4 Conversation Grouping

**File**: `app/Http/Controllers/Api/MessageController.php` (Lines 56-87)

```php
// Group by organization to get conversations
$conversations = $messages->groupBy('organization_id')->map(function ($msgs) {
    $latest = $msgs->first();
    $org = $latest->organization;

    return [
        'id' => $latest->organization_id,
        'organization' => [
            'id' => $org->id ?? null,
            'name' => $org->name ?? 'Unknown Organization',
            'logo_url' => $org->logo_url ?? null,
        ],
        'last_message' => [
            'message' => $latest->body,
            'is_from_admin' => $latest->is_from_admin,
            'created_at' => $latest->created_at
        ],
        'unread_count' => $msgs->where('is_from_admin', true)->where('status', '!=', 'seen')->count()
    ];
})->values();
```

**🧠 What This Creates:**

Think of this like your phone's messaging app showing a list of chats:

```
┌─────────────────────────────────────────────┐
│ 💬 Messages                                 │
├─────────────────────────────────────────────┤
│ 🏢 Computer Science Society        (2) 🔴   │
│    "Your order is ready for pickup"         │
│    Today, 2:30 PM                           │
├─────────────────────────────────────────────┤
│ 🏢 Engineering Club                         │
│    "Thanks for your order!"                 │
│    Yesterday                                │
├─────────────────────────────────────────────┤
│ 🏢 Sports Club                     (1) 🔴   │
│    "Do you have size L?"                    │
│    Dec 24                                   │
└─────────────────────────────────────────────┘
```

The `(2)` and `(1)` are the `unread_count` calculated in the code!

---

## 14. Favorites/Wishlist System

### 14.1 What is the Favorites System?

It's like a **bookmark** for products. When you see something you like but aren't ready to buy, you tap the heart ❤️ to save it for later.

### 14.2 Toggle Favorite (Heart Button)

**File**: `app/Http/Controllers/Api/FavoriteController.php` (Lines 118-151)

```php
public function toggle(Request $request)
{
    $request->validate([
        'merchandise_id' => 'required|integer|exists:merchandises,id'
    ]);

    $userId = $request->user()->id;
    $merchandiseId = $request->merchandise_id;

    // Check if already favorited
    $existing = Favorite::where('user_id', $userId)
        ->where('merchandise_id', $merchandiseId)
        ->first();

    if ($existing) {
        // Already favorited? Remove it!
        $existing->delete();
        return response()->json([
            'success' => true,
            'message' => 'Removed from favorites',
            'is_favorited' => false
        ]);
    }

    // Not favorited? Add it!
    $favorite = Favorite::create([
        'user_id' => $userId,
        'merchandise_id' => $merchandiseId
    ]);

    return response()->json([
        'success' => true,
        'message' => 'Added to favorites',
        'is_favorited' => true,
        'favorite_id' => $favorite->id
    ]);
}
```

**🧠 Beginner Explanation:**

This is like a light switch - tap once to turn ON (add to favorites), tap again to turn OFF (remove from favorites).

```
User taps ❤️ on T-Shirt
        │
        ▼
Is T-Shirt already in favorites?
        │
   ┌────┴────┐
   YES       NO
   │         │
   ▼         ▼
Remove it   Add it
(♡ empty)   (❤️ filled)
```

### 14.3 Efficient Favorite Checking

**File**: `app/Http/Controllers/Api/FavoriteController.php` (Lines 184-194)

```php
public function ids(Request $request)
{
    $ids = Favorite::where('user_id', $request->user()->id)
        ->pluck('merchandise_id')
        ->toArray();

    return response()->json([
        'success' => true,
        'favorite_ids' => $ids
    ]);
}
```

**🧠 Why This Exists:**

Instead of checking each product one-by-one ("Is product #1 favorited? Is product #2 favorited?"), this gives you ALL your favorite product IDs at once: `[5, 12, 47, 89]`.

Now when showing 100 products, the frontend can instantly check:

-   Is product #5 in the list? YES → Show ❤️
-   Is product #6 in the list? NO → Show ♡

This is **much faster** than making 100 separate requests!

---

## 15. Preorder Queue System

### 15.1 Preorder vs Regular Order - What's the Difference?

| Aspect       | Regular Order                   | Preorder                           |
| ------------ | ------------------------------- | ---------------------------------- |
| **When**     | Item is IN stock                | Item is OUT of stock               |
| **Stock**    | Deducted immediately            | NOT deducted (no stock to deduct!) |
| **Discount** | Member discount (if applicable) | 10% preorder discount              |
| **Status**   | Pending → Processing → Ready    | Pending → Confirmed → Ready        |
| **Purpose**  | Buy now, get now                | Reserve now, get later             |

### 15.2 Creating a Preorder

**File**: `app/Http/Controllers/Api/PreorderController.php` (Lines 55-113)

```php
public function store(Request $request)
{
    $validator = Validator::make($request->all(), [
        'merchandise_id' => 'required|exists:merchandises,id',
        'quantity' => 'required|integer|min:1',
        'expected_delivery_date' => 'nullable|date|after:today'
    ]);

    // Check if merchandise exists
    $merchandise = Merchandise::find($request->merchandise_id);

    if (!$merchandise) {
        return response()->json([
            'success' => false,
            'message' => 'Merchandise not found'
        ], 404);
    }

    // Check if merchandise is actually out of stock
    if ($merchandise->stock > 0) {
        return response()->json([
            'success' => false,
            'message' => 'This item is in stock. Please add it to cart instead.'
        ], 400);
    }

    // Calculate total amount
    $price = $merchandise->discount > 0
        ? ($merchandise->price - $merchandise->discount)
        : $merchandise->price;
    $totalAmount = $price * $request->quantity;

    // Create preorder
    $preorder = Preorder::create([
        'user_id' => $request->user()->id,
        'merchandise_id' => $request->merchandise_id,
        'quantity' => $request->quantity,
        'total_amount' => $totalAmount,
        'status' => 'pending',
        'expected_delivery_date' => $request->expected_delivery_date ?? now()->addWeeks(2)
    ]);

    return response()->json([
        'success' => true,
        'message' => 'Preorder created successfully. You will be notified when the item is available.',
        'preorder' => $preorder
    ], 201);
}
```

**🧠 Step-by-Step What Happens:**

```
┌─────────────────────────────────────────────────────────────────────┐
│             PREORDER CREATION FLOW                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1️⃣ User sees: "T-Shirt - OUT OF STOCK"                             │
│         └── Button says "Pre-order" instead of "Add to Cart"        │
│                                                                      │
│  2️⃣ User clicks "Pre-order"                                         │
│         │                                                            │
│         ▼                                                            │
│  3️⃣ System checks: Is it REALLY out of stock?                       │
│         │                                                            │
│    ┌────┴────┐                                                       │
│    │ stock=0 │ YES → Continue to preorder                            │
│    │ stock>0 │ NO  → "Item is in stock, add to cart instead!"       │
│    └─────────┘                                                       │
│         │                                                            │
│         ▼                                                            │
│  4️⃣ Create preorder record with:                                    │
│         ├── User ID (who's ordering)                                 │
│         ├── Product ID (what they want)                              │
│         ├── Quantity (how many)                                      │
│         ├── Expected date (when it might arrive)                     │
│         └── Status = "pending"                                       │
│                                                                      │
│  5️⃣ User gets message: "You'll be notified when available!"         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 15.3 Admin Preorder Dashboard

**File**: `app/Http/Controllers/Api/PreorderController.php` (Lines 152-188)

```php
public function adminIndex(Request $request)
{
    $admin = $request->attributes->get('org_admin');

    // Get all preorders for merchandises belonging to this organization
    $preorders = Preorder::whereHas('merchandise', function($query) use ($admin) {
        $query->where('organization_id', $admin->organization_id);
    })
    ->with(['user', 'merchandise'])
    ->orderBy('created_at', 'desc')
    ->get();

    // Group by status for easier management
    $grouped = [
        'pending' => $preorders->where('status', 'pending')->values(),
        'confirmed' => $preorders->where('status', 'confirmed')->values(),
        'processing' => $preorders->where('status', 'processing')->values(),
        'ready' => $preorders->where('status', 'ready')->values(),
        'completed' => $preorders->where('status', 'completed')->values(),
        'cancelled' => $preorders->where('status', 'cancelled')->values(),
    ];

    return response()->json([
        'preorders' => $preorders,
        'grouped_by_status' => $grouped,
        'summary' => [
            'total' => $preorders->count(),
            'pending' => $grouped['pending']->count(),
            // ... etc
        ]
    ]);
}
```

**🧠 What The Admin Sees:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  📦 PREORDER MANAGEMENT                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Summary:  Total: 45 | Pending: 12 | Confirmed: 8 | Ready: 5        │
│                                                                      │
│  ┌─── PENDING (12) ────────────────────────────────────────────┐    │
│  │ Juan Dela Cruz    | T-Shirt Blue M  | Qty: 2 | Dec 20       │    │
│  │ Maria Santos      | Hoodie Black L  | Qty: 1 | Dec 21       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─── CONFIRMED (8) ───────────────────────────────────────────┐    │
│  │ Pedro Garcia      | Cap White       | Qty: 3 | Dec 18       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 16. Admin Notification Broadcasting

### 16.1 What is Broadcasting?

It's like a **school PA system announcement**. Instead of messaging students one-by-one, the admin can send one message that reaches:

-   ALL users in the system
-   Only their organization's MEMBERS
-   One SPECIFIC person

### 16.2 Flexible Notification Targeting

**File**: `app/Http/Controllers/Api/AdminNotificationController.php` (Lines 49-125)

```php
public function send(Request $request)
{
    $validated = $request->validate([
        'target_type' => 'required|string|in:all,members,specific',
        'notification_type' => 'required|string|in:info,success,warning,error',
        'title' => 'required|string|max:100',
        'content' => 'required|string|max:500',
        'target_id' => 'required_if:target_type,specific',
        'image_urls' => 'nullable|array|max:10',
    ]);

    $admin = $request->attributes->get('org_admin');

    $recipients = [];

    switch ($validated['target_type']) {
        case 'all':
            // Send to all users in the system
            $recipients = User::all();
            break;

        case 'members':
            // Send only to organization members
            $recipients = User::whereHas('organizations', function ($query) use ($admin) {
                $query->where('organizations.id', $admin->organization_id);
            })->get();
            break;

        case 'specific':
            // Send to specific user (by ID or email)
            $targetId = $validated['target_id'];
            $user = User::where('id', $targetId)
                ->orWhere('email', $targetId)
                ->orWhere('username', $targetId)
                ->orWhere('student_id', $targetId)
                ->first();

            if (!$user) {
                return response()->json([
                    'message' => 'User not found with ID/email: ' . $targetId
                ], 404);
            }

            $recipients = collect([$user]);
            break;
    }

    // Create notifications for all recipients
    $notifications = [];
    foreach ($recipients as $user) {
        $notifications[] = Notification::create([
            'user_id' => $user->id,
            'organization_id' => $admin->organization_id,
            'title' => $validated['title'],
            'body' => $validated['content'],
            'data' => json_encode([
                'type' => $validated['notification_type'],
                'title' => $validated['title'],
                'message' => $validated['content']
            ]),
            'sent_at' => now(),
        ]);
    }

    return response()->json([
        'message' => 'Notification sent successfully',
        'recipients_count' => count($notifications),
    ], 201);
}
```

**🧠 Visual Explanation:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                 ADMIN NOTIFICATION TARGETING                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Admin selects: "Send New Announcement"                              │
│                                                                      │
│  ┌─── Who should receive this? ────────────────────────────────┐    │
│  │                                                              │    │
│  │  ○ ALL USERS          → Everyone using the entire app       │    │
│  │                          (1,245 people)                      │    │
│  │                                                              │    │
│  │  ● OUR MEMBERS ONLY   → Only students who joined our org    │    │
│  │                          (156 members)                       │    │
│  │                                                              │    │
│  │  ○ SPECIFIC PERSON    → Type email/student ID/username      │    │
│  │                          (1 person)                          │    │
│  │                                                              │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Title: [🏷️ FLASH SALE! 50% OFF TODAY ONLY!              ]         │
│                                                                      │
│  Type:  ⓘ Info  ✓ Success  ⚠️ Warning  🚫 Error                     │
│                                                                      │
│  Message: [📝 Visit our booth at the main lobby...        ]         │
│                                                                      │
│  [📷 Add Images]                                                     │
│                                                                      │
│                                          [ 🚀 Send Notification ]    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**🔍 The Magic of `switch` Statement:**

```php
switch ($validated['target_type']) {
    case 'all':      // Like choosing "Everyone" in a group chat
    case 'members':  // Like choosing "Only my friends"
    case 'specific': // Like choosing "Just this one person"
}
```

---

## 17. Password Reset Flow

### 17.1 The Complete Password Reset Journey

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PASSWORD RESET FLOW                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1️⃣ FORGOT PASSWORD                                                 │
│     User clicks "Forgot Password?" on login page                     │
│     └── Enters their email address                                   │
│                                                                      │
│  2️⃣ TOKEN GENERATION                                                │
│     System creates a secret code (like a one-time key)               │
│     └── Stored in database with 60-minute expiry                     │
│                                                                      │
│  3️⃣ EMAIL SENT                                                      │
│     Email with reset link: yoursite.com/reset?token=abc123           │
│     └── User clicks the link                                         │
│                                                                      │
│  4️⃣ TOKEN VERIFICATION                                              │
│     System checks: Is this code valid? Not expired?                  │
│     └── If yes, show password reset form                             │
│                                                                      │
│  5️⃣ PASSWORD CHANGED                                                │
│     New password saved, token deleted, all sessions logged out       │
│     └── User can now login with new password                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.2 Sending Reset Link (Security Best Practice)

**File**: `app/Http/Controllers/Api/PasswordResetController.php` (Lines 17-88)

```php
public function sendResetLink(Request $request)
{
    $request->validate([
        'email' => 'required|email',
    ]);

    $user = User::where('email', $request->email)->first();

    if (!$user) {
        // Return success even if user doesn't exist (security best practice)
        return response()->json([
            'message' => 'If your email is registered, you will receive a password reset link shortly.'
        ]);
    }

    // Check if user is OAuth user (they can't reset password)
    if ($user->provider && $user->provider !== 'email') {
        return response()->json([
            'message' => 'This account uses ' . ucfirst($user->provider) . ' authentication.',
            'is_oauth' => true,
            'provider' => $user->provider
        ], 422);
    }

    // Generate token
    $token = Str::random(64);

    // Delete any existing reset tokens for this email
    DB::table('password_reset_tokens')->where('email', $request->email)->delete();

    // Store the new token
    DB::table('password_reset_tokens')->insert([
        'email' => $request->email,
        'token' => Hash::make($token),
        'created_at' => Carbon::now()
    ]);

    // Generate reset URL
    $frontendUrl = config('app.frontend_url', 'http://localhost:3000');
    $resetUrl = $frontendUrl . '/reset-password?token=' . $token . '&email=' . urlencode($request->email);

    // Send email
    Mail::send('emails.password-reset', [
        'user' => $user,
        'resetUrl' => $resetUrl,
    ], function ($message) use ($user) {
        $message->to($user->email)
            ->subject('Reset Your Password - University Merch Store');
    });

    return response()->json([
        'message' => 'If your email is registered, you will receive a password reset link shortly.'
    ]);
}
```

**🧠 Security Secrets Explained:**

| Security Feature                                  | Why It's Important                                                           |
| ------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Same message for existing/non-existing emails** | Hackers can't test which emails are registered by seeing different responses |
| **OAuth users blocked**                           | Google login users don't HAVE a password to reset!                           |
| **64-character random token**                     | Impossible to guess (like a 64-digit lottery number)                         |
| **Token is hashed before storing**                | Even if database is stolen, hackers can't use the tokens                     |
| **Old tokens deleted first**                      | Prevents having multiple valid codes floating around                         |

### 17.3 Token Expiration Check

**File**: `app/Http/Controllers/Api/PasswordResetController.php` (Lines 118-127)

```php
// Check if token is expired (60 minutes)
if (Carbon::parse($record->created_at)->addMinutes(60)->isPast()) {
    DB::table('password_reset_tokens')->where('email', $request->email)->delete();
    return response()->json([
        'valid' => false,
        'message' => 'This reset link has expired. Please request a new one.'
    ], 422);
}
```

**🧠 What This Does:**

```
Token created at: 2:00 PM
     + 60 minutes = 3:00 PM (expiration time)

Current time: 2:30 PM → ✅ Valid (before 3:00 PM)
Current time: 3:15 PM → ❌ Expired (after 3:00 PM)
```

---

## 18. User Notification Filtering

### 18.1 Why Filter Notifications?

The app has TWO separate "inbox" areas:

1. **🔔 Notification Bell** - Order updates, membership approvals, announcements
2. **💬 Message Icon** - Chat conversations with orgs

They must NOT overlap! A message from an org admin should appear in Messages, NOT in Notifications.

### 18.2 Filtering Out Message/Membership Notifications

**File**: `app/Http/Controllers/Api/NotificationController.php` (Lines 12-37)

```php
public function index(Request $request)
{
    $user = $request->user();

    // EXCLUDE message-type notifications - those are handled separately via the Messages system
    // EXCLUDE membership_request notifications - those are for admins, not users
    $notifications = Notification::with('organization')
        ->where('user_id', $user->id)
        ->where(function($query) {
            // Exclude notifications where data contains type = 'message' or 'membership_request'
            $query->whereNull('data')
                ->orWhere(function($q) {
                    $q->where('data', 'not like', '%"type":"message"%')
                      ->where('data', 'not like', '%"type": "message"%')
                      ->where('data', 'not like', '%"type":"membership_request"%')
                      ->where('data', 'not like', '%"type": "membership_request"%');
                });
        })
        ->orderBy('sent_at', 'desc')
        ->get();

    return response()->json([
        'notifications' => $notifications,
        'total' => $notifications->count()
    ]);
}
```

**🧠 Think of it Like Mail Sorting:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NOTIFICATION SORTING                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   📬 All Incoming Notifications                                      │
│           │                                                          │
│           ▼                                                          │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │                    SORTING ROOM                            │     │
│   │                                                            │     │
│   │   Check the "type" field in each notification:             │     │
│   │                                                            │     │
│   │   type = "message"            → Goes to 💬 Messages        │     │
│   │   type = "membership_request" → Goes to 📋 Admin Panel     │     │
│   │   type = anything else        → Goes to 🔔 Notifications   │     │
│   │                                                            │     │
│   └───────────────────────────────────────────────────────────┘     │
│           │                                                          │
│           ├────────────────┬────────────────┐                        │
│           ▼                ▼                ▼                        │
│        🔔 Bell          💬 Chat         📋 Admin                     │
│        (Orders,         (Conversations  (Membership                  │
│         Promos)          with orgs)      requests)                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**🔍 The Weird-Looking Query Explained:**

```php
->where('data', 'not like', '%"type":"message"%')
->where('data', 'not like', '%"type": "message"%')  // Note the space after colon
```

Why TWO conditions? Because JSON can be formatted differently:

-   `{"type":"message"}` (no space)
-   `{"type": "message"}` (with space)

Both mean the same thing, so we filter out both!

---

## 🎓 BONUS: The Most Complex Part Explained Simply

### The Variant Stock Matching Algorithm

This is arguably the **most complex logic** in the entire system. Let me break it down like telling a story.

**The Problem:**

-   Admin creates a T-shirt with Size and Color options
-   Admin enters stock as: `{"Red - S": 10, "Red - M": 15, "Blue - S": 20}`
-   User adds to cart: `{"size": "S", "color": "Red"}`

**The Challenge:**
How does the system know that `{"size": "S", "color": "Red"}` should match `"Red - S"`? They look completely different!

**The Solution (Fuzzy Matching):**

```php
// User selected: {"size": "S", "color": "Red"}
// We need to find: "Red - S" in stock

// Step 1: Extract just the values
$values = array_values($variantOptions);
// Result: ["S", "Red"]

// Step 2: Build ALL possible key combinations
$possibleKeys = [];

// Try: "s - red" (lowercase, original order)
$possibleKeys[] = strtolower(implode(' - ', $values));  // "s - red"

// Try: "S - Red" (as typed)
$possibleKeys[] = implode(' - ', $values);  // "S - Red"

// Try: REVERSED order (maybe admin typed Color first)
$reversedValues = array_reverse($values);  // ["Red", "S"]
$possibleKeys[] = strtolower(implode(' - ', $reversedValues));  // "red - s"
$possibleKeys[] = implode(' - ', $reversedValues);  // "Red - S"

// Now we have: ["s - red", "S - Red", "red - s", "Red - S"]
```

**Why This Matters:**

```
Without fuzzy matching:
  User selects: {"size": "S", "color": "Red"}
  System looks for: "S - Red" or "size: S, color: Red"
  Stock has: "Red - S"
  Result: ❌ "Out of stock!" (even though we have 10!)

With fuzzy matching:
  System tries: "s - red", "S - Red", "red - s", "Red - S"
  Stock has: "Red - S"
  Match found: "Red - S" ✓
  Result: ✅ "Added to cart!"
```

**Real-World Analogy:**

Imagine you're looking for your friend "John Smith" in a phonebook, but the phonebook lists him as "Smith, John". Without being clever, you'd never find him! The fuzzy matching is like being smart enough to try different name formats until you find a match.

---

## Quick Reference: File Locations

| Feature                 | File                              | Key Lines |
| ----------------------- | --------------------------------- | --------- |
| User Login              | `AuthController.php`              | 77-103    |
| Admin Login             | `AuthController.php`              | 193-223   |
| Registration            | `AuthController.php`              | 15-75     |
| OAuth Onboarding        | `OnboardingController.php`        | 42-126    |
| Cart Add                | `CartController.php`              | 137-294   |
| Checkout (Full)         | `CheckoutController.php`          | 17-406    |
| Direct Checkout         | `CheckoutController.php`          | 473-721   |
| Partial Checkout        | `CheckoutController.php`          | 841-1105  |
| Order Cancel            | `OrderController.php`             | 170-224   |
| Refund Request          | `OrderController.php`             | 412-456   |
| Admin Order Status      | `AdminOrderController.php`        | 61-126    |
| Refund Approve          | `AdminOrderController.php`        | 362-408   |
| Membership Approve      | `AdminController.php`             | 437-466   |
| Analytics               | `AdminController.php`             | 33-198    |
| Stock Sync              | `Merchandise.php`                 | 207-217   |
| **Messaging**           | `MessageController.php`           | 15-375    |
| **Favorites**           | `FavoriteController.php`          | 15-207    |
| **Preorder Queue**      | `PreorderController.php`          | 16-230    |
| **Broadcasting**        | `AdminNotificationController.php` | 49-125    |
| **Password Reset**      | `PasswordResetController.php`     | 17-192    |
| **Notification Filter** | `NotificationController.php`      | 12-103    |

---

_Generated for ORG MERCH University Merchandise Store - Complete System Documentation_
_Last Updated: December 26, 2024_
