# Microservicio: MS-RECLAMOS

**Tipo**: Claims/Complaints Management Service  
**Base de Datos**: PostgreSQL:5432 (Prisma ORM)  
**Comunicación**: NATS (Pub/Sub)  
**Framework**: NestJS  

---

## 📋 Tabla de Contenidos

1. [Propósito](#propósito)
2. [Esquema de Base de Datos](#esquema-de-base-de-datos)
3. [NATS Interface](#nats-interface)
4. [Controladores y Servicios](#controladores-y-servicios)
5. [Entidades y Enums](#entidades-y-enums)
6. [Flujos de Negocio](#flujos-de-negocio)
7. [Validaciones](#validaciones)
8. [Error Handling](#error-handling)
9. [Configuración](#configuración)

---

## 🎯 Propósito

Gestionar el ciclo de vida completo de reclamos:
- ✅ Crear reclamos nuevos
- ✅ Almacenar información persistente
- ✅ Recuperar y listar reclamos
- ✅ Actualizar estados
- ✅ Historial de mensajes
- ✅ Tracking y auditoría

---

## 🗄️ Esquema de Base de Datos

### Tabla: Reclamo
```sql
CREATE TABLE "Reclamo" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Identificación
  "numeroReclamo" VARCHAR(50) UNIQUE NOT NULL,
  "canal" VARCHAR(50) NOT NULL,                -- whatsapp, web, email, etc.
  "canalIdentidad" JSONB NOT NULL,            -- { type, value }
  
  -- Contenido
  "descripcion" TEXT NOT NULL,
  "categoria" VARCHAR(100) NOT NULL,
  "prioridad" VARCHAR(50) NOT NULL,           -- BAJA, NORMAL, ALTA, URGENTE
  
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
  "asignado_a" UUID REFERENCES "User"("id"),
  "area_responsable" VARCHAR(100),
  
  -- Metadata
  "evidencia" JSONB,                         -- { imageUrls: [], documentUrls: [] }
  "tags" TEXT[],
  "observaciones" TEXT,
  
  -- Timestamps
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  "resueltoEn" TIMESTAMP,
  
  -- Auditoría
  "creado_por" UUID,
  "ultima_modificacion_por" UUID,
  
  CONSTRAINT valid_prioridad CHECK (prioridad IN ('BAJA', 'NORMAL', 'ALTA', 'URGENTE')),
  CONSTRAINT valid_estado CHECK (estado IN ('PENDIENTE', 'EN_PROCESO', 'RESUELTO', 'RECHAZADO', 'DUPLICADO'))
);

INDEXES:
- numeroReclamo (UNIQUE)
- estado
- categoria
- prioridad
- latitud, longitud (spatial)
- createdAt DESC
- asignado_a
```

### Tabla: MensajeHistorial
```sql
CREATE TABLE "MensajeHistorial" (
  "id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "reclamoId" UUID NOT NULL REFERENCES "Reclamo"("id") ON DELETE CASCADE,
  
  "tipo" VARCHAR(50),                -- entrada, estado_cambio, asignacion, resolucion
  "contenido" TEXT NOT NULL,
  "remitente" VARCHAR(255),
  "canal" VARCHAR(50),
  
  "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_tipo CHECK (
    tipo IN ('entrada', 'estado_cambio', 'asignacion', 'resolucion', 'nota_interna')
  )
);

INDEXES:
- reclamoId
- createdAt DESC
```

### Enums

**ReclamoEstado**:
```typescript
enum ReclamoEstado {
  PENDIENTE = 'PENDIENTE',        // Nuevo, no procesado
  EN_PROCESO = 'EN_PROCESO',      // Asignado, en trabajo
  RESUELTO = 'RESUELTO',          // Completado satisfactoriamente
  RECHAZADO = 'RECHAZADO',        // No aplica/fuera de jurisdicción
  DUPLICADO = 'DUPLICADO'         // Duplicado de otro reclamo
}
```

**ReclamoCategoria**:
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

**ReclamoPrioridad**:
```typescript
enum ReclamoPrioridad {
  BAJA = 'BAJA',
  NORMAL = 'NORMAL',
  ALTA = 'ALTA',
  URGENTE = 'URGENTE'
}
```

**Canal**:
```typescript
enum Canal {
  WHATSAPP = 'whatsapp',
  WEB = 'web',
  EMAIL = 'email',
  INSTAGRAM = 'instagram',
  LLAMADA_TELEFONICA = 'llamada_telefonica'
}
```

---

## 📡 NATS Interface

### Suscripciones

#### 1. reclamos.create (Pub/Sub)
**Origen**: MS-AI  
**Descripción**: Crear nuevo reclamo  

**Message**:
```json
{
  "canal": "whatsapp",
  "canalIdentidad": { "type": "phone", "value": "+5493415551234" },
  "descripcion": "Hay un bache grande en la Av. Principal entre 5ta y 6ta",
  "categoria": "baches_y_pavimento",
  "prioridad": "NORMAL",
  "ubicacion": {
    "latitud": -32.9432,
    "longitud": -60.6537,
    "direccion": "Av. Principal 500, Funes",
    "zona": "Centro"
  },
  "evidencia": {
    "imagenes": ["url1", "url2"]
  },
  "timestamp": 1715425800000
}
```

**Procesamiento**:
1. Validar estructura del mensaje
2. Generar numero único para reclamo
3. Crear registro en PostgreSQL tabla "Reclamo"
4. Crear entrada en historial de mensajes
5. Retornar confirmación

**Response** (implícito en DB):
```json
{
  "success": true,
  "reclamoId": "uuid-123",
  "numeroReclamo": "REC-2026-000001",
  "estado": "PENDIENTE"
}
```

#### 2. reclamos.find-all (Request/Reply)
**Origen**: Gateway, admin-dashboard  
**Descripción**: Listar reclamos con filtros  

**Request**:
```json
{
  "filters": {
    "estado": "PENDIENTE",
    "categoria": "baches_y_pavimento",
    "prioridad": "ALTA",
    "dateFrom": "2026-05-01",
    "dateTo": "2026-05-31"
  },
  "pagination": {
    "page": 1,
    "limit": 20
  },
  "sort": "createdAt:DESC"
}
```

**Response**:
```json
{
  "data": [
    {
      "id": "uuid-123",
      "numeroReclamo": "REC-2026-000001",
      "descripcion": "Bache...",
      "categoria": "baches_y_pavimento",
      "prioridad": "NORMAL",
      "estado": "PENDIENTE",
      "ubicacion": { "lat": -32.94, "lng": -60.65 },
      "createdAt": "2026-05-11T10:30:00Z"
    }
  ],
  "total": 150,
  "page": 1,
  "limit": 20,
  "pages": 8
}
```

#### 3. reclamos.find-one (Request/Reply)
**Origen**: Gateway, admin-dashboard  
**Descripción**: Obtener detalle de reclamo  

**Request**:
```json
{
  "reclamoId": "uuid-123"
}
```

**Response**:
```json
{
  "id": "uuid-123",
  "numeroReclamo": "REC-2026-000001",
  "descripcion": "Hay un bache grande...",
  "categoria": "baches_y_pavimento",
  "prioridad": "NORMAL",
  "estado": "EN_PROCESO",
  "ubicacion": {
    "latitud": -32.9432,
    "longitud": -60.6537,
    "direccion": "Av. Principal 500, Funes"
  },
  "asignado_a": "uuid-user-456",
  "area_responsable": "Obras Públicas",
  "historial": [
    {
      "tipo": "entrada",
      "contenido": "Reclamo ingresado vía WhatsApp",
      "createdAt": "2026-05-11T10:30:00Z"
    },
    {
      "tipo": "asignacion",
      "contenido": "Asignado a John Doe",
      "createdAt": "2026-05-11T10:35:00Z"
    }
  ],
  "createdAt": "2026-05-11T10:30:00Z",
  "updatedAt": "2026-05-11T10:35:00Z"
}
```

#### 4. reclamos.update (Request/Reply)
**Origen**: Admin-dashboard, operarios  
**Descripción**: Actualizar estado o información de reclamo  

**Request**:
```json
{
  "reclamoId": "uuid-123",
  "update": {
    "estado": "EN_PROCESO",
    "asignado_a": "uuid-user-456",
    "observaciones": "Iniciado trabajo en sitio",
    "evidencia": {
      "imagenes": ["url-nueva"]
    }
  }
}
```

**Procesamiento**:
1. Obtener reclamo actual
2. Validar cambios permitidos según estado actual
3. Actualizar en DB
4. Registrar en historial
5. Notificar cambio (futuro: ms-notifications)

#### 5. reclamos.remove (Request/Reply)
**Origen**: Admin-dashboard (solo PlatformAdmin)  
**Descripción**: Eliminar reclamo (lógicamente)  

**Request**:
```json
{
  "reclamoId": "uuid-123",
  "razon": "Eliminado por error de entrada"
}
```

---

## 🎮 Controladores y Servicios

### ReclamosController
```typescript
@Controller('reclamos')
export class ReclamosController {
  
  @Post()
  async createReclamo(@Body() dto: CreateReclamoDto): Promise<ReclamoDto> {
    // Recibe de NATS o gateway
  }

  @Get()
  async listReclamos(@Query() filters: FiltersDto): Promise<PaginatedDto<ReclamoDto>> {
    // Lista con filtros
  }

  @Get(':id')
  async getReclamo(@Param('id') id: string): Promise<ReclamoDto> {
    // Obtiene uno por ID
  }

  @Patch(':id')
  async updateReclamo(
    @Param('id') id: string,
    @Body() dto: UpdateReclamoDto
  ): Promise<ReclamoDto> {
    // Actualiza estado, asignación, etc.
  }

  @Delete(':id')
  async removeReclamo(@Param('id') id: string): Promise<void> {
    // Eliminación lógica
  }
}
```

### ReclamosService
**Responsabilidades**:
- CRUD operaciones en PostgreSQL
- Validaciones de negocio
- Transiciones de estado
- Auditoría de cambios

```typescript
async createReclamo(data: ReclamoCreateDto): Promise<Reclamo> {
  // Generar numero único
  const numeroReclamo = await this.generateReclamoNumber();

  // Crear con prioridad basada en categoría
  const prioridad = this.calculatePriority(data.categoria);

  // Insertar en DB (con transacción)
  const reclamo = await this.prisma.reclamo.create({
    data: {
      numeroReclamo,
      canal: data.canal,
      canalIdentidad: data.canalIdentidad,
      descripcion: data.descripcion,
      categoria: data.categoria,
      prioridad,
      latitud: data.ubicacion?.latitud,
      longitud: data.ubicacion?.longitud,
      direccion: data.ubicacion?.direccion,
      estado: 'PENDIENTE',
      createdAt: new Date()
    }
  });

  // Registrar en historial
  await this.prisma.mensajeHistorial.create({
    data: {
      reclamoId: reclamo.id,
      tipo: 'entrada',
      contenido: `Reclamo creado vía ${data.canal}`,
      remitente: 'SISTEMA'
    }
  });

  return reclamo;
}

async updateReclamo(id: string, data: ReclamoUpdateDto): Promise<Reclamo> {
  const reclamo = await this.prisma.reclamo.findUnique({ where: { id } });
  
  if (!reclamo) throw new NotFoundException();

  // Validar transición de estado
  this.validateStateTransition(reclamo.estado, data.estado);

  // Actualizar
  const updated = await this.prisma.reclamo.update({
    where: { id },
    data: {
      ...data,
      updatedAt: new Date()
    }
  });

  // Registrar cambios en historial
  if (data.estado && data.estado !== reclamo.estado) {
    await this.recordStateChange(id, reclamo.estado, data.estado);
  }

  return updated;
}

private validateStateTransition(currentState: string, newState: string): void {
  const validTransitions = {
    'PENDIENTE': ['EN_PROCESO', 'RECHAZADO', 'DUPLICADO'],
    'EN_PROCESO': ['RESUELTO', 'PENDIENTE'],
    'RESUELTO': [],
    'RECHAZADO': [],
    'DUPLICADO': []
  };

  if (!validTransitions[currentState]?.includes(newState)) {
    throw new BadRequestException(
      `Cannot transition from ${currentState} to ${newState}`
    );
  }
}
```

---

## 📋 Entidades y DTOs

### ReclamoDto (Response)
```typescript
export class ReclamoDto {
  id: string;
  numeroReclamo: string;
  canal: string;
  descripcion: string;
  categoria: string;
  prioridad: string;
  estado: string;
  latitud?: number;
  longitud?: number;
  direccion?: string;
  asignado_a?: string;
  area_responsable?: string;
  createdAt: Date;
  updatedAt: Date;
  historial?: MensajeHistorialDto[];
}
```

### CreateReclamoDto (Input)
```typescript
export class CreateReclamoDto {
  @IsEnum(['whatsapp', 'web', 'email', 'instagram'])
  canal: string;

  @IsObject()
  @ValidateNested()
  canalIdentidad: { type: string; value: string };

  @IsString()
  @MaxLength(2000)
  descripcion: string;

  @IsEnum(ReclamoCategoria)
  categoria: string;

  @IsOptional()
  @IsEnum(ReclamoPrioridad)
  prioridad?: string;

  @IsOptional()
  @IsObject()
  ubicacion?: {
    latitud: number;
    longitud: number;
    direccion: string;
    zona?: string;
  };

  @IsOptional()
  @IsObject()
  evidencia?: {
    imagenes?: string[];
    documentos?: string[];
  };
}
```

---

## 🔄 Flujos de Negocio

### Flujo 1: Creación de Reclamo
```
MS-AI publica reclamos.create
  ↓
MS-Reclamos recibe (NATS subscription)
  ↓
ValidaReclamoCreateDto
  ↓
Genera numeroReclamo único
  ↓
Inserta en tabla Reclamo (estado: PENDIENTE)
  ↓
Registra en MensajeHistorial
  ↓
✅ Reclamo creado
```

### Flujo 2: Cambio de Estado
```
Admin-dashboard actualiza estado
  ↓
PATCH /api/reclamos/{id}
  ↓
Gateway valida JWT
  ↓
Envía reclamos.update (NATS)
  ↓
MS-Reclamos valida transición de estado
  ↓
Verifica permisos del usuario
  ↓
Actualiza tabla Reclamo
  ↓
Registra cambio en historial
  ↓
✅ Estado actualizado
  ↓
(Futuro) Publica evento para notificaciones
```

### Flujo 3: Asignación a Área
```
Admin asigna reclamo a área/operario
  ↓
Actualiza campo "asignado_a" y "area_responsable"
  ↓
Registra en historial: "Asignado a {nombre}"
  ↓
(Futuro) Envía notificación al asignado
```

---

## ✅ Validaciones

### A Nivel DTO (class-validator)
- Campos requeridos
- Longitud máxima de strings
- Valores enum válidos
- Estructura JSONB válida

### A Nivel de Negocio
- Coordenadas geográficas válidas (lat/lng en rango)
- Transiciones de estado permitidas
- Categorías válidas
- Prioridades válidas

### A Nivel de BD
- Constraints UNIQUE en numeroReclamo
- Constraints CHECK en prioridad y estado
- Foreign Keys en referencias
- NOT NULL en campos requeridos

---

## ⚠️ Error Handling

```typescript
// Not found
throw new NotFoundException(`Reclamo ${id} not found`);

// Invalid state transition
throw new BadRequestException(`Cannot transition from ${current} to ${new}`);

// Duplicate entry
throw new ConflictException(`Reclamo number already exists`);

// Unauthorized
throw new ForbiddenException(`User does not have permission to update reclamo`);

// Database error
throw new InternalServerErrorException(`Database error: ${error.message}`);
```

---

## ⚙️ Configuración

### .env
```bash
DATABASE_URL=postgresql://user:password@localhost:5432/smartservice_reclamos
NATS_SERVERS=nats://localhost:4222
NODE_ENV=development
```

---

## 📊 Queries Típicas

### Reclamos pendientes por prioridad
```sql
SELECT * FROM "Reclamo"
WHERE estado = 'PENDIENTE'
ORDER BY prioridad DESC, createdAt ASC
LIMIT 20;
```

### Reclamos sin asignar
```sql
SELECT COUNT(*) as sin_asignar
FROM "Reclamo"
WHERE estado = 'PENDIENTE' AND asignado_a IS NULL;
```

### Timeline de reclamo
```sql
SELECT tipo, contenido, createdAt
FROM "MensajeHistorial"
WHERE "reclamoId" = 'uuid-123'
ORDER BY createdAt ASC;
```

---

## 🔗 Dependencias

```json
{
  "@nestjs/common": "^11.0.0",
  "@prisma/client": "^5.x.x",
  "class-validator": "^0.14.0",
  "class-transformer": "^0.5.x"
}
```

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
