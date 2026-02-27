# Authorization and RBAC

## What it is

**Authorization** is the process of determining **what an authenticated identity is allowed to do** — which resources they can access and which actions they can perform. It answers: "Can user-123 delete this post?" after authentication has already confirmed who user-123 is.

**Role-Based Access Control (RBAC)** is the dominant authorization model, where permissions are assigned to **roles**, and roles are assigned to users. Users inherit permissions through their roles.

---

## How it works

### Authorization Models

| Model | Description | Use Case |
|---|---|---|
| **RBAC** | Permissions via roles (user → roles → permissions) | Enterprise apps, SaaS, most systems |
| **ABAC** | Attribute-Based Access Control: policies use attributes of user, resource, environment | Fine-grained, contextual access (cloud IAM) |
| **ACL** | Access Control List: explicit per-resource per-user entries | Files, S3 bucket policies |
| **DAC** | Discretionary: resource owner grants access | Unix file permissions |
| **ReBAC** | Relationship-Based: access based on relationship graph (user "owns" resource) | Google Zanzibar, Google Docs |

### Core RBAC Model

```
Users ──► Roles ──► Permissions

User: alice
Roles: [editor, billing-viewer]
Permissions from editor:    [create:post, update:post, delete:own-post]
Permissions from billing-viewer: [read:invoice]
Alice's effective permissions: all of the above combined
```

```python
# RBAC data model
users = {
    'alice': {'roles': ['editor', 'billing-viewer']},
    'bob':   {'roles': ['admin']},
    'carol': {'roles': ['viewer']}
}

roles = {
    'admin': ['create:*', 'read:*', 'update:*', 'delete:*'],
    'editor': ['create:post', 'read:post', 'update:post', 'delete:own-post'],
    'billing-viewer': ['read:invoice'],
    'viewer': ['read:post']
}

def has_permission(user_id: str, permission: str) -> bool:
    user = users.get(user_id, {})
    for role in user.get('roles', []):
        perms = roles.get(role, [])
        if permission in perms:
            return True
        # Wildcard check: 'create:*' matches 'create:anything'
        if any(p.endswith(':*') and permission.startswith(p[:-1]) for p in perms):
            return True
    return False

has_permission('alice', 'create:post')  # True
has_permission('carol', 'create:post')  # False
has_permission('bob', 'delete:invoice') # True (admin)
```

### Hierarchical RBAC

Roles can inherit from parent roles:

```
viewer (read:post)
  └── editor (+ create:post, update:post)
        └── admin (+ delete:*, manage:users)
```

### ABAC (Attribute-Based Access Control)

ABAC evaluates policies based on attributes of the user, resource, and environment:

```python
# ABAC policy engine
def can_access(user, resource, action, context):
    # Example rules:
    
    # Rule 1: Users can edit their own resources
    if action == 'edit' and resource.owner_id == user.id:
        return True
    
    # Rule 2: Managers can read reports from their department
    if action == 'read' and resource.type == 'report':
        if 'manager' in user.roles and resource.department == user.department:
            return True
    
    # Rule 3: Admin can do anything during business hours
    if 'admin' in user.roles and 9 <= context.hour <= 18:
        return True
    
    return False

# More expressive than RBAC, but complex to manage.
# ABAC is the model behind AWS IAM policies.
```

```json
// AWS IAM policy (ABAC example):
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringEquals": {"aws:RequestedRegion": "us-east-1"},
    "IpAddress": {"aws:SourceIp": "192.168.0.0/24"}
  }
}
```

### Google Zanzibar (ReBAC)

Google's authorization system (used for Google Docs, Calendar, Drive) uses **Relationship-Based Access Control**:

```
Tuple: (object, relation, user)
Examples:
  (doc:report-2024, owner, user:alice)
  (doc:report-2024, viewer, user-group:finance-team)
  (folder:q4-reports, parent, doc:report-2024)

Query: "Can user:bob view doc:report-2024?"
  → Is bob in finance-team? → yes → finance-team has viewer on doc → YES
  → Or does doc:report-2024 inherit permissions from its parent folder? → checked transitively
```

Open-source Zanzibar implementations: **OpenFGA** (by Auth0/Okta), **Permify**, **Ory Keto**.

### JWT Claims for Authorization

Include authorization data in JWT payload:

```json
{
  "sub": "user-123",
  "roles": ["editor", "billing-viewer"],
  "permissions": ["create:post", "read:invoice"],
  "tenant_id": "org-456"
}
```

The service reads claims from the trusted JWT without a database call per request. Works well for coarse-grained authorization.

**Limitation**: JWT claims are baked at token issuance. If roles change mid-session, the token still carries old claims until it expires. For real-time role changes:
- Use short-lived tokens (15 min).
- Or call an authorization service (OPA, Casbin) on each request.

### Open Policy Agent (OPA)

OPA is a general-purpose policy engine. Write policies in Rego language:

```rego
# policy.rego
package authz

default allow = false

allow {
    input.user.roles[_] == "admin"
}

allow {
    input.action == "read"
    input.user.roles[_] == "viewer"
}

allow {
    input.action == "edit"
    input.user.id == input.resource.owner_id
}
```

```python
# Query OPA at runtime
import requests

response = requests.post('http://opa:8181/v1/data/authz/allow', json={
    "input": {
        "user": {"id": "alice", "roles": ["editor"]},
        "action": "edit",
        "resource": {"owner_id": "alice"}
    }
})
is_allowed = response.json()['result']  # True
```

OPA is used in Kubernetes (admission controllers), Istio (envoy auth), and as a sidecar for application-level policies.

---

## Key Trade-offs

| Model | Pros | Cons |
|---|---|---|
| RBAC | Simple, easy to audit | Role explosion with complex systems |
| ABAC | Fine-grained, contextual | Complex policies, harder to audit |
| ReBAC | Natural for ownership hierarchies | Complex query engine required |
| ACL | Simple per-resource control | Scales poorly to many users |

---

## When to use

- **RBAC**: business applications with clear user types (admin, manager, viewer). Most SaaS products.
- **ABAC**: cloud infrastructure (AWS IAM), systems with contextual access rules (time-of-day, IP, department).
- **ReBAC**: applications with hierarchical resource ownership (Google Drive, Notion, GitHub repositories).
- **OPA**: when authorization logic is complex and must be decoupled from service code.

---

## Common Pitfalls

- **Checking authorization in the wrong layer**: don't rely solely on UI hiding buttons — validate permissions server-side on every API call. The UI is not a security boundary.
- **Role explosion**: creating a new role for every tiny permission combination leads to unmaintainable RBAC. Use hierarchical roles and additive permission sets.
- **Coarse-grained roles in multi-tenant systems**: "admin" with blanket permissions in a multi-tenant SaaS can allow cross-tenant data access. Scope all permission checks to `tenant_id`.
- **Not logging authorization decisions**: authorization failures are the audit trail. Log denied access attempts with user, resource, action, and timestamp for security incident response.
