# 5. Algorithm

This section documents the core algorithms used in the Email Agent microservice.

## 5.1 Token Management Algorithm (Microsoft Graph)

### 5.1.1 Multi-Client Token Initialization and Decryption
The service supports multiple Graph client credentials (default and onboarding). Credentials are base64-encoded JSON with fields that are AES-encrypted. On startup, credentials are decoded and decrypted, and tokens are prefetched.

**Algorithm Steps:**
1. Decode `email.clients.info` and `email.onboarding.clients.info` from Base64
2. Decrypt `clientId`, `clientSecret`, `tenantId` using AES
3. Populate in-memory maps for regular and onboarding clients
4. Prefetch and cache access tokens for all configured clients

```java
@PostConstruct
public void init() {
  // Decode → decrypt → load clients → prefetch tokens
}
```

### 5.1.2 Access Token Retrieval with Expiry Buffer
Thread-safe access with per-tenant keys and proactive refresh using an expiry buffer.

**Algorithm Flow:**
1. Resolve `clientKey` from `UserContextHolder` (fallback to `default`)
2. If cached token missing or expired (now ≥ expiry - buffer), refresh
3. Return token for `clientKey`

```java
public synchronized String getAccessToken(UserContextHolder ctx) throws InterruptedException {
  String clientKey = Optional.ofNullable(ctx.getCurrentUser())
      .map(UserInfo::getCustomerName).orElse(TokenUtils.DEFAULT_CLIENT);
  TokenData t = tokenCache.get(clientKey);
  if (t == null || Instant.now().isAfter(t.expiryTime)) {
    refreshToken(clientKey);
  }
  return tokenCache.get(clientKey).accessToken();
}
```

### 5.1.3 Robust Token Fetch with Retry and Backoff
Obtain tokens via OAuth2 client credentials with bounded retries.

**Parameters:**
- Max retries: 5
- Delay: 5s between attempts
- Expiry buffer: Additional time added to configured `token.refresh.interval`

```java
private TokenData fetchNewAccessToken(String clientKey, boolean onboarding) throws InterruptedException {
  int retry = 0;
  while (retry < MAX_TOKEN_RETRIES) {
    try (CloseableHttpClient http = HttpClients.createDefault()) {
      // Build form request to /{tenantId}/oauth2/v2.0/token
      // On 2xx: parse access_token and set expiry = now + refreshInterval + buffer
      return parsedToken;
    } catch (Exception ex) {
      retry++;
      if (retry < MAX_TOKEN_RETRIES) Thread.sleep(TOKEN_RETRY_DELAY_MS);
      else throw new BusinessException("Failed after retries", ex);
    }
  }
  return null;
}
```

## 5.2 Availability and Slotting Algorithms

### 5.2.1 Working Hours Intersection (Multi-User)
Compute common UTC time ranges where all users are within working hours.

**Inputs:** list of users’ working hours and time zones (Graph format)

**Steps:**
1. Normalize Graph time zones → IANA (`GRAPH_API_TO_IANA_TIMEZONE`)
2. For each user/day: convert local [start, end] to UTC minute ranges
3. Merge all ranges per day and compute intersection via sweep line
4. Output `WorkingHoursSummary` with common UTC ranges per day

```java
// Sweep line:
for (range : ranges) timeline[range.start]++ ; timeline[range.end]--;
active += delta; when(active == userCount) start = minute; when(active < userCount) push [start, minute]
```

### 5.2.2 Free Slot Discovery Over Meetings
Given busy meetings across attendees, find continuous free slots of a given duration between a time window.

**Steps:**
1. Merge all attendees’ meetings → single list sorted by start time
2. Walk from window start; for each meeting, emit fixed-size slots that fit in the gap
3. Advance cursor to meeting end if it overlaps
4. After last meeting, keep emitting slots until window end

```java
List<Meeting> findAvailableTimeSlots(Map<String, List<Meeting>> meetings, Date from, Date till, long duration)
```

## 5.3 Date/Time Normalization Algorithms

### 5.3.1 Local → UTC Conversion (String Time)
Convert `yyyy-MM-dd'T'HH:mm:ss` in a given zone to UTC ISO with `Z`.

```java
String convertToUTCString(String date, String zone)
```

### 5.3.2 LocalTime → UTC Instant on Relative Day
Build UTC instant for a LocalTime on today or +N days in a zone.

```java
String convertToUTCString(LocalTime time, String zone, int daysToAdd)
```

### 5.3.3 Rounding to Next 30-min Boundary
Round a LocalDateTime to the next half hour or next hour.

```java
LocalDateTime roundOffTime(LocalDateTime t)
```

## 5.4 Retry, Fallback, and Degradation

### 5.4.1 Exponential Backoff (Service Calls)
Apply bounded exponential backoff for transient Graph/EWS failures.

- Attempts: 3 (typical), base: 1s, multiplier: 2x, max delay: 30s
- On non-retryable errors (4xx auth, validation), do not retry

### 5.4.2 Provider Fallback (Graph → EWS)
When Graph APIs are unavailable or rate limited:
1. Retry with backoff
2. If terminal failure, route operation to `EwsEventService`
3. Normalize responses to common DTOs
4. Audit fallback usage

## 5.5 Caching Strategy

### 5.5.1 Token Cache
- Per-tenant token maps for regular and onboarding clients
- Proactive refresh with expiry buffer
- Thread-safe `ConcurrentHashMap` and synchronized refresh

### 5.5.2 Preferences and Timezone Cache
- Cache `EmailPreferences` lookups per user for repeated operations
- Evict/update on user preference changes

## 5.6 User and Event Operations

### 5.6.1 Event Creation/Update Normalization
- Normalize time fields to UTC
- Default `meetingType` to Teams if absent
- Enrich with user’s `timeZone` from preferences when missing
- Persist minimal metadata for idempotency and traceability

### 5.6.2 License Assignment Flow
- After user creation, assign SKU IDs from `graph.license.skuIds`
- Retry transient errors; surface detailed messages on failure

## 5.7 Error Handling and Auditing

- Wrap external failures as `ApiException`/`BusinessException` with context
- Log structured details (operation, tenant, user, endpoint, status)
- Surface safe messages to REST callers; avoid leaking secrets

---

*These algorithms ensure the Email Agent delivers reliable scheduling, robust token handling, accurate time normalization across time zones, and resilient integrations with Microsoft Graph and EWS.*
