# Microservicio: MS-AUTH

**Tipo**: Authentication & Authorization Service  
**Base de Datos**: PostgreSQL:5433 (Prisma ORM)  
**Comunicación**: NATS (Request/Reply)  
**Framework**: NestJS + Passport.js + JWT  

---

## 📋 Tabla de Contenidos

1. [Propósito](#propósito)
2. [Endpoints y Operaciones](#endpoints-y-operaciones)
3. [Esquema de Base de Datos](#esquema-de-base-de-datos)
4. [RBAC - Roles y Permisos](#rbac---roles-y-permisos)
5. [Flujos de Autenticación](#flujos-de-autenticación)
6. [Servicios Principales](#servicios-principales)
7. [Seguridad](#seguridad)
8. [NATS Messages](#nats-messages)
9. [Error Handling](#error-handling)
10. [Configuración](#configuración)

---

## 🎯 Propósito

Proveer servicios de autenticación centralizados:
- ✅ Emisión de JWT
- ✅ Validación de tokens
- ✅ Gestión de usuarios
- ✅ Gestión de roles y permisos
- ✅ Tokens de refresco
- ✅ Revocación de sesiones

---

## 📨 Endpoints y Operaciones

### NATS Topics (Request/Reply)

#### 1. auth.authenticate
**Descripción**: Validar credenciales y generar JWT  
**Requester**: Gateway  
**Timeout**: 5 segundos  

**Request**:
```json
{
  "username": "operator@smartservice.com",
  "password": "secure-password"
}
```

**Response (Success)**:
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "refresh-token-xyz",
  "user": {
    "id": "uuid-123",
    "username": "operator@smartservice.com",
    "email": "operator@smartservice.com",
    "roles": ["operator"],
    "permissions": ["read:reclamos", "create:reclamo"],
    "isActive": true
  }
}
```

**Response (Error)**:
```json
{
  "error": "INVALID_CREDENTIALS",
  "message": "Username or password is incorrect"
}
```

#### 2. auth.validate-token
**Descripción**: Validar JWT y obtener información del usuario  
**Requester**: Gateway (para cada request HTTP)  
**Timeout**: 2 segundos  

**Request**:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response (Valid)**:
```json
{
  "valid": true,
  "userId": "uuid-123",
  "username": "operator@smartservice.com",
  "roles": ["operator"],
  "permissions": ["read:reclamos", "create:reclamo"],
  "expiresAt": 1715428800
}
```

**Response (Invalid)**:
```json
{
  "valid": false,
  "error": "TOKEN_EXPIRED",
  "message": "Token has expired"
}
```

#### 3. auth.refresh-token
**Descripción**: Generar nuevo accessToken usando refreshToken  
**Requester**: Gateway  

**Request**:
```json
{
  "refreshToken": "refresh-token-xyz"
}
```

**Response**:
```json
{
  "accessToken": "new-access-token...",
  "expiresIn": 900
}
```

#### 4. auth.get-user
**Descripción**: Obtener información completa del usuario  
**Requester**: Otros microservicios  

**Request**:
```json
{
  "userId": "uuid-123"
}
```

**Response**:
```json
{
  "id": "uuid-123",
  "username": "operator@smartservice.com",
  "email": "operator@smartservice.com",
  "roles": ["operator"],
  "permissions": [...],
  "lastLogin": "2026-05-11T10:30:00Z"
}
```

#### 5. auth.check-permission
**Descripción**: Verificar si usuario tiene permiso específico  
**Requester**: MS-Reclamos, otros servicios  

**Request**:
```json
{
  "userId": "uuid-123",
  "permission": "delete:reclamo"
}
```

**Response**:
```json
{
  "has": true
}
```

---

## 🗄️ Esquema de Base de Datos

### Tabla: User
```sql
CREATE TABLE "User" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "username" VARCHAR(255) UNIQUE NOT NULL,
  "email" VARCHAR(255) UNIQUE NOT NULL,
  "passwordHash" VARCHAR(255) NOT NULL,
  "firstName" VARCHAR(100),
  "lastName" VARCHAR(100),
  "isActive" BOOLEAN DEFAULT true,
  "lastLogin" TIMESTAMP,
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_email CHECK (email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);

INDEXES:
- username (UNIQUE)
- email (UNIQUE)
- isActive
```

### Tabla: RefreshToken
```sql
CREATE TABLE "RefreshToken" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "userId" UUID NOT NULL REFERENCES "User"("id") ON DELETE CASCADE,
  "token" VARCHAR(512) NOT NULL,
  "expiresAt" TIMESTAMP NOT NULL,
  "revokedAt" TIMESTAMP,
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT token_revoked_xor CHECK (
    ("revokedAt" IS NULL) OR ("revokedAt" IS NOT NULL)
  )
);

INDEXES:
- userId
- token
- expiresAt
```

### Tabla: Role
```sql
CREATE TABLE "Role" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" VARCHAR(50) UNIQUE NOT NULL,
  "description" TEXT,
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_role_name CHECK (
    name IN ('operator', 'area_manager', 'official', 'municipal_admin', 'platform_admin')
  )
);

PREDEFINED ROLES:
- operator: Operador básico, puede crear y ver reclamos
- area_manager: Gestor de área, puede asignar y supervisar
- official: Oficial municipal, puede actualizar estados
- municipal_admin: Administrador municipal, gestión completa
- platform_admin: Administrador de plataforma, acceso total
```

### Tabla: UserRole (Relación M:N)
```sql
CREATE TABLE "UserRole" (
  "userId" UUID NOT NULL REFERENCES "User"("id") ON DELETE CASCADE,
  "roleId" UUID NOT NULL REFERENCES "Role"("id") ON DELETE CASCADE,
  "assignedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  PRIMARY KEY ("userId", "roleId")
);

INDEXES:
- userId
- roleId
```

### Tabla: Permission
```sql
CREATE TABLE "Permission" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" VARCHAR(100) UNIQUE NOT NULL,
  "description" TEXT,
  "resource" VARCHAR(50),
  "action" VARCHAR(50),
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

PERMISSIONS (Ejemplos):
- read:reclamos
- create:reclamo
- update:reclamo
- delete:reclamo
- read:users
- create:user
- update:user
- manage:roles
- manage:permissions
```

### Tabla: RolePermission (Relación M:N)
```sql
CREATE TABLE "RolePermission" (
  "roleId" UUID NOT NULL REFERENCES "Role"("id") ON DELETE CASCADE,
  "permissionId" UUID NOT NULL REFERENCES "Permission"("id") ON DELETE CASCADE,
  "grantedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  PRIMARY KEY ("roleId", "permissionId")
);

INDEXES:
- roleId
- permissionId
```

### Tabla: UserPermission (Permisos directos)
```sql
CREATE TABLE "UserPermission" (
  "userId" UUID NOT NULL REFERENCES "User"("id") ON DELETE CASCADE,
  "permissionId" UUID NOT NULL REFERENCES "Permission"("id") ON DELETE CASCADE,
  "grantedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  PRIMARY KEY ("userId", "permissionId")
);
```

### Enums

**UserRole Enum**:
```typescript
enum UserRole {
  OPERATOR = 'operator',
  AREA_MANAGER = 'area_manager',
  OFFICIAL = 'official',
  MUNICIPAL_ADMIN = 'municipal_admin',
  PLATFORM_ADMIN = 'platform_admin'
}
```

---

## 🔐 RBAC - Roles y Permisos

### Matriz de Roles y Permisos

```
┌─────────────────────┬────────────┬────────────┬──────────┬──────────────┬─────────────┐
│ Permission          │ Operator   │ Area Mgr   │ Official │ Mun. Admin   │ Plat. Admin │
├─────────────────────┼────────────┼────────────┼──────────┼──────────────┼─────────────┤
│ read:reclamos       │ ✅         │ ✅         │ ✅       │ ✅           │ ✅          │
│ create:reclamo      │ ✅         │ ✅         │ ✅       │ ✅           │ ✅          │
│ update:reclamo      │ ❌         │ ✅         │ ✅       │ ✅           │ ✅          │
│ delete:reclamo      │ ❌         │ ❌         │ ❌       │ ✅           │ ✅          │
│ assign:reclamo      │ ❌         │ ✅         │ ✅       │ ✅           │ ✅          │
│ read:users          │ ❌         │ ❌         │ ❌       │ ✅           │ ✅          │
│ create:user         │ ❌         │ ❌         │ ❌       │ ✅           │ ✅          │
│ update:user         │ ❌         │ ❌         │ ❌       │ ✅           │ ✅          │
│ delete:user         │ ❌         │ ❌         │ ❌       │ ❌           │ ✅          │
│ manage:roles        │ ❌         │ ❌         │ ❌       │ ❌           │ ✅          │
│ manage:permissions  │ ❌         │ ❌         │ ❌       │ ❌           │ ✅          │
└─────────────────────┴────────────┴────────────┴──────────┴──────────────┴─────────────┘
```

---

## 🔄 Flujos de Autenticación

### Flujo 1: Login Inicial
```
1. Usuario envía credenciales al Gateway
   POST /api/auth/login
   Body: { username, password }

2. Gateway envía a MS-Auth (NATS request/reply)
   Topic: auth.authenticate
   Payload: { username, password }

3. MS-Auth:
   a. Busca usuario en PostgreSQL por username
   b. Verifica si existe y está activo
   c. Valida contraseña con bcrypt.compare()
   d. Si válido:
      - Crea JWT con: sub (userId), username, roles, permissions
      - Crea/rota RefreshToken
      - Guarda RefreshToken en PostgreSQL
      - Retorna tokens + info usuario
   e. Si inválido:
      - Log de intento fallido
      - Retorna error INVALID_CREDENTIALS

4. Gateway retorna al cliente:
   { accessToken, refreshToken, user }

5. Cliente almacena tokens:
   - accessToken en memory o sessionStorage
   - refreshToken en cookie HTTP-only
```

### Flujo 2: Validación de JWT en Cada Request
```
1. Cliente envía HTTP request con JWT
   Headers: { Authorization: "Bearer {accessToken}" }

2. Gateway JwtAuthGuard intercepta:
   a. Extrae token del header
   b. Envía a MS-Auth (NATS request/reply)
      Topic: auth.validate-token
   c. Espera respuesta validación

3. MS-Auth valida:
   a. Descifra JWT con JWT_ACCESS_SECRET
   b. Verifica firma y expiración
   c. Si válido: retorna { valid: true, userId, roles, permissions }
   d. Si expirado/inválido: retorna { valid: false, error }

4. Si válido:
   - Gateway continúa procesamiento
   - Injetta user en request.user

5. Si inválido:
   - Gateway retorna 401 Unauthorized
```

### Flujo 3: Refresh Token
```
1. AccessToken expirado
2. Cliente envía RefreshToken
   POST /api/auth/refresh
   Body: { refreshToken }

3. Gateway envía a MS-Auth
   Topic: auth.refresh-token

4. MS-Auth:
   a. Busca RefreshToken en PostgreSQL
   b. Verifica no esté revocado
   c. Verifica no haya expirado
   d. Crea nuevo AccessToken
   e. Retorna nuevo token

5. Cliente usa nuevo AccessToken
```

---

## 🔧 Servicios Principales

### AuthService
```typescript
async authenticate(username: string, password: string): Promise<AuthDto> {
  const user = await this.prisma.user.findUnique({ 
    where: { username },
    include: { 
      userRoles: { include: { role: { include: { permissions: true } } } }
    }
  });

  if (!user || !user.isActive) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const isPasswordValid = await bcrypt.compare(password, user.passwordHash);
  if (!isPasswordValid) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const roles = user.userRoles.map(ur => ur.role.name);
  const permissions = user.userRoles
    .flatMap(ur => ur.role.rolePermissions)
    .map(rp => rp.permission.name);

  const accessToken = this.jwtService.sign({
    sub: user.id,
    username: user.username,
    roles,
    permissions
  }, { expiresIn: this.configService.get('JWT_ACCESS_EXPIRATION') });

  const refreshToken = this.generateRefreshToken();
  await this.prisma.refreshToken.create({
    data: {
      userId: user.id,
      token: refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
    }
  });

  return {
    accessToken,
    refreshToken,
    user: {
      id: user.id,
      username: user.username,
      email: user.email,
      roles,
      permissions,
      isActive: user.isActive
    }
  };
}

async validateToken(token: string): Promise<TokenPayload> {
  try {
    const payload = this.jwtService.verify(token);
    return {
      valid: true,
      userId: payload.sub,
      username: payload.username,
      roles: payload.roles,
      permissions: payload.permissions
    };
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return { valid: false, error: 'TOKEN_EXPIRED' };
    }
    return { valid: false, error: 'INVALID_TOKEN' };
  }
}

async refreshToken(refreshToken: string): Promise<{ accessToken: string }> {
  const stored = await this.prisma.refreshToken.findFirst({
    where: { token: refreshToken, revokedAt: null }
  });

  if (!stored || stored.expiresAt < new Date()) {
    throw new UnauthorizedException('Invalid refresh token');
  }

  const user = await this.prisma.user.findUnique({
    where: { id: stored.userId },
    include: { userRoles: { include: { role: true } } }
  });

  const roles = user.userRoles.map(ur => ur.role.name);

  const newAccessToken = this.jwtService.sign({
    sub: user.id,
    username: user.username,
    roles
  }, { expiresIn: this.configService.get('JWT_ACCESS_EXPIRATION') });

  return { accessToken: newAccessToken };
}
```

### RoleService
```typescript
async getUserRoles(userId: string): Promise<string[]> {
  const userRoles = await this.prisma.userRole.findMany({
    where: { userId },
    include: { role: true }
  });
  return userRoles.map(ur => ur.role.name);
}

async hasRole(userId: string, role: string): Promise<boolean> {
  const userRole = await this.prisma.userRole.findFirst({
    where: {
      userId,
      role: { name: role }
    }
  });
  return !!userRole;
}

async assignRoleToUser(userId: string, roleId: string): Promise<void> {
  await this.prisma.userRole.create({
    data: { userId, roleId }
  });
}
```

### PermissionService
```typescript
async getUserPermissions(userId: string): Promise<string[]> {
  const userRoles = await this.prisma.userRole.findMany({
    where: { userId },
    include: {
      role: {
        include: {
          rolePermissions: { include: { permission: true } }
        }
      }
    }
  });

  const rolePermissions = userRoles
    .flatMap(ur => ur.role.rolePermissions)
    .map(rp => rp.permission.name);

  const directPermissions = await this.prisma.userPermission.findMany({
    where: { userId },
    include: { permission: true }
  });

  return [
    ...new Set([
      ...rolePermissions,
      ...directPermissions.map(up => up.permission.name)
    ])
  ];
}

async hasPermission(userId: string, permission: string): Promise<boolean> {
  const permissions = await this.getUserPermissions(userId);
  return permissions.includes(permission);
}
```

---

## 🔒 Seguridad

### Password Hashing
```typescript
// En creación de usuario
const hashedPassword = await bcrypt.hash(plainPassword, 10); // salt rounds = 10

// En validación de login
const isValid = await bcrypt.compare(plainPassword, hashedPassword);
```

### JWT Security
- **Secret Key**: Mínimo 32 caracteres (env variable)
- **Algorithm**: HS256 (HMAC SHA-256)
- **Expiration**: 15 minutos para accessToken, 7 días para refreshToken
- **Payload**: Incluye userId, roles, permissions (no datos sensibles)

### Token Revocation
- RefreshTokens guardados en DB
- Campo `revokedAt` para revocación
- Logout: marcar refreshToken como revocado

### CORS & CSRF
- CORS configurado en Gateway
- CSRF tokens para operaciones sensibles (planeado)

---

## 📡 NATS Messages

### Publicaciones (Outbound)
- No publica directamente a otros servicios
- Solo responde a request/reply

### Subscripciones (Inbound)
```
auth.authenticate          → Request/Reply
auth.validate-token        → Request/Reply
auth.refresh-token         → Request/Reply
auth.get-user              → Request/Reply
auth.check-permission      → Request/Reply
auth.assign-role           → Request/Reply
auth.revoke-token          → Request/Reply
```

---

## ⚠️ Error Handling

### Excepciones Comunes

```typescript
// Invalid credentials
throw new UnauthorizedException('Username or password is incorrect');

// Token expired
throw new UnauthorizedException('Token has expired');

// User not found
throw new NotFoundException('User not found');

// Insufficient permissions
throw new ForbiddenException('You do not have permission to perform this action');

// User already exists
throw new ConflictException('Username already in use');
```

---

## ⚙️ Configuración

### .env
```bash
# Database
DATABASE_URL=postgresql://user:password@localhost:5433/smartservice_auth

# JWT
JWT_ACCESS_SECRET=your-very-long-secret-key-min-32-chars
JWT_REFRESH_SECRET=your-refresh-secret-min-32-chars
JWT_ACCESS_EXPIRATION=900          # 15 minutos
JWT_REFRESH_EXPIRATION=604800      # 7 días

# NATS
NATS_SERVERS=nats://localhost:4222

# App
NODE_ENV=development
```

---

## 🔗 Dependencias Clave

```json
{
  "@nestjs/common": "^11.0.0",
  "@nestjs/jwt": "^12.x.x",
  "@nestjs/passport": "^10.x.x",
  "@prisma/client": "^5.x.x",
  "passport": "^0.7.0",
  "passport-jwt": "^4.0.1",
  "bcrypt": "^6.0.0",
  "jsonwebtoken": "^9.x.x"
}
```

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
