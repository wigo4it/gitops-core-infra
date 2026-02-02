# Keycloak Operator

## Overview

Keycloak is an open-source Identity and Access Management (IAM) solution that provides authentication, authorization, and user management capabilities. This Helm chart deploys Keycloak using the Bitnami/Codecentric chart.

**Official Documentation:** https://www.keycloak.org/documentation  
**Helm Chart:** https://github.com/codecentric/helm-charts/tree/master/charts/keycloakx  
**GitHub:** https://github.com/keycloak/keycloak

## What Does Keycloak Do?

Keycloak provides enterprise-grade identity and access management:

- **Single Sign-On (SSO)**: One login for multiple applications
- **Identity Brokering**: Social login (Google, GitHub, Azure AD, etc.)
- **User Federation**: LDAP, Active Directory integration
- **OAuth 2.0 / OpenID Connect**: Modern authentication protocols
- **SAML 2.0**: Support for legacy enterprise systems
- **User Management**: Self-service registration, password policies, 2FA
- **Authorization**: Fine-grained authorization with policies
- **Client Management**: Manage applications and services

### Common Use Cases

✅ **Microservices Authentication**: Centralized auth for microservice architectures  
✅ **Web Application SSO**: Single sign-on across multiple web apps  
✅ **API Gateway Integration**: JWT-based API authentication  
✅ **Mobile Apps**: OAuth 2.0 flows for mobile applications  
✅ **Enterprise SSO**: SAML federation with corporate identity providers  
✅ **Social Login**: Allow users to login with Google, Facebook, GitHub, etc.  

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Keycloak                           │
├──────────────────────────────────────────────────────┤
│                                                       │
│  ┌────────────────────────────────────────────────┐  │
│  │            Keycloak Server(s)                  │  │
│  │  ┌──────────────┐  ┌──────────────┐          │  │
│  │  │   Node 1     │  │   Node 2     │          │  │
│  │  │  (Master)    │──│  (Replica)   │          │  │
│  │  └──────────────┘  └──────────────┘          │  │
│  │         │                  │                   │  │
│  │         └──────────────────┘                   │  │
│  │         Infinispan Cache (Session Sharing)     │  │
│  └────────────────────────────────────────────────┘  │
│                      │                                │
│                      ▼                                │
│  ┌────────────────────────────────────────────────┐  │
│  │           PostgreSQL Database                  │  │
│  │  • Realms                                      │  │
│  │  • Users                                       │  │
│  │  • Clients                                     │  │
│  │  • Roles & Permissions                         │  │
│  └────────────────────────────────────────────────┘  │
│                                                       │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
          ┌─────────────────────────────────┐
          │     External Identity           │
          │     Providers (Optional)        │
          │  • Azure AD                     │
          │  • Google                       │
          │  • LDAP/AD                      │
          │  • SAML IdPs                    │
          └─────────────────────────────────┘
                         │
                         ▼
          ┌─────────────────────────────────┐
          │    Client Applications          │
          │  • Web Apps (OAuth/OIDC)        │
          │  • APIs (JWT tokens)            │
          │  • Mobile Apps                  │
          │  • Legacy Apps (SAML)           │
          └─────────────────────────────────┘
```

## Deployment Configuration

### Version Management

Versions are managed per-cluster in [.argocd.yaml](.argocd.yaml):

```yaml
clusters:
  local-dev:
    enabled: true
    chartVersion: "2.3.0"       # Latest for testing
  # IKV Clusters - Disabled
  aks-ikv-*:
    enabled: false              # Not deployed on IKV
  # Commonground Clusters - Enabled
  aks-commonground-nonprod:
    enabled: true
    chartVersion: "2.3.0"
  aks-commonground-prod:
    enabled: true
    chartVersion: "2.2.0"
```

**Deployment Scope:** Commonground clusters + local development only  
**Current Chart Version:** `2.3.0` (nonprod) / `2.2.0` (prod)  
**Chart Repository:** https://codecentric.github.io/helm-charts  
**Target Namespace:** `keycloak-system`

### Configuration Files

| File | Purpose | Applies To |
|------|---------|------------|
| [values.yaml](values.yaml) | Base configuration | All enabled clusters |
| [values.local-dev.yaml](values.local-dev.yaml) | Local development overrides | Local testing |
| [values.aks-commonground-nonprod.yaml](values.aks-commonground-nonprod.yaml) | Commonground pre-prod | nonprod |
| [values.aks-commonground-prod.yaml](values.aks-commonground-prod.yaml) | Commonground production | prod |

**Note:** No IKV values files - Keycloak is disabled for IKV clusters

## Key Configuration Options

### Resource Limits

**Default (base):**
```yaml
resources:
  limits: { cpu: 500m, memory: 512Mi }
  requests: { cpu: 100m, memory: 256Mi }
```

**Production:**
```yaml
resources:
  limits: { cpu: 2000m, memory: 2Gi }
  requests: { cpu: 1000m, memory: 1Gi }
```

### High Availability (Production)

Production deployments should use:

```yaml
replicas: 3  # Multiple instances

postgresql:
  enabled: true
  primary:
    persistence:
      enabled: true
      size: 50Gi

# Session replication with Infinispan
cache:
  enabled: true
  stack: kubernetes
```

### Database Configuration

**Development (local-dev):**
```yaml
postgresql:
  enabled: false  # Use H2 in-memory database
```

**Production:**
```yaml
postgresql:
  enabled: true
  auth:
    username: keycloak
    password: ${POSTGRES_PASSWORD}  # From secret
    database: keycloak
  primary:
    persistence:
      enabled: true
      storageClass: premium-ssd
      size: 50Gi
```

## Keycloak Configuration

### Creating a Realm

Realms are isolated spaces for managing users, clients, and settings:

```bash
# Access Keycloak admin console
kubectl port-forward svc/keycloak 8080:80 -n keycloak-system
# Open: http://localhost:8080

# Get admin credentials
kubectl get secret keycloak -n keycloak-system -o jsonpath='{.data.admin-password}' | base64 -d
```

**Via Admin UI:**
1. Login to admin console
2. Hover over realm dropdown (top-left)
3. Click "Add realm"
4. Enter name: `my-realm`
5. Click "Create"

### Creating a Client (Application)

Clients represent applications that use Keycloak:

**Web Application (Authorization Code Flow):**
```json
{
  "clientId": "my-webapp",
  "rootUrl": "https://my-app.example.com",
  "redirectUris": ["https://my-app.example.com/*"],
  "webOrigins": ["https://my-app.example.com"],
  "protocol": "openid-connect",
  "publicClient": false,
  "standardFlowEnabled": true
}
```

**API/Service (Client Credentials Flow):**
```json
{
  "clientId": "my-api-client",
  "protocol": "openid-connect",
  "publicClient": false,
  "serviceAccountsEnabled": true,
  "standardFlowEnabled": false
}
```

### User Management

**Create User:**
```bash
# Via API using Keycloak Admin CLI
kubectl exec -it keycloak-0 -n keycloak-system -- \
  /opt/keycloak/bin/kcadm.sh create users \
  -r my-realm \
  -s username=john.doe \
  -s email=john.doe@example.com \
  -s enabled=true
```

**Set Password:**
```bash
kubectl exec -it keycloak-0 -n keycloak-system -- \
  /opt/keycloak/bin/kcadm.sh set-password \
  -r my-realm \
  --username john.doe \
  --new-password mypassword
```

## Integration Examples

### Spring Boot Application

**application.yml:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: my-webapp
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            scope: openid, profile, email
        provider:
          keycloak:
            issuer-uri: https://keycloak.example.com/realms/my-realm
```

### Node.js Application

```javascript
const session = require('express-session');
const Keycloak = require('keycloak-connect');

const memoryStore = new session.MemoryStore();
const keycloak = new Keycloak({
  store: memoryStore
}, {
  realm: 'my-realm',
  'auth-server-url': 'https://keycloak.example.com/',
  'ssl-required': 'external',
  resource: 'my-webapp',
  'public-client': true
});

app.use(session({
  secret: 'secret',
  resave: false,
  saveUninitialized: true,
  store: memoryStore
}));

app.use(keycloak.middleware());

// Protected route
app.get('/protected', keycloak.protect(), (req, res) => {
  res.send('Protected resource');
});
```

### Python Application (FastAPI)

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2AuthorizationCodeBearer
from keycloak import KeycloakOpenID

keycloak_openid = KeycloakOpenID(
    server_url="https://keycloak.example.com/",
    client_id="my-api",
    realm_name="my-realm",
    client_secret_key="client-secret"
)

oauth2_scheme = OAuth2AuthorizationCodeBearer(
    authorizationUrl="https://keycloak.example.com/realms/my-realm/protocol/openid-connect/auth",
    tokenUrl="https://keycloak.example.com/realms/my-realm/protocol/openid-connect/token"
)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        return keycloak_openid.introspect(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/protected")
async def protected_route(user=Depends(get_current_user)):
    return {"message": "Protected resource", "user": user}
```

## Upgrading Keycloak

### Step-by-Step Process

⚠️ **Always backup database before upgrading!**

**1. Backup database:**
```bash
kubectl exec -it keycloak-postgresql-0 -n keycloak-system -- \
  pg_dump -U keycloak keycloak > keycloak-backup-$(date +%Y%m%d).sql
```

**2. Test locally:**
```bash
vim .argocd.yaml
# Change: local-dev: chartVersion: "2.3.0" → "2.4.0"

git commit -am "test(keycloak): upgrade to 2.4.0 locally"
git push
```

**3. Verify existing realms and clients still work:**
```bash
# Test login flows
# Verify API clients can still authenticate
# Check admin console accessibility
```

**4. Promote:** local-dev → nonprod → prod

### Version Compatibility

Check Keycloak version compatibility:  
https://www.keycloak.org/docs/latest/upgrading/

| Chart Version | Keycloak Version | Notes |
|--------------|------------------|-------|
| 2.3.x | 23.x | Current stable |
| 2.2.x | 22.x | Previous stable |
| 2.1.x | 21.x | Older version |

## Monitoring

### Health Checks

```bash
# Check Keycloak pods
kubectl get pods -n keycloak-system

# View logs
kubectl logs -n keycloak-system -l app.kubernetes.io/name=keycloak --tail=100

# Check database
kubectl get pods -n keycloak-system -l app.kubernetes.io/name=postgresql

# Access admin console
kubectl port-forward svc/keycloak 8080:80 -n keycloak-system
```

### Metrics

Keycloak exposes metrics at `/metrics` endpoint:
- `keycloak_logins_total` - Total login count
- `keycloak_failed_login_attempts_total` - Failed login attempts
- `keycloak_sessions_total` - Active sessions
- `keycloak_registrations_total` - User registrations

## Troubleshooting

### Keycloak Not Starting

**Symptoms:** Pods crash or fail to start

**Check:**
```bash
# Check pod status
kubectl describe pod -n keycloak-system -l app.kubernetes.io/name=keycloak

# View logs
kubectl logs -n keycloak-system -l app.kubernetes.io/name=keycloak

# Common issues:
# - Database connection failure
# - Insufficient memory
# - Configuration errors
```

### Database Connection Issues

**Symptoms:** Cannot connect to PostgreSQL

**Check:**
```bash
# Verify PostgreSQL is running
kubectl get pods -n keycloak-system -l app.kubernetes.io/name=postgresql

# Test database connection
kubectl exec -it keycloak-0 -n keycloak-system -- \
  pg_isready -h keycloak-postgresql -U keycloak

# Check database credentials in secret
kubectl get secret keycloak-postgresql -n keycloak-system -o yaml
```

### Login Issues

**Symptoms:** Users cannot login

**Check:**
- Verify realm is configured correctly
- Check client redirect URIs match application URLs
- Ensure user exists and is enabled
- Check for account lockout policies
- Review Keycloak logs for authentication errors

## Best Practices

✅ **Production**: Use external PostgreSQL with replication  
✅ **High Availability**: Run 3+ replicas with load balancing  
✅ **Security**: Always use HTTPS, enable HTTPS-only mode  
✅ **Secrets**: Store credentials in Kubernetes secrets or Azure Key Vault  
✅ **Backups**: Regular database backups before upgrades  
✅ **Monitoring**: Set up alerts for failed logins, system health  
✅ **Resource Limits**: Size appropriately based on user count  
✅ **Session Management**: Configure appropriate session timeouts  

## Additional Resources

- **Keycloak Docs:** https://www.keycloak.org/documentation
- **Admin Guide:** https://www.keycloak.org/docs/latest/server_admin/
- **Securing Apps:** https://www.keycloak.org/docs/latest/securing_apps/
- **Chart Repository:** https://github.com/codecentric/helm-charts
- **Community:** https://github.com/keycloak/keycloak/discussions

## Support

For deployment issues:
1. Check pod logs in `keycloak-system` namespace
2. Verify database connectivity
3. Review configuration in values files
4. Check resource constraints

For Keycloak-specific questions:
- GitHub Discussions: https://github.com/keycloak/keycloak/discussions
- Mailing List: https://lists.jboss.org/mailman/listinfo/keycloak-user
- Stack Overflow: Tag `keycloak`
