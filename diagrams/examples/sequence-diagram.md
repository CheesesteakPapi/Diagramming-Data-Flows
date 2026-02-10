# Sequence Diagram Example

This diagram shows the sequence of interactions in a user authentication flow.

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant A as Auth Service
    participant D as Database

    U->>F: Enter credentials
    F->>A: POST /login
    A->>D: Query user
    D-->>A: User data
    A->>A: Verify password
    alt Authentication successful
        A-->>F: JWT Token
        F-->>U: Login success
    else Authentication failed
        A-->>F: Error message
        F-->>U: Login failed
    end
```

## Authentication Flow

1. User enters credentials in the frontend
2. Frontend sends login request to auth service
3. Auth service queries database for user
4. Password verification is performed
5. On success, JWT token is returned
6. On failure, error message is shown
