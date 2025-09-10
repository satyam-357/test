# 1. Introduction

## 1.1 Objective and Scope

### Project Overview

The **Email Agent** is a Spring Boot microservice that provides email and calendar management functionality through Microsoft Graph API and Exchange Web Services (EWS) integration. This LLD document details the technical implementation of the service's core components, algorithms, and integration patterns.

### Technical Scope

This Low Level Design document covers the implementation details of:

- **REST API Layer**: 22+ endpoints for calendar and user management operations
- **Service Layer**: Business logic implementation with Microsoft Graph and EWS integration
- **Data Layer**: JPA entities and DAO implementations for MariaDB persistence
- **Utility Layer**: Core algorithms for availability checking, conflict detection, and working hours calculation
- **Integration Layer**: External API integrations with token management and retry mechanisms

### Technical Architecture

The Email Agent implements a layered architecture with the following key components:

#### Core Implementation Components
- **REST Controllers**: `GraphIntegrationRest` with 22+ endpoints for calendar and user operations
- **Service Layer**: `EventService` interface with `GraphIntegrationService` and `EwsEventService` implementations
- **Data Access**: `EmailUserDao` and `EmailPreferencesDao` for MariaDB persistence
- **Utility Classes**: Specialized utilities for token management, date handling, and availability algorithms
- **External Integrations**: Microsoft Graph API, Exchange Web Services, and Zoom API clients

#### Key Technical Features
- **Multi-Platform Support**: Configurable switching between Microsoft Graph and EWS via `email.platform` property
- **Token Management**: OAuth 2.0 token caching and refresh with `TokenUtils`
- **Availability Algorithms**: `findAvailableTimeSlots` and `findUnavailableTimeSlots` for meeting scheduling
- **Working Hours Processing**: `WorkingHoursUtil` with 50+ timezone mappings and intersection algorithms
- **Data Transformation**: DTOs for API communication and JPA entities for persistence

## 1.2 Technical Implementation

### Technology Stack
- **Framework**: Spring Boot 3.5.5 with Java 21
- **Database**: MariaDB 10.6+ with JPA/Hibernate (MySQL dialect)
- **External APIs**: Microsoft Graph API, Exchange Web Services, Zoom API
- **Security**: OAuth 2.0 with multi-client token management
- **Caching**: In-memory token caching (55-minute refresh interval)
- **Documentation**: Swagger/OpenAPI 3.0 annotations
- **Configuration**: Environment-based with Base64 encoded credentials

### Core Implementation Components

#### 1. REST API Layer (`GraphIntegrationRest`)
- **22+ Endpoints**: Calendar events, meeting scheduling, user management, availability checking
- **Key Endpoints**: 
  - `getAvailableMeetingSlots` - Next 7 days available slots
  - `getAvailableSlotsAndConflict` - Conflict detection with working hours filtering
  - `getSchedule` - Microsoft Graph schedule API
  - `getAvailability` - Detailed availability status per user
  - `scheduleEvent`, `updateEvent`, `cancelEvent` - Event management
  - `createUser`, `updateUserDetails`, `assignLicense` - User management
- **Security**: OAuth 2.0 with role-based access control
- **Documentation**: Swagger/OpenAPI 3.0 annotations

#### 2. Service Layer Implementation
- **`EventService` Interface**: 20+ methods defining contract for calendar and user operations
- **`GraphIntegrationService`**: Microsoft Graph API implementation with token management
- **`EwsEventService`**: Exchange Web Services implementation for legacy support
- **Business Logic**: Data transformation, external API coordination, error handling

#### 3. Data Access Layer
- **`EmailUserDao`**: JPA repository for user data (`findByEmail`, `existsByEmailAndNotDeleted`)
- **`EmailPreferencesDao`**: User preferences management (`getEmailPreferencesByUserId`)
- **JPA Entities**: `EmailUser`, `EmailPreferences` for MariaDB persistence
- **Multi-tenant Support**: Domain validation and soft deletes

#### 4. Utility Layer Implementation
- **`TokenUtils`**: OAuth 2.0 token management with multi-client support and caching
- **`WorkingHoursUtil`**: 50+ timezone mappings with intersection algorithms
- **`EventUtils`**: `findAvailableTimeSlots` and `findUnavailableTimeSlots` algorithms
- **`DateUtils`**: Timezone conversion and date manipulation
- **`CommonUtils`**: Meeting request conversion and JSON manipulation
- **`EWSUtils`**: Exchange Web Services utility methods
- **`ZoomClient`**: Zoom API integration utilities

#### 5. Data Transfer Objects
- **Event Management**: `EventDto`, `Meeting`, `AvailableSlots`, `AvailabilityResponse`
- **User Management**: `CreateUserRequestDto`, `GraphUserDto`
- **Availability**: `AvailabilityStatus`, `TimeSlot`, `WorkingHoursSummary`, `ConflictMeetingWrapper`
- **Event Operations**: `ForwardEventDto`, `SetAutomaticRepliesRequest`

### Core Algorithms Implementation

#### Availability and Conflict Detection
- **`findAvailableTimeSlots`**: Gap analysis algorithm for finding available time slots between existing meetings
- **`findUnavailableTimeSlots`**: Overlap detection algorithm creating `ConflictMeetingWrapper` objects
- **`WorkingHoursUtil`**: Intersection algorithms for multi-user working hours with 50+ timezone mappings
- **`getAvailableMeetingSlots`**: 7-day lookahead algorithm with configurable duration and working hours filtering

## 1.3 Document Scope

This Low Level Design document provides detailed implementation specifications for:

- **Component Design**: Detailed class structures, method signatures, and implementation patterns
- **Algorithm Implementation**: Core algorithms for availability checking, conflict detection, and working hours processing
- **Integration Details**: External API integration patterns, token management, and error handling
- **Data Models**: JPA entities, DTOs, and database schema implementation
- **Configuration**: Environment-based configuration and property management

### Implementation Focus Areas

- **REST API Implementation**: Endpoint specifications, request/response handling, and security integration
- **Service Layer Logic**: Business logic implementation, data transformation, and external API coordination
- **Utility Algorithms**: Availability checking, conflict detection, and working hours calculation algorithms
- **Data Persistence**: JPA entity implementation and DAO patterns for MariaDB integration
- **External Integrations**: Microsoft Graph API, Exchange Web Services, and Zoom API implementation patterns

---

*This LLD document provides the technical implementation details necessary for developers to understand, maintain, and extend the Email Agent microservice.*
