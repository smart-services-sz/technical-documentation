# 📚 TECHNICAL_DOCUMENTATION

Documentación técnica completa del proyecto SmartService. Incluye arquitectura global, especificaciones de cada microservicio, bases de datos y frontend.

---

## 📖 Índice de Documentos

### 1. [000_GLOBAL_ARCHITECTURE.md](000_GLOBAL_ARCHITECTURE.md)
**Descripción**: Vista general del sistema  
**Contenido**:
- Descripción general de SmartService
- Diagrama de arquitectura
- Matriz de tecnologías
- Relaciones entre servicios
- Flujo principal de creación de reclamos
- Patrones NATS (pub/sub vs request/reply)
- Configuración Docker Compose
- Template de variables de entorno

**Para quién**: Architects, project managers, nuevos desarrolladores

---

### 2. [001_GATEWAY.md](001_GATEWAY.md)
**Descripción**: API Gateway - Punto de entrada único  
**Contenido**:
- Endpoints HTTP disponibles
- Flujo de autenticación
- Debouncing y idempotencia
- Integración con NATS
- Validación y sanitización
- Estructura de errors
- Rate limiting

**Para quién**: Backend developers, frontend developers, DevOps

---

### 3. [002_MS_AUTH.md](002_MS_AUTH.md)
**Descripción**: Servicio de autenticación y RBAC  
**Contenido**:
- Métodos de autenticación
- JWT y refresh tokens
- Matriz de roles y permisos
- NATS request/reply topics
- Database schema completo
- Bcrypt password hashing
- Rate limiting por usuario

**Para quién**: Security team, backend developers

---

### 4. [003_MS_RECLAMOS.md](003_MS_RECLAMOS.md)
**Descripción**: Servicio de gestión de reclamos/complaints  
**Contenido**:
- CRUD de reclamos
- Máquina de estados (state transitions)
- Database schema
- NATS topics (pub/sub y request/reply)
- Validaciones de negocio
- Auditoría y historial

**Para quién**: Business analysts, backend developers

---

### 5. [004_MS_AI.md](004_MS_AI.md)
**Descripción**: Servicio de procesamiento conversacional con IA  
**Contenido**:
- Máquina de estados (4 fases)
- OpenAI GPT-4o-mini integration
- Seguridad: detección de prompt injection
- Redis state management
- Rate limiting y circuit breaker
- Validaciones de entrada
- Fallback strategies

**Para quién**: ML engineers, backend developers, security team

---

### 6. [005_MS_GEO.md](005_MS_GEO.md)
**Descripción**: Servicio de geocodificación y validación geográfica  
**Contenido**:
- Google Maps Geocoding API integration
- Validación de restricciones geográficas
- Extracción de componentes de dirección
- Retry logic con exponential backoff
- Manejo de errores

**Para quién**: Backend developers, DevOps

---

### 7. [006_FRONTEND.md](006_FRONTEND.md)
**Descripción**: Aplicaciones frontend (Admin Dashboard + Public Site)  
**Contenido**:
- **Public Site**: HTML/CSS/JS estático, landing page
- **Admin Dashboard**: Next.js 16, React 19, admin panel
- Autenticación y cookies
- API integration patterns
- Componentes principales
- Rutas y funcionalidades

**Para quién**: Frontend developers, product managers

---

### 8. [007_DATABASE_SCHEMA.md](007_DATABASE_SCHEMA.md)
**Descripción**: Referencia completa de schemas de bases de datos  
**Contenido**:
- MS-AUTH schema (User, Role, Permission, etc.)
- MS-RECLAMOS schema (Reclamo, MensajeHistorial)
- Enums y tipos
- Índices y optimizaciones
- Consultas comunes
- ERD (Entity Relationship Diagram)
- Estrategia de backup

**Para quién**: Database administrators, backend developers

---

## 🚀 Quick Start

### Iniciarse rápidamente
1. Leer [000_GLOBAL_ARCHITECTURE.md](000_GLOBAL_ARCHITECTURE.md) para entender el sistema
2. Según tu rol:
   - **Backend**: Léé los microservicio específico (001-005)
   - **Frontend**: Léé [006_FRONTEND.md](006_FRONTEND.md)
   - **DBA**: Léé [007_DATABASE_SCHEMA.md](007_DATABASE_SCHEMA.md)

### Entender un flujo específico
- Flujo de creación de reclamo vía WhatsApp: [000_GLOBAL_ARCHITECTURE.md](000_GLOBAL_ARCHITECTURE.md#-flujo-principal) → [001_GATEWAY.md](001_GATEWAY.md) → [004_MS_AI.md](004_MS_AI.md) → [005_MS_GEO.md](005_MS_GEO.md) → [003_MS_RECLAMOS.md](003_MS_RECLAMOS.md)
- Autenticación y autorización: [002_MS_AUTH.md](002_MS_AUTH.md)

### Encontrar algo específico
- Usar Ctrl+F en el documento relevante
- Buscar por "NATS" para ver tópicos
- Buscar por "Tabla:" para ver schemas

---

## 📊 Technology Stack

```
┌─────────────────────────────────────────┐
│ Frontend                                 │
├─────────────────────────────────────────┤
│ • Next.js 16.2.6 (Admin Dashboard)      │
│ • React 19.2.4                          │
│ • Tailwind CSS                          │
│ • React Query (@tanstack/react-query)   │
│ • HTML5/CSS3/JS (Public Site)           │
└─────────────────────────────────────────┘
         ↓ HTTP REST API
┌─────────────────────────────────────────┐
│ API Gateway (001_GATEWAY)               │
│ • NestJS 11.0.0                         │
│ • Express                               │
│ • Passport.js                           │
└─────────────────────────────────────────┘
         ↓ NATS Pub/Sub + Request/Reply
┌─────────────────────────────────────────┐
│ Microservices (NestJS 11.0.0)           │
├─────────────────────────────────────────┤
│ • MS-Auth (002): Autenticación + RBAC   │
│ • MS-Reclamos (003): CRUD de claims     │
│ • MS-AI (004): NLP + OpenAI             │
│ • MS-Geo (005): Geocoding               │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│ External APIs                           │
├─────────────────────────────────────────┤
│ • OpenAI GPT-4o-mini (004_MS_AI)        │
│ • Google Maps Geocoding (005_MS_GEO)    │
│ • WhatsApp Business API (webhook)       │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│ Data Layer                              │
├─────────────────────────────────────────┤
│ • PostgreSQL 16 (2 instancias)          │
│   - ms-auth:5433                        │
│   - ms-reclamos:5432                    │
│ • Redis 5 (Session state, idempotency)  │
└─────────────────────────────────────────┘
```

---

## 🔄 Inter-Service Communication

### NATS Topics Map

**Request/Reply (Sync)**:
```
Gateway → Auth Service
  auth.authenticate
  auth.validate-token
  auth.refresh-token
  auth.get-user
  auth.check-permission

AI Service → Geo Service
  geo.geocode-address (30s timeout)
```

**Pub/Sub (Async)**:
```
AI Service → Reclamos Service
  reclamos.create

Reclamos Service → (broadcast)
  reclamos.updated
  reclamos.resolved
```

Ver [000_GLOBAL_ARCHITECTURE.md](000_GLOBAL_ARCHITECTURE.md) para detalles completos.

---

## 🗄️ Database Map

```
PostgreSQL (ms-auth):5433
├── User
├── RefreshToken
├── Role (predefined: operator, area_manager, official, etc.)
├── UserRole (M:N)
├── Permission
├── RolePermission (M:N)
└── UserPermission (M:N)

PostgreSQL (ms-reclamos):5432
├── Reclamo
│   ├── numeroReclamo (UNIQUE)
│   ├── estado (PENDIENTE, EN_PROCESO, RESUELTO, RECHAZADO, DUPLICADO)
│   ├── categoria (agua_y_cloacas, alumbrado, baches_y_pavimento, ...)
│   └── prioridad (BAJA, NORMAL, ALTA, URGENTE)
└── MensajeHistorial (audit log)

Redis
├── conversation:{channelId}:{channel} → state, messages, extractedData
├── idempotency:{externalMessageId} → messageId (24h TTL)
└── debounce:{channelId} → count (8s window)
```

---

## 🔐 Security Considerations

### Authentication
- JWT con RS256 (RS = asymmetric)
- Refresh token rotation
- Token revocation tracking
- Rate limiting por usuario

### API Security
- CORS configurado
- HTTPS enforced (produción)
- Input validation (class-validator)
- SQL injection prevention (Prisma ORM)

### AI Security
- Detección de prompt injection (regex patterns)
- Rate limiting global: 100/min, 1000/hora
- Circuit breaker para OpenAI (3 fallos consecutivos)
- Fallback a regex si OpenAI falla

Ver [004_MS_AI.md](004_MS_AI.md) para detalles de medidas de seguridad.

---

## 📋 Checklist para desarrollo

### Setup inicial
- [ ] Clonar repositorio
- [ ] Instalar Docker
- [ ] Copiar `.env.example` → `.env`
- [ ] `docker-compose up`
- [ ] Revisar [000_GLOBAL_ARCHITECTURE.md](000_GLOBAL_ARCHITECTURE.md)

### Antes de hacer un PR
- [ ] Leer documentación del servicio relevante
- [ ] Verificar NATS topics no cambien sin coordinar
- [ ] Database changes → ejecutar migrations
- [ ] Tests unitarios pasen

### Agregar una nueva feature
1. Documentar en el archivo del servicio relevante
2. Actualizar este README si es necesario
3. Agregar tests
4. Documentar cambios NATS si hay

---

## 📝 Notas de Implementación

### Limitaciones Actuales
- ❌ No hay S3/object storage (evidencia guardada en JSON)
- ❌ No hay sistema de notificaciones (email, SMS)
- ❌ No hay audit trail completo de cambios a nivel de sistema
- ⚠️ Admin dashboard sin mapas interactivos

### Planeado para futuro
- [ ] Integración S3 para almacenar evidencia
- [ ] Sistema de notificaciones (Twilio, SendGrid)
- [ ] Audit trail centralizado
- [ ] Analytics dashboard
- [ ] Mobile app para ciudadanos
- [ ] AI en tiempo real para categorización automática

---

## 🤝 Contribuir

Cuando agregues cambios:
1. Actualiza el documento del servicio correspondiente
2. Mantén ejemplos de código actualizados
3. Agrega notas de breaking changes
4. Sincroniza con changelog

---

## 📞 Contact & Support

- **Architecture Questions**: Ver [000_GLOBAL_ARCHITECTURE.md](000_GLOBAL_ARCHITECTURE.md)
- **Microservice Specifics**: Ver documento correspondiente (001-005)
- **Database**: Ver [007_DATABASE_SCHEMA.md](007_DATABASE_SCHEMA.md)
- **Frontend**: Ver [006_FRONTEND.md](006_FRONTEND.md)

---

**Última actualización**: 11 de Mayo de 2026  
**Versión de documentación**: 1.0.0  
**Estado**: Completa y actualizada
