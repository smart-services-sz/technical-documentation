# Microservicio: GATEWAY

**Puerto**: 3000 (HTTP REST)  
**Tipo**: API Gateway + Entry Point  
**Frameworks**: NestJS, Express  
**Comunicación**: HTTP IN → NATS OUT  

---

## 📋 Tabla de Contenidos

1. [Propósito y Responsabilidades](#propósito-y-responsabilidades)
2. [Puntos de Entrada HTTP](#puntos-de-entrada-http)
3. [Flujo de Procesamiento](#flujo-de-procesamiento)
4. [Módulos Principales](#módulos-principales)
5. [Servicios Clave](#servicios-clave)
6. [Middleware y Guards](#middleware-y-guards)
7. [Integración NATS](#integración-nats)
8. [Redis Operations](#redis-operations)
9. [Validación y Sanitización](#validación-y-sanitización)
10. [Manejo de Errores](#manejo-de-errores)
11. [Configuración](#configuración)
12. [Dependencias](#dependencias)

---

## 🎯 Propósito y Responsabilidades

### Objetivo Principal
Ser el punto de entrada único (Single Entry Point) para todos los clientes externos. Actúa como:
- **Orquestador**: Coordina mensajes entre clientes y microservicios
- **Autenticador**: Valida JWT y RBAC
- **Sanitizador**: Limpia y valida entrada
- **Deduplicador**: Evita procesamiento de mensajes duplicados
- **Acumulador**: Agrupa mensajes en ventanas temporales (debouncing)

### Responsabilidades
1. ✅ Recibir webhooks de WhatsApp/Meta
2. ✅ Validar y sanitizar entrada
3. ✅ Verificar autenticación JWT
4. ✅ Deduplicar mensajes (idempotencia)
5. ✅ Acumular/debounce mensajes
6. ✅ Despachar a NATS
7. ✅ Manejar respuestas de microsservicios
8. ✅ Retornar respuestas a clientes
9. ✅ Proveer Swagger UI para documentación

---

## 📨 Puntos de Entrada HTTP

### 1. POST /api/inbound/messages
**Descripción**: Recibe mensajes de ciudadanos (WhatsApp, web, email)  
**Autenticación**: No requerida (webhooks externos)  
**Rate Limit**: Configurado globalmente  

**Request Body**:
```json
{
  "channel": "whatsapp",
  "channelIdentity": {
    "type": "phone",
    "value": "+5493415551234"
  },
  "externalMessageId": "wamid-unique-id-001",
  "messageText": "Hola quiero hacer un reclamo",
  "responseWindow": {
    "debounceMs": 2000,
    "delayMs": 0,
    "maxAccumulationMs": 8000
  }
}
```

**Response**:
```json
{
  "status": "accepted",
  "messageId": "uuid-xxx",
  "processingStarted": true,
  "eta": 5000
}
```

**Status Codes**:
- `202 Accepted`: Mensaje aceptado para procesamiento
- `400 Bad Request`: Entrada inválida
- `429 Too Many Requests`: Rate limit excedido
- `500 Internal Server Error`: Error de servidor

**Flujo**:
```
1. Validación del payload (class-validator)
2. Verificación de idempotencia en Redis
   - Si existe: retorna respuesta cacheada
   - Si no existe: continúa
3. Sanitización del mensaje
4. Debouncing:
   - Acumula en Redis por `responseWindow.maxAccumulationMs`
   - Dispara procesamiento después de ventana
5. Publica a NATS: ai.process-inbound-message
6. Retorna 202 Accepted
```

### 2. POST /api/auth/login
**Descripción**: Autentica usuario y retorna JWT  
**Autenticación**: No requerida  

**Request**:
```json
{
  "username": "operator@smartservice.com",
  "password": "secure-password"
}
```

**Response**:
```json
{
  "accessToken": "eyJhbGc...",
  "refreshToken": "refresh...",
  "user": {
    "id": "uuid",
    "username": "operator@smartservice.com",
    "roles": ["operator"]
  }
}
```

**Flujo**:
```
1. Recibe credenciales
2. Envía auth.login a MS-Auth (NATS request/reply)
3. MS-Auth valida y retorna JWT
4. Gateway envuelve en cookie HTTP-only (opcional)
5. Retorna tokens al cliente
```

### 3. GET /api/docs (Swagger UI)
**Descripción**: Documentación interactiva de API  
**URL**: http://localhost:3000/api/docs  

### 4. GET /api/health
**Descripción**: Health check del servicio  
**Response**: `{ "status": "ok", "timestamp": "..." }`

---

## 🔄 Flujo de Procesamiento

### Flujo Detallado de un Mensaje Inbound

```
┌─ INPUT HTTP POST /api/inbound/messages ─────────────────────┐
│  Body: { channel, channelIdentity, externalMessageId, ... }  │
└──────────────────────┬──────────────────────────────────────┘
                       │
            ┌──────────▼──────────┐
            │  DTO Validation     │ (class-validator)
            │  - Estructura OK?   │
            │  - Campos requeridos?
            └──────────┬──────────┘
                       │
            ┌──────────▼──────────────────────┐
            │  Check Redis Idempotency        │
            │  Key: idempotency:{id}:{channel}│
            └──────┬───────────────┬──────────┘
                   │ EXISTS        │ NOT EXISTS
                   ▼               ▼
            ┌───────────────┐   ┌──────────────────────────┐
            │ Return cached │   │ Sanitize message text    │
            │ response      │   │ - Remove special chars   │
            └───────────────┘   │ - Normalize whitespace   │
                                │ - Verify length < 2000   │
                                └──────────┬───────────────┘
                                           │
                        ┌──────────────────▼──────────────────┐
                        │  Store in Redis (TTL 24h)           │
                        │  - Key: idempotency:...             │
                        │  - Value: response                  │
                        └──────────────┬─────────────────────┘
                                       │
                        ┌──────────────▼──────────────────┐
                        │  Debouncing Logic               │
                        │  - Add to batch queue (Redis)   │
                        │  - Set timer: responseWindow    │
                        │  - If batch reaches limit: fire │
                        │  - Else: wait for timeout       │
                        └──────────────┬─────────────────┘
                                       │
                        ┌──────────────▼──────────────────┐
                        │  Dispatch to NATS               │
                        │  Topic: ai.process-inbound-msg  │
                        │  Payload: sanitized + metadata  │
                        └──────────────┬─────────────────┘
                                       │
            ┌──────────────────────────▼──────────────────────┐
            │  Return 202 Accepted to Client                  │
            │  { status: "accepted", messageId: "...", ... }  │
            └─────────────────────────────────────────────────┘
```

---

## 🏛️ Módulos Principales

### 1. **AppModule** (app.module.ts)
Módulo raíz que configura:
- `TypeOrmModule`: Inyección de dependencias
- `ConfigModule`: Variables de entorno (dotenv)
- `NatsModule`: Cliente NATS
- `RedisModule`: Cliente Redis
- `JwtModule`: Autenticación
- Controllers globales
- Middleware global

### 2. **InboundModule** (inbound/)
Responsable del procesamiento de mensajes inbound
```
inbound/
├── inbound.controller.ts   → HTTP POST endpoint
├── inbound.service.ts      → Lógica de procesamiento
├── dto/
│   ├── inbound-payload.dto.ts
│   └── response.dto.ts
└── interfaces/
    ├── channel.interface.ts
    └── message.interface.ts
```

### 3. **AuthModule** (auth/)
Manejo de autenticación y JWT
```
auth/
├── auth.controller.ts
├── auth.service.ts
├── jwt.strategy.ts  → Passport JWT strategy
├── guards/
│   ├── jwt.guard.ts
│   └── roles.guard.ts
└── dto/
    └── login.dto.ts
```

### 4. **CommonModule** (common/)
Código compartido
```
common/
├── decorators/
│   ├── public.decorator.ts
│   └── roles.decorator.ts
├── guards/
│   └── jwt-auth.guard.ts
├── filters/
│   └── http-exception.filter.ts
└── interceptors/
    └── response.interceptor.ts
```

### 5. **NatsModule** (nats/)
Integración con NATS message broker

### 6. **RedisModule** (redis/)
Integración con Redis
- Idempotencia
- Debouncing
- Session state

---

## 🔧 Servicios Clave

### InboundService
**Responsabilidad**: Procesar mensajes entrantes

**Métodos principales**:
```typescript
async processInboundMessage(dto: InboundPayloadDto): Promise<void> {
  // 1. Verificar idempotencia
  const cached = await this.redis.get(`idempotency:${dto.externalMessageId}`);
  if (cached) return JSON.parse(cached);

  // 2. Sanitizar
  const sanitized = this.sanitizeMessage(dto.messageText);

  // 3. Debouncing
  await this.addToBatch(dto.channelIdentity, {
    ...dto,
    messageText: sanitized
  });

  // 4. Despachar a NATS
  await this.nats.publish('ai.process-inbound-message', JSON.stringify({
    channel: dto.channel,
    channelIdentity: dto.channelIdentity,
    messageText: sanitized,
    timestamp: Date.now()
  }));

  // 5. Cachear respuesta
  const response = { status: 'accepted', ... };
  await this.redis.setex(
    `idempotency:${dto.externalMessageId}`,
    86400,
    JSON.stringify(response)
  );

  return response;
}

private sanitizeMessage(text: string): string {
  // Remover caracteres especiales peligrosos
  // Normalizar espacios en blanco
  // Limitar longitud
  // Validar que contenga keywords de reclamos
}

private async addToBatch(identity: ChannelIdentity, message: any): Promise<void> {
  // Agrupar mensajes por canal+usuario
  // Aplicar debouncing según responseWindow
  // Disparar procesamiento cuando ventana expira
}
```

### AuthService
**Responsabilidad**: Autenticación y JWT

```typescript
async login(username: string, password: string) {
  // 1. Buscar usuario en MS-Auth
  const user = await this.nats.request('auth.find-by-username', username);
  
  // 2. Validar contraseña (bcrypt)
  const isValid = await bcrypt.compare(password, user.passwordHash);
  if (!isValid) throw new UnauthorizedException();
  
  // 3. Generar JWT
  const token = this.jwtService.sign({
    sub: user.id,
    username: user.username,
    roles: user.roles
  });
  
  return { accessToken: token, user };
}

async validateJwt(token: string) {
  try {
    const payload = this.jwtService.verify(token);
    return payload;
  } catch (err) {
    throw new UnauthorizedException('Invalid token');
  }
}
```

---

## 🔐 Middleware y Guards

### 1. **JwtAuthGuard** (guards/jwt-auth.guard.ts)
Validar JWT en cada request
```typescript
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace('Bearer ', '');
    
    if (!token) throw new UnauthorizedException();
    
    try {
      request.user = this.jwtService.verify(token);
      return true;
    } catch {
      throw new UnauthorizedException();
    }
  }
}
```

### 2. **RolesGuard** (guards/roles.guard.ts)
Validar roles y permisos
```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const roles = Reflect.getMetadata('roles', context.getHandler());
    if (!roles) return true; // Si no hay metadata, permitir
    
    const request = context.switchToHttp().getRequest();
    const userRoles = request.user.roles;
    
    return roles.some((role: string) => userRoles.includes(role));
  }
}
```

### 3. **GlobalExceptionFilter** (filters/http-exception.filter.ts)
Manejo centralizado de excepciones
```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const status = exception.getStatus();
    
    response.status(status).json({
      statusCode: status,
      message: exception.message,
      timestamp: new Date().toISOString()
    });
  }
}
```

---

## 📡 Integración NATS

### Publicaciones (Outbound)

#### 1. ai.process-inbound-message (Pub)
**Descripción**: Enviar mensaje a MS-AI para procesamiento  
**Tipo**: Pub/Sub Asincrónico  
**Payload**:
```json
{
  "channel": "whatsapp",
  "channelIdentity": { "type": "phone", "value": "+549..." },
  "messageText": "Hola...",
  "timestamp": 1715425800000,
  "externalMessageId": "wamid-001"
}
```

### Suscripciones (Inbound)

#### 1. auth.* (Request/Reply)
**Descripción**: Validar usuario, obtener roles/permisos  
**Timeout**: 5 segundos  

```typescript
// Request
const user = await this.nats.request('auth.validate-token', token);

// MS-Auth responde
{ id: "uuid", username: "...", roles: [...], permissions: [...] }
```

---

## 💾 Redis Operations

### 1. Idempotencia
```
Key: idempotency:{externalMessageId}:{channel}
Value: Cached response
TTL: INBOUND_IDEMPOTENCY_TTL_SECONDS (86400 = 24h)
Type: String (JSON)
```

### 2. Debouncing/Batching
```
Key: batch:{channel}:{channelIdentity}
Value: Array de mensajes
TTL: INBOUND_MAX_ACCUMULATION_MS
Type: List
Operation:
  1. RPUSH (agregar mensaje)
  2. EXPIRE (set TTL)
  3. LRANGE (obtener batch)
  4. DEL (limpiar)
```

### 3. Message History
```
Key: history:{channelIdentity}:{channel}
Value: Últimos N mensajes
TTL: INBOUND_CONTEXT_TTL_SECONDS
Type: List (rolling window)
Max Size: INBOUND_MESSAGE_HISTORY_LIMIT (20)
```

---

## ✅ Validación y Sanitización

### DTO Validation (class-validator)
```typescript
export class InboundPayloadDto {
  @IsEnum(['whatsapp', 'web', 'email', 'instagram'])
  channel: string;

  @IsObject()
  @ValidateNested()
  channelIdentity: ChannelIdentity;

  @IsString()
  @MaxLength(2000)
  messageText: string;

  @IsString()
  @Length(1, 255)
  externalMessageId: string;

  @IsOptional()
  @IsObject()
  responseWindow?: ResponseWindow;
}
```

### Sanitización de Mensaje
```typescript
private sanitizeMessage(text: string): string {
  // 1. Remover caracteres de control
  text = text.replace(/[\x00-\x1F\x7F]/g, '');
  
  // 2. Normalizar espacios
  text = text.replace(/\s+/g, ' ').trim();
  
  // 3. Limitar longitud
  if (text.length > 2000) {
    text = text.substring(0, 2000);
  }
  
  // 4. Verificar que no sea solo spam
  if (!this.containsRelevantKeywords(text)) {
    throw new BadRequestException('Message does not contain relevant keywords');
  }
  
  return text;
}

private containsRelevantKeywords(text: string): boolean {
  const keywords = ['reclamo', 'problema', 'queja', 'servicio', 'falla', 'denuncia'];
  const lower = text.toLowerCase();
  return keywords.some(kw => lower.includes(kw));
}
```

---

## ⚠️ Manejo de Errores

### Exception Hierarchy
```
HttpException (base)
├── BadRequestException (400)
├── UnauthorizedException (401)
├── ForbiddenException (403)
├── NotFoundException (404)
├── ConflictException (409)
└── InternalServerErrorException (500)
```

### Error Response Format
```json
{
  "statusCode": 400,
  "message": "Validation failed: messageText must not be empty",
  "error": "BadRequest",
  "timestamp": "2026-05-11T10:30:00Z",
  "path": "/api/inbound/messages"
}
```

### Manejo de Errores NATS
```typescript
try {
  const reply = await firstValueFrom(
    this.nats.request('ai.process-inbound-message', msg, {
      timeout: 30000
    })
  );
} catch (err) {
  if (err.name === 'TimeoutError') {
    logger.error('NATS request timeout');
    // Retry logic o fallback
  } else {
    logger.error('NATS request failed:', err.message);
    throw new InternalServerErrorException();
  }
}
```

---

## ⚙️ Configuración

### Archivo .env
```bash
# Server
NODE_ENV=development
PORT=3000

# NATS
NATS_SERVERS=nats://localhost:4222

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_ACCESS_SECRET=your-very-secret-key-min-32-chars
JWT_REFRESH_SECRET=your-refresh-secret-min-32-chars
JWT_ACCESS_EXPIRATION=900           # 15 minutos
JWT_REFRESH_EXPIRATION=604800       # 7 días

# Inbound Processing
INBOUND_IDEMPOTENCY_TTL_SECONDS=86400
INBOUND_CONTEXT_TTL_SECONDS=86400
INBOUND_MESSAGE_HISTORY_LIMIT=20
INBOUND_MIN_DEBOUNCE_MS=800
INBOUND_MIN_DEBOUNCE_WHATSAPP_MS=2000
INBOUND_MAX_BATCH_SIZE=10

# WhatsApp Webhook
WHATSAPP_BUSINESS_ACCOUNT_ID=123456789
WHATSAPP_ACCESS_TOKEN=EAAxxxx...
WHATSAPP_VERIFY_TOKEN=verify_token_123
WHATSAPP_API_VERSION=v18.0

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000          # 1 minuto
RATE_LIMIT_MAX_REQUESTS=100          # 100 req/min
```

---

## 📦 Dependencias

### Principales
```json
{
  "@nestjs/common": "^11.0.0",
  "@nestjs/core": "^11.0.0",
  "@nestjs/jwt": "^12.x.x",
  "@nestjs/passport": "^10.x.x",
  "@nestjs/swagger": "^8.x.x",
  "passport": "^0.7.0",
  "passport-jwt": "^4.0.1",
  "jsonwebtoken": "^9.x.x",
  "bcrypt": "^6.0.0",
  "nats": "^2.29.3",
  "ioredis": "^5.x.x",
  "class-validator": "^0.14.0",
  "class-transformer": "^0.5.x",
  "axios": "^1.7.7",
  "dotenv": "^16.x.x"
}
```

---

## 🔗 Flujos Completos

### 1. Login → JWT
```
POST /api/auth/login
  ↓
AuthController.login()
  ↓
AuthService.authenticate()
  ↓
NATS Request: auth.validate-credentials
  ↓
MS-Auth validates & returns JWT
  ↓
Gateway returns: { accessToken, refreshToken, user }
```

### 2. Inbound Message → Reclamo Creation
```
POST /api/inbound/messages (webhook)
  ↓
InboundController.handleMessage()
  ↓
InboundService.processInboundMessage()
  │
  ├─ Validate DTO
  ├─ Check Redis idempotency
  ├─ Sanitize input
  ├─ Add to batch (debounce)
  │
  └─ After debounce window:
     ↓
     NATS Pub: ai.process-inbound-message
     ↓
     MS-AI receives & processes
     ↓
     NATS Pub: reclamos.create
     ↓
     MS-Reclamos persists
     ↓
     ✅ Reclamo created

Return: 202 Accepted
```

---

## 🚨 Limitaciones Actuales

- ❌ No hay almacenamiento de archivos (imágenes adjuntas)
- ⚠️ Sin notificaciones de confirmación integradas
- ⚠️ Sin audit trail de requests
- ⚠️ Rate limiting básico (necesita refinamiento)

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
