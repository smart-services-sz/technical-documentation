# SmartService - Arquitectura Global del Sistema

**Fecha de Creación**: 11 de Mayo de 2026  
**Estado**: Análisis Completo del Código Actual  
**Versión**: 1.0

---

## 📋 Tabla de Contenidos

1. [Visión General](#visión-general)
2. [Componentes Principales](#componentes-principales)
3. [Flujo de Datos](#flujo-de-datos)
4. [Tecnologías Utilizadas](#tecnologías-utilizadas)
5. [Relaciones Entre Servicios](#relaciones-entre-servicios)
6. [Configuración Global](#configuración-global)
7. [Patrones de Comunicación](#patrones-de-comunicación)
8. [Seguridad](#seguridad)
9. [Entorno y Despliegue](#entorno-y-despliegue)

---

## 🎯 Visión General

SmartService es una **plataforma de gestión inteligente de reclamos ciudadanos** basada en microservicios. Permite que ciudadanos reporten problemas a través de WhatsApp o web, utiliza IA para clasificar y procesar reclamos, valida ubicaciones geográficas y los almacena en una base de datos para seguimiento operativo.

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENTES EXTERNOS                          │
│            WhatsApp · Web · Email · Instagram                   │
└─────────────────────┬───────────────────────────────────────────┘
                      │ HTTP POST
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                 GATEWAY (API REST)  :3000                       │
│  • InboundController (recibe mensajes)                          │
│  • Autenticación JWT                                            │
│  • Debouncing & Idempotencia (Redis)                            │
│  • Despacha a NATS                                              │
└────────────┬────────────────────────────────────────────────────┘
             │ NATS pub/sub + request/reply
      ┌──────┴─────────────────┬──────────────────────┬────────────┐
      ▼                        ▼                      ▼            ▼
┌──────────────┐  ┌──────────────────┐  ┌──────────────┐  ┌──────────────┐
│   MS-AUTH    │  │     MS-AI        │  │   MS-RECLAMOS│  │    MS-GEO    │
│              │  │                  │  │              │  │              │
│• JWT issuer  │  │• NLP + GPT-4o    │  │• CRUD        │  │• Google Maps │
│• Users/Roles │  │• State mgmt      │  │• Persistence │  │• Geocoding   │
│• Permissions │  │• Classification  │  │• History     │  │• Validation  │
│              │  │• Orchestration   │  │              │  │              │
│PostgreSQL    │  │Redis + OpenAI    │  │PostgreSQL    │  │(Stateless)   │
│:5433         │  │                  │  │:5432         │  │              │
└──────────────┘  └──────────────────┘  └──────────────┘  └──────────────┘
                      ▲
                      │ (2) geo.geocode-address
                      └──────┬──────┬────────────────────────────┐
                             │      │                            │
                      ┌──────┴─┐    │ (3) reclamos.create        │
                      │ Google │    │                            │
                      │ Maps   │    ▼                            │
                      │ API    │
                      └────────┘
```

---

## 🏗️ Componentes Principales

### 1. **FRONTEND** (Presentación)
- **public-site**: Sitio estático para ciudadanos
- **admin-dashboard**: Panel administrativo Next.js

### 2. **API GATEWAY** (Entrada única)
- Puerto 3000 (HTTP REST)
- Autenticación JWT
- Manejo de webhooks (WhatsApp, Meta)
- Debouncing y deduplicación

### 3. **MICROSERVICIOS** (Procesamiento)
- **ms-auth**: Autenticación y autorización
- **ms-ai**: Procesamiento de IA y orquestación
- **ms-reclamos**: Persistencia de reclamos
- **ms-geo**: Geolocalización

### 4. **INFRAESTRUCTURA**
- **PostgreSQL**: Bases de datos relacionales (2 instancias)
- **Redis**: State management, caché, debouncing
- **NATS**: Message broker (pub/sub + request/reply)
- **OpenAI**: NLP y clasificación
- **Google Maps**: Geolocalización

---

## 🔄 Flujo de Datos Principal

### Flujo Completo de un Reclamo

```
1. ENTRADA DEL MENSAJE
   ┌─────────────────────────────────────────────────────────┐
   │ Ciudadano envía WhatsApp a número Business              │
   │ → Webhook Meta → Gateway (/api/inbound/messages)        │
   └──────────────────────┬──────────────────────────────────┘

2. PROCESAMIENTO EN GATEWAY
   ┌─────────────────────────────────────────────────────────┐
   │ • Sanitización de entrada                               │
   │ • Verificación de idempotencia (Redis)                  │
   │ • Debouncing (acumula mensajes por 2-8 segundos)        │
   │ • Despacha a NATS: ai.process-inbound-message           │
   └──────────────────────┬──────────────────────────────────┘

3. PROCESAMIENTO EN MS-AI
   ┌─────────────────────────────────────────────────────────┐
   │ • Recibe mensaje en NATS                                │
   │ • Recupera estado conversacional (Redis)                │
   │ • Máquina de estados:                                   │
   │   - greeting → collecting_data → ready_for_claim        │
   │ • Llama OpenAI GPT-4o-mini para NLP                     │
   │ • Extrae: descripción, categoría, prioridad, dirección  │
   │ • Si dirección: publica geo.geocode-address (NATS)      │
   │ • Salva estado en Redis (TTL 24h)                       │
   └──────────────────────┬──────────────────────────────────┘

4. GEOLOCALIZACIÓN
   ┌─────────────────────────────────────────────────────────┐
   │ • MS-GEO recibe geo.geocode-address                     │
   │ • Consulta Google Maps Geocoding API                    │
   │ • Valida zona permitida                                 │
   │ • Retorna: lat, lng, addressComponents                  │
   │ • Responde a MS-AI (request/reply)                      │
   └──────────────────────┬──────────────────────────────────┘

5. CREACIÓN DE RECLAMO
   ┌─────────────────────────────────────────────────────────┐
   │ • MS-AI publica reclamos.create (NATS)                  │
   │ • Estructura:                                           │
   │   {                                                     │
   │     "canal": "whatsapp",                                │
   │     "categoria": "baches_y_pavimento",                  │
   │     "prioridad": "NORMAL",                              │
   │     "descripcion": "Bache en Av. Principal",            │
   │     "ubicacion": { "lat": -32.94, "lng": -60.65 },      │
   │     "estado": "PENDIENTE"                               │
   │   }                                                     │
   └──────────────────────┬──────────────────────────────────┘

6. PERSISTENCIA
   ┌─────────────────────────────────────────────────────────┐
   │ • MS-RECLAMOS recibe reclamos.create (NATS)             │
   │ • Inserta en PostgreSQL:5432                            │
   │   - tabla: Reclamo                                      │
   │   - con estado PENDIENTE, timestamp, tracking           │
   │ • Genera ID único (UUID)                                │
   │ • Retorna confirmación                                  │
   └──────────────────────┬──────────────────────────────────┘

7. RESPUESTA AL USUARIO
   ┌─────────────────────────────────────────────────────────┐
   │ • Gateway recibe confirmación                           │
   │ • Envía respuesta a ciudadano (SMS/WhatsApp)            │
   │ • "Tu reclamo fue registrado con ID: xxx"              │
   └─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tecnologías Utilizadas

### Backend
| Componente | Tecnología | Versión | Uso |
|-----------|-----------|---------|-----|
| **Framework** | NestJS | 11.x | Framework para microservicios |
| **Lenguaje** | TypeScript | 5.x | Tipado estático |
| **ORM** | Prisma | 7.x | Acceso a datos type-safe |
| **Validación** | class-validator | - | DTO validation |
| **Autenticación** | Passport.js | - | JWT strategy |
| **JWT** | jsonwebtoken | - | Token generation |
| **Hashing** | bcrypt | 6.x | Password hashing |

### Messaging & Cache
| Componente | Tecnología | Versión | Uso |
|-----------|-----------|---------|-----|
| **Message Broker** | NATS | 2.29.3 | Pub/sub entre servicios |
| **Cache** | Redis | 5.x | State, idempotencia, debounce |
| **Redis Client** | ioredis | - | Conexión a Redis |

### Bases de Datos
| Componente | Tecnología | Versión | Uso |
|-----------|-----------|---------|-----|
| **SQL** | PostgreSQL | 16 alpine | Persistencia relacional |
| **Driver** | pg | - | Conexión a PostgreSQL |

### Integración Externa
| Componente | Tecnología | Versión | Uso |
|-----------|-----------|---------|-----|
| **IA** | OpenAI API | v6.36.0 | GPT-4o-mini para NLP |
| **Geolocalización** | Google Maps API | - | Geocoding |
| **Chat** | WhatsApp Business API | Meta Graph | Webhooks/mensajes |

### Frontend
| Componente | Tecnología | Versión | Uso |
|-----------|-----------|---------|-----|
| **Framework** | Next.js | 16.2.6 | React framework |
| **UI** | React | 19.2.4 | Componentes |
| **Styling** | Tailwind CSS | - | Utilidades CSS |
| **HTTP Client** | React Query | - | Data fetching |
| **Seguridad** | nookies | - | Cookie management |

### DevOps
| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| **Containerización** | Docker | Containers |
| **Orquestación** | Docker Compose | Local development |
| **Version Control** | GitHub | Repository |

---

## 🔗 Relaciones Entre Servicios

### 1. **Gateway ↔ MS-Auth**
```
NATS Sync (Request/Reply):
- Gateway envía: auth.validate-token
- MS-Auth responde: { valid: boolean, user: User }
- Usado en: Autenticación de requests HTTP
```

### 2. **Gateway ↔ MS-AI**
```
NATS Async (Pub/Sub):
- Gateway publica: ai.process-inbound-message
  Payload: { channelIdentity, messageText, channel, externalMessageId }
- MS-AI procesa y publica: reclamos.create, geo.geocode-address
```

### 3. **MS-AI ↔ MS-Geo**
```
NATS Sync (Request/Reply):
- MS-AI envía: geo.geocode-address
  Payload: { address, allowedCountries, allowedAreas }
- MS-Geo responde: { latitude, longitude, addressComponents, valid }
- Timeout: 30 segundos
```

### 4. **MS-AI ↔ MS-Reclamos**
```
NATS Async (Pub/Sub):
- MS-AI publica: reclamos.create
  Payload: { categoria, prioridad, descripcion, ubicacion, ... }
- MS-Reclamos persiste en PostgreSQL
```

### 5. **MS-Auth ↔ PostgreSQL** (ms-auth:5433)
```
Sincrónico:
- Login: busca usuario, valida contraseña, genera JWT
- Verificación: descifra y valida token
- Roles/Permisos: consulta relaciones
```

### 6. **MS-Reclamos ↔ PostgreSQL** (ms-reclamos:5432)
```
Sincrónico:
- CRUD de reclamos
- Almacena: descripción, ubicación, categoría, estado, timeline
- Consultas: find-all, find-one, update, remove
```

### 7. **MS-AI ↔ Redis**
```
Estado Conversacional:
- Clave: `conversation:{channelIdentity}:{channel}`
- TTL: 24 horas
- Contenido: phase (greeting/collecting_data/ready_for_claim/done),
             messages[], extractedData, context
```

### 8. **Gateway ↔ Redis**
```
Idempotencia & Debouncing:
- Clave: `idempotency:{externalMessageId}:{channel}`
- TTL: 86400 segundos (24 horas)
- Debouncing: acumula mensajes en ventana temporal
```

---

## ⚙️ Configuración Global

### Variables de Entorno Requeridas

#### Gateway (.env)
```bash
# Node
NODE_ENV=development
PORT=3000

# NATS
NATS_SERVERS=nats://localhost:4222

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_ACCESS_SECRET=your-secret-key
JWT_REFRESH_SECRET=refresh-secret-key
JWT_ACCESS_EXPIRATION=900           # 15 min
JWT_REFRESH_EXPIRATION=604800       # 7 days

# Inbound Processing
INBOUND_IDEMPOTENCY_TTL_SECONDS=86400      # 24 horas
INBOUND_CONTEXT_TTL_SECONDS=86400
INBOUND_MESSAGE_HISTORY_LIMIT=20
INBOUND_MIN_DEBOUNCE_MS=800
INBOUND_MIN_DEBOUNCE_WHATSAPP_MS=2000
INBOUND_MAX_BATCH_SIZE=10

# WhatsApp
WHATSAPP_BUSINESS_ACCOUNT_ID=your-account-id
WHATSAPP_ACCESS_TOKEN=your-access-token
WHATSAPP_VERIFY_TOKEN=verify-token
WHATSAPP_API_VERSION=v18.0
```

#### MS-Auth (.env)
```bash
NODE_ENV=development
NATS_SERVERS=nats://localhost:4222

# Database (ms-auth:5433)
DATABASE_URL=postgresql://user:password@postgres-auth:5433/smartservice_auth

# JWT
JWT_ACCESS_SECRET=your-secret-key
JWT_REFRESH_SECRET=refresh-secret-key
JWT_ACCESS_EXPIRATION=900
JWT_REFRESH_EXPIRATION=604800
```

#### MS-Reclamos (.env)
```bash
NODE_ENV=development
NATS_SERVERS=nats://localhost:4222

# Database (ms-reclamos:5432)
DATABASE_URL=postgresql://user:password@postgres:5432/smartservice_reclamos
```

#### MS-AI (.env)
```bash
NODE_ENV=development
NATS_SERVERS=nats://localhost:4222

# Redis
REDIS_URL=redis://localhost:6379

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
OPENAI_MAX_TOKENS=500

# AI Configuration
AI_CONVERSATION_TTL_SECONDS=86400

# Geographic Constraints
GEO_ALLOWED_COUNTRIES=Argentina
GEO_ALLOWED_ADMIN_AREAS=Funes,Santa Fe

# Rate Limiting
AI_RATE_LIMIT_PER_MINUTE=100
AI_RATE_LIMIT_PER_HOUR=1000

# Circuit Breaker
AI_CIRCUIT_BREAKER_THRESHOLD=3
AI_CIRCUIT_BREAKER_TIMEOUT_MS=300000
```

#### MS-Geo (.env)
```bash
NODE_ENV=development
NATS_SERVERS=nats://localhost:4222

# Google Maps
GOOGLE_MAPS_API_KEY=your-api-key

# Geographic Constraints
GEO_ALLOWED_COUNTRIES=Argentina
GEO_ALLOWED_ADMIN_AREAS=Funes,Santa Fe
GEO_ALLOWED_ZONES=Centro,Norte,Sur,Oeste
```

---

## 📡 Patrones de Comunicación

### 1. Pub/Sub (Asincrónico)
**Usado para**: Eventos que no requieren respuesta inmediata
```typescript
// Publicador
natsClient.publish('ai.process-inbound-message', JSON.stringify(payload));

// Suscriptor
natsClient.subscribe('ai.process-inbound-message', (msg) => {
  const data = JSON.parse(msg.data);
  // procesar
});
```

### 2. Request/Reply (Sincrónico)
**Usado para**: Operaciones que requieren respuesta inmediata
```typescript
// Cliente (MS-AI)
const reply = await firstValueFrom(
  natsClient.request('geo.geocode-address', JSON.stringify(address), { timeout: 30000 })
);

// Servidor (MS-Geo)
natsClient.subscribe('geo.geocode-address', (msg) => {
  const result = geocodeAddress(msg.data);
  msg.respond(JSON.stringify(result));
});
```

### 3. JWT Authentication
**Flujo**:
1. Usuario envía credenciales a Gateway
2. Gateway solicita auth a MS-Auth
3. MS-Auth valida y retorna JWT
4. Gateway valida JWT en cada request
5. Incluye claims: userId, roles, permissions

---

## 🔒 Seguridad

### Autenticación
- **Método**: JWT (JSON Web Tokens)
- **Storage**: HTTP-only cookies o Authorization header
- **Refresh**: Tokens de refresco rotados
- **Revocación**: Tracked en PostgreSQL (revokedAt field)

### Protección AI
- **Prompt Injection**: Detección de patrones maliciosos
- **Rate Limiting**: 100/min, 1000/hora por cliente
- **Input Validation**: Límite 2000 caracteres
- **Output Validation**: Verificación de tipos de respuesta
- **Circuit Breaker**: Fallback a regex si OpenAI falla 3 veces

### RBAC (Role-Based Access Control)
- **Roles**: Operator, AreaManager, Official, MunicipalAdmin, PlatformAdmin
- **Enforcement**: Guards en Gateway y MS-Auth
- **Permissions**: Granular por acción (create, read, update, delete)

### CORS & Rate Limiting
- Configurado en Gateway
- Por IP del cliente
- Por JWT token (clientId)

---

## 🚀 Entorno y Despliegue

### Desarrollo Local
```bash
# 1. Docker Compose levanta PostgreSQL + Redis + NATS
docker-compose up -d

# 2. Instalar dependencias en cada servicio
cd gateway && npm install
cd ../ms-auth && npm install
# ... etc

# 3. Ejecutar migraciones Prisma
cd ms-auth && npx prisma migrate dev
cd ../ms-reclamos && npx prisma migrate dev

# 4. Iniciar servicios
npm run start:dev
```

### Docker Compose Services
```yaml
services:
  postgres-auth:
    image: postgres:16-alpine
    ports: [5433:5432]
    environment:
      POSTGRES_DB: smartservice_auth
      POSTGRES_PASSWORD: password

  postgres:
    image: postgres:16-alpine
    ports: [5432:5432]
    environment:
      POSTGRES_DB: smartservice_reclamos
      POSTGRES_PASSWORD: password

  redis:
    image: redis:5-alpine
    ports: [6379:6379]

  nats:
    image: nats:latest
    ports: [4222:4222]
```

### Producción (Planeado)
- **Cloud**: Railway o AWS
- **Containers**: Docker
- **CI/CD**: GitHub Actions
- **Monitoreo**: Prometheus + Grafana (planeado)
- **Logs**: Centralizados (planeado)

---

## 📊 Resumen de Servicios

| Servicio | Puerto | Responsabilidad | Base de Datos |
|----------|--------|-----------------|---------------|
| **gateway** | 3000 | HTTP entry, JWT auth, NATS orchestration | - |
| **ms-auth** | interno | User auth, JWT issuing, RBAC | PostgreSQL:5433 |
| **ms-reclamos** | interno | Claims CRUD, persistence | PostgreSQL:5432 |
| **ms-ai** | interno | NLP processing, conversation state, orchestration | Redis |
| **ms-geo** | interno | Address geocoding, validation | - |

---

## 🔗 Véase También
- [001_GATEWAY.md](001_GATEWAY.md) - Detalles del Gateway
- [002_MS_AUTH.md](002_MS_AUTH.md) - Autenticación y autorización
- [003_MS_RECLAMOS.md](003_MS_RECLAMOS.md) - Gestión de reclamos
- [004_MS_AI.md](004_MS_AI.md) - Procesamiento de IA
- [005_MS_GEO.md](005_MS_GEO.md) - Geolocalización
- [006_FRONTEND.md](006_FRONTEND.md) - Aplicaciones frontend
- [007_DATABASE_SCHEMA.md](007_DATABASE_SCHEMA.md) - Esquemas de datos

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
