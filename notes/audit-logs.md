# Audit Logs

## Log Streaming

Stream to SIEM (Datadog, Splunk, Microsoft Sentinel), object storage (S3, GCS), or custom HTTPS endpoints.

## Events

| Action | Actor | Target |
|--------|-------|--------|
| invitation-created | User | Invitation |
| invitation-accepted | User | Invitation |
| invitation-revoked | User | Invitation |
| user-added | System | User |
| user-removed | User | User |
| api-key-created | User | API Key |
| api-key-marked-as-expired | User | API Key |
| session-created | User | Session |
| session-revoked | User | Session |

## Schemas

### AuditLogEvent

```typescript
type AuditLogEvent = {
  action: string;
  occurred_at: string;             // ISO 8601 datetime
  actor: Actor;
  targets: Target[];
  context: {
    location: string;              // client IP address
  };
  metadata: {
    session_id?: string;
    impersonator?: string;         // email of turbopuffer admin acting on behalf of customer
    impersonation_reason?: string;
  };
};

type Actor = User | System;
type Target = User | ApiKey | Invitation | Session;
```

### User

```typescript
type User = {
  type: "user";
  id: string;
  name: string;                    // email address
};
```

### API Key

```typescript
type ApiKey = {
  type: "api-key";
  id: string;
  name: string;                    // display name of the key
  metadata: {
    suffix: string;                // last 4 characters of the key
  };
};
```

### Invitation

```typescript
type Invitation = {
  type: "invitation";
  id: string;
  name: string;                    // email address of invited user
  metadata?: {
    invited_user_id: string;         
  };
};
```

### Session

```typescript
type Session = {
  type: "session";
  id: string;                        
  name: string;                    // session ID
  metadata: {
    user_agent: string;
    impersonator?: string;         // email of turbopuffer admin acting on behalf of customer
    impersonation_reason?: string;
  };
};
```

### System

```typescript
type System = {
  type: "system";
  id: "system";
  name: "System";
};
```

## Example Events

**API key created:**

```json
{
  "action": "api-key-created",
  "occurred_at": "2026-04-13T14:22:08Z",
  "actor": {
    "type": "user",
    "id": "V1StGXR8_Z5jdHi6B-myTq",
    "name": "ada@example.com"
  },
  "targets": [
    {
      "type": "api-key",
      "id": "8fW3zNcY6tRo1kGpLvAe2b/production",
      "name": "production",
      "metadata": { "suffix": "a1b2" }
    }
  ],
  "context": { "location": "203.0.113.42" },
  "metadata": {
    "session_id": "sess_lK9eT0vB3xYq2pNwA4fJ7"
  }
}
```

**API key expired:**

```json
{
  "action": "api-key-marked-as-expired",
  "occurred_at": "2026-04-13T14:25:11Z",
  "actor": {
    "type": "user",
    "id": "V1StGXR8_Z5jdHi6B-myTq",
    "name": "ada@example.com"
  },
  "targets": [
    {
      "type": "api-key",
      "id": "8fW3zNcY6tRo1kGpLvAe2b/production",
      "name": "production",
      "metadata": { "suffix": "a1b2" }
    }
  ],
  "context": { "location": "203.0.113.42" },
  "metadata": {
    "session_id": "sess_lK9eT0vB3xYq2pNwA4fJ7"
  }
}
```
