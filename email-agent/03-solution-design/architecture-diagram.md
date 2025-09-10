# 3.1 Architecture Diagram

This section contains the high-level architecture diagrams of the Email Agent microservice.

## System Architecture Overview

The Email Agent follows a layered architecture pattern with clear separation of concerns:

```mermaid
graph TB
    subgraph "Spring Boot Application"
        APP[Application Code]
        REST[GraphIntegrationRest]
    end
    
    subgraph "Service Layer"
        ES[EventService]
        GIS[GraphIntegrationService]
        EWS[EwsEventService]
    end
    
    subgraph "Data Access Layer"
        EUD[EmailUserDao]
        EPD[EmailPreferencesDao]
        ENTITY[JPA Entities]
    end
    
    subgraph "External APIs"
        MSGRAPH[Microsoft Graph API]
        EWSAPI[Exchange Web Services]
        ZOOM[Zoom API]
    end
    
    subgraph "Database"
        MARIADB[(MariaDB)]
    end
    
    subgraph "Utilities"
        AUTH[Authentication Utils]
        TIMEZONE[Timezone Utils]
        WORKING[Working Hours Utils]
        DATE[Date Utils]
        EMAIL[Email Utils]
    end
    
    APP --> REST
    REST --> ES
    ES --> GIS
    ES --> EWS
    ES --> EUD
    ES --> EPD
    
    GIS --> MSGRAPH
    EWS --> EWSAPI
    ES --> ZOOM
    
    EUD --> ENTITY
    EPD --> ENTITY
    ENTITY --> MARIADB
    
    ES --> AUTH
    ES --> TIMEZONE
    ES --> WORKING
    ES --> DATE
    ES --> EMAIL
```

## Component Relationships

### REST Controller Layer
- **GraphIntegrationRest**: Primary REST controller for all email and calendar operations
- **API Documentation**: Swagger/OpenAPI annotations for comprehensive API documentation
- **Security Integration**: Role-based access control and authentication

### Service Layer
- **EventService**: Core interface for all event and calendar operations
- **GraphIntegrationService**: Microsoft Graph API integration service
- **EwsEventService**: Exchange Web Services integration service

### Data Access Layer
- **EmailUserDao**: User data access operations
- **EmailPreferencesDao**: User preferences and settings management
- **JPA Entities**: Data persistence with MariaDB

### External Integration Layer
- **Microsoft Graph API**: Modern calendar and user management
- **Exchange Web Services**: Legacy Exchange server integration
- **Zoom API**: Meeting creation and management

## Data Flow Architecture

### Calendar Event Management Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant MSGRAPH as Microsoft Graph
    participant DB as MariaDB
    
    CLIENT->>REST: POST /scheduleEvent
    REST->>ES: scheduleEvent(email, requestBody)
    ES->>ES: Validate request and timezone
    ES->>GIS: createEvent(eventData)
    GIS->>MSGRAPH: POST /me/events
    MSGRAPH-->>GIS: Return event details
    GIS-->>ES: Return event response
    ES->>DB: Save event metadata
    ES-->>REST: Return success response
    REST-->>CLIENT: Return event details
```

### Availability Checking Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: POST /v2/getAvailability
    REST->>ES: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    ES->>ES: Process timezone and working hours
    ES->>GIS: getSchedule(emails, timeRange, slotDuration)
    GIS->>MSGRAPH: POST /me/calendar/getSchedule
    MSGRAPH-->>GIS: Return availability data
    GIS->>GIS: Process availability and conflicts
    GIS-->>ES: Return availability response
    ES-->>REST: Return formatted availability
    REST-->>CLIENT: Return available slots
```

### User Management Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant MSGRAPH as Microsoft Graph
    participant DB as MariaDB
    
    CLIENT->>REST: POST /createUser
    REST->>ES: createUser(userRequest)
    ES->>ES: Validate user data
    ES->>GIS: createUserInGraph(userData)
    GIS->>MSGRAPH: POST /users
    MSGRAPH-->>GIS: Return user details
    GIS->>MSGRAPH: POST /users/{id}/assignLicense
    MSGRAPH-->>GIS: Return license assignment
    GIS-->>ES: Return user creation response
    ES->>DB: Save user preferences
    ES-->>REST: Return user details
    REST-->>CLIENT: Return created user
```

## Key Architectural Principles

### 1. Separation of Concerns
- **REST Layer**: API endpoints and request/response handling
- **Service Layer**: Business logic and orchestration
- **Data Layer**: Data persistence and retrieval
- **Integration Layer**: External API communication

### 2. Dependency Injection
- All components are Spring-managed beans
- Loose coupling through interface-based design
- Easy testing and mocking capabilities

### 3. Caching Strategy
- **Token Caching**: OAuth tokens cached for performance
- **User Preferences**: User settings cached in memory
- **Configuration Caching**: External API configurations cached

### 4. Error Handling
- **Retry Logic**: Automatic retry with exponential backoff
- **Fallback Support**: Fallback between Graph API and EWS
- **Graceful Degradation**: Service continues with reduced functionality

### 5. Security Model
- **OAuth 2.0**: Secure authentication with Microsoft Graph
- **Role-Based Access**: Fine-grained permissions for different operations
- **Token Management**: Automatic token refresh and validation

## Integration Patterns

### 1. Microsoft Graph Integration
- **Authentication**: OAuth 2.0 with client credentials flow
- **API Calls**: RESTful API calls with proper error handling
- **Rate Limiting**: Respect for Microsoft Graph rate limits
- **Retry Logic**: Exponential backoff for transient failures

### 2. Exchange Web Services Integration
- **Authentication**: Basic authentication or modern auth
- **SOAP Calls**: EWS SOAP API for legacy Exchange servers
- **Error Handling**: Proper SOAP fault handling
- **Fallback Support**: Fallback to EWS when Graph API fails

### 3. Database Integration
- **JPA/Hibernate**: Object-relational mapping
- **Connection Pooling**: Efficient database connection management
- **Transaction Management**: ACID compliance for data operations
- **Query Optimization**: Efficient database queries

---

*This architecture ensures the Email Agent provides a robust, scalable, and maintainable solution for email and calendar integration in Spring Boot applications.*
