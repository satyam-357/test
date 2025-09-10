# 3.2 Component Diagram

This section contains detailed component diagrams of the Email Agent microservice.

## Core Components

### Primary Interface Components
```mermaid
classDiagram
    class GraphIntegrationRest {
        -EventService eventService
        -EmailPreferencesDao preferencesDao
        -UserContextHolder userContextHolder
        +getAvailableMeetingSlots(Map request) List~Meeting~
        +getCalendarEvents(Map request) Map~String, Object~
        +scheduleEvent(Map jsonBody) Map~String, Object~
        +getAvailability(Map request) AvailabilityResponse
        +createUser(Map userRequest) Map~String, Object~
        +updateUserDetails(GraphUserDto userDto) Map~String, Object~
        +setAutomaticRepliesSettings(SetAutomaticRepliesRequest request) Map~String, Object~
    }
    
    class EventService {
        <<interface>>
        +getCalendarEventsV1(String email, String startDateTime, String endDateTime, String subject) List~EventDto~
        +scheduleEvent(String email, Map requestBody) Map~String, Object~
        +getAvailability(List emails, String startDateTime, String endDateTime, int slotDuration, Boolean workingFlag) AvailabilityResponse
        +createUser(Map userRequest) Map~String, Object~
        +updateUserDetails(GraphUserDto userDto) Map~String, Object~
        +setAutomaticRepliesSettings(String userId, SetAutomaticRepliesRequest request) Map~String, Object~
        +declineMeeting(String eventId) Map~String, String~
        +acceptMeeting(String eventId) Map~String, String~
        +cancelEvent(String eventId, String comment) Map~String, String~
    }
    
    class UserContextHolder {
        -ThreadLocal~UserInfo~ userContext
        +getCurrentUser() UserInfo
        +setCurrentUser(UserInfo userInfo) void
        +clear() void
    }
```

### Service Implementation Components
```mermaid
classDiagram
    class GraphIntegrationService {
        -EmailUserDao emailUserDao
        -EmailPreferencesDao emailPreferencesDao
        -TokenUtils tokenUtils
        -DateUtils dateUtils
        -WorkingHoursUtil workingHoursUtil
        +getCalendarEventsV1(String email, String startDateTime, String endDateTime, String subject) List~EventDto~
        +scheduleEvent(String email, Map requestBody) Map~String, Object~
        +getAvailability(List emails, String startDateTime, String endDateTime, int slotDuration, Boolean workingFlag) AvailabilityResponse
        +createUser(Map userRequest) Map~String, Object~
        +updateUserDetails(GraphUserDto userDto) Map~String, Object~
        +setAutomaticRepliesSettings(String userId, SetAutomaticRepliesRequest request) Map~String, Object~
        -makeGraphApiCall(String endpoint, String method, Object requestBody) String
        -processAvailabilityResponse(String response) AvailabilityResponse
    }
    
    class EwsEventService {
        -EmailUserDao emailUserDao
        -EmailPreferencesDao emailPreferencesDao
        -EWSUtils ewsUtils
        -DateUtils dateUtils
        -WorkingHoursUtil workingHoursUtil
        +getCalendarEventsV1(String email, String startDateTime, String endDateTime, String subject) List~EventDto~
        +scheduleEvent(String email, Map requestBody) Map~String, Object~
        +getAvailability(List emails, String startDateTime, String endDateTime, int slotDuration, Boolean workingFlag) AvailabilityResponse
        +createUser(Map userRequest) Map~String, Object~
        +updateUserDetails(GraphUserDto userDto) Map~String, Object~
        +setAutomaticRepliesSettings(String userId, SetAutomaticRepliesRequest request) Map~String, Object~
        -makeEwsApiCall(String operation, Object parameters) String
        -processEwsResponse(String response) List~EventDto~
    }
```

## Data Access Components

### DAO and Entity Components
```mermaid
classDiagram
    class EmailUserDao {
        -JpaRepository~EmailUser, String~ repository
        +findByEmail(String email) EmailUser
        +save(EmailUser emailUser) EmailUser
        +findAll() List~EmailUser~
        +deleteByEmail(String email) void
    }
    
    class EmailPreferencesDao {
        -JpaRepository~EmailPreferences, String~ repository
        +getEmailPreferencesByUserId(String userId) EmailPreferences
        +save(EmailPreferences preferences) EmailPreferences
        +findByUserId(String userId) Optional~EmailPreferences~
        +updateTimeZone(String userId, String timeZone) void
    }
    
    class EmailUser {
        -String email
        -String displayName
        -String timeZone
        -Date createdDate
        -Date lastModifiedDate
        +getEmail() String
        +setEmail(String email) void
        +getDisplayName() String
        +setDisplayName(String displayName) void
    }
    
    class EmailPreferences {
        -String userId
        -String timeZone
        -String workingHours
        -Boolean autoReplyEnabled
        -String autoReplyMessage
        +getUserId() String
        +setUserId(String userId) void
        +getTimeZone() String
        +setTimeZone(String timeZone) void
    }
```

## Utility Components

### Authentication and Token Management
```mermaid
classDiagram
    class TokenUtils {
        <<utility>>
        +getAccessToken(String clientId, String clientSecret, String tenantId) String
        +refreshAccessToken(String refreshToken) String
        +validateToken(String token) boolean
        +extractClaimsFromToken(String token) Map~String, Object~
        -makeTokenRequest(String endpoint, Map parameters) String
    }
    
    class UserInfo {
        -String email
        -String displayName
        -String tenantId
        -List~String~ roles
        +getEmail() String
        +setEmail(String email) void
        +getDisplayName() String
        +setDisplayName(String displayName) void
        +getTenantId() String
        +setTenantId(String tenantId) void
    }
```

### Date and Time Management
```mermaid
classDiagram
    class DateUtils {
        <<utility>>
        +convertToUTCString(String dateTime, String timeZone) String
        +convertFromUTCString(String utcDateTime, String timeZone) String
        +formatDateTime(Date date, String pattern) String
        +parseDateTime(String dateTime, String pattern) Date
        +getCurrentUTCTime() String
        -getTimeZoneOffset(String timeZone) int
    }
    
    class WorkingHoursUtil {
        <<utility>>
        +calculateWorkingHours(String timeZone, String startTime, String endTime) WorkingHoursSummary
        +isWithinWorkingHours(String timeZone, String dateTime) boolean
        +getNextWorkingDay(String timeZone, String date) String
        +calculateAvailableSlots(List~String~ emails, String startTime, String endTime, int slotDuration) List~TimeSlot~
        -getWorkingDays(String timeZone) List~DayOfWeek~
    }
```

### Email and Event Processing
```mermaid
classDiagram
    class EmailUtils {
        <<utility>>
        +validateEmailAddress(String email) boolean
        +extractDomainFromEmail(String email) String
        +formatEmailList(List~String~ emails) String
        +parseEmailAddresses(String emailString) List~String~
        -isValidEmailFormat(String email) boolean
    }
    
    class EventUtils {
        <<utility>>
        +convertToEventDto(Map eventData) EventDto
        +convertToMeetingDto(Map eventData) Meeting
        +formatEventResponse(Map response) Map~String, Object~
        +validateEventData(Map eventData) boolean
        -extractEventDetails(Map eventData) EventDto
    }
```

## External Integration Components

### Microsoft Graph API Integration
```mermaid
classDiagram
    class GraphApiClient {
        -String baseUrl
        -String accessToken
        -RestTemplate restTemplate
        +getCalendarEvents(String email, String startTime, String endTime) String
        +createEvent(String email, Map eventData) String
        +updateEvent(String eventId, Map eventData) String
        +deleteEvent(String eventId) String
        +getAvailability(List emails, String startTime, String endTime, int slotDuration) String
        +createUser(Map userData) String
        +updateUser(String userId, Map userData) String
        +assignLicense(String userId, String skuId) String
        -makeApiCall(String endpoint, String method, Object requestBody) String
    }
    
    class EwsApiClient {
        -String ewsUrl
        -String username
        -String password
        -ExchangeService exchangeService
        +getCalendarEvents(String email, String startTime, String endTime) String
        +createEvent(String email, Map eventData) String
        +updateEvent(String eventId, Map eventData) String
        +deleteEvent(String eventId) String
        +getAvailability(List emails, String startTime, String endTime, int slotDuration) String
        -makeEwsCall(String operation, Object parameters) String
    }
```

## Component Interaction Patterns

### 1. Calendar Event Creation Pattern
```mermaid
sequenceDiagram
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    participant DB as Database
    
    REST->>ES: scheduleEvent(email, requestBody)
    ES->>ES: Validate and process request
    ES->>GIS: createEvent(eventData)
    GIS->>GAC: createEvent(email, eventData)
    GAC->>MSGRAPH: POST /me/events
    MSGRAPH-->>GAC: Return event details
    GAC-->>GIS: Return response
    GIS-->>ES: Return event response
    ES->>DB: Save event metadata
    ES-->>REST: Return success response
```

### 2. Availability Checking Pattern
```mermaid
sequenceDiagram
    participant REST as GraphIntegrationRest
    participant ES as EventService
    participant GIS as GraphIntegrationService
    participant GAC as GraphApiClient
    participant MSGRAPH as Microsoft Graph
    
    REST->>ES: getAvailability(emails, startTime, endTime, slotDuration, workingFlag)
    ES->>ES: Process timezone and working hours
    ES->>GIS: getSchedule(emails, timeRange, slotDuration)
    GIS->>GAC: getAvailability(emails, startTime, endTime, slotDuration)
    GAC->>MSGRAPH: POST /me/calendar/getSchedule
    MSGRAPH-->>GAC: Return availability data
    GAC-->>GIS: Return response
    GIS->>GIS: Process availability and conflicts
    GIS-->>ES: Return availability response
    ES-->>REST: Return formatted availability
```

## Key Design Patterns

### 1. Service Layer Pattern
- **EventService**: Interface defining business operations
- **GraphIntegrationService**: Microsoft Graph implementation
- **EwsEventService**: EWS implementation

### 2. Data Access Object (DAO) Pattern
- **EmailUserDao**: User data access operations
- **EmailPreferencesDao**: Preferences data access operations
- **JPA Entities**: Data persistence objects

### 3. Factory Pattern
- **TokenUtils**: Creates and manages authentication tokens
- **DateUtils**: Creates formatted date/time objects
- **WorkingHoursUtil**: Creates working hours calculations

### 4. Strategy Pattern
- **GraphIntegrationService**: Microsoft Graph strategy
- **EwsEventService**: EWS strategy
- **Service Selection**: Based on configuration or availability

### 5. Template Method Pattern
- **GraphApiClient**: Common HTTP request handling
- **EwsApiClient**: Common EWS operation handling
- **Utility Classes**: Common processing patterns

---

*This component design ensures modularity, testability, and maintainability of the Email Agent while providing a clean separation of concerns.*
