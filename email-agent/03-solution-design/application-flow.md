# 3.3 Application Flow (Sequence Diagrams)

This section contains sequence diagrams showing the flow of operations in the Email Agent microservice.

## Calendar Event Management Flow

### Schedule Event Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant DB as MariaDB
    
    CLIENT->>REST: POST /scheduleEvent
    REST->>REST: Extract user context and validate request
    REST->>ES: scheduleEvent(email, requestBody)
    ES->>ES: Validate timezone and meeting type
    ES->>GIS: createEvent(eventData)
    GIS->>GAC: createEvent(email, eventData)
    GAC->>MSGRAPH: POST /me/events
    MSGRAPH-->>GAC: Return event details
    GAC-->>GIS: Return response
    GIS->>GIS: Process event response
    GIS-->>ES: Return event details
    ES->>DB: Save event metadata
    ES-->>REST: Return success response
    REST-->>CLIENT: Return event details
```

### Update Event Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant DB as MariaDB
    
    CLIENT->>REST: PATCH /v1/updateExistingEvent
    REST->>REST: Extract eventId and user context
    REST->>ES: updateEventV1(eventId, email, jsonBody)
    ES->>ES: Validate event data and timezone
    ES->>GIS: updateEvent(eventId, eventData)
    GIS->>GAC: updateEvent(eventId, eventData)
    GAC->>MSGRAPH: PATCH /me/events/{eventId}
    MSGRAPH-->>GAC: Return updated event
    GAC-->>GIS: Return response
    GIS-->>ES: Return update result
    ES->>DB: Update event metadata
    ES-->>REST: Return update result
    REST-->>CLIENT: Return success response
```

### Cancel Event Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: POST /cancelEvent
    REST->>REST: Extract eventId and comment
    REST->>ES: cancelEvent(eventId, comment)
    ES->>ES: Validate eventId and prepare cancellation
    ES->>GIS: cancelEvent(eventId, comment)
    GIS->>GAC: cancelEvent(eventId, comment)
    GAC->>MSGRAPH: DELETE /me/events/{eventId}
    MSGRAPH-->>GAC: Return cancellation confirmation
    GAC-->>GIS: Return response
    GIS-->>ES: Return cancellation result
    ES-->>REST: Return success response
    REST-->>CLIENT: Return cancellation confirmation
```

## Availability and Meeting Management Flow

### Get Available Meeting Slots Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant WHU as WorkingHoursUtil
    
    CLIENT->>REST: POST /v1/getAvailableMeetingSlots
    REST->>REST: Extract attendees and time range
    REST->>ES: getAvailableMeetingSlots(emails, startTime, endTime, slotDuration)
    ES->>ES: Process timezone conversion
    ES->>GIS: getAvailableSlots(emails, timeRange, slotDuration)
    GIS->>GAC: getSchedule(emails, startTime, endTime, slotDuration)
    GAC->>MSGRAPH: POST /me/calendar/getSchedule
    MSGRAPH-->>GAC: Return availability data
    GAC-->>GIS: Return response
    GIS->>WHU: calculateAvailableSlots(availabilityData, slotDuration)
    WHU-->>GIS: Return available slots
    GIS-->>ES: Return available slots
    ES-->>REST: Return List~Meeting~
    REST-->>CLIENT: Return available meeting slots
```

### Get Availability with Conflicts Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant WHU as WorkingHoursUtil
    
    CLIENT->>REST: POST /v2/getAvailability
    REST->>REST: Extract schedules and parameters
    REST->>ES: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    ES->>ES: Process timezone and working hours
    ES->>GIS: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    GIS->>GAC: getSchedule(emails, startTime, endTime, slotDuration)
    GAC->>MSGRAPH: POST /me/calendar/getSchedule
    MSGRAPH-->>GAC: Return availability data
    GAC-->>GIS: Return response
    GIS->>WHU: processAvailabilityResponse(availabilityData, workingFlag)
    WHU-->>GIS: Return processed availability
    GIS-->>ES: Return AvailabilityResponse
    ES-->>REST: Return availability with conflicts
    REST-->>CLIENT: Return availability response
```

## User Management Flow

### Create User Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant DB as MariaDB
    
    CLIENT->>REST: POST /createUser
    REST->>REST: Extract user request data
    REST->>ES: createUser(userRequest)
    ES->>ES: Validate user data and required fields
    ES->>GIS: createUserInGraph(userData)
    GIS->>GAC: createUser(userData)
    GAC->>MSGRAPH: POST /users
    MSGRAPH-->>GAC: Return user details
    GAC-->>GIS: Return response
    GIS->>GAC: assignLicense(userId, skuId)
    GAC->>MSGRAPH: POST /users/{userId}/assignLicense
    MSGRAPH-->>GAC: Return license assignment
    GAC-->>GIS: Return license response
    GIS-->>ES: Return user creation response
    ES->>DB: Save user preferences
    ES-->>REST: Return user details
    REST-->>CLIENT: Return created user
```

### Update User Details Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant DB as MariaDB
    
    CLIENT->>REST: PATCH /updateUserDetails
    REST->>REST: Extract GraphUserDto
    REST->>ES: updateUserDetails(graphUserDto)
    ES->>ES: Validate user data
    ES->>GIS: updateUserInGraph(userData)
    GIS->>GAC: updateUser(userId, userData)
    GAC->>MSGRAPH: PATCH /users/{userId}
    MSGRAPH-->>GAC: Return updated user
    GAC-->>GIS: Return response
    GIS-->>ES: Return update result
    ES->>DB: Update user preferences
    ES-->>REST: Return update result
    REST-->>CLIENT: Return success response
```

## Meeting Response Flow

### Accept Meeting Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: GET /acceptMeeting?eventId={eventId}
    REST->>REST: Extract and sanitize eventId
    REST->>ES: acceptMeeting(eventId)
    ES->>ES: Validate eventId format
    ES->>GIS: acceptMeeting(eventId)
    GIS->>GAC: acceptEvent(eventId)
    GAC->>MSGRAPH: POST /me/events/{eventId}/accept
    MSGRAPH-->>GAC: Return acceptance confirmation
    GAC-->>GIS: Return response
    GIS-->>ES: Return acceptance result
    ES-->>REST: Return success response
    REST-->>CLIENT: Return acceptance confirmation
```

### Decline Meeting Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: GET /declineMeeting?eventId={eventId}
    REST->>REST: Extract and sanitize eventId
    REST->>ES: declineMeeting(eventId)
    ES->>ES: Validate eventId format
    ES->>GIS: declineMeeting(eventId)
    GIS->>GAC: declineEvent(eventId)
    GAC->>MSGRAPH: POST /me/events/{eventId}/decline
    MSGRAPH-->>GAC: Return decline confirmation
    GAC-->>GIS: Return response
    GIS-->>ES: Return decline result
    ES-->>REST: Return success response
    REST-->>CLIENT: Return decline confirmation
```

## Calendar Event Retrieval Flow

### Get Calendar Events Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: POST /getCalendarEvents
    REST->>REST: Extract time range and subject
    REST->>ES: getCalendarEventsV1(email, startDateTime, endDateTime, subject)
    ES->>ES: Process timezone conversion
    ES->>GIS: getCalendarEvents(email, startTime, endTime, subject)
    GIS->>GAC: getEvents(email, startTime, endTime, subject)
    GAC->>MSGRAPH: GET /me/events?startDateTime={start}&endDateTime={end}
    MSGRAPH-->>GAC: Return calendar events
    GAC-->>GIS: Return response
    GIS->>GIS: Process and filter events
    GIS-->>ES: Return List~EventDto~
    ES-->>REST: Return calendar events
    REST-->>CLIENT: Return events with count
```

### Get Event Details Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: POST /getEventDetails
    REST->>REST: Extract eventId and time range
    REST->>ES: getEventDetails(email, eventId, startDateTime, endDateTime)
    ES->>ES: Validate eventId and time range
    ES->>GIS: getEventDetails(eventId, timeRange)
    GIS->>GAC: getEvent(eventId)
    GAC->>MSGRAPH: GET /me/events/{eventId}
    MSGRAPH-->>GAC: Return event details
    GAC-->>GIS: Return response
    GIS-->>ES: Return EventDto
    ES-->>REST: Return event details
    REST-->>CLIENT: Return event information
```

## Error Handling Flow

### Exception Handling and Recovery
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant EH as ExceptionHandler
    
    CLIENT->>REST: POST /scheduleEvent
    REST->>ES: scheduleEvent(email, requestBody)
    ES->>GIS: createEvent(eventData)
    GIS->>GAC: createEvent(email, eventData)
    GAC->>MSGRAPH: POST /me/events
    MSGRAPH-->>GAC: Return error (rate limit/authentication)
    GAC-->>GIS: Throw ApiException
    GIS-->>ES: Throw BusinessException
    ES-->>REST: Throw ApiException
    REST->>EH: Handle exception
    EH->>EH: Log error and determine response
    EH-->>REST: Return error response
    REST-->>CLIENT: Return error with details
```

### Retry and Fallback Flow
```mermaid
sequenceDiagram
    participant CLIENT as Client Application
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant EWS as EwsEventService
    participant GAC as GraphApiClient
    participant EWSAPI as EWS API
    participant MSGRAPH as Microsoft Graph
    
    CLIENT->>REST: POST /getAvailability
    REST->>ES: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    ES->>GIS: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    GIS->>GAC: getSchedule(emails, startTime, endTime, slotDuration)
    GAC->>MSGRAPH: POST /me/calendar/getSchedule
    MSGRAPH-->>GAC: Return error (service unavailable)
    GAC-->>GIS: Throw ApiException
    GIS-->>ES: Throw BusinessException
    ES->>ES: Check fallback configuration
    ES->>EWS: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    EWS->>EWSAPI: Get availability via EWS
    EWSAPI-->>EWS: Return availability data
    EWS-->>ES: Return availability response
    ES-->>REST: Return availability
    REST-->>CLIENT: Return availability response
```

---

*These sequence diagrams illustrate the detailed flow of operations within the Email Agent, showing how different components interact to provide seamless email and calendar integration capabilities.*
