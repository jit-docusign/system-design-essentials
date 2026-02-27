# Zero Trust Architecture

## What it is

**Zero Trust** is a security model built on the principle: **"Never trust, always verify."** Instead of trusting everything inside a corporate network perimeter (the traditional "castle and moat" model), Zero Trust assumes the network is always hostile — whether traffic originates from the internet or from inside the data center.

Every request for access must be:
1. **Authenticated** — who is this identity (user, service, device)?
2. **Authorized** — do they have permission for this specific resource?
3. **Continuously evaluated** — not just at login, but on every request.

Google's BeyondCorp (2014) pioneered Zero Trust for corporate access. The NSA and NIST now mandate it for US federal systems (NIST SP 800-207).

---

## How it works

### The Contrast: Perimeter vs Zero Trust

```
Traditional Perimeter Model:
┌─────────────────────────────────────────────────┐
│  "Trusted" Internal Network                     │
│                                                 │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐       │
│  │  App │  │  DB  │  │  App │  │  App │       │
│  └──────┘  └──────┘  └──────┘  └──────┘       │
│     ↑           ↑        ↑         ↑           │
│   trust all lateral movement                   │
│                                                 │
└─────────────────────────────────────────────────┘
              🏰 Firewall 🏰
         ↑
  Internet (untrusted)
Problem: Once inside (via VPN, phishing, insider threat),
         attacker can move freely to any internal resource.
```

```
Zero Trust Model:
                                         Policy Engine
                                         (Identity, Device, Context)
Internet         Identity-Aware Proxy         │
  │                      │                   ▼
  │  Every request ──────►│ ──────── Authorize? ──────►  App/Resource
  │  carries identity     │ ◄─────── YES/NO ◄─────────
  │  (user + device)      │
  │                       │    Each service also verifies
  │                       │    the next hop (mTLS)

No implicit trust based on network location.
```

### Five Pillars of Zero Trust

| Pillar | Description | Technologies |
|---|---|---|
| **Identity** | Verify every user, service, workload | SSO (OIDC), MFA, SPIFFE/SPIRE |
| **Device** | Verify device health and posture | MDM, certificate-based device auth, EDR |
| **Network** | Microsegmentation, encrypt all traffic | mTLS, network policies, eBPF |
| **Application** | App-layer authorization, per-request | OPA, Identity-Aware Proxy |
| **Data** | Classify and protect data itself | DLP, encryption at rest, RBAC |

### Identity-Aware Proxy (IAP)

An IAP sits in front of all applications and enforces identity-based access. There is no VPN:

```
                         ┌─────────────────────┐
                         │  Identity-Aware Proxy│
User ── HTTPS ──────────►│  1. AuthN (OIDC SSO) │──────► App
(no VPN needed)          │  2. AuthZ (check IAM)│
                         │  3. Audit log        │
                         └─────────────────────┘

If user is not authenticated → redirect to SSO login
If user is authenticated but not authorized → 403
User context (email, groups, device cert) forwarded to app as headers
```

```python
# IAP-protected app reads forwarded identity
from flask import Flask, request

app = Flask(__name__)

@app.route('/api/data')
def get_data():
    # IAP sets these headers after verifying the JWT
    user_email = request.headers.get('X-Goog-Authenticated-User-Email')
    user_id = request.headers.get('X-Goog-Authenticated-User-ID')
    
    # Additional app-level authorization
    if not is_authorized(user_email, resource='data', action='read'):
        return {'error': 'Forbidden'}, 403
    
    return {'data': '...', 'accessed_by': user_email}
```

Open-source IAP implementations: **Pomerium**, **Ory Oathkeeper**, **Teleport**.

### Microsegmentation

Divide the network into small segments where services can only communicate with explicitly allowed peers:

```
Without microsegmentation:                With microsegmentation:
┌────────────────────────┐                ┌─────────────────────────┐
│  Order Service  ───────────────────►    │  Order Service          │
│  │                     │                │  │  NetworkPolicy:       │
│  Payment Service       │                │  │  allow → Payment only │
│  │                     │                │  ▼                       │
│  Inventory Service     │                │  Payment Service         │
│  │                     │                │  (NetworkPolicy: allow   │
│  HR Database   ◄──────────────────      │   → DB only)             │
│  (no business reason)  │                │                          │
└────────────────────────┘                │  HR DB: no route from   │
                                          │  Order Service at all    │
                                          └─────────────────────────┘
```

```yaml
# Kubernetes NetworkPolicy: Payment service only accepts from Order service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-policy
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: order-service
      ports:
        - protocol: TCP
          port: 8080
```

### mTLS (Mutual TLS) Everywhere

Standard TLS only authenticates the server. **mTLS** adds client authentication — both sides present certificates:

```
Service A                              Service B
  │── TLS ClientHello ───────────────► │
  │◄── ServerHello + Cert ─────────── │
  │ [Verify B's cert against CA]       │
  │── Client Cert + ClientKeyExchange ►│
  │                                    │ [Verify A's cert against CA]
  │◄──── Finished ──────────────────── │
  │═══════ Encrypted + Mutually Auth'd ════
```

In Kubernetes, **SPIFFE** (Secure Production Identity Framework For Everyone) issues **SVIDs** (SPIFFE Verifiable Identity Documents) to workloads — cryptographic identities tied to the service itself, not an IP address:

```
SPIFFE ID format: spiffe://trust-domain/path
Example:          spiffe://example.org/ns/default/sa/payment-service

Each pod gets a short-lived X.509 certificate signed by the SPIFFE CA (SPIRE).
Istio/Linkerd auto-rotate these certificates and enforce mTLS between all services.
```

### Continuous Verification & Contextual Access

Zero Trust access decisions re-evaluate continuously, not just at login:

```python
# Conceptual policy decision for each request
def make_access_decision(request_context: dict) -> bool:
    user = lookup_user(request_context['user_id'])
    device = lookup_device(request_context['device_id'])
    
    # Identity signals
    if not user.mfa_verified_recently:
        return False  # Require step-up auth
    
    # Device posture
    if device.os_version < MINIMUM_OS_VERSION:
        return False  # Unpatched OS
    if device.last_security_scan_hours_ago > 24:
        return False  # Stale EDR scan
    
    # Behavioral signals
    risk_score = compute_risk_score(
        user=user,
        location=request_context['geo'],
        time=request_context['timestamp'],
        resource=request_context['resource'],
    )
    if risk_score > THRESHOLD:
        trigger_step_up_auth(user)
        return False  # Force re-authentication
    
    return True
```

### BeyondCorp Architecture (Google's Implementation)

```
Flow:
1. User opens laptop (no VPN)
2. Request hits frontend proxy (GFE — not an IAP)
3. Access Proxy checks:
   a. Is the device in the device inventory? (cert on device)
   b. Is the user authenticated? (OIDC cookie / JWT)
   c. Is the user + device authorized for this app/resource?
4. If all checks pass → request forwarded to internal app
5. Internal app trusts forwarded identity headers (not the network)
6. All inter-service calls use mTLS (service-to-service)

Components:
- Device Inventory Service: tracks managed devices + their certs
- User Database + SSO: identity provider (Okta, Google Workspace)
- Access Policy Engine: evaluates user + device + resource
- Access Proxy: the single enforcement point (IAP)
```

---

## Key Trade-offs

| Choice | Pros | Cons |
|---|---|---|
| **Full Zero Trust** | Blast radius minimized, insider threats neutralized | Complex to implement, significant infra changes |
| **Perimeter-only security** | Simple, low overhead | Single point of failure; lateral movement unchecked after breach |
| **mTLS everywhere** | No implicit trust between services | Certificate lifecycle management complexity |
| **IAP vs VPN** | Per-app access control, no network-wide trust | IAP must handle all auth; availability-critical component |
| **Continuous evaluation** | Detects anomalies in active sessions | Latency overhead on every request; complex policy engine |

---

## When to use

- **Remote workforce**: when users work outside the corporate network, Zero Trust is more practical than VPN (which grants too-broad network access).
- **Multi-cloud / hybrid**: perimeter security is meaningless when workloads span AWS, GCP, and on-prem. Zero Trust applies consistently everywhere.
- **Sensitive data**: healthcare (HIPAA), finance (PCI-DSS), government — where insider threats and lateral movement attacks must be prevented.
- **Microservices**: when dozens of services communicate, uncontrolled lateral movement is a major risk. mTLS + network policies implement Zero Trust at the service mesh layer.

---

## Common Pitfalls

- **Implementing only the network layer**: Zero Trust fails if you microsegment the network but still grant every service the same DB credentials. Identity, not network location, is the control plane.
- **Over-relying on VPN as Zero Trust**: moving from a firewall to a VPN is not Zero Trust. VPN grants broad network access once authenticated; Zero Trust grants per-resource access with continuous validation.
- **Ignoring device identity**: Zero Trust requires both user identity AND device health. An authenticated user on a compromised device is still a threat. Integrate MDM and certificate-based device authentication.
- **Not logging everything**: Zero Trust is "always verify" — but verification is only valuable if you log every access decision and have anomaly detection on those logs. Comprehensive audit logs are non-negotiable.
- **Treating Zero Trust as a product**: Zero Trust is a strategy, not a single vendor product. Avoid "zero trust washing" — buying a single tool and declaring done. It requires continuous improvement across all five pillars.
