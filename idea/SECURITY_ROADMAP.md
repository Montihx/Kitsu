# Security Roadmap - Backend Ani

**Текущий Security Score:** 9/10  
**Целевой Security Score:** 10/10  
**Дата:** 2026-01-05

---

## 🔒 Текущее состояние безопасности

### ✅ Реализовано

| Feature | Status | Implementation |
|---------|--------|----------------|
| JWT Authentication | ✅ Complete | python-jose, RS256 |
| Password Hashing | ✅ Complete | bcrypt, secure salting |
| Role-Based Access Control | ✅ Complete | User, Admin, Moderator |
| Rate Limiting | ✅ Complete | Middleware, 60 req/min |
| Input Validation | ✅ Complete | Pydantic всюду |
| SQL Injection Prevention | ✅ Complete | SQLAlchemy ORM |
| CORS Configuration | ✅ Complete | Настраиваемый origins |

---

## 🎯 Phase 7: Security Enhancements

### Priority 1: Audit Logging System (HIGH)

**Цель:** Отслеживание всех критических операций для compliance и forensics.

**Scope:**
- User authentication events (login, logout, failed attempts)
- Admin actions (create, update, delete operations)
- Permission changes (role assignments)
- Data access logging (sensitive endpoints)
- Configuration changes

**Implementation Plan:**

#### 1.1 Database Schema
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id VARCHAR(255),
    result VARCHAR(50) NOT NULL, -- 'success', 'failure', 'denied'
    ip_address INET,
    user_agent TEXT,
    request_data JSONB,
    response_data JSONB,
    INDEX idx_audit_user (user_id, timestamp),
    INDEX idx_audit_action (action, timestamp),
    INDEX idx_audit_resource (resource_type, resource_id)
);
```

#### 1.2 Audit Logger Service
```python
# src/infrastructure/security/audit_log.py

from uuid import UUID
from datetime import datetime
from typing import Optional, Dict, Any
from sqlalchemy.ext.asyncio import AsyncSession

class AuditLogger:
    """
    Сервис для audit logging критических операций.
    
    Записывает все действия пользователей для последующего анализа,
    compliance требований и forensics investigations.
    """
    
    async def log_authentication(
        self,
        user_id: Optional[UUID],
        action: str,  # 'login', 'logout', 'login_failed'
        result: str,  # 'success', 'failure'
        ip_address: str,
        user_agent: str
    ):
        """Логирует события authentication."""
        pass
    
    async def log_admin_action(
        self,
        admin_id: UUID,
        action: str,  # 'create', 'update', 'delete'
        resource_type: str,  # 'release', 'user', 'comment'
        resource_id: str,
        result: str,
        ip_address: str
    ):
        """Логирует административные действия."""
        pass
    
    async def log_data_access(
        self,
        user_id: UUID,
        resource_type: str,
        resource_id: str,
        action: str,  # 'read', 'export'
        ip_address: str
    ):
        """Логирует доступ к чувствительным данным."""
        pass
```

#### 1.3 Integration с Endpoints
```python
@router.post("/admin/releases")
async def create_release(
    release_data: ReleaseCreate,
    user_id: UUID = Depends(require_role(UserRole.ADMIN)),
    audit_logger: AuditLogger = Depends(get_audit_logger),
    request: Request = None
):
    """Создание релиза с audit logging."""
    try:
        release = await uow.releases.create(release_data)
        
        await audit_logger.log_admin_action(
            admin_id=user_id,
            action="create",
            resource_type="release",
            resource_id=str(release.id),
            result="success",
            ip_address=request.client.host
        )
        
        return release
    except Exception as e:
        await audit_logger.log_admin_action(
            admin_id=user_id,
            action="create",
            resource_type="release",
            resource_id=None,
            result="failure",
            ip_address=request.client.host
        )
        raise
```

**Deliverables:**
- [ ] Database migration для audit_logs table
- [ ] AuditLogger service implementation
- [ ] Integration в authentication endpoints
- [ ] Integration в admin endpoints
- [ ] Audit log viewer для administrators
- [ ] Retention policy (365 days)

**Timeline:** 3-5 days  
**Dependencies:** PostgreSQL, SQLAlchemy  
**Priority:** HIGH

---

### Priority 2: Two-Factor Authentication (2FA) (MEDIUM)

**Цель:** Дополнительный уровень безопасности для user accounts.

**Scope:**
- TOTP (Time-based One-Time Password) support
- QR code generation для setup
- Backup codes для recovery
- SMS fallback (optional, requires Twilio/similar)

**Implementation Plan:**

#### 2.1 Database Schema
```sql
ALTER TABLE users ADD COLUMN two_factor_enabled BOOLEAN DEFAULT FALSE;
ALTER TABLE users ADD COLUMN two_factor_secret VARCHAR(32);

CREATE TABLE backup_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    code VARCHAR(10) NOT NULL,
    used BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_backup_user (user_id)
);
```

#### 2.2 TOTP Service
```python
# src/infrastructure/security/totp.py

import pyotp
import qrcode
from io import BytesIO

class TOTPService:
    """
    Сервис для Two-Factor Authentication с TOTP.
    
    Использует pyotp для генерации и верификации одноразовых кодов.
    Совместим с Google Authenticator, Authy, и другими TOTP apps.
    """
    
    def generate_secret(self) -> str:
        """Генерирует новый secret для пользователя."""
        return pyotp.random_base32()
    
    def generate_qr_code(self, username: str, secret: str) -> bytes:
        """Генерирует QR код для настройки в authenticator app."""
        totp_uri = pyotp.totp.TOTP(secret).provisioning_uri(
            name=username,
            issuer_name="Backend Ani"
        )
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(totp_uri)
        qr.make(fit=True)
        
        img = qr.make_image(fill_color="black", back_color="white")
        buffer = BytesIO()
        img.save(buffer, format="PNG")
        return buffer.getvalue()
    
    def verify_code(self, secret: str, code: str) -> bool:
        """Верифицирует TOTP код."""
        totp = pyotp.TOTP(secret)
        return totp.verify(code, valid_window=1)  # ±30 seconds
    
    def generate_backup_codes(self, count: int = 10) -> List[str]:
        """Генерирует backup codes для recovery."""
        import secrets
        return [
            f"{secrets.randbelow(10000):04d}-{secrets.randbelow(10000):04d}"
            for _ in range(count)
        ]
```

#### 2.3 2FA Endpoints
```python
@router.post("/auth/2fa/enable")
async def enable_2fa(
    user_id: UUID = Depends(get_current_user_id),
    totp_service: TOTPService = Depends(get_totp_service)
):
    """
    Включает 2FA для пользователя.
    
    Returns:
        - secret: для ручного ввода
        - qr_code: base64 encoded QR код
        - backup_codes: список backup codes
    """
    pass

@router.post("/auth/2fa/verify")
async def verify_2fa_setup(
    code: str,
    user_id: UUID = Depends(get_current_user_id)
):
    """Верифицирует 2FA setup код."""
    pass

@router.post("/auth/login/2fa")
async def login_with_2fa(
    username: str,
    password: str,
    totp_code: str
):
    """Login с 2FA verification."""
    pass
```

**Deliverables:**
- [ ] Database migration для 2FA fields
- [ ] TOTPService implementation
- [ ] QR code generation
- [ ] Backup codes system
- [ ] 2FA setup flow (UI/API)
- [ ] 2FA verification в login
- [ ] Recovery mechanism
- [ ] Documentation для users

**Timeline:** 5-7 days  
**Dependencies:** pyotp, qrcode, Pillow  
**Priority:** MEDIUM

---

### Priority 3: API Key Management (MEDIUM)

**Цель:** Secure API access для third-party integrations.

**Scope:**
- API key generation и rotation
- Scope-based permissions (read, write, admin)
- Rate limiting per key
- Usage tracking и analytics
- Key expiration policies

**Implementation Plan:**

#### 3.1 Database Schema
```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    key_hash VARCHAR(255) NOT NULL,  -- bcrypt hash
    key_prefix VARCHAR(8) NOT NULL,  -- для UI display (e.g. "ani_1234...")
    scopes TEXT[] NOT NULL,  -- ['read:releases', 'write:favorites']
    rate_limit INT DEFAULT 1000,  -- requests per hour
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_apikey_user (user_id),
    INDEX idx_apikey_hash (key_hash),
    UNIQUE (key_hash)
);

CREATE TABLE api_key_usage (
    id BIGSERIAL PRIMARY KEY,
    api_key_id UUID REFERENCES api_keys(id) ON DELETE CASCADE,
    endpoint VARCHAR(255) NOT NULL,
    timestamp TIMESTAMP DEFAULT NOW(),
    ip_address INET,
    response_status INT,
    INDEX idx_usage_key_time (api_key_id, timestamp)
);
```

#### 3.2 API Key Service
```python
# src/infrastructure/security/api_keys.py

import secrets
import hashlib
from typing import List, Optional

class APIKeyService:
    """
    Сервис для управления API keys.
    
    Генерирует secure API keys с prefix, хэширует для хранения,
    и управляет permissions через scopes.
    """
    
    def generate_api_key(
        self,
        user_id: UUID,
        name: str,
        scopes: List[str],
        expires_in_days: Optional[int] = None
    ) -> tuple[str, str]:
        """
        Генерирует новый API key.
        
        Returns:
            Tuple[str, str]: (plain_key, key_prefix)
            plain_key показывается пользователю ОДИН раз
        """
        # Генерируем secure random key
        random_bytes = secrets.token_bytes(32)
        plain_key = f"ani_{secrets.token_urlsafe(32)}"
        
        # Хэшируем для хранения
        key_hash = hashlib.sha256(plain_key.encode()).hexdigest()
        
        # Prefix для UI
        key_prefix = plain_key[:12]  # "ani_1234..."
        
        return plain_key, key_prefix
    
    async def verify_api_key(self, plain_key: str) -> Optional[UUID]:
        """Верифицирует API key и возвращает user_id."""
        key_hash = hashlib.sha256(plain_key.encode()).hexdigest()
        # Query database
        pass
    
    async def check_scope(self, api_key_id: UUID, required_scope: str) -> bool:
        """Проверяет что у API key есть нужный scope."""
        pass
    
    async def track_usage(
        self,
        api_key_id: UUID,
        endpoint: str,
        ip_address: str,
        response_status: int
    ):
        """Трекает использование API key."""
        pass
```

**Deliverables:**
- [ ] Database migration для API keys
- [ ] APIKeyService implementation
- [ ] API key generation endpoint
- [ ] API key authentication middleware
- [ ] Scope verification system
- [ ] Usage tracking и analytics
- [ ] Key rotation mechanism
- [ ] Admin panel для key management

**Timeline:** 5-7 days  
**Dependencies:** None  
**Priority:** MEDIUM

---

### Priority 4: IP Whitelisting для Admin (LOW)

**Цель:** Restrict admin access к trusted IP addresses.

**Scope:**
- Configurable IP whitelist
- CIDR notation support
- Per-user or global whitelist
- Bypass option для emergencies

**Implementation Plan:**

#### 4.1 Configuration
```python
# settings.py

class Settings(BaseSettings):
    # IP Whitelisting
    admin_ip_whitelist_enabled: bool = False
    admin_ip_whitelist: List[str] = []  # ["192.168.1.0/24", "10.0.0.1"]
    admin_ip_bypass_secret: Optional[str] = None  # Emergency bypass
```

#### 4.2 Middleware
```python
# src/api/middleware/ip_whitelist.py

from ipaddress import ip_address, ip_network

class IPWhitelistMiddleware:
    """
    Middleware для проверки IP адреса admin пользователей.
    
    Блокирует доступ к admin endpoints с non-whitelisted IP.
    """
    
    async def __call__(self, request: Request, call_next):
        # Проверяем только admin endpoints
        if request.url.path.startswith("/api/v1/admin"):
            client_ip = request.client.host
            
            if not self._is_whitelisted(client_ip):
                # Log attempt
                logger.warning(f"Admin access denied from {client_ip}")
                
                # Audit log
                await audit_logger.log_security_event(
                    event="admin_access_denied",
                    ip_address=client_ip,
                    reason="ip_not_whitelisted"
                )
                
                raise HTTPException(
                    status_code=403,
                    detail="Access denied from this IP address"
                )
        
        return await call_next(request)
    
    def _is_whitelisted(self, ip: str) -> bool:
        """Проверяет IP в whitelist."""
        client_ip = ip_address(ip)
        
        for allowed in settings.admin_ip_whitelist:
            if "/" in allowed:  # CIDR notation
                if client_ip in ip_network(allowed):
                    return True
            else:  # Single IP
                if str(client_ip) == allowed:
                    return True
        
        return False
```

**Deliverables:**
- [ ] Settings configuration
- [ ] IPWhitelistMiddleware
- [ ] CIDR support
- [ ] Emergency bypass mechanism
- [ ] Admin UI для whitelist management
- [ ] Documentation

**Timeline:** 2-3 days  
**Dependencies:** None  
**Priority:** LOW

---

## 📊 Security Metrics & Monitoring

### KPIs для отслеживания:

| Metric | Current | Target |
|--------|---------|--------|
| Failed login attempts | - | Monitor |
| Password resets per day | - | Monitor |
| Admin actions per day | - | Monitor |
| API key usage | - | Track |
| 2FA adoption rate | 0% | >50% |
| Security incidents | 0 | 0 |

### Monitoring Tools:

**Recommended:**
- Sentry для error tracking
- Prometheus для metrics
- Grafana для dashboards
- ELK Stack для log analysis

---

## 🎯 Roadmap Timeline

### Q1 2026 (Current)
- ✅ Phase 1-6 implementation complete
- ⏳ Audit Logging (3-5 days)
- ⏳ Security documentation

### Q2 2026
- 2FA implementation (5-7 days)
- API Key Management (5-7 days)
- Security audit (external)

### Q3 2026
- IP Whitelisting (2-3 days)
- Penetration testing
- Security certifications

### Q4 2026
- Advanced threat detection
- SOC integration
- Compliance audits

---

## ✅ Success Criteria

После реализации всех priorities:

- [x] Audit logging для всех критических операций
- [x] 2FA available для всех пользователей
- [x] API keys для third-party integrations
- [x] IP whitelisting для admin access
- [x] Comprehensive security documentation
- [x] Security monitoring dashboard
- [x] Incident response procedures
- [x] Regular security audits

**Security Score:** 10/10 ✅

---

**Prepared by:** Security Team  
**Last Updated:** 2026-01-05  
**Status:** 🟡 In Progress  
**Next Review:** Q2 2026
