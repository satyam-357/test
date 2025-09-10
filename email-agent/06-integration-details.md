# 6. Integration Details

This section documents the integration details of the Email Agent microservice.

## 6.1 Microsoft Graph Integration

### 6.1.1 Service Communication
The service integrates with Microsoft Graph for modern calendar and user management operations.

**Integration Points:**
- **Calendar Management**: Create, update, cancel, forward events; get events and details
- **Availability**: Use `getSchedule` to compute availability and conflicts
- **User Management**: Create users, update user details, assign licenses
- **Mailbox Settings**: Configure automatic replies (OOO)

**Graph Endpoints (illustrative):**
```http
POST https://graph.microsoft.com/v1.0/users/{userId}/events
PATCH https://graph.microsoft.com/v1.0/users/{userId}/events/{eventId}
DELETE https://graph.microsoft.com/v1.0/users/{userId}/events/{eventId}
POST https://graph.microsoft.com/v1.0/users/{userId}/calendar/getSchedule
POST https://graph.microsoft.com/v1.0/users
PATCH https://graph.microsoft.com/v1.0/users/{userId}
POST https://graph.microsoft.com/v1.0/users/{userId}/assignLicense
PATCH https://graph.microsoft.com/v1.0/users/{userId}/mailboxSettings
```

### 6.1.2 Authentication & Token Management
- OAuth 2.0 Client Credentials flow per tenant
- Multi-client support (default and onboarding) via `TokenUtils`
- Token caches per tenant with proactive refresh and retry/backoff

```java
String token = tokenUtils.getAccessToken(userContextHolder);
```

### 6.1.3 Data Normalization
- All times normalized to UTC `yyyy-MM-dd'T'HH:mm:ss'Z'`
- Meeting type defaulted (Teams) when absent
- User preferences (timezone) applied when not provided in request

## 6.2 Exchange Web Services (EWS) Integration

### 6.2.1 Service Communication
EWS is used for legacy Exchange environments where Graph is unavailable or for fallback.

**Integration Points:**
- **Calendar Operations**: Create, read, update, delete events
- **Availability**: Resolve availability via EWS APIs
- **OOO Settings**: Manage out-of-office where applicable

**Configuration:**
```properties
ews.ewsURL=https://exchange.example.com/EWS/Exchange.asmx
ews.serviceAccountUsername=poller@example.com
ews.serviceAccountPassword=******
```

### 6.2.2 Fallback Strategy
- Primary path uses Microsoft Graph
- On transient/terminal Graph failures, fallback to EWS when enabled
- Responses normalized to common DTOs

## 6.3 Zoom Integration

### 6.3.1 Service Communication
Zoom is integrated for meeting creation and management when the meeting type is set to Zoom.

**Configuration:**
```properties
zoom.clientId=...
zoom.clientSecret=...
zoom.accountId=...
zoom.tokenUrl=https://zoom.us/oauth/token
zoom.apiURL=https://api.zoom.us/v2
```

**Usage:**
- OAuth with account-level app to obtain access tokens
- Create/update Zoom meetings as part of event scheduling flows

## 6.4 Spring Boot Integration

### 6.4.1 Configuration Properties
Key properties used by the Email Agent:

```properties
# Application
spring.application.name=Email-agent
server.port=8088
server.servlet.context-path=/emailagent

# Logging
logging.level.com.enttribe=DEBUG

# Database
spring.datasource.url=jdbc:mariadb://127.0.0.1:3306/email
spring.datasource.username=...
spring.datasource.password=...
spring.jpa.hibernate.ddl-auto=none

# Token/Clients
token.refresh.interval=3300000
email.clients.info=base64_encoded_clients_json
email.onboarding.clients.info=base64_encoded_onboarding_clients_json

# Platform
email.platform=ews
org.domain.name=vision.com,visionwaves.com

# Graph
graph.license.skuIds=f245ecc8-75af-4f8e-b61f-27d8114de5f3

# EWS
ews.serviceAccountUsername=...
ews.serviceAccountPassword=...
ews.ewsURL=https://exchange.example.com/EWS/Exchange.asmx

# Zoom
zoom.clientId=...
zoom.clientSecret=...
zoom.accountId=...
zoom.tokenUrl=https://zoom.us/oauth/token
zoom.apiURL=https://api.zoom.us/v2
```

### 6.4.2 Auto-Configuration and Beans
- Standard Spring Boot auto-configuration for web and JPA
- `ObjectMapper` bean configured for ISO date/time
- Service beans (`EventService`, `GraphIntegrationService`, `EwsEventService`)
- Utility beans (`TokenUtils`, timezone/date utils)

```java
@Bean
public ObjectMapper objectMapper() {
  ObjectMapper mapper = new ObjectMapper();
  mapper.registerModule(new JavaTimeModule());
  mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'"));
  return mapper;
}
```

### 6.4.3 Security and Access Control
- OAuth tokens acquired per tenant via `TokenUtils`
- Role-based scopes validated per endpoint (see `GraphIntegrationRest`)
- User context injected via `UserContextHolder`

## 6.5 Retry, Fallback, and Resilience

### 6.5.1 Retry
- Exponential backoff on transient failures for Graph/EWS/Zoom calls
- Bounded max attempts; configurable delays

### 6.5.2 Fallback
- Prefer Graph; fallback to EWS when configured or on persistent errors
- Maintain consistent response DTOs across providers

### 6.5.3 Circuit Breaking and Timeouts
- Reasonable HTTP timeouts for external calls
- Optional circuit breaker to avoid cascade failures

## 6.6 Data Persistence and Caching

### 6.6.1 Persistence
- MariaDB for users and preferences via JPA DAOs

### 6.6.2 Caching
- In-memory caches for tokens (per tenant) and preferences
- Proactive token refresh with expiry buffer

## 6.7 Observability and Auditing
- Structured logs for all external integrations (operation, tenant, endpoint, status)
- Audit important lifecycle operations (event creation, updates, user changes)
- Health checks and metrics for critical dependencies

---

*These integration details describe how the Email Agent securely and reliably integrates with Microsoft Graph, EWS, and Zoom while leveraging Spring Boot for configuration, resilience, and observability.*
