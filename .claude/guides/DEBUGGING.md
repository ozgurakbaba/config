# Systematic Debugging Guide

## Core Principle

**ALWAYS find the root cause. NEVER fix symptoms or add workarounds.**

---

## The Process

### 1. Reproduce
Create a minimal reproduction case that consistently triggers the issue.

**Questions to answer:**
- What are the exact steps to reproduce?
- Does it happen every time or intermittently?
- What's the smallest code/data set that triggers it?

---

### 2. Hypothesize
Form testable hypotheses about the root cause.

**Good hypothesis characteristics:**
- Specific and falsifiable
- Based on evidence (logs, error messages, stack traces)
- Explains all symptoms, not just some

**Example:**
- Bad: "Something's wrong with the database"
- Good: "The connection pool is exhausted because we're not closing connections in error cases"

---

### 3. Test
Systematically test each hypothesis.

**Testing methods:**
- Add logging at key points
- Use debugger breakpoints
- Isolate components (comment out code, use mocks strategically)
- Check assumptions (data types, null checks, API responses)

**Important:** Test one variable at a time.

---

### 4. Verify
Confirm the fix resolves the root cause, not just symptoms.

**Verification checklist:**
- [ ] Issue no longer reproduces with original steps
- [ ] Fix doesn't introduce new issues
- [ ] All related tests pass
- [ ] Edge cases are handled
- [ ] Logs are clean (no unexpected errors/warnings)

---

### 5. Document
Record findings in journal for future reference.

**What to document:**
- Root cause analysis
- Why the issue occurred
- How it was fixed
- What was learned
- Related issues to watch for

---

## When You're Stuck

**STOP and ask Ozgur for help when:**
- You've spent 30+ minutes without progress
- The issue involves unfamiliar systems/code
- You need domain knowledge or business context
- Multiple hypotheses seem equally plausible
- The fix would require significant architectural changes

---

## Common Anti-Patterns

### ❌ Symptom Fixing
```typescript
// Bad: Hiding the symptom
if (data === undefined) {
  return []; // Why is it undefined? Fix the source!
}

// Good: Fix the root cause
// After investigation: data is undefined because API call fails silently
// Fix: Add proper error handling to API client
```

### ❌ Cargo Cult Debugging
```typescript
// Bad: Random changes without understanding
setTimeout(() => fetchData(), 100); // "It worked once, so..."

// Good: Understand the race condition
await Promise.all([fetchUsers(), fetchPosts()]); // Properly wait for both
```

### ❌ Assumption-Based Debugging
```typescript
// Bad: Assuming without verification
console.log('User should be logged in here');

// Good: Verify assumptions
console.log('User auth state:', { isAuthenticated, userId, token: !!token });
```

---

## Debugging Tools

### Logging Best Practices
```typescript
// Include context in logs
console.log('Fetching user data', { userId, timestamp: Date.now() });

// Log both input and output
console.log('API request:', { endpoint, params });
const response = await api.get(endpoint, params);
console.log('API response:', { status: response.status, data: response.data });

// Use structured logging
logger.error('Payment failed', {
  userId,
  orderId,
  amount,
  error: error.message,
  stack: error.stack,
});
```

### Browser DevTools
- **Network tab:** Inspect API calls, check timing, view payloads
- **Console:** Check for errors, warnings, logs
- **Sources/Debugger:** Set breakpoints, step through code
- **React DevTools:** Inspect component props/state
- **Performance tab:** Profile rendering, identify bottlenecks

---

## Debugging Checklist

When debugging, systematically check:

- [ ] **Error messages:** Read the full error, including stack trace
- [ ] **Logs:** Check console, network tab, server logs
- [ ] **Data:** Verify data types, null/undefined, unexpected values
- [ ] **Timing:** Race conditions, async/await issues, event ordering
- [ ] **Environment:** Check env vars, build mode (dev/prod), feature flags
- [ ] **Dependencies:** Version conflicts, breaking changes, peer deps
- [ ] **Browser/Platform:** Cross-browser issues, mobile vs desktop
- [ ] **State:** Component state, global state, cache state
- [ ] **Side effects:** Unintended mutations, closure scope issues
