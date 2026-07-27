# Spring Security Process - Step by Step Guide

## Table of Contents
1. [Overview](#overview)
2. [Authentication Flow](#authentication-flow)
3. [Authorization Flow](#authorization-flow)
4. [JWT Token Process](#jwt-token-process)
5. [Security Filter Chain](#security-filter-chain)
6. [Role-Based Access Control](#role-based-access-control)
7. [Implementation Details](#implementation-details)

---

## Overview

Spring Security is a powerful and highly customizable authentication and access-control framework. This document outlines the complete process of Spring Security with JWT tokens and role-based access control using SecurityFilterChain.

### Key Components
- **Authentication**: Verifying the identity of a user
- **Authorization**: Determining what authenticated users are allowed to do
- **JWT Token**: JSON Web Token for stateless authentication
- **SecurityFilterChain**: Chain of filters that process security-related requests
- **Roles**: User roles for fine-grained access control

---

## Authentication Flow

### Step 1: User Login Request
```
Client sends HTTP POST request with credentials:
{
  "username": "user@example.com",
  "password": "password123"
}
```

### Step 2: Authentication Manager Processes Credentials
1. Request is intercepted by Spring Security filters
2. `UsernamePasswordAuthenticationFilter` extracts username and password
3. `AuthenticationManager` is called to authenticate the user

### Step 3: UserDetailsService Lookup
1. `AuthenticationManager` delegates to `UserDetailsService`
2. `UserDetailsService.loadUserByUsername()` retrieves user from database
3. Returns `UserDetails` object containing:
   - Username
   - Password (encrypted)
   - Authorities (roles)
   - Account status flags

### Step 4: Password Verification
1. `PasswordEncoder` compares provided password with stored encrypted password
2. Uses BCrypt, Argon2, or other encoding algorithms
3. If passwords match, authentication succeeds

### Step 5: Generate Authentication Object
1. `Authentication` object is created with:
   - Principal: User details
   - Credentials: User password (cleared after authentication)
   - Authorities: User roles/permissions
   - Authenticated: true

### Step 6: Return JWT Token
1. If authentication successful, `AuthenticationProvider` returns `Authentication` object
2. `AuthenticationSuccessHandler` is invoked
3. JWT token is generated with:
   - User information (claims)
   - Expiration time
   - Digital signature

---

## JWT Token Process

### JWT Structure

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Three Parts:**

1. **Header (Base64 Encoded)**
   - Algorithm: HS256 (HMAC with SHA-256)
   - Token type: JWT

2. **Payload (Claims - Base64 Encoded)**
   - `sub`: Subject (user ID)
   - `name`: User name
   - `iat`: Issued at (timestamp)
   - `exp`: Expiration time
   - `roles`: User roles
   - Custom claims

3. **Signature (Base64 Encoded)**
   - Created by encoding header and payload
   - Signed with server's secret key
   - Ensures token integrity

### JWT Token Validation

```
Step 1: Decode token (Base64 decode header & payload)
Step 2: Extract signature from token
Step 3: Recreate signature using secret key
Step 4: Compare original signature with recreated signature
Step 5: Verify token hasn't expired
Step 6: Extract claims and create Authentication object
```

---

## Authorization Flow

### Step 1: Request with JWT Token
```
Client sends HTTP request with Authorization header:
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Step 2: JWT Filter Intercepts Request
1. `JwtAuthenticationFilter` or similar custom filter intercepts request
2. Extracts JWT token from Authorization header
3. Validates token signature and expiration

### Step 3: Token Validation
1. **Signature Verification**: Recreate signature and compare
2. **Expiration Check**: Verify token hasn't expired
3. **Claims Extraction**: Extract user ID, roles, and other claims

### Step 4: Create Authentication Object
1. If token is valid, extract user details
2. Load user from database (or cache) using user ID
3. Create `Authentication` object with:
   - User principal
   - Authorities (roles from token)
   - Authenticated: true

### Step 5: Set Authentication in SecurityContext
```java
SecurityContextHolder.getContext().setAuthentication(authentication);
```

### Step 6: Access Control Decision
1. Request is passed through `AccessDecisionManager`
2. `ConfigAttributes` define required roles for endpoint
3. Compare user's authorities with required authorities

### Step 7: Grant or Deny Access
- **Allow**: User has required role → Continue to controller
- **Deny**: User lacks required role → Throw `AccessDeniedException`

---

## Security Filter Chain

### Default Filter Order

```
1. SecurityContextPersistenceFilter
   ├─ Restores SecurityContext from cache
   └─ Saves SecurityContext after request

2. HeaderWriterFilter
   └─ Adds security headers to response

3. LogoutFilter
   └─ Processes logout requests

4. UsernamePasswordAuthenticationFilter
   ├─ Only for form login
   └─ Creates Authentication from username/password

5. DefaultLoginPageGeneratingFilter
   └─ Generates default login page

6. DefaultLogoutPageGeneratingFilter
   └─ Generates default logout page

7. BasicAuthenticationFilter
   └─ Processes HTTP Basic authentication

8. RequestCacheAwareFilter
   └─ Restores original request after login

9. SecurityContextHolderAwareRequestFilter
   └─ Wraps request to provide security methods

10. AnonymousAuthenticationFilter
    └─ Creates anonymous authentication if none exists

11. SessionManagementFilter
    └─ Manages session for authenticated user

12. ExceptionTranslationFilter
    ├─ Catches Spring Security exceptions
    └─ Handles authentication/authorization failures

13. FilterSecurityInterceptor
    ├─ Enforces URL-based authorization
    └─ Checks if user has permission for URL
```

### Custom JWT Filter Position

```
SecurityFilterChain
    │
    ├─ BasicAuthenticationFilter
    │
    ├─ JwtAuthenticationFilter (Custom - added here)
    │  ├─ Extracts JWT from Authorization header
    │  ├─ Validates JWT signature
    │  ├─ Creates Authentication object
    │  └─ Sets in SecurityContext
    │
    ├─ AnonymousAuthenticationFilter
    │
    └─ FilterSecurityInterceptor
       ├─ Checks URL authorization
       └─ Denies unauthorized access
```

---

## Role-Based Access Control (RBAC)

### Step 1: Define Roles

```java
public enum Role {
    ADMIN("ADMIN"),
    USER("USER"),
    MANAGER("MANAGER");
    
    private final String authority;
    
    Role(String authority) {
        this.authority = "ROLE_" + authority;
    }
}
```

### Step 2: Assign Roles to Users

```
User Entity:
    id: 1
    username: "john"
    roles: [ROLE_ADMIN, ROLE_USER]

User Entity:
    id: 2
    username: "jane"
    roles: [ROLE_USER]
```

### Step 3: Include Roles in JWT Token

```java
claims.put("roles", user.getRoles());
// Payload includes:
// "roles": ["ROLE_ADMIN", "ROLE_USER"]
```

### Step 4: Extract Roles from Token

```java
List<String> roles = token.getClaim("roles").asList(String.class);
List<GrantedAuthority> authorities = roles.stream()
    .map(SimpleGrantedAuthority::new)
    .collect(Collectors.toList());
```

### Step 5: Apply Authorization Rules

```java
@PostMapping("/api/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<User> createUser(@RequestBody User user) {
    // Only ADMIN role can access
    return ResponseEntity.ok(userService.save(user));
}

@GetMapping("/api/user/profile")
@PreAuthorize("hasAnyRole('ADMIN', 'USER')")
public ResponseEntity<User> getUserProfile() {
    // ADMIN or USER role can access
    return ResponseEntity.ok(userService.getCurrentUser());
}

@DeleteMapping("/api/admin/users/{id}")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    // Only ADMIN role can access
    userService.deleteById(id);
    return ResponseEntity.ok().build();
}
```

---

## Implementation Details

### 1. SecurityConfig Configuration

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeHttpRequests()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .addFilterBefore(jwtAuthenticationFilter(), 
                             UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter(jwtTokenProvider());
    }
    
    @Bean
    public JwtTokenProvider jwtTokenProvider() {
        return new JwtTokenProvider();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 2. JWT Token Provider

```java
@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration}")
    private int jwtExpirationMs;
    
    public String generateToken(Authentication authentication) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        
        return Jwts.builder()
            .setSubject(userPrincipal.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
            .claim("roles", userPrincipal.getAuthorities())
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public String getUsernameFromToken(String token) {
        return Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

### 3. JWT Authentication Filter

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenProvider tokenProvider;
    
    public JwtAuthenticationFilter(JwtTokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        try {
            String jwt = getJwtFromRequest(request);
            
            if (jwt != null && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromToken(jwt);
                
                // Load user details and create Authentication
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                List<GrantedAuthority> authorities = 
                    (List<GrantedAuthority>) userDetails.getAuthorities();
                
                Authentication authentication = new UsernamePasswordAuthenticationToken(
                    userDetails, null, authorities);
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("Could not set user authentication", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 4. Login Controller

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                loginRequest.getUsername(),
                loginRequest.getPassword()));
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        String jwt = tokenProvider.generateToken(authentication);
        
        return ResponseEntity.ok(new AuthResponse(jwt));
    }
}
```

### 5. User Details Service

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) 
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));
        
        return UserPrincipal.create(user);
    }
}
```

---

## Complete Request-Response Cycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    1. LOGIN REQUEST                              │
│  POST /api/auth/login                                            │
│  {                                                               │
│    "username": "john",                                           │
│    "password": "password123"                                     │
│  }                                                               │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│            2. AUTHENTICATION PROCESSING                          │
│  └─ UsernamePasswordAuthenticationFilter intercepts              │
│  └─ AuthenticationManager processes credentials                 │
│  └─ UserDetailsService loads user from DB                       │
│  └─ PasswordEncoder verifies password                            │
│  └─ Authentication object created with roles                    │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│               3. JWT TOKEN GENERATION                            │
│  └─ Claims populated (username, roles, expiration)              │
│  └─ Token signed with secret key                                │
│  └─ JWT returned in response                                    │
│                                                                  │
│  RESPONSE:                                                       │
│  {                                                               │
│    "token": "eyJhbGciOiJIUzI1NiIs..."                           │
│  }                                                               │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│             4. API REQUEST WITH JWT TOKEN                        │
│  GET /api/user/profile                                          │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...                  │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│          5. JWT FILTER PROCESSES TOKEN                           │
│  └─ JwtAuthenticationFilter extracts token                      │
│  └─ Token signature validated                                   │
│  └─ Token expiration checked                                    │
│  └─ Claims extracted (username, roles)                          │
│  └─ User loaded from database                                   │
│  └─ Authentication object created                               │
│  └─ Set in SecurityContext                                      │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│         6. AUTHORIZATION CHECK                                   │
│  └─ FilterSecurityInterceptor checks URL rules                  │
│  └─ Compare user roles with required roles                      │
│  └─ @PreAuthorize evaluated                                     │
│  └─ Access granted/denied                                       │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│        7. EXECUTE CONTROLLER METHOD                              │
│  └─ UserPrincipal available in controller                       │
│  └─ Process business logic                                      │
│  └─ Return response                                             │
│                                                                  │
│  RESPONSE:                                                       │
│  {                                                               │
│    "id": 1,                                                      │
│    "username": "john",                                           │
│    "email": "john@example.com",                                  │
│    "roles": ["ROLE_ADMIN", "ROLE_USER"]                         │
│  }                                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Security Best Practices

1. **Secret Key Management**
   - Store secret key in environment variables
   - Use strong secret key (at least 256 bits)
   - Never commit secret key to version control

2. **Token Expiration**
   - Set reasonable expiration time (15-60 minutes)
   - Use refresh tokens for long-lived sessions
   - Implement token refresh mechanism

3. **HTTPS Only**
   - Always use HTTPS in production
   - Prevents token interception
   - Required for secure communication

4. **Password Encoding**
   - Use BCrypt with sufficient rounds (10+)
   - Never store plain text passwords
   - Use strong password policy

5. **CORS Configuration**
   - Explicitly configure allowed origins
   - Restrict allowed methods (GET, POST, etc.)
   - Validate incoming requests

6. **SQL Injection Prevention**
   - Use parameterized queries
   - Use ORM frameworks (Hibernate, JPA)
   - Validate all user input

7. **Logout Mechanism**
   - Invalidate tokens on logout
   - Implement token blacklist/whitelist
   - Clear SecurityContext

8. **Error Handling**
   - Don't expose sensitive information
   - Log security events
   - Handle exceptions gracefully

---

## Useful Annotations

```java
@PreAuthorize("hasRole('ADMIN')")
// Method execution allowed only if user has ADMIN role

@PreAuthorize("hasAnyRole('ADMIN', 'USER')")
// Method execution allowed if user has ADMIN or USER role

@PreAuthorize("#id == authentication.principal.id")
// Method execution allowed if parameter ID matches current user ID

@PostAuthorize("returnObject.owner == authentication.principal")
// Checks permission after method execution

@Secured("ROLE_ADMIN")
// Legacy annotation, use @PreAuthorize instead

@PermitAll
// All users can access

@DenyAll
// No user can access, even authenticated users
```

---

## Troubleshooting Common Issues

### 1. Token Not Being Recognized
- Check Authorization header format: `Bearer <token>`
- Verify JWT secret key matches
- Check token expiration time

### 2. 401 Unauthorized Errors
- Verify JWT token is valid
- Check token hasn't expired
- Ensure token signature is correct

### 3. 403 Forbidden Errors
- Check user roles/permissions
- Verify @PreAuthorize conditions
- Check SecurityFilterChain configuration

### 4. User Not Found
- Verify username in JWT matches database
- Check UserDetailsService implementation
- Ensure database connection is working

### 5. CORS Errors
- Configure CORS properly in SecurityConfig
- Check allowed origins
- Verify request headers

---

## References

- [Spring Security Documentation](https://spring.io/projects/spring-security)
- [JWT Introduction](https://jwt.io/introduction)
- [OWASP Authentication](https://owasp.org/www-community/authentication)
- [Spring Security Architecture](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-architecture)

---

## Version Information

- Spring Boot: 3.x+
- Spring Security: 6.x+
- Java: 11+
- JWT Library: jjwt (JSON Web Token)

---

**Last Updated**: 2024
**Author**: Spring Security Documentation & Best Practices
