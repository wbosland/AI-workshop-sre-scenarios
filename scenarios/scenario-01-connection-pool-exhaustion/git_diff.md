# Git Diff: v2.4.0 → v2.4.1

## Commits in Release 2.4.1

### Commit 1: Add exponential backoff retry logic
**Author:** alex.rodriguez@company.com  
**Date:** 2024-07-06 10:30 UTC  
**Commit Hash:** a7f3d9e

**Change Description:** Add new retry logic with exponential backoff for transaction failures

```diff
--- a/src/payment/PaymentRetryHandler.java
+++ b/src/payment/PaymentRetryHandler.java
@@ -1,25 +1,85 @@
 public class PaymentRetryHandler {
 
-    private static final int MAX_RETRIES = 3;
-    private static final long RETRY_DELAY_MS = 1000;
+    private static final int MAX_RETRIES = 5;
+    private static final long INITIAL_BACKOFF_MS = 100;
+    private static final double BACKOFF_MULTIPLIER = 2.0;
 
-    public PaymentResult processTransaction(Transaction txn) {
-        for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
+    public PaymentResult processTransaction(Transaction txn) throws SQLException {
+        long backoffMs = INITIAL_BACKOFF_MS;
+        SQLException lastException = null;
+        
+        for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
             try {
-                return executePayment(txn);
-            } catch (SQLException e) {
-                if (attempt < MAX_RETRIES) {
-                    Thread.sleep(RETRY_DELAY_MS);
+                Connection conn = null;
+                try {
+                    conn = getConnection();
+                    return executePayment(txn, conn);
+                } catch (SQLException e) {
+                    if (attempt < MAX_RETRIES) {
+                        log.warn("Connection acquisition failed, retrying", 
+                                 "transaction_id", txn.getId(), 
+                                 "retry_count", attempt,
+                                 "backoff_ms", backoffMs);
+                        Thread.sleep(backoffMs);
+                        backoffMs = (long) (backoffMs * BACKOFF_MULTIPLIER);
+                        lastException = e;
+                    } else {
+                        throw e;
+                    }
+                }
             }
         }
-        throw new PaymentException("Max retries exceeded");
+        if (lastException != null) {
+            throw lastException;
+        }
+        throw new PaymentException("Max retries exceeded");
     }
-
-    private PaymentResult executePayment(Transaction txn) throws SQLException {
+    
-        // Implementation
+    private PaymentResult executePayment(Transaction txn, Connection conn) throws SQLException {
         // ...
     }
 }
```

**ISSUE IDENTIFIED:** 

⚠️ **Connection not explicitly closed in the retry loop**  
In the retry logic, when a `SQLException` is caught in the inner try-catch, the `Connection conn` variable goes out of scope but may not be properly closed if an exception occurs. The connection might be leaking because:

1. The `try-catch` block catches the exception
2. The code attempts a retry with a new `getConnection()` call
3. The old connection reference is lost but the connection pool may not immediately release it

---

### Commit 2: Update database connection pool configuration
**Author:** sarah.chen@company.com  
**Date:** 2024-07-06 14:15 UTC  
**Commit Hash:** b2e8c4f

**Change Description:** Optimize connection pool for high-volume scenarios

```diff
--- a/config/database.properties
+++ b/config/database.properties
-datasource.hikari.maximum-pool-size=50
-datasource.hikari.minimum-idle-connections=10
-datasource.hikari.connection-timeout=30000
-datasource.hikari.idle-timeout=600000
-datasource.hikari.max-lifetime=1800000
+datasource.hikari.maximum-pool-size=50
+datasource.hikari.minimum-idle-connections=5
+datasource.hikari.connection-timeout=30000
+datasource.hikari.idle-timeout=300000
+datasource.hikari.max-lifetime=900000
```

**Analysis:**
- Reduced `minimum-idle-connections` from 10 to 5
- Reduced `idle-timeout` from 600s to 300s (aggressive cleanup)
- Reduced `max-lifetime` from 1800s to 900s
- These settings alone wouldn't cause exhaustion, but combined with leaked connections become problematic

---

### Commit 3: Improve error handling for failed transactions
**Author:** michael.johnson@company.com  
**Date:** 2024-07-07 09:45 UTC  
**Commit Hash:** c1a5d2g

**Change Description:** Better error messages and logging

```diff
--- a/src/payment/PaymentService.java
+++ b/src/payment/PaymentService.java
@@ -45,7 +45,15 @@
     public ResponseEntity<?> processPayment(PaymentRequest req) {
         try {
             PaymentResult result = paymentHandler.processTransaction(txn);
             return ResponseEntity.ok(result);
         } catch (SQLException e) {
+            log.error("Database error processing payment",
+                    "error_code", e.getErrorCode(),
+                    "message", e.getMessage(),
+                    "sql_state", e.getSQLState());
+            
+            // Note: Retry logic already in PaymentRetryHandler,
+            // this catch block handles final failures
             return ResponseEntity.status(503).body(new ErrorResponse(...));
         } catch (Exception e) {
             log.error("Unexpected error", e);
```

---

## Root Cause Analysis

### The Problem

The combination of:

1. **Connection Leak in Retry Logic** (Commit 1)
   - When `getConnection()` throws `SQLException`, the code catches it and retries
   - If a connection was partially allocated before the exception, it may not be properly closed
   - This is exacerbated by the new retry logic which tries up to 5 times instead of 3

2. **Aggressive Connection Cleanup Settings** (Commit 2)
   - Reduced `idle-timeout` and `max-lifetime` mean connections are cleaned up faster
   - But this doesn't happen fast enough to prevent exhaustion
   - Leaked connections prevent proper cleanup

3. **High Load Scenario**
   - Campaign launch increases traffic 1.5x
   - Normal retry rate: ~200/min
   - With new logic: ~400/min (due to 5 retries vs 3)
   - Each retry attempt holds connections longer
   - Leaked connections accumulate

### Why Health Checks Passed

- Health checks run at normal traffic levels
- They don't stress-test the retry logic
- The leak is subtle and only manifests under high load with multiple concurrent failures

### Why the Failure Cascaded

1. First instance exhausts connections → 503 errors begin
2. Clients see errors → application-level retries kick in
3. More retries = more connection attempts
4. Other instances receive traffic surge
5. All instances hit pool exhaustion simultaneously
6. Deadlocks occur as connections are held longer
7. System completely frozen

---

## The Fix

The issue is in `PaymentRetryHandler.java`. The connection should be properly managed:

```java
public PaymentResult processTransaction(Transaction txn) throws SQLException {
    long backoffMs = INITIAL_BACKOFF_MS;
    SQLException lastException = null;
    
    for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
        Connection conn = null;
        try {
            conn = getConnection();
            return executePayment(txn, conn);
        } catch (SQLException e) {
            if (conn != null) {
                conn.close();  // ← CRITICAL: Close on error
            }
            if (attempt < MAX_RETRIES) {
                Thread.sleep(backoffMs);
                backoffMs = (long) (backoffMs * BACKOFF_MULTIPLIER);
                lastException = e;
            } else {
                throw e;
            }
        }
    }
}
```

**Better approach:** Use try-with-resources:

```java
for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
    try (Connection conn = getConnection()) {
        return executePayment(txn, conn);
    } catch (SQLException e) {
        if (attempt < MAX_RETRIES) {
            Thread.sleep(backoffMs);
            backoffMs = (long) (backoffMs * BACKOFF_MULTIPLIER);
            lastException = e;
        } else {
            throw e;
        }
    }
}
```

The `try-with-resources` ensures the connection is always closed, even if an exception occurs.