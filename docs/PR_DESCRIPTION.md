# Tier 1: Bug Hunting & Critical Fixes

## 1. JWT Expiration Ignored
- **Location**: `backend/app/core/security.py`, line 56
- **Severity**: Critical
- **Reason**: The option `{"verify_exp": False}` is explicitly passed to `jwt.decode()`, meaning expired tokens are still considered valid. This is a severe security flaw that defeats the purpose of token expiration.
- **Fix Proposal**: Remove the `options={"verify_exp": False}` argument so that the JWT library automatically verifies the expiration time.

## 2. Cross-User Data Leak in Cache
- **Location**: `backend/app/api/v1/todos.py`, line 37
- **Severity**: Critical
- **Reason**: The Redis cache key is statically set to `"todos:list"`. This means if User A fetches their todos, the response is cached under this key. If User B subsequently fetches their todos, they will receive User A's cached data.
- **Fix Proposal**: Scope the cache key by the user ID: `cache_key = f"todos:list:{current_user.id}"`.

## 3. Missing Authorization Check on Todos
- **Location**: `backend/app/api/v1/todos.py`, in `get_todo`, `update_existing_todo`, and `delete_existing_todo`
- **Severity**: High
- **Reason**: The API endpoints fetch a todo by ID but fail to verify if the fetched todo belongs to the authenticated user. This allows any user to read, modify, or delete any other user's todos by guessing the UUID.
- **Fix Proposal**: Add a check: `if todo.user_id != current_user.id: raise HTTPException(status_code=403, detail="Not authorized")` after fetching the todo from the database.

## 4. Cannot Toggle Todo Completion to False
- **Location**: `backend/app/api/v1/todos.py`, line 123
- **Severity**: Medium
- **Reason**: The update logic uses `if todo_data.completed: todo.completed = todo_data.completed`. In Python, `False` is falsy, so if the frontend sends `{ completed: false }`, the `if` block is skipped and the todo cannot be unchecked.
- **Fix Proposal**: Change the condition to explicitly check for `None`: `if todo_data.completed is not None:`.

## 5. Missing Cache Invalidation on Mutations
- **Location**: `backend/app/api/v1/todos.py`, `create_new_todo`, `update_existing_todo`, `delete_existing_todo`
- **Severity**: Medium
- **Reason**: When a user creates, updates, or deletes a todo, the Redis cache for their todo list is not cleared. They will see stale data until the cache TTL (5 minutes) expires.
- **Fix Proposal**: Add `await redis.delete(f"todos:list:{current_user.id}")` after successful mutations.

## 6. Frontend: Missing Optimistic Update Rollback
- **Location**: `frontend/src/features/todos/api/todos.ts`, line 95
- **Severity**: Low
- **Reason**: The `useUpdateTodo` hook performs an optimistic update in `onMutate`, but fails to roll back the state in `onError` if the API request fails. This leaves the UI in an inconsistent state compared to the backend.
- **Fix Proposal**: Implement rollback in `onError` using the `previousTodos` context returned from `onMutate`.

## 7. Secret Leak Risk (.env not ignored)
- **Location**: `.gitignore`, line 9
- **Severity**: High
- **Reason**: The `.env` entry is commented out (`# .env`), meaning environment variables and secrets could accidentally be committed and pushed to the repository.
- **Fix Proposal**: Uncomment `.env` in the `.gitignore` file.

## 8. Documentation Files Ignored
- **Location**: `.gitignore`, line 35
- **Severity**: Medium
- **Reason**: The `docs/` directory is explicitly ignored, which prevents tracking necessary architectural or specification documents (like `TODO_SHARING_SPEC.md` required in Tier 3).
- **Fix Proposal**: Remove the `docs/` entry from `.gitignore`.
