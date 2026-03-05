# Dependency Injection (DI)

**Dependency Injection** is a design pattern where a class receives its dependencies from the outside rather than creating them itself. It's a specific implementation of the **Dependency Inversion Principle (DIP)** from SOLID.

---

## What Could Go Wrong with DI?

### 1. Constructor Over-Injection

When a class has too many injected dependencies, it's a sign that the class is doing too much — violating the **Single Responsibility Principle**.

```php
<?php

class OrderService {
    public function __construct(
        private UserRepository $users,
        private ProductRepository $products,
        private PaymentGateway $payment,
        private ShippingService $shipping,
        private NotificationService $notification,
        private TaxCalculator $tax,
        private DiscountService $discount,
        private Logger $logger,
        private CacheManager $cache,
        private EventDispatcher $events
    ) {}
}
```

**Problem:** 10 dependencies = the class is doing too much.
**Fix:** Break it into smaller, focused classes.

---

### 2. Captive Dependency (Wrong Lifetime/Scope)

Injecting a short-lived (request-scoped) dependency into a long-lived (singleton) one. The short-lived object becomes "captive" — it never refreshes.

```php
<?php

// ❌ Bad — singleton holds a request-scoped object
class ReportGenerator { // Registered as singleton
    public function __construct(
        private CurrentUser $currentUser // Request-scoped
    ) {}

    public function generate() {
        // $this->currentUser is STALE after the first request!
        // Every request sees the SAME user from when the singleton was created
        return "Report for: " . $this->currentUser->name();
    }
}
```

**Fix:** Inject a factory or resolver instead:

```php
<?php

// ✅ Good — resolve the dependency fresh each time
class ReportGenerator {
    public function __construct(
        private Closure $currentUserResolver
    ) {}

    public function generate() {
        $currentUser = ($this->currentUserResolver)();
        return "Report for: " . $currentUser->name();
    }
}
```

---

### 3. Circular Dependencies

A depends on B, B depends on A — causes infinite loops or resolution failures.

```php
<?php

// ❌ Bad — circular dependency
class AuthService {
    public function __construct(private UserService $userService) {}
}

class UserService {
    public function __construct(private AuthService $authService) {}
}

// Container tries to create AuthService → needs UserService → needs AuthService → 💥 infinite loop
```

**Fix:** Extract the shared logic into a third class, or use an interface to break the cycle.

```php
<?php

// ✅ Good — break the cycle with a third class
class AuthService {
    public function __construct(private UserRepository $userRepo) {}
}

class UserService {
    public function __construct(private UserRepository $userRepo) {}
}
```

---

### 4. Service Locator Anti-Pattern

Injecting the **entire container** instead of specific dependencies. This hides what the class actually needs.

```php
<?php

// ❌ Bad — depends on the entire container
class OrderService {
    public function __construct(private Container $container) {}

    public function process(int $orderId) {
        $repo = $this->container->get(OrderRepository::class);
        $payment = $this->container->get(PaymentGateway::class);
        // Dependencies are hidden — you can't tell from the constructor what this class needs
    }
}
```

```php
<?php

// ✅ Good — explicit dependencies
class OrderService {
    public function __construct(
        private OrderRepository $repo,
        private PaymentGateway $payment
    ) {}

    public function process(int $orderId) {
        // Dependencies are clear from the constructor
    }
}
```

---

### 5. Ambient Context (The Silent Killer)

Relying on global/static state instead of explicit injection. Works in some contexts, silently fails in others.

```php
<?php

// ❌ Bad — reads company from HTTP request implicitly
class InvoiceService {
    public function getInvoices() {
        // This global scope reads from the HTTP request header
        // Works in controllers, SILENTLY RETURNS EMPTY in jobs/queues!
        return Invoice::all(); // Global scope filters by company from request
    }
}
```

```php
<?php

// ✅ Good — explicit company ID parameter
class InvoiceService {
    public function getInvoices(int $companyId) {
        return Invoice::withoutGlobalScope(CompanyScope::class)
            ->where('company_id', $companyId)
            ->get();
    }
}
```

**This is the #1 source of silent bugs in Laravel apps with multi-tenancy.** The code works perfectly in HTTP context but silently returns empty results in queues, jobs, and artisan commands.

---

### 6. Late Resolution Failures

DI errors only appear at **runtime** when the class is first resolved — not at startup. You might not discover missing bindings until a specific feature is used in production.

```php
<?php

// This interface has no binding registered
interface PaymentGateway {
    public function charge(float $amount): bool;
}

class OrderController {
    public function store(PaymentGateway $gateway) {
        // 💥 Runtime error: "Target [PaymentGateway] is not instantiable"
        // Only fails when someone actually hits this endpoint
    }
}
```

**Fix:** Register all bindings explicitly and write tests that resolve key services:

```php
<?php

// In a test
public function test_all_services_can_be_resolved()
{
    $this->assertInstanceOf(PaymentGateway::class, app(PaymentGateway::class));
}
```

---

## Summary Table

| Problem | Symptom | Fix |
|:--------|:--------|:----|
| **Constructor over-injection** | 5+ dependencies in constructor | Split class into smaller ones (SRP) |
| **Captive dependency** | Stale data after first request | Inject factory/resolver instead |
| **Circular dependency** | Infinite loop or resolution error | Extract shared logic into third class |
| **Service locator** | Hidden dependencies, hard to test | Inject explicit dependencies |
| **Ambient context** | Silent failures in non-HTTP context | Pass context explicitly as parameters |
| **Late resolution** | Runtime errors in production | Write resolution tests, register bindings early |
