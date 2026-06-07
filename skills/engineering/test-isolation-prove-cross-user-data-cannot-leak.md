---
title: Test isolation: prove cross-user data cannot leak
category: engineering
tags: ['testing', 'security', 'safety']
source: Frappe community
---

For every user-scoped feature, write a test where User B tries to read User A's data and verify it gets 404 or 403.

## When
Any new feature that stores or retrieves user-specific data.

## Why
Authorization bugs are some of the most damaging in production — not because they are hard to fix, but because they are invisible until exploited. Unit tests for the happy path (User A reads User A's data) pass even when the authorization logic is completely absent.

The only test that proves authorization is actually working is a test where it should fail: User B attempts to access User A's data and is denied.

Note: return 404, not 403, for cross-user resource access. 403 confirms the resource exists and the caller doesn't have permission — which leaks information. 404 says "no such resource for you" without confirming existence.

## How
```js
// Test structure
describe('Todo authorization', () => {
  let userA, userB, todoId;

  beforeEach(async () => {
    // Create two independent users and one resource
    userA = await createTestUser('a@test.com');
    userB = await createTestUser('b@test.com');
    todoId = await createTodo(userA.id, { title: 'User A todo' });
  });

  it('User A reads own todo — 200', async () => {
    const res = await api.get(`/todos/${todoId}`, { auth: userA.token });
    expect(res.status).toBe(200);
  });

  it('User B reads User A todo — 404 (not 403)', async () => {
    const res = await api.get(`/todos/${todoId}`, { auth: userB.token });
    expect(res.status).toBe(404);  // NOT 403 — don't confirm existence
  });

  it('User B updates User A todo — 404', async () => {
    const res = await api.patch(`/todos/${todoId}`, { done: true }, { auth: userB.token });
    expect(res.status).toBe(404);
    // Also verify the DB row was NOT changed
    const todo = await getTodoFromDB(todoId);
    expect(todo.done).toBe(false);
  });
});
```

## Anti-pattern
Only testing the happy path. "I added user_id to the WHERE clause" — without a test that proves it, you don't know if the WHERE clause is actually being used.
