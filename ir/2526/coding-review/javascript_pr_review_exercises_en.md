# JavaScript Pull Request Review Exercises

This document contains 5 JavaScript code review exercises. Each exercise simulates a **Pull Request that modifies a single file**.

The goal is to review each snippet as if it had been submitted by another engineer on the team: identify bugs, design issues, edge cases, readability problems, performance concerns, async pitfalls, or maintainability issues.

The exercises are ordered from lower to higher complexity.

---

# 1. Shopping Cart Summary

## Context

This PR adds a function that calculates a shopping cart summary.

Each product has a name, price, and quantity. The function should calculate the subtotal, apply an optional discount, and also return the total number of purchased units.

The code is simple, but there are several details worth reviewing around validation, data types, and edge cases.

### File: `cart.js`

```javascript
function calculateCartSummary(items, discountPercent) {
  let subtotal = 0;
  let totalItems = 0;

  for (let i = 0; i < items.length; i++) {
    const item = items[i];

    subtotal += item.price * item.quantity;
    totalItems += item.quantity;
  }

  let discount = 0;

  if (discountPercent) {
    discount = subtotal * (discountPercent / 100);
  }

  const total = subtotal - discount;

  return {
    subtotal: subtotal.toFixed(2),
    discount: discount.toFixed(2),
    total: total.toFixed(2),
    totalItems,
  };
}

const cart = [
  {
    name: "Keyboard",
    price: 79.99,
    quantity: 1,
  },
  {
    name: "Mouse",
    price: 39.5,
    quantity: 2,
  },
];

console.log(calculateCartSummary(cart, 10));

module.exports = {
  calculateCartSummary,
};
```

## Interesting Review Points

Even though the code looks straightforward, there are several useful review questions:

- `toFixed()` returns a **string**, not a number. This may cause unexpected behavior if another part of the application tries to perform arithmetic with those fields.
- There is no validation for `items`, `price`, `quantity`, or `discountPercent`.
- A negative discount or a discount greater than 100 would probably produce invalid results.
- The condition `if (discountPercent)` mixes the idea of “value is defined” with “value is truthy”.
- Floating-point arithmetic may introduce small precision errors when dealing with money.
- The `console.log` lives in the same file that is exported as a module, which creates a side effect when the file is imported.

---

# 2. Grouping Orders by Customer

## Context

This PR adds a function used by a small reporting system.

The function receives a list of orders and builds a customer summary containing the number of orders, total amount spent, and the date of the most recent order.

The implementation works for the sample data, but there are decisions around mutability, sorting, and date handling that should be reviewed.

### File: `orders.js`

```javascript
function buildCustomerReport(orders) {
  const report = {};

  orders.sort((a, b) => {
    return new Date(a.createdAt) - new Date(b.createdAt);
  });

  for (const order of orders) {
    if (!report[order.userId]) {
      report[order.userId] = {
        userId: order.userId,
        orders: 0,
        totalSpent: 0,
        lastOrderAt: null,
      };
    }

    const customer = report[order.userId];

    customer.orders += 1;
    customer.totalSpent += order.total;
    customer.lastOrderAt = order.createdAt;
  }

  const result = [];

  for (const userId in report) {
    const customer = report[userId];

    customer.totalSpent = Number(
      customer.totalSpent.toFixed(2)
    );

    result.push(customer);
  }

  return result;
}

const orders = [
  {
    id: "o-1",
    userId: "u-1",
    total: 25.5,
    createdAt: "2026-04-10T10:00:00Z",
  },
  {
    id: "o-2",
    userId: "u-2",
    total: 12,
    createdAt: "2026-04-10T11:00:00Z",
  },
  {
    id: "o-3",
    userId: "u-1",
    total: 30,
    createdAt: "2026-04-12T09:00:00Z",
  },
];

console.log(buildCustomerReport(orders));
```

## Interesting Review Points

This example already introduces a few less obvious concerns:

- `orders.sort(...)` **mutates the original array** passed into the function. This may surprise code that calls `buildCustomerReport`.
- The algorithm relies on ascending order to determine `lastOrderAt`, but that requirement is not explicit.
- Objects are created inside `report` and then mutated again before being returned.
- A plain object is used as an index by `userId`; in some cases, a `Map` may be more appropriate.
- Repeatedly calling `new Date(...)` inside the comparator may become expensive for large collections.
- Invalid dates are not handled.
- There is no check that `total` is actually a number.
- The function combines multiple responsibilities: sorting, grouping, aggregating, rounding, and transforming the final result.

---

# 3. Loading User Profiles from an API

## Context

This PR introduces a function for loading multiple user profiles from an HTTP API.

The goal is to request all profiles in parallel and return only those that were successfully retrieved.

This example introduces common asynchronous programming concerns: error handling, concurrency limits, and the behavior of `Promise.all`.

### File: `profiles.js`

```javascript
async function fetchProfile(userId) {
  const response = await fetch(
    `https://api.example.com/users/${userId}`
  );

  if (!response.ok) {
    throw new Error(
      `Could not load user ${userId}: ${response.status}`
    );
  }

  return response.json();
}

async function loadProfiles(userIds) {
  const requests = userIds.map(async (userId) => {
    try {
      const profile = await fetchProfile(userId);

      return {
        id: profile.id,
        name: profile.name,
        email: profile.email,
      };
    } catch (error) {
      console.error(
        "Error loading profile",
        userId,
        error.message
      );

      return null;
    }
  });

  const profiles = await Promise.all(requests);

  return profiles.filter(Boolean);
}

async function main() {
  const ids = [
    "u-100",
    "u-101",
    "u-102",
    "u-103",
    "u-104",
  ];

  const profiles = await loadProfiles(ids);

  console.log(
    `Loaded ${profiles.length} profiles`
  );

  console.log(profiles);
}

main();
```

## Interesting Review Points

This snippet introduces several important async design decisions:

- All requests start almost at the same time. With 5 users this is fine, but with thousands of IDs it could overload both the client and the server.
- Errors are silently converted into `null`, which means the caller loses information about which users failed.
- `console.error` is mixed directly into domain logic. In a real application, a logging abstraction or error propagation may be preferable.
- There is no timeout for requests.
- HTTP errors, network errors, and invalid JSON responses are not treated differently.
- `userIds` are not deduplicated, so the same user may trigger multiple identical requests.
- The function depends on the global `fetch`, which may make unit testing more difficult.
- `main()` runs automatically when the file is imported, creating a side effect.

---

# 4. Async Result Cache with Expiration

## Context

This PR adds a small in-memory cache to avoid repeating expensive calls to a `loader` function.

Each value should remain cached for a configurable amount of time (`ttlMs`). Once the value expires, the loader runs again.

The implementation looks compact, but it introduces more subtle problems around concurrency, rejected promises, and memory growth.

### File: `async-cache.js`

```javascript
class AsyncCache {
  constructor(ttlMs = 5000) {
    this.ttlMs = ttlMs;
    this.entries = new Map();
  }

  async get(key, loader) {
    const now = Date.now();
    const cached = this.entries.get(key);

    if (cached && cached.expiresAt > now) {
      return cached.value;
    }

    const value = await loader();

    this.entries.set(key, {
      value,
      expiresAt: Date.now() + this.ttlMs,
    });

    return value;
  }

  has(key) {
    const entry = this.entries.get(key);

    if (!entry) {
      return false;
    }

    return entry.expiresAt > Date.now();
  }

  delete(key) {
    return this.entries.delete(key);
  }

  clear() {
    this.entries.clear();
  }

  size() {
    return this.entries.size;
  }
}

async function loadProduct(productId) {
  const response = await fetch(
    `https://api.example.com/products/${productId}`
  );

  if (!response.ok) {
    throw new Error(`Product not found: ${productId}`);
  }

  return response.json();
}

const cache = new AsyncCache(10_000);

async function getProduct(productId) {
  return cache.get(productId, () => loadProduct(productId));
}

module.exports = {
  AsyncCache,
  getProduct,
};
```

## Interesting Review Points

This example contains concurrency issues commonly seen in real systems:

- If two calls execute `get("abc")` at the same time when the value is not cached, both will execute `loader()`. This is sometimes called a **cache stampede** or *thundering herd*.
- One common solution is to temporarily cache the `Promise` itself while the value is being loaded.
- `size()` also counts expired entries because they are never automatically removed.
- Expired entries remain in memory indefinitely unless they are queried again or `clear()` is called manually.
- The TTL starts counting **after** `loader()` finishes. Depending on the intended contract, this may or may not be correct.
- There is no validation that `ttlMs` is positive.
- There is also no validation that `loader` is a function.
- If Promises are cached, rejected Promises require special care: a failed request should not remain cached forever.
- The class mixes storage policy, expiration behavior, and loading strategy; in larger systems these responsibilities are sometimes separated.

---

# 5. Concurrent Job Processor with Retries

## Context

This PR adds a job processor that receives a list of jobs and a `handler` function.

The goal is to execute several jobs in parallel, limit the maximum number of concurrent operations, and automatically retry failed jobs.

This example is complex enough to require a careful review: worker coordination, retries, result consistency, error handling, and shared state all come into play.

### File: `job-runner.js`

```javascript
async function runJobs(
  jobs,
  handler,
  options = {}
) {
  const concurrency = options.concurrency || 3;
  const maxRetries = options.maxRetries || 2;

  const queue = [...jobs];

  const results = [];
  const failures = [];

  async function worker(workerId) {
    while (queue.length > 0) {
      const job = queue.shift();

      if (!job) {
        return;
      }

      let attempt = 0;
      let completed = false;

      while (
        attempt <= maxRetries &&
        !completed
      ) {
        try {
          const value = await handler(
            job,
            {
              workerId,
              attempt,
            }
          );

          results.push({
            jobId: job.id,
            value,
          });

          completed = true;
        } catch (error) {
          attempt += 1;

          if (attempt > maxRetries) {
            failures.push({
              jobId: job.id,
              error: error.message,
            });
          } else {
            await new Promise((resolve) =>
              setTimeout(
                resolve,
                100 * attempt
              )
            );
          }
        }
      }
    }
  }

  const workers = [];

  for (
    let i = 0;
    i < concurrency;
    i++
  ) {
    workers.push(worker(i + 1));
  }

  await Promise.all(workers);

  return {
    results,
    failures,
  };
}

module.exports = {
  runJobs,
};
```

## Interesting Review Points

This code raises several design questions:

- `options.concurrency || 3` and `options.maxRetries || 2` silently replace values such as `0` with the defaults.
- There is no validation for `concurrency`. Negative, fractional, or extremely high values could result in unexpected behavior.
- Results are appended with `push`, so their order depends on which worker finishes first. The final result does not preserve the original order of `jobs`.
- The queue is shared through `queue.shift()`. In JavaScript running on a single event loop, this does not create a classic memory race, but it is still shared mutable state across multiple async workers.
- `Array.prototype.shift()` has linear cost on large arrays because remaining elements must be reindexed.
- The `attempt` counter can be confusing: the first execution runs with `attempt = 0`.
- Retry delay uses a very basic linear backoff and has no *jitter*. In distributed systems, many workers could retry at exactly the same time.
- The code assumes `error` is an `Error` instance with a `message` property.
- There is no cancellation support through `AbortSignal`.
- If the `handler` hangs forever, one of the workers will also remain blocked forever.
- Failures are accumulated and returned instead of causing `runJobs` itself to reject. That may be a valid design, but it should be explicitly defined as part of the function contract.
- The design does not distinguish between retryable and non-retryable errors. For example, a validation error probably should not be retried.
- The result does not include enough information to observe how long each job took, how many attempts it required, or which worker ultimately completed it.
- For very large workloads, copying all jobs into `queue` may be less desirable than using a shared index or an iterable/asynchronous source.

---

# Suggested Usage

One way to work through these exercises is to review each example as if it were a real Pull Request and write review comments grouped by severity:

- **Bug:** incorrect behavior or an edge case that produces wrong results.
- **Maintainability:** code that works but will be difficult to understand or modify.
- **Performance:** operations that may degrade at larger scale.
- **Reliability:** issues involving errors, retries, timeouts, or concurrency.
- **Nit:** small improvements related to style or clarity.

It can also be useful to propose a corrected implementation after completing each review.
