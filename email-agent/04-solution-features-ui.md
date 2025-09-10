# 4. Solution Features and User Interface

This section documents the major features and user interfaces of the Email Agent microservice.

## 4.1 Core Implementation Features

### 4.1.1 Unified Calendar Management
REST API implementation for calendar operations across Microsoft Graph API and Exchange Web Services.

**Implementation Details:**
- **REST Endpoints**: 12 calendar management endpoints with Swagger documentation
- **Platform Switching**: Configurable via `email.platform` property (graph/ews)
- **DTO Mapping**: Automatic conversion between JSON and strongly typed DTOs
- **Timezone Processing**: `DateUtils` utility with 50+ timezone mappings

**Example Usage:**
```java
@Autowired
private EventService eventService;

// Schedule a meeting
Map<String, Object> eventData = Map.of(
    "subject", "Project Review Meeting",
    "startTime", "2024-01-15T10:00:00",
    "endTime", "2024-01-15T11:00:00",
    "attendees", List.of("john@company.com", "jane@company.com")
);
Map<String, Object> result = eventService.scheduleEvent("user@company.com", eventData);

// Get calendar events
List<EventDto> events = eventService.getCalendarEventsV1(
    "user@company.com", 
    "2024-01-15T00:00:00", 
    "2024-01-15T23:59:59", 
    "Meeting"
);
```

### 4.1.2 Availability and Conflict Detection Algorithms
Implementation of meeting scheduling algorithms with availability checking and conflict detection.

**Algorithm Implementation:**
- **`findAvailableTimeSlots`**: Gap analysis algorithm for finding available slots
- **`findUnavailableTimeSlots`**: Overlap detection creating `ConflictMeetingWrapper` objects
- **`WorkingHoursUtil`**: Multi-user working hours intersection with timezone conversion
- **`getAvailableMeetingSlots`**: Configurable lookahead period with configurable duration

**Example Usage:**
```java
@Autowired
private EventService eventService;

// Get available meeting slots
List<String> attendees = List.of("john@company.com", "jane@company.com");
List<Meeting> availableSlots = eventService.getAvailableMeetingSlots(
    attendees, 
    "2024-01-15T09:00:00", 
    "2024-01-15T17:00:00", 
    30
);

// Get availability with conflicts
AvailabilityResponse availability = eventService.getAvailability(
    attendees,
    "2024-01-15T09:00:00",
    "2024-01-15T17:00:00",
    30,
    true
);
```

### 4.1.3 User Management Implementation
JPA-based user lifecycle management with Microsoft Graph integration and license assignment.

**Implementation Details:**
- **User Creation**: `createUser` and `createUserV1` endpoints with `CreateUserRequestDto`
- **User Updates**: `updateUserDetails` endpoint with `GraphUserDto` mapping
- **License Management**: `assignLicense` endpoint with Microsoft 365 SKU assignment
- **Data Persistence**: `EmailUser` and `EmailPreferences` JPA entities with MariaDB

**Example Usage:**
```java
@Autowired
private EventService eventService;

// Create a new user
Map<String, Object> userData = Map.of(
    "displayName", "John Doe",
    "mail", "john.doe@company.com",
    "userPrincipalName", "john.doe@company.com",
    "password", "TempPassword123!"
);
Map<String, Object> user = eventService.createUser(userData);

// Update user details
GraphUserDto userUpdate = new GraphUserDto();
userUpdate.setDisplayName("John Smith");
userUpdate.setMail("john.smith@company.com");
Map<String, Object> result = eventService.updateUserDetails(userUpdate);
```

## 4.2 REST API Interfaces

### 4.2.1 Calendar Event Management

#### POST /scheduleEvent
Create new calendar events with attendees and meeting details.

**Request:**
```json
{
  "email": "user@company.com",
  "subject": "Project Review Meeting",
  "startTime": "2024-01-15T10:00:00",
  "endTime": "2024-01-15T11:00:00",
  "attendees": [
    {
      "email": "john@company.com",
      "type": "required"
    },
    {
      "email": "jane@company.com", 
      "type": "optional"
    }
  ],
  "location": "Conference Room A",
  "body": "Weekly project review meeting",
  "isOnlineMeeting": true,
  "meetingType": "Teams"
}
```

**Response:**
```json
{
  "eventId": "AAMkAGI2...",
  "subject": "Project Review Meeting",
  "startTime": "2024-01-15T10:00:00Z",
  "endTime": "2024-01-15T11:00:00Z",
  "joinUrl": "https://teams.microsoft.com/l/meetup-join/...",
  "status": "created",
  "attendees": [
    {
      "email": "john@company.com",
      "responseStatus": "accepted"
    },
    {
      "email": "jane@company.com",
      "responseStatus": "tentative"
    }
  ]
}
```

#### PATCH /v1/updateExistingEvent
Update existing calendar events.

**Request:**
```json
{
  "eventId": "AAMkAGI2...",
  "email": "user@company.com",
  "subject": "Updated Project Review Meeting",
  "startTime": "2024-01-15T14:00:00",
  "endTime": "2024-01-15T15:00:00",
  "body": "Updated meeting details"
}
```

**Response:**
```json
{
  "eventId": "AAMkAGI2...",
  "subject": "Updated Project Review Meeting",
  "startTime": "2024-01-15T14:00:00Z",
  "endTime": "2024-01-15T15:00:00Z",
  "status": "updated",
  "lastModified": "2024-01-15T09:30:00Z"
}
```

#### POST /cancelEvent
Cancel calendar events with optional comment.

**Request:**
```json
{
  "eventId": "AAMkAGI2...",
  "comment": "Meeting cancelled due to scheduling conflict"
}
```

**Response:**
```json
{
  "eventId": "AAMkAGI2...",
  "status": "cancelled",
  "cancelledTime": "2024-01-15T09:30:00Z",
  "comment": "Meeting cancelled due to scheduling conflict"
}
```

#### POST /forwardEvent
Forward events to multiple recipients.

**Request:**
```json
{
  "eventId": "AAMkAGI2...",
  "emailIds": ["newuser1@company.com", "newuser2@company.com"],
  "comment": "Please review this meeting"
}
```

**Response:**
```json
{
  "eventId": "AAMkAGI2...",
  "forwardedTo": ["newuser1@company.com", "newuser2@company.com"],
  "status": "forwarded",
  "forwardedTime": "2024-01-15T09:30:00Z"
}
```

### 4.2.2 Event Response Management

#### GET /acceptMeeting?eventId={id}
Accept meeting invitations.

**Response:**
```json
{
  "eventId": "AAMkAGI2...",
  "responseStatus": "accepted",
  "responseTime": "2024-01-15T09:30:00Z"
}
```

#### GET /declineMeeting?eventId={id}
Decline meeting invitations.

**Response:**
```json
{
  "eventId": "AAMkAGI2...",
  "responseStatus": "declined",
  "responseTime": "2024-01-15T09:30:00Z"
}
```

### 4.2.3 Event Retrieval

#### POST /getCalendarEvents
Get calendar events by time range.

**Request:**
```json
{
  "email": "user@company.com",
  "startDateTime": "2024-01-15T00:00:00",
  "endDateTime": "2024-01-15T23:59:59",
  "subject": "Meeting"
}
```

**Response:**
```json
[
  {
    "id": "AAMkAGI2...",
    "subject": "Project Review Meeting",
    "organizer": "user@company.com",
    "attendees": ["john@company.com", "jane@company.com"],
    "meetingStartTime": "2024-01-15T10:00:00Z",
    "meetingEndTime": "2024-01-15T11:00:00Z",
    "location": "Conference Room A",
    "joinUrl": "https://teams.microsoft.com/l/meetup-join/...",
    "isCancelled": false,
    "createdDateTime": "2024-01-14T15:30:00Z",
    "lastModifiedDateTime": "2024-01-14T15:30:00Z"
  }
]
```

### 4.2.4 Availability and Meeting Management

#### POST /v1/getAvailableMeetingSlots
Get available meeting slots for multiple attendees.

**Request:**
```json
{
  "emails": ["john@company.com", "jane@company.com", "bob@company.com"],
  "startDateTime": "2024-01-15T09:00:00",
  "endDateTime": "2024-01-15T17:00:00",
  "slotDuration": 30
}
```

**Response:**
```json
[
  {
    "startTime": "2024-01-15T09:00:00Z",
    "endTime": "2024-01-15T09:30:00Z",
    "duration": 30,
    "available": true
  },
  {
    "startTime": "2024-01-15T10:00:00Z",
    "endTime": "2024-01-15T10:30:00Z",
    "duration": 30,
    "available": true
  },
  {
    "startTime": "2024-01-15T14:00:00Z",
    "endTime": "2024-01-15T14:30:00Z",
    "duration": 30,
    "available": true
  }
]
```

#### POST /v1/getAvailableSlotsAndConflict
Get available slots with conflict information.

**Request:**
```json
{
  "emails": ["john@company.com", "jane@company.com"],
  "startDateTime": "2024-01-15T09:00:00",
  "endDateTime": "2024-01-15T17:00:00",
  "slotDuration": 60,
  "workingFlag": true
}
```

**Response:**
```json
{
  "slots": [
    {
      "startTime": "2024-01-15T09:00:00Z",
      "endTime": "2024-01-15T10:00:00Z",
      "duration": 60,
      "available": true
    },
    {
      "startTime": "2024-01-15T14:00:00Z",
      "endTime": "2024-01-15T15:00:00Z",
      "duration": 60,
      "available": true
    }
  ],
  "conflicts": [
    {
      "userId": "john@company.com",
      "meetings": [
        {
          "startTime": "2024-01-15T10:00:00Z",
          "endTime": "2024-01-15T11:00:00Z",
          "subject": "Existing Meeting"
        }
      ]
    }
  ],
  "filtered": true
}
```

#### POST /v2/getAvailability
Get detailed availability using Microsoft Graph getSchedule API.

**Request:**
```json
{
  "emails": ["john@company.com", "jane@company.com"],
  "startDateTime": "2024-01-15T09:00:00",
  "endDateTime": "2024-01-15T17:00:00",
  "slotDuration": 30,
  "workingFlag": true
}
```

**Response:**
```json
{
  "timeSlots": [
    {
      "startTime": "2024-01-15T09:00:00Z",
      "endTime": "2024-01-15T09:30:00Z",
      "availability": [
        {
          "email": "john@company.com",
          "status": "free",
          "meeting": null
        },
        {
          "email": "jane@company.com",
          "status": "free",
          "meeting": null
        }
      ]
    },
    {
      "startTime": "2024-01-15T10:00:00Z",
      "endTime": "2024-01-15T10:30:00Z",
      "availability": [
        {
          "email": "john@company.com",
          "status": "busy",
          "meeting": {
            "subject": "Existing Meeting",
            "startTime": "2024-01-15T10:00:00Z",
            "endTime": "2024-01-15T11:00:00Z"
          }
        },
        {
          "email": "jane@company.com",
          "status": "free",
          "meeting": null
        }
      ]
    }
  ],
  "availableSlots": [
    {
      "startTime": "2024-01-15T09:00:00Z",
      "endTime": "2024-01-15T09:30:00Z"
    },
    {
      "startTime": "2024-01-15T14:00:00Z",
      "endTime": "2024-01-15T14:30:00Z"
    }
  ]
}
```

### 4.2.3 User Management Interface
Complete user lifecycle management through REST API.

**User Management Endpoints:**
- `POST /createUser`: Create new users in Microsoft Graph
- `POST /v1/createUser`: Create users via bulk upload
- `PATCH /updateUserDetails`: Update user information
- `POST /assignLicense`: Assign Microsoft 365 licenses

**User Preferences:**
- `PATCH /setAutomaticRepliesSettings`: Configure out-of-office settings
- Timezone management and working hours configuration
- Email preferences and notification settings

## 4.3 Configuration Management

### 4.3.1 Multi-Platform Integration
Seamless integration with multiple email and calendar platforms.

**Supported Platforms:**
- **Microsoft Graph API**: Modern calendar and user management
- **Exchange Web Services (EWS)**: Legacy Exchange server support
- **Zoom API**: Meeting creation and management
- **Multi-Tenant Support**: Multiple organization support

**Platform Features:**
- **Automatic Selection**: Platform selection based on configuration
- **Fallback Support**: Automatic failover between platforms
- **Configuration Caching**: Cached platform configurations for performance
- **Error Handling**: Graceful handling of platform-specific errors

### 4.3.2 Dynamic Configuration Loading
Automatic loading and management of external API configurations.

**Configuration Sources:**
- **Environment Variables**: Sensitive configuration via environment variables
- **Application Properties**: Standard Spring Boot configuration
- **Database Configuration**: User preferences and settings
- **External API Configs**: Microsoft Graph and EWS credentials

**Configuration Features:**
- **Startup Loading**: All configurations loaded at application startup
- **Runtime Updates**: Dynamic configuration updates without restart
- **Encryption Support**: Encrypted storage of sensitive credentials
- **Validation**: Configuration validation and error reporting

### 4.3.3 Timezone and Working Hours Management
Intelligent timezone handling and working hours calculation.

**Timezone Features:**
- **Automatic Conversion**: UTC to local timezone conversion
- **User Preferences**: Individual user timezone settings
- **Working Hours**: Configurable working hours per user
- **Holiday Support**: Support for organization holidays and exceptions

**Working Hours Configuration:**
- **Flexible Schedules**: Support for different working schedules
- **Multi-Timezone**: Handle meetings across multiple timezones
- **Conflict Resolution**: Intelligent conflict detection and resolution
- **Availability Windows**: Configurable availability checking windows

## 4.4 Enterprise Features

### 4.4.1 Comprehensive Audit Logging
Built-in audit logging for compliance and monitoring.

**Audit Types:**
- **Event Audit**: Track all calendar event operations
- **User Audit**: Monitor user management operations
- **API Audit**: Log all external API calls and responses
- **Error Audit**: Detailed error logging and tracking

**Audit Features:**
- **Automatic Logging**: No additional code required
- **Structured Data**: JSON-formatted audit records
- **Performance Metrics**: Response times and operation metrics
- **Security Tracking**: Authentication and authorization events

### 4.4.2 Security and Access Control
Enterprise-grade security and access control features.

**Security Features:**
- **OAuth 2.0 Integration**: Secure authentication with Microsoft Graph
- **Role-Based Access**: Fine-grained permissions for different operations
- **Token Management**: Automatic token refresh and validation
- **API Key Encryption**: Encrypted storage of external API credentials

**Access Control:**
- **Operation-Level Permissions**: Granular control over API operations
- **User Context**: Automatic user context extraction and validation
- **Multi-Tenant Isolation**: Data segregation by organization
- **Audit Trail**: Complete audit trail for security compliance

### 4.4.3 Error Handling and Recovery
Robust error handling with automatic recovery mechanisms.

**Error Handling Features:**
- **Retry Logic**: Exponential backoff for transient failures
- **Fallback Support**: Automatic fallback between Graph API and EWS
- **Circuit Breaker**: Prevent cascade failures
- **Graceful Degradation**: Service continues with reduced functionality

**Recovery Mechanisms:**
- **Automatic Retry**: Configurable retry attempts for failed operations
- **Provider Failover**: Switch between Microsoft Graph and EWS
- **Data Consistency**: Ensure data consistency during failures
- **Monitoring Integration**: Real-time error monitoring and alerting

### 4.4.4 Performance Optimization
High-performance email and calendar operations.

**Performance Features:**
- **Connection Pooling**: Efficient database and API connection management
- **Caching Strategy**: Intelligent caching of frequently accessed data
- **Batch Operations**: Support for bulk operations where possible
- **Async Processing**: Asynchronous processing for long-running operations

**Optimization Techniques:**
- **Token Caching**: Cached authentication tokens for performance
- **Response Caching**: Cached API responses where appropriate
- **Database Optimization**: Optimized database queries and indexing
- **Memory Management**: Efficient memory usage and garbage collection

## 4.5 Developer Experience

### 4.5.1 Zero Boilerplate Integration
Minimal code required for email and calendar integration.

**Benefits:**
- **Single Service**: One service for all email and calendar operations
- **Auto-Configuration**: Automatic Spring Boot configuration
- **Type Safety**: Compile-time type checking with DTOs
- **IDE Support**: Full IDE support with autocomplete and documentation

### 4.5.2 Comprehensive API Documentation
Extensive API documentation with Swagger/OpenAPI integration.

**Documentation Features:**
- **Interactive API**: Swagger UI for testing and exploration
- **Request/Response Examples**: Real-world usage examples
- **Error Codes**: Comprehensive error code documentation
- **Authentication Guide**: Step-by-step authentication setup

### 4.5.3 Testing Support
Built-in testing capabilities and utilities.

**Testing Features:**
- **Mock Support**: Easy mocking of external API responses
- **Test Utilities**: Helper classes for testing calendar operations
- **Integration Tests**: End-to-end testing support
- **Performance Testing**: Load testing capabilities for calendar operations

### 4.5.4 Monitoring and Observability
Comprehensive monitoring and observability features.

**Monitoring Features:**
- **Health Checks**: Built-in health check endpoints
- **Metrics Collection**: Performance and usage metrics
- **Log Aggregation**: Centralized logging for distributed systems
- **Alerting**: Real-time alerting for errors and performance issues

---

*The Email Agent provides a comprehensive set of features that simplify email and calendar integration while providing enterprise-grade capabilities for production applications.*
