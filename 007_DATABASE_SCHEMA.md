# DATABASE SCHEMAS

---

## 🗄️ MS-AUTH Database (PostgreSQL:5433)

### Database: smartservice_auth

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
  "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_email CHECK (
    email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'
  )
);

CREATE INDEX idx_user_username ON "User"("username");
CREATE INDEX idx_user_email ON "User"("email");
CREATE INDEX idx_user_isActive ON "User"("isActive");
```

**Relaciones**:
- 1:N con UserRole
- 1:N con RefreshToken
- 1:N con UserPermission

---

### Tabla: RefreshToken
```sql
CREATE TABLE "RefreshToken" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "userId" UUID NOT NULL REFERENCES "User"("id") ON DELETE CASCADE,
  "token" VARCHAR(512) NOT NULL,
  "expiresAt" TIMESTAMP NOT NULL,
  "revokedAt" TIMESTAMP,
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_revocation CHECK (
    ("revokedAt" IS NULL) OR ("revokedAt" IS NOT NULL)
  )
);

CREATE INDEX idx_refresh_token_userId ON "RefreshToken"("userId");
CREATE INDEX idx_refresh_token_token ON "RefreshToken"("token");
CREATE INDEX idx_refresh_token_expiresAt ON "RefreshToken"("expiresAt");
```

---

### Tabla: Role
```sql
CREATE TABLE "Role" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" VARCHAR(50) UNIQUE NOT NULL,
  "description" TEXT,
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_role_name CHECK (
    name IN (
      'operator',
      'area_manager',
      'official',
      'municipal_admin',
      'platform_admin'
    )
  )
);

-- Predefined roles
INSERT INTO "Role" (name, description) VALUES
  ('operator', 'Operador básico, puede crear y ver reclamos'),
  ('area_manager', 'Gestor de área, puede asignar y supervisar'),
  ('official', 'Oficial municipal, puede actualizar estados'),
  ('municipal_admin', 'Administrador municipal, gestión completa'),
  ('platform_admin', 'Administrador de plataforma, acceso total');
```

---

### Tabla: UserRole (M:N)
```sql
CREATE TABLE "UserRole" (
  "userId" UUID NOT NULL REFERENCES "User"("id") ON DELETE CASCADE,
  "roleId" UUID NOT NULL REFERENCES "Role"("id") ON DELETE CASCADE,
  "assignedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  PRIMARY KEY ("userId", "roleId")
);

CREATE INDEX idx_user_role_userId ON "UserRole"("userId");
CREATE INDEX idx_user_role_roleId ON "UserRole"("roleId");
```

---

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

-- Predefined permissions
INSERT INTO "Permission" (name, description, resource, action) VALUES
  ('read:reclamos', 'Leer reclamos', 'reclamos', 'read'),
  ('create:reclamo', 'Crear reclamo', 'reclamos', 'create'),
  ('update:reclamo', 'Actualizar reclamo', 'reclamos', 'update'),
  ('delete:reclamo', 'Eliminar reclamo', 'reclamos', 'delete'),
  ('assign:reclamo', 'Asignar reclamo', 'reclamos', 'assign'),
  ('read:users', 'Leer usuarios', 'users', 'read'),
  ('create:user', 'Crear usuario', 'users', 'create'),
  ('update:user', 'Actualizar usuario', 'users', 'update'),
  ('delete:user', 'Eliminar usuario', 'users', 'delete'),
  ('manage:roles', 'Gestionar roles', 'roles', 'manage'),
  ('manage:permissions', 'Gestionar permisos', 'permissions', 'manage');
```

---

### Tabla: RolePermission (M:N)
```sql
CREATE TABLE "RolePermission" (
  "roleId" UUID NOT NULL REFERENCES "Role"("id") ON DELETE CASCADE,
  "permissionId" UUID NOT NULL REFERENCES "Permission"("id") ON DELETE CASCADE,
  "grantedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  PRIMARY KEY ("roleId", "permissionId")
);

CREATE INDEX idx_role_permission_roleId ON "RolePermission"("roleId");
CREATE INDEX idx_role_permission_permissionId ON "RolePermission"("permissionId");
```

---

### Tabla: UserPermission (Permisos directos)
```sql
CREATE TABLE "UserPermission" (
  "userId" UUID NOT NULL REFERENCES "User"("id") ON DELETE CASCADE,
  "permissionId" UUID NOT NULL REFERENCES "Permission"("id") ON DELETE CASCADE,
  "grantedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  PRIMARY KEY ("userId", "permissionId")
);

CREATE INDEX idx_user_permission_userId ON "UserPermission"("userId");
```

---

## 🗄️ MS-RECLAMOS Database (PostgreSQL:5432)

### Database: smartservice_reclamos

### Tabla: Reclamo
```sql
CREATE TABLE "Reclamo" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Identificación
  "numeroReclamo" VARCHAR(50) UNIQUE NOT NULL,
  "canal" VARCHAR(50) NOT NULL,
  "canalIdentidad" JSONB NOT NULL,  -- { "type": "phone", "value": "+549..." }
  
  -- Contenido
  "descripcion" TEXT NOT NULL,
  "categoria" VARCHAR(100) NOT NULL,
  "prioridad" VARCHAR(50) NOT NULL,
  
  -- Ubicación
  "latitud" DOUBLE PRECISION,
  "longitud" DOUBLE PRECISION,
  "direccion" TEXT,
  "zonaGeografica" VARCHAR(100),
  "municipio" VARCHAR(100),
  
  -- Estado
  "estado" VARCHAR(50) NOT NULL DEFAULT 'PENDIENTE',
  "estado_anterior" VARCHAR(50),
  "fecha_resolucion" TIMESTAMP,
  
  -- Asignación
  "asignado_a" UUID,
  "area_responsable" VARCHAR(100),
  
  -- Metadata
  "evidencia" JSONB,  -- { "imageUrls": [...], "documentUrls": [...] }
  "tags" TEXT[],
  "observaciones" TEXT,
  
  -- Timestamps
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  "resueltoEn" TIMESTAMP,
  
  -- Auditoría
  "creado_por" UUID,
  "ultima_modificacion_por" UUID,
  
  CONSTRAINT valid_prioridad CHECK (
    prioridad IN ('BAJA', 'NORMAL', 'ALTA', 'URGENTE')
  ),
  CONSTRAINT valid_estado CHECK (
    estado IN ('PENDIENTE', 'EN_PROCESO', 'RESUELTO', 'RECHAZADO', 'DUPLICADO')
  ),
  CONSTRAINT valid_categoria CHECK (
    categoria IN (
      'agua_y_cloacas',
      'alumbrado',
      'baches_y_pavimento',
      'arbolado',
      'residuos',
      'electricidad',
      'gas',
      'transporte',
      'infraestructura',
      'otros'
    )
  ),
  CONSTRAINT valid_canal CHECK (
    canal IN ('whatsapp', 'web', 'email', 'instagram', 'llamada_telefonica')
  )
);

-- Índices para búsqueda
CREATE INDEX idx_reclamo_numeroReclamo ON "Reclamo"("numeroReclamo");
CREATE INDEX idx_reclamo_estado ON "Reclamo"("estado");
CREATE INDEX idx_reclamo_categoria ON "Reclamo"("categoria");
CREATE INDEX idx_reclamo_prioridad ON "Reclamo"("prioridad");
CREATE INDEX idx_reclamo_createdAt ON "Reclamo"("createdAt" DESC);
CREATE INDEX idx_reclamo_asignado_a ON "Reclamo"("asignado_a");
CREATE INDEX idx_reclamo_ubicacion ON "Reclamo" USING GIST (
  ST_MakePoint("longitud", "latitud")
);  -- Spatial index (si se habilita PostGIS)
```

---

### Tabla: MensajeHistorial
```sql
CREATE TABLE "MensajeHistorial" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "reclamoId" UUID NOT NULL REFERENCES "Reclamo"("id") ON DELETE CASCADE,
  
  "tipo" VARCHAR(50),
  "contenido" TEXT NOT NULL,
  "remitente" VARCHAR(255),
  "canal" VARCHAR(50),
  
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_tipo CHECK (
    tipo IN ('entrada', 'estado_cambio', 'asignacion', 'resolucion', 'nota_interna')
  )
);

CREATE INDEX idx_historial_reclamoId ON "MensajeHistorial"("reclamoId");
CREATE INDEX idx_historial_createdAt ON "MensajeHistorial"("createdAt" DESC);
```

---

## 📊 Enums

### ReclamoEstado
```typescript
enum ReclamoEstado {
  PENDIENTE = 'PENDIENTE',        // Nuevo, no procesado
  EN_PROCESO = 'EN_PROCESO',      // Asignado, en trabajo
  RESUELTO = 'RESUELTO',          // Completado satisfactoriamente
  RECHAZADO = 'RECHAZADO',        // No aplica
  DUPLICADO = 'DUPLICADO'         // Duplicado de otro
}
```

### ReclamoCategoria
```typescript
enum ReclamoCategoria {
  AGUA_Y_CLOACAS = 'agua_y_cloacas',
  ALUMBRADO = 'alumbrado',
  BACHES_Y_PAVIMENTO = 'baches_y_pavimento',
  ARBOLADO = 'arbolado',
  RESIDUOS = 'residuos',
  ELECTRICIDAD = 'electricidad',
  GAS = 'gas',
  TRANSPORTE = 'transporte',
  INFRAESTRUCTURA = 'infraestructura',
  OTROS = 'otros'
}
```

### ReclamoPrioridad
```typescript
enum ReclamoPrioridad {
  BAJA = 'BAJA',
  NORMAL = 'NORMAL',
  ALTA = 'ALTA',
  URGENTE = 'URGENTE'
}
```

### Canal
```typescript
enum Canal {
  WHATSAPP = 'whatsapp',
  WEB = 'web',
  EMAIL = 'email',
  INSTAGRAM = 'instagram',
  LLAMADA_TELEFONICA = 'llamada_telefonica'
}
```

### MensajeHistorialTipo
```typescript
enum MensajeHistorialTipo {
  ENTRADA = 'entrada',
  ESTADO_CAMBIO = 'estado_cambio',
  ASIGNACION = 'asignacion',
  RESOLUCION = 'resolucion',
  NOTA_INTERNA = 'nota_interna'
}
```

---

## 📈 Consultas Comunes

### Reclamos por estado
```sql
SELECT estado, COUNT(*) as cantidad
FROM "Reclamo"
GROUP BY estado;
```

**Resultado esperado**:
```
estado      | cantidad
------------|----------
PENDIENTE   | 45
EN_PROCESO  | 23
RESUELTO    | 112
RECHAZADO   | 8
DUPLICADO   | 5
```

### Reclamos pendientes sin asignar
```sql
SELECT id, numeroReclamo, descripcion, prioridad, createdAt
FROM "Reclamo"
WHERE estado = 'PENDIENTE' AND asignado_a IS NULL
ORDER BY prioridad DESC, createdAt ASC
LIMIT 20;
```

### Tiempo promedio de resolución
```sql
SELECT
  categoria,
  ROUND(AVG(EXTRACT(EPOCH FROM (resueltoEn - createdAt)) / 3600), 2) as horas_promedio
FROM "Reclamo"
WHERE estado = 'RESUELTO'
GROUP BY categoria
ORDER BY horas_promedio DESC;
```

### Reclamos por área geográfica
```sql
SELECT zonaGeografica, COUNT(*) as cantidad
FROM "Reclamo"
GROUP BY zonaGeografica
ORDER BY cantidad DESC;
```

### Historial completo de un reclamo
```sql
SELECT
  tipo,
  contenido,
  remitente,
  createdAt
FROM "MensajeHistorial"
WHERE "reclamoId" = 'uuid-123'
ORDER BY createdAt ASC;
```

---

## 🔍 Vista de Estadísticas (Optional - Materialized View)

```sql
CREATE MATERIALIZED VIEW reclamo_stats AS
SELECT
  DATE(r."createdAt") as fecha,
  r.categoria,
  r.estado,
  COUNT(*) as cantidad,
  ROUND(AVG(r.prioridad::INT), 2) as prioridad_promedio
FROM "Reclamo" r
GROUP BY DATE(r."createdAt"), r.categoria, r.estado;

CREATE INDEX idx_reclamo_stats_fecha ON reclamo_stats(fecha DESC);
```

---

## 🔗 Relaciones ERD

```
┌─────────────────────────────────────────────────────────────────┐
│ MS-AUTH Database                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐      1:N       ┌──────────────┐                  │
│  │   User   │─────────────→ │RefreshToken  │                  │
│  └──────────┘                └──────────────┘                  │
│       │ M:N                                                     │
│       ├──────────┬──────────┐                                   │
│       │          │          │                                   │
│  ┌────────────┐  │    ┌──────────────┐                         │
│  │ UserRole   │  │    │UserPermission│                         │
│  │ (M:N join) │  │    │  (M:N join)  │                         │
│  └────────────┘  │    └──────────────┘                         │
│       │          │          │                                   │
│       └─→┌──────────┐       │                                   │
│          │  Role    │←──────┘                                   │
│          │ M:N join │                                           │
│          └──────────┘                                           │
│               │ M:N                                             │
│               └─→ ┌──────────────┐                              │
│                  │ Permission   │                              │
│                  └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ MS-RECLAMOS Database                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐         1:N        ┌─────────────────────┐  │
│  │   Reclamo    │────────────────→ │ MensajeHistorial    │  │
│  └──────────────┘                  └─────────────────────┘  │
│       │                                                       │
│       │ Foreign Key                                          │
│       └─→ User.id (de ms-auth)                              │
│                                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Seguridad de Datos

### Campos Sensibles
- User.passwordHash: Hashed con bcrypt (min 10 rounds)
- RefreshToken.token: JWT firmado
- Reclamo.canalIdentidad: Puede contener teléfono (PII)

### Backup Strategy
- **Frequency**: Daily snapshots
- **Retention**: 30 días
- **Storage**: Encrypted on disk

### GDPR Compliance
- Derecho a ser olvidado: Soft delete de usuarios
- Exportación de datos: Query completa del usuario
- Consentimiento: Registrado en auditoría

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
