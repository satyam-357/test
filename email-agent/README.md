# Email Agent - Low Level Design (LLD)

## Table of Contents

1. [Introduction](./01-introduction.md)
   - 1.1 Objective and Scope
   - 1.2 Project Overview
   - 1.3 Architecture Context

2. [Assumptions](./02-assumptions.md)
   - 2.1 Technical Assumptions
   - 2.2 Integration Assumptions
   - 2.3 Performance Assumptions
   - 2.4 Business Assumptions

3. [Solution Design](./03-solution-design/README.md)
   - 3.1 [Architecture Diagram](./03-solution-design/architecture-diagram.md)
   - 3.2 [Component Diagram](./03-solution-design/component-diagram.md)
   - 3.3 [Application Flow (Sequence Diagrams)](./03-solution-design/application-flow.md)
   - 3.4 [Database Schema](./03-solution-design/database-schema.md)

4. [Solution Features and User Interface](./04-solution-features-ui.md)
   - 4.1 Core Features
   - 4.2 REST API Interfaces
   - 4.3 Configuration Management
   - 4.4 Enterprise Features
   - 4.5 Developer Experience

5. [Algorithm](./05-algorithms.md)
   - 5.1 Token Management Algorithm
   - 5.2 Availability and Slotting Algorithms
   - 5.3 Date/Time Normalization Algorithms
   - 5.4 Retry, Fallback, and Resilience
   - 5.5 Caching Strategy
   - 5.6 User and Event Operations

6. [Integration Details](./06-integration-details.md)
   - 6.1 Microsoft Graph Integration
   - 6.2 Exchange Web Services (EWS) Integration
   - 6.3 Zoom Integration
   - 6.4 Spring Boot Integration
   - 6.5 Retry, Fallback, and Resilience
   - 6.6 Data Persistence and Caching

7. [Operation Runbook](./07-operation-runbook/README.md)
   - 7.1 [Common Issues & Debugging](./07-operation-runbook/common-issues-debugging.md)
   - 7.2 [Performance Tuning Guide](./07-operation-runbook/performance-tuning-guide.md)

8. [Appendices](./08-appendices.md)
   - 8.1 Configuration Reference
   - 8.2 Troubleshooting Guide
   - 8.3 Migration Guide
   - 8.4 API Reference
   - 8.5 Glossary

---

## Document Information

- **Project**: Email Agent
- **Version**: 1.0.0
- **Type**: Spring Boot Microservice
- **Target**: Enterprise Email and Calendar Management
- **Last Updated**: [Current Date]

## Technical Implementation Reference

- **Main Package**: `com.enttribe.emailagent`
- **REST Controller**: `GraphIntegrationRest` (22+ endpoints)
- **Service Layer**: `EventService` interface with `GraphIntegrationService` and `EwsEventService` implementations
- **Data Layer**: `EmailUserDao`, `EmailPreferencesDao` with JPA entities
- **Utilities**: `TokenUtils`, `WorkingHoursUtil`, `EventUtils`, `DateUtils`
- **Database**: MariaDB 10.6+ with JPA/Hibernate
- **External APIs**: Microsoft Graph API, Exchange Web Services, Zoom API

## Database Schema

### Core Tables
- **email_users**: User information and preferences
- **email_preferences**: User-specific settings and timezone configuration

### JPA Entities
- **EmailUser**: User data persistence with soft delete support
- **EmailPreferences**: User preferences and timezone settings

## API Endpoints

### Calendar Management (12 endpoints)
- `POST /scheduleEvent` - Create calendar events
- `PATCH /v1/updateExistingEvent` - Update existing events
- `POST /cancelEvent` - Cancel events
- `POST /forwardEvent` - Forward events
- `POST /rescheduleEvent` - Reschedule events
- `GET /acceptMeeting` - Accept meeting invitations
- `GET /declineMeeting` - Decline meeting invitations
- `GET /tentativelyAccept` - Tentatively accept meetings
- `POST /getCalendarEvents` - Retrieve calendar events
- `POST /getEventDetails` - Get specific event details
- `POST /getEventDetailsBySubjectAndTime` - Find events by criteria
- `POST /getCalendarEventByEventId` - Get event by ID

### Availability Management (4 endpoints)
- `POST /v1/getAvailableMeetingSlots` - Find available slots
- `POST /v1/getAvailableSlotsAndConflict` - Get slots with conflicts
- `POST /v2/getAvailability` - Microsoft Graph availability
- `POST /getSchedule` - Raw schedule data

### User Management (4 endpoints)
- `POST /createUser` - Create users
- `POST /v1/createUser` - Bulk user creation
- `PATCH /updateUserDetails` - Update user information
- `POST /assignLicense` - Assign Microsoft 365 licenses

### Automation (2 endpoints)
- `PATCH /setAutomaticRepliesSettings` - Configure out-of-office
- `GET /ping` - Health check

## Core Algorithms

### Availability Checking
- **`findAvailableTimeSlots`**: Gap analysis between existing meetings
- **`findUnavailableTimeSlots`**: Conflict detection and overlap identification
- **`WorkingHoursUtil`**: Multi-user working hours intersection with 50+ timezone mappings
- **`getAvailableMeetingSlots`**: 7-day lookahead with configurable duration

### Token Management
- **OAuth 2.0**: Multi-client token management with automatic refresh
- **Caching**: 55-minute token refresh interval with proactive refresh
- **Retry Logic**: Exponential backoff for token refresh failures

---

*This LLD provides complete technical implementation details for developers to understand, maintain, and extend the Email Agent microservice.*
