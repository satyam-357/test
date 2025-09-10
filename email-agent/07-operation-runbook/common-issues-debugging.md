# 7.1 Common Issues & Debugging

*This section documents common issues and debugging procedures for the Email Agent microservice.*

## Common Issues

### 7.1.1 Application Startup Failures

#### Issue: Token Decryption Failures During Startup
**Symptom:**
```
BusinessException: Error in decrypting graph credentials
BusinessException: Error in initializing token utility
```

**Root Cause:**
The application has incorrect or missing values for the encrypted client credentials in `email.clients.info` or `email.onboarding.clients.info` environment variables. These contain AES-encrypted Microsoft Graph client credentials.

**Solution:**
1. Verify the `email.clients.info` environment variable is set correctly
2. Ensure the `email.onboarding.clients.info` is properly configured if using onboarding features
3. Check that the AES encryption key used to encrypt credentials matches the decryption key
4. Validate the Base64 encoding of the JSON configuration

**Prevention:**
- Document the correct encrypted credentials for each environment
- Use secure key management practices for AES encryption keys
- Implement credential validation during startup

#### Issue: Database Connection Failures
**Symptom:**
```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource'
java.sql.SQLException: Access denied for user 'root'@'localhost'
```

**Root Cause:**
Database connection parameters are incorrect or the MariaDB service is not running.

**Solution:**
1. Verify MariaDB is running: `systemctl status mariadb`
2. Check database connection parameters in `application.properties`
3. Ensure database exists: `CREATE DATABASE email;`
4. Verify user permissions and password

**Configuration Check:**
```properties
spring.datasource.url=jdbc:mariadb://127.0.0.1:3306/email
spring.datasource.username=root
spring.datasource.password=your_password
```

### 7.1.2 Microsoft Graph API Issues

#### Issue: Authentication Token Expired
**Symptom:**
```
401 Unauthorized
{"error":{"code":"InvalidAuthenticationToken","message":"Access token has expired"}}
```

**Root Cause:**
The cached access token has expired and the automatic refresh mechanism failed.

**Solution:**
1. Check if the `token.refresh.interval` is set correctly (default: 3300000)
2. Verify the client credentials are valid
3. Ensure network connectivity to Microsoft Graph
4. Check the token cache in `TokenUtils`

**Debug Steps:**
```java
// Check token cache
TokenData token = tokenCache.get(clientKey);
if (token == null || Instant.now().isAfter(token.expiryTime)) {
    // Token needs refresh
}
```

#### Issue: Rate Limiting
**Symptom:**
```
429 Too Many Requests
{"error":{"code":"TooManyRequests","message":"Application is throttled"}}
```

**Root Cause:**
The application has exceeded Microsoft Graph rate limits.

**Solution:**
1. Implement exponential backoff retry logic
2. Reduce concurrent API calls
3. Use batch operations where possible
4. Monitor and respect rate limit headers

**Rate Limit Handling:**
```java
// Implement retry with backoff
if (statusCode == 429) {
    long retryAfter = response.getHeader("Retry-After");
    Thread.sleep(retryAfter * 1000);
    // Retry request
}
```

### 7.1.3 Calendar and Event Issues

#### Issue: Timezone Conversion Errors
**Symptom:**
```
DateTimeParseException: Text could not be parsed at index X
Invalid timezone: America/New_York
```

**Root Cause:**
Invalid timezone format or unsupported timezone identifier.

**Solution:**
1. Validate timezone format using `ZoneId.of(timezone)`
2. Use IANA timezone identifiers (e.g., "America/New_York")
3. Check the `GRAPH_API_TO_IANA_TIMEZONE` mapping in `WorkingHoursUtil`
4. Provide fallback to UTC for invalid timezones

**Timezone Validation:**
```java
try {
    ZoneId zoneId = ZoneId.of(timezone);
    // Process timezone
} catch (DateTimeException e) {
    // Fallback to UTC
    ZoneId zoneId = ZoneId.of("UTC");
}
```

#### Issue: Meeting Scheduling Failures
**Symptom:**
```
400 Bad Request
{"error":{"code":"ErrorInvalidRecipients","message":"The specified recipients are invalid"}}
```

**Root Cause:**
Invalid attendee email addresses or insufficient permissions.

**Solution:**
1. Validate email addresses using `EmailUtils.validateEmailAddress()`
2. Check if attendees exist in the organization
3. Verify the user has permission to schedule meetings with these attendees
4. Ensure attendee email addresses are in the correct domain

### 7.1.4 User Management Issues

#### Issue: User Creation Failures
**Symptom:**
```
400 Bad Request
{"error":{"code":"Request_BadRequest","message":"Another object with the same value for property userPrincipalName already exists"}}
```

**Root Cause:**
Attempting to create a user with an existing `userPrincipalName`.

**Solution:**
1. Check if user already exists before creation
2. Use a unique `userPrincipalName` (e.g., add timestamp or random suffix)
3. Implement proper error handling for duplicate users
4. Consider using update instead of create for existing users

#### Issue: License Assignment Failures
**Symptom:**
```
400 Bad Request
{"error":{"code":"Request_BadRequest","message":"The license assignment failed"}}
```

**Root Cause:**
Invalid SKU ID or insufficient license availability.

**Solution:**
1. Verify the `graph.license.skuIds` configuration
2. Check if licenses are available in the tenant
3. Ensure the user has a valid account before license assignment
4. Validate SKU ID format and availability

### 7.1.5 EWS Integration Issues

#### Issue: EWS Authentication Failures
**Symptom:**
```
401 Unauthorized
SOAP fault: The request failed with HTTP status 401: Unauthorized
```

**Root Cause:**
Invalid EWS service account credentials or incorrect EWS URL.

**Solution:**
1. Verify `ews.serviceAccountUsername` and `ews.serviceAccountPassword`
2. Check `ews.ewsURL` is correct and accessible
3. Ensure the service account has proper permissions
4. Test EWS connectivity manually

**EWS Configuration Check:**
```properties
ews.serviceAccountUsername=pollservice@domain.com
ews.serviceAccountPassword=secure_password
ews.ewsURL=https://exchange.domain.com/EWS/Exchange.asmx
```

## Debugging Procedures

### 7.1.6 Startup Debugging
1. **Check Environment Variables**: Verify all required environment variables are set
2. **Validate Database**: Ensure MariaDB is running and accessible
3. **Test Graph Connectivity**: Verify Microsoft Graph API accessibility
4. **Review Logs**: Check application logs for specific error messages
5. **Validate Credentials**: Ensure encrypted credentials can be decrypted

### 7.1.7 Runtime Debugging
1. **Monitor Token Cache**: Check if tokens are properly cached and refreshed
2. **Validate API Calls**: Ensure all required parameters are provided
3. **Check External Services**: Verify Microsoft Graph, EWS, and Zoom APIs are accessible
4. **Review Error Messages**: Analyze error messages for specific failure points
5. **Monitor Performance**: Check response times and resource usage

### 7.1.8 Calendar Operation Debugging
1. **Timezone Validation**: Ensure all timezone conversions are correct
2. **Event Data Validation**: Verify event data format and required fields
3. **Attendee Validation**: Check attendee email addresses and permissions
4. **Availability Calculation**: Verify working hours and availability logic
5. **Conflict Detection**: Ensure meeting conflicts are properly detected

### 7.1.9 User Management Debugging
1. **User Data Validation**: Check user creation data format
2. **License Availability**: Verify license SKU IDs and availability
3. **Permission Checks**: Ensure proper permissions for user operations
4. **Domain Validation**: Check user email domains against allowed domains
5. **Bulk Operations**: Monitor bulk user creation and license assignment

---

*This section provides comprehensive troubleshooting guidance for common issues encountered when operating the Email Agent microservice.*
