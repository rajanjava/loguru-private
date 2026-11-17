# Issue Resolution: BaseExceptionGroup members are missing from exception output

## Metadata
- **Target Repository**: `Delgan/loguru`
- **Private Repository**: `rajanjava/loguru-private`
- **Upstream Issue**: [#1542: BaseExceptionGroup members are missing from exception output](https://github.com/Delgan/loguru/issues/1542)
- **Status**: Resolved & Verified

---

## Problem Description & Root Cause
In `loguru._better_exceptions`, `is_exception_group(exc)` was defined as:
```python
if sys.version_info >= (3, 11):
    def is_exception_group(exc):
        return isinstance(exc, ExceptionGroup)
else:
    try:
        from exceptiongroup import ExceptionGroup
    except ImportError:
        def is_exception_group(exc):
            return False
    else:
        def is_exception_group(exc):
            return isinstance(exc, ExceptionGroup)
```

In Python 3.11+, `ExceptionGroup` inherits from `BaseExceptionGroup`. When an exception group contains non-`Exception` subclasses (such as `KeyboardInterrupt`, `asyncio.CancelledError`, `SystemExit`, or `GeneratorExit`), Python constructs an instance of `BaseExceptionGroup` rather than `ExceptionGroup`.

Because `is_exception_group` only checked `isinstance(exc, ExceptionGroup)`:
- `is_exception_group(exc)` returned `False` for any `BaseExceptionGroup` containing non-`Exception` errors.
- As a result, `ExceptionFormatter._format_exception()` fell back to formatting the exception as a regular single exception via `traceback.format_exception_only`. It printed `BaseExceptionGroup: mixed failures (2 sub-exceptions)` and completely skipped iterating through `value.exceptions` (lines 526–553).
- The sub-exceptions and their tracebacks were completely omitted from the log output.

---

## Solution
Update `is_exception_group(exc)` to check `BaseExceptionGroup` instead of `ExceptionGroup`, both for Python 3.11+ built-in support and the Python < 3.11 `exceptiongroup` backport:
```python
if sys.version_info >= (3, 11):
    def is_exception_group(exc):
        return isinstance(exc, BaseExceptionGroup)
else:
    try:
        from exceptiongroup import BaseExceptionGroup
    except ImportError:
        def is_exception_group(exc):
            return False
    else:
        def is_exception_group(exc):
            return isinstance(exc, BaseExceptionGroup)
```
Since `ExceptionGroup` is a subclass of `BaseExceptionGroup`, `isinstance(exc, BaseExceptionGroup)` correctly matches both `BaseExceptionGroup` and `ExceptionGroup`.

---

## Code Changes
```diff
diff --git a/loguru/_better_exceptions.py b/loguru/_better_exceptions.py
index 3bb53ab..9854a16 100644
--- a/loguru/_better_exceptions.py
+++ b/loguru/_better_exceptions.py
@@ -13,11 +13,11 @@ import traceback
 if sys.version_info >= (3, 11):
 
     def is_exception_group(exc):
-        return isinstance(exc, ExceptionGroup)
+        return isinstance(exc, BaseExceptionGroup)
 
 else:
     try:
-        from exceptiongroup import ExceptionGroup
+        from exceptiongroup import BaseExceptionGroup
     except ImportError:
 
         def is_exception_group(exc):
@@ -26,7 +26,7 @@ else:
     else:
 
         def is_exception_group(exc):
-            return isinstance(exc, ExceptionGroup)
+            return isinstance(exc, BaseExceptionGroup)
```

---

## Verification
- Added reproduction test `test_base_exception_group_members_formatted` in `tests/test_exceptions_formatting.py` raising a `BaseExceptionGroup` with mixed errors (`ValueError` and `KeyboardInterrupt`).
- Verified failure prior to fix:
  `AssertionError: assert 'ValueError: data failed' in ... BaseExceptionGroup: mixed failures (2 sub-exceptions)`
- Verified pass after fix:
  `101 passed, 3 skipped in 14.58s`
- Verified `ruff check` on modified files:
  `All checks passed!`
