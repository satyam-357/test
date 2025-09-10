# 8. Appendices

*This section contains additional reference materials for the Email Agent microservice.*

## 8.1 Configuration Reference

### Application Properties (Key Settings)
```properties
# Core
spring.application.name=Email-agent
server.port=8088
server.servlet.context-path=/emailagent

# Logging
logging.level.org.springframework=ERROR
logging.level.org.hibernate=ERROR
logging.level.com.enttribe=DEBUG

# Database
spring.datasource.url=jdbc:mariadb://127.0.0.1:3306/email?autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.username=...
spring.datasource.password=...
spring.jpa.hibernate.ddl-auto=none
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl

# Token/Clients
token.refresh.interval=3300000
email.clients.info=base64_encoded_clients_json
email.onboarding.clients.info=base64_encoded_onboarding_clients_json

# Platform Selection
email.platform=ews
org.domain.name=vision.com,visionwaves.com
outsiders.events.lookup.allow=false
total.meeting.limit=500

# Graph Licensing
graph.license.skuIds=f245ecc8-75af-4f8a-b61f-27d8114de5f3

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

### Environment Variables (Examples)
```bash
export MYSQL_URL="jdbc:mariadb://db:3306/email"
export MYSQL_USERNAME="email_user"
export MYSQL_CHECKSUM="strong_password"
export EMAIL_CLIENTS_INFO="<base64 of encrypted client JSON>"
export EMAIL_ONBOARDING_CLIENTS_INFO="<base64 of encrypted onboarding client JSON>"
export ORG_DOMAIN="vision.com,visionwaves.com"
export EMAIL_PLATFORM="ews"
```

### Encrypted Client JSON (Structure)
```json
{
  "default": {"clientId":"enc_id","clientSecret":"enc_secret","tenantId":"enc_tenant"},
  "BNTV":    {"clientId":"enc_id","clientSecret":"enc_secret","tenantId":"enc_tenant"}
}
```

## 8.2 Troubleshooting Guide

See `07-operation-runbook/common-issues-debugging.md` for:
- Token decryption and cache refresh
- Graph rate limiting and 401 handling
- Timezone conversion and scheduling issues
- User creation/licensing problems
- EWS authentication/connectivity

## 8.3 Migration Guide

### From EWS-Only to Graph-First
- Set `email.platform=ews` → progressively migrate to `graph`
- Implement dual-write/read paths behind `EventService`
- Normalize DTOs to keep REST layer unchanged
- Validate parity for: create, update, cancel, availability

### Version Upgrades
- Spring Boot 3.5.x → keep Java 21 toolchain
- MariaDB driver and dialect compatibility
- Microsoft Graph SDK/API updates: verify breaking changes on events and users

## 8.4 API Reference (Key Endpoints)

Base path: `/emailagent/emailservice`

### Calendar Operations
- `POST /scheduleEvent`
- `PATCH /v1/updateExistingEvent`
- `POST /cancelEvent`
- `POST /forwardEvent`
- `POST /rescheduleEvent`
- `POST /getCalendarEvents`
- `POST /getEventDetails`
- `POST /getEventDetailsBySubjectAndTime`
- `POST /getCalendarEventByEventId`

### Availability
- `POST /v1/getAvailableMeetingSlots`
- `POST /v1/getAvailableSlotsAndConflict`
- `POST /v2/getAvailability`
- `POST /getSchedule`

### User and Mailbox
- `POST /createUser`
- `POST /v1/createUser`
- `PATCH /updateUserDetails`
- `POST /assignLicense`
- `PATCH /setAutomaticRepliesSettings`

## 8.5 Glossary

- **UTC**: Coordinated Universal Time; standardized time reference
- **EWS**: Exchange Web Services; SOAP-based Exchange API
- **Microsoft Graph**: REST API for Microsoft 365 services
- **SKU (License)**: Microsoft 365 license identifier applied to a user
- **OOO**: Out-of-office (automatic replies settings)
- **DTO**: Data Transfer Object used for REST contracts
- **Tenant**: Microsoft 365 organization boundary
- **Working Hours Summary**: Intersection of users’ working hours across time zones

---

*This appendices section provides configuration references, operational links, migration notes, API endpoints, and terminology to support operating and evolving the Email Agent.*
