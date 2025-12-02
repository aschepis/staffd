# Rate Limit Handler Refactoring Suggestions

Low-hanging fruit refactorings to make `rate_limit.go` more idiomatic Go code.

## 1. String-based error checking is fragile

**Location:** `agent/rate_limit.go:27-39`

**Issue:** The `IsRateLimitError` function uses string matching, which is not idiomatic and fragile.

**Recommendation:** Use proper error wrapping with `fmt.Errorf("%w", err)` and check with `errors.Is()` or `errors.As()`, or create a custom error type that can be checked with type assertions.

**Example:**
```go
// Option 1: Use error wrapping at creation site
return fmt.Errorf("API request failed: %w", &RateLimitError{...})

// Then check with:
var rateLimitErr *RateLimitError
if errors.As(err, &rateLimitErr) {
    // handle rate limit
}

// Option 2: Check for specific sentinel error
var ErrRateLimit = errors.New("rate limit exceeded")
```

## 2. Redundant type assertion in CreateBackoff

**Location:** `agent/rate_limit.go:91-118`

**Issue:** Lines 97 and 107 create a `backoff.BackOff` interface then immediately cast it back to `*backoff.ExponentialBackOff`.

**Recommendation:** Work with the concrete type directly, then wrap at the end:

```go
func (h *RateLimitHandler) CreateBackoff(retryAfter time.Duration) backoff.BackOff {
    eb := backoff.NewExponentialBackOff()

    if retryAfter > 0 {
        eb.InitialInterval = retryAfter
        eb.Multiplier = 1.5
        eb.RandomizationFactor = 0.1
    } else {
        eb.InitialInterval = 1 * time.Second
        eb.Multiplier = 2.0
        eb.RandomizationFactor = 0.2
    }

    eb.MaxInterval = 5 * time.Minute
    eb.MaxElapsedTime = h.maxElapsedTime
    eb.Reset()

    return backoff.WithMaxRetries(eb, h.maxRetries)
}
```

## 3. Eliminate code duplication in CreateBackoff

**Location:** `agent/rate_limit.go:91-118`

**Issue:** The two branches differ only in a few parameter values, leading to duplication of common configuration.

**Recommendation:** See the refactored version in suggestion #2 above, which eliminates the duplication.

## 4. Extract magic numbers to constants

**Location:** `agent/rate_limit.go:66, 81-82`, and throughout

**Issue:** Magic numbers scattered throughout reduce maintainability.

**Recommendation:**
```go
const (
    DefaultRetryAfter     = 60 * time.Second
    DefaultMaxRetries     = 5
    DefaultMaxElapsedTime = 5 * time.Minute
    DefaultMaxInterval    = 5 * time.Minute
    DefaultInitialDelay   = 1 * time.Second
)
```

## 5. Simplify WaitForRetry

**Location:** `agent/rate_limit.go:185-195`

**Issue:** Creating and cleaning up a timer when `time.After` can be used directly.

**Recommendation:**
```go
func (h *RateLimitHandler) WaitForRetry(ctx context.Context, delay time.Duration) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(delay):
        return nil
    }
}
```

**Note:** The current implementation with `timer.Stop()` is more efficient for very long delays since `time.After` cannot be garbage collected until it fires. For rate limiting (typically seconds to minutes), this is unlikely to matter, but keep the current implementation if memory efficiency is critical.

## 6. Define callback type

**Location:** `agent/rate_limit.go:74`

**Issue:** Inline function signature reduces readability.

**Recommendation:**
```go
// RateLimitCallback is called when a rate limit is encountered
type RateLimitCallback func(agentID string, retryAfter time.Duration, attempt int) error

type RateLimitHandler struct {
    maxRetries      uint64
    maxElapsedTime  time.Duration
    stateManager    *StateManager
    onRateLimitFunc RateLimitCallback
    logger          zerolog.Logger
}
```

## 7. Questionable backoff usage pattern

**Location:** `agent/rate_limit.go:131-134`

**Issue:** Creating a new backoff instance on each call defeats the purpose of exponential backoff state. The backoff should track attempts across retries, but here it's recreated each time `HandleRateLimit` is called, resetting the state.

**Recommendation:** Consider one of these approaches:
- Store the backoff in the struct per agent (map[string]backoff.BackOff)
- Pass the backoff as a parameter to preserve state between retries
- If the current behavior is intentional, add a comment explaining why

**Example:**
```go
type RateLimitHandler struct {
    // ... existing fields ...
    agentBackoffs map[string]backoff.BackOff
    mu            sync.RWMutex
}

func (h *RateLimitHandler) getOrCreateBackoff(agentID string, retryAfter time.Duration) backoff.BackOff {
    h.mu.Lock()
    defer h.mu.Unlock()

    if b, exists := h.agentBackoffs[agentID]; exists {
        return b
    }

    b := h.CreateBackoff(retryAfter)
    h.agentBackoffs[agentID] = b
    return b
}
```

## 8. Variable shadowing

**Location:** `agent/rate_limit.go:52`

**Issue:** The `err` variable shadows the parameter. While legal, it's clearer to use a different name.

**Recommendation:**
```go
if seconds, parseErr := strconv.Atoi(retryAfterStr); parseErr == nil {
    return time.Duration(seconds) * time.Second
}
```

## Summary

Priority order for implementation:
1. Extract constants (easy, immediate readability improvement)
2. Eliminate code duplication in CreateBackoff
3. Define callback type
4. Fix variable shadowing
5. Consider backoff state management approach
6. Improve error checking (requires broader changes to error handling)
7. Simplify WaitForRetry (only if memory efficiency isn't critical)
