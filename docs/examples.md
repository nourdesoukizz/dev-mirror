# Example Quiz Sessions

These examples show what a dev-mirror session looks like in practice.

---

## Session 1: Developer Passes

The developer has just implemented a caching layer. dev-mirror triggers.

**dev-mirror:**

> I have been reviewing what you shipped in the last few commits. Answer these:
>
> 1. In `lib/cache.ts`, you initialize the LRU cache with a `maxSize` of 500 entries at line 12. Why 500? What happens to performance if this value is too low for your traffic pattern?
>
> 2. `getCachedUser()` at line 34 falls through to a database call on cache miss, but you do not populate the cache after the DB read. Is this intentional? Where does cache population happen?
>
> 3. You added a `ttl` parameter to `setCacheEntry()` with a default of 300 seconds. The auth token in `middleware/auth.ts` expires at 3600 seconds. Why is the cache TTL 12x shorter than the token lifetime?

**Developer:**

> 1. 500 is based on our p95 active user count from the last month of logs. If it is too low, we get cache thrashing — hot entries get evicted before they are re-requested, and the cache becomes useless overhead. We would see DB query rates climb back to pre-cache levels.
>
> 2. Intentional. Cache population happens in the write path at `services/user.ts` line 78, inside `updateUser()`. We only cache after writes to avoid serving stale reads from a DB replica. The read path is always fresh.
>
> 3. Because the cache stores derived session data, not the token itself. If a user's permissions change mid-session, we want the cache to reflect that within 5 minutes. The token lifetime is irrelevant — it is the data freshness that matters.

**dev-mirror:**

> You own this code. Move on.

---

## Session 2: Developer Fails

The developer has just fixed a bug in the payment processing pipeline. dev-mirror triggers.

**dev-mirror:**

> I have been reviewing what you shipped in the last few commits. Answer these:
>
> 1. In `services/payment.ts`, you changed the `processRefund()` function at line 67. The original code called `validateRefundAmount()` before `checkRefundEligibility()`. You reversed that order. Why?
>
> 2. You added a try-catch around the Stripe API call at line 82, but the catch block only logs the error and returns `null`. What does the calling function in `routes/refunds.ts` do when it receives `null`? Walk me through the error propagation.
>
> 3. You removed the `REFUND_COOLDOWN_HOURS` check from `checkRefundEligibility()`. Where is that rate limiting enforced now?
>
> 4. The `calculateProration()` helper you added at line 103 divides `amount` by `daysRemaining`. What happens when `daysRemaining` is zero?

**Developer:**

> 1. I reversed it because... I think the eligibility check was faster so it made sense to run it first? To fail fast.
>
> 2. I am not sure what the calling function does with null actually. I think it probably returns a 500?
>
> 3. Oh, I did not realize there was a cooldown check. I just removed it because the tests were failing with it in place.
>
> 4. It would... divide by zero. I did not think about that case.

**dev-mirror:**

> You do not understand what you shipped. Here is what you need to go read:
>
> - `services/payment.ts` lines 60-110: You cannot explain why you reordered the validation calls or what happens on error. Read the entire `processRefund()` function and trace what happens when the Stripe call fails.
> - `routes/refunds.ts` lines 30-55: You do not know what happens when `processRefund()` returns `null`. Read the route handler and trace the response path.
> - `services/payment.ts` line 45: You removed a rate limiting check because tests failed. That is not a reason. Read what `REFUND_COOLDOWN_HOURS` was protecting against.
> - `services/payment.ts` line 103: `calculateProration()` has a division by zero. Fix it.
>
> I am not writing more code in the payment module until you can explain the refund flow end to end. Come back when you can.

---

## Session 3: Developer Re-attempts After Reading

The developer from Session 2 returns after reading the flagged code.

**Developer:**

> OK, I read through the payment module. Let me explain the refund flow.

**dev-mirror:**

> Go ahead. Explain why you reordered `validateRefundAmount()` and `checkRefundEligibility()`, and what happens when `processRefund()` returns null to the route handler.

**Developer:**

> The reorder was wrong. `validateRefundAmount()` needs to run first because `checkRefundEligibility()` depends on the validated amount to calculate the refund window. With them reversed, `checkRefundEligibility()` was operating on the raw, unvalidated input. I need to revert that order.
>
> When `processRefund()` returns null, the route handler at line 38 in `routes/refunds.ts` checks for null and returns a 400 with a generic "refund failed" message. But it does not distinguish between a Stripe API failure and a validation failure — both return null. That is a problem because the client cannot tell if they should retry or not.
>
> The `REFUND_COOLDOWN_HOURS` check prevents duplicate refund requests within 24 hours. I removed it because my test fixtures did not account for the cooldown, so existing refund records in the test data caused failures. I need to fix the test data, not remove the guard.

**dev-mirror:**

> You understand the refund flow now. Block lifted. Fix the validation order, the error discrimination, and restore the cooldown check.

---

## Session 4: Focused Quiz with Arguments

The developer runs `/dev-mirror error handling` to specifically test their understanding of error handling patterns.

**dev-mirror:**

> I have been reviewing error handling in your recent changes. Answer these:
>
> 1. In `middleware/errorHandler.ts`, you catch all errors at line 15 and format them using `formatErrorResponse()`. But `formatErrorResponse()` itself can throw if the error object has a circular reference. What happens then?
>
> 2. The `services/api.ts` file has three different error handling patterns: try-catch at line 22, `.catch()` at line 58, and error-first callback at line 91. Why three different patterns in the same file?
>
> 3. You added `isRetryableError()` at line 44 in `utils/errors.ts`. It checks for status codes 429, 502, 503, and 504. Why not 500?

This is what targeted quizzing looks like — deep and specific to the area you asked about.
