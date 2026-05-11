# Frontend: PUBLIC-SITE & ADMIN-DASHBOARD

**Tecnologías**:
- **public-site**: HTML5, CSS3, JavaScript puro
- **admin-dashboard**: Next.js 16.2.6, React 19.2.4, TypeScript, Tailwind CSS

---

## 📋 Tabla de Contenidos

1. [Public Site](#public-site)
2. [Admin Dashboard](#admin-dashboard)
3. [Autenticación y Cookies](#autenticación-y-cookies)
4. [API Integration](#api-integration)

---

## 🌐 PUBLIC-SITE

### Propósito
Sitio estático para ciudadanos. Información sobre el servicio y punto de entrada para crear reclamos vía web.

### Estructura
```
public-site/
├── index.html          → Landing page
├── app.js             → JavaScript principal
├── styles.css         → Estilos
└── README.md
```

### Funcionalidades

#### 1. Landing Page (index.html)
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SmartService - Gestión de Reclamos</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <header>
    <nav><!-- Navegación --></nav>
  </header>

  <main>
    <!-- Hero Section -->
    <section id="hero">
      <h1>SmartService</h1>
      <p>Reporta problemas en tu ciudad de forma fácil</p>
      <button onclick="openForm()">Crear Reclamo</button>
    </section>

    <!-- Formulario de Reclamo (Modal) -->
    <div id="reclamoModal" class="modal hidden">
      <form id="reclamoForm">
        <input type="text" name="descripcion" placeholder="Describe el problema" required>
        <select name="categoria" required>
          <option value="">Selecciona categoría</option>
          <option value="baches_y_pavimento">Baches y Pavimento</option>
          <option value="alumbrado">Alumbrado</option>
          <!-- ... más categorías ... -->
        </select>
        <input type="text" name="direccion" placeholder="Dirección" required>
        <button type="submit">Enviar Reclamo</button>
      </form>
    </div>

    <!-- Información -->
    <section id="info">
      <h2>¿Cómo funciona?</h2>
      <p>...</p>
    </section>
  </main>

  <footer>
    <p>&copy; 2026 SmartService</p>
  </footer>

  <script src="app.js"></script>
</body>
</html>
```

#### 2. JavaScript (app.js)
```javascript
// Enviar reclamo vía web
async function submitReclamo(event) {
  event.preventDefault();

  const formData = new FormData(document.getElementById('reclamoForm'));
  const data = Object.fromEntries(formData);

  try {
    const response = await fetch('/api/inbound/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        channel: 'web',
        channelIdentity: {
          type: 'email',
          value: data.email || 'anonymous@web'
        },
        messageText: `${data.descripcion}\nDirección: ${data.direccion}`,
        externalMessageId: generateUUID()
      })
    });

    if (response.ok) {
      alert('¡Reclamo enviado exitosamente!');
      closeForm();
    } else {
      alert('Error al enviar reclamo');
    }
  } catch (err) {
    console.error('Error:', err);
    alert('Error de conexión');
  }
}

function generateUUID() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    var r = Math.random() * 16 | 0,
      v = c == 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
}
```

### Categorías de Reclamos
```javascript
const categorias = [
  { value: 'agua_y_cloacas', label: 'Agua y Cloacas' },
  { value: 'alumbrado', label: 'Alumbrado' },
  { value: 'baches_y_pavimento', label: 'Baches y Pavimento' },
  { value: 'arbolado', label: 'Arbolado' },
  { value: 'residuos', label: 'Residuos' },
  { value: 'electricidad', label: 'Electricidad' },
  { value: 'gas', label: 'Gas' },
  { value: 'transporte', label: 'Transporte' },
  { value: 'infraestructura', label: 'Infraestructura' },
  { value: 'otros', label: 'Otros' }
];
```

---

## 💼 ADMIN-DASHBOARD

### Propósito
Panel administrativo para operarios, gestores y administradores municipales. Permite:
- Visualizar reclamos
- Actualizar estados
- Asignar a personal
- Ver historial y tracking
- Generar reportes

### Estructura
```
admin-dashboard/
├── src/
│   ├── app/
│   │   ├── layout.tsx          → Layout principal
│   │   ├── page.tsx            → Dashboard home
│   │   ├── api/                → API routes (backend)
│   │   ├── reclamos/
│   │   │   ├── page.tsx        → Listado de reclamos
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx    → Detalle de reclamo
│   │   │   └── new/
│   │   │       └── page.tsx    → Crear reclamo
│   │   ├── login/
│   │   │   └── page.tsx        → Login
│   │   └── no-access/
│   │       └── page.tsx        → 403 Forbidden
│   ├── components/             → Componentes reutilizables
│   ├── services/               → Servicios HTTP
│   ├── lib/                    → Utilities
│   └── config/                 → Configuración
├── next.config.ts
├── tsconfig.json
└── package.json
```

### Rutas Principales

#### 1. /login
**Descripción**: Autenticación  
**Acceso**: Público  
**Componente**: LoginPage  

```typescript
// login/page.tsx
export default function LoginPage() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      const res = await fetch('/api/auth/login', {
        method: 'POST',
        body: JSON.stringify({ username, password })
      });
      if (res.ok) {
        // Redirect to dashboard
        router.push('/');
      }
    } catch (err) {
      setError('Invalid credentials');
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input value={username} onChange={e => setUsername(e.target.value)} />
      <input type="password" value={password} onChange={e => setPassword(e.target.value)} />
      <button type="submit">Iniciar Sesión</button>
    </form>
  );
}
```

#### 2. / (Dashboard Home)
**Descripción**: Vista general de reclamos  
**Acceso**: Requiere autenticación  
**Información**:
- KPIs: Total, pendientes, en proceso, resueltos
- Gráficos por categoría, prioridad
- Últimos reclamos
- Alertas/pendientes de resolver

#### 3. /reclamos
**Descripción**: Listado completo de reclamos  
**Acceso**: operator, area_manager, official, admin  
**Features**:
- Tabla con búsqueda y filtros
- Paginación
- Sort por columnas
- Selección múltiple
- Acciones en lote (cambiar estado, asignar, etc.)

```typescript
// reclamos/page.tsx
export default function ReclamosPage() {
  const [reclamos, setReclamos] = useState([]);
  const [filters, setFilters] = useState({ estado: '', categoria: '' });
  const [page, setPage] = useState(1);

  useEffect(() => {
    fetchReclamos();
  }, [filters, page]);

  async function fetchReclamos() {
    const query = new URLSearchParams({
      estado: filters.estado,
      categoria: filters.categoria,
      page: page.toString()
    });
    const res = await fetch(`/api/reclamos?${query}`);
    const data = await res.json();
    setReclamos(data.data);
  }

  return (
    <div>
      <h1>Reclamos</h1>
      
      {/* Filtros */}
      <div className="filters">
        <select onChange={e => setFilters({...filters, estado: e.target.value})}>
          <option value="">Todos los estados</option>
          <option value="PENDIENTE">Pendiente</option>
          <option value="EN_PROCESO">En Proceso</option>
          <option value="RESUELTO">Resuelto</option>
        </select>
      </div>

      {/* Tabla */}
      <table>
        <thead>
          <tr>
            <th>Número</th>
            <th>Descripción</th>
            <th>Categoría</th>
            <th>Prioridad</th>
            <th>Estado</th>
            <th>Acciones</th>
          </tr>
        </thead>
        <tbody>
          {reclamos.map(reclamo => (
            <tr key={reclamo.id}>
              <td><Link href={`/reclamos/${reclamo.id}`}>{reclamo.numeroReclamo}</Link></td>
              <td>{reclamo.descripcion.substring(0, 50)}...</td>
              <td>{reclamo.categoria}</td>
              <td><Badge priority={reclamo.prioridad} /></td>
              <td><Badge estado={reclamo.estado} /></td>
              <td>
                <button onClick={() => openDetailModal(reclamo.id)}>Ver</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Paginación */}
      <Pagination currentPage={page} onPageChange={setPage} />
    </div>
  );
}
```

#### 4. /reclamos/[id]
**Descripción**: Detalle de reclamo individual  
**Acceso**: operator+  
**Información**:
- Descripción completa
- Ubicación en mapa
- Historial de cambios
- Fotos/evidencia
- Botones para cambiar estado, asignar, etc.

```typescript
// reclamos/[id]/page.tsx
export default function ReclamoDetailPage({ params }: { params: { id: string } }) {
  const [reclamo, setReclamo] = useState(null);

  useEffect(() => {
    fetch(`/api/reclamos/${params.id}`)
      .then(r => r.json())
      .then(data => setReclamo(data));
  }, [params.id]);

  if (!reclamo) return <div>Cargando...</div>;

  return (
    <div className="detalle-reclamo">
      <h1>Reclamo {reclamo.numeroReclamo}</h1>

      <div className="grid">
        <section>
          <h3>Información</h3>
          <p><strong>Descripción:</strong> {reclamo.descripcion}</p>
          <p><strong>Categoría:</strong> {reclamo.categoria}</p>
          <p><strong>Prioridad:</strong> {reclamo.prioridad}</p>
          <p><strong>Estado:</strong> {reclamo.estado}</p>
          <p><strong>Ubicación:</strong> {reclamo.direccion}</p>
        </section>

        <section>
          <h3>Mapa</h3>
          <MapComponent lat={reclamo.latitud} lng={reclamo.longitud} />
        </section>
      </div>

      <section>
        <h3>Historial</h3>
        <Timeline historial={reclamo.historial} />
      </section>

      <section>
        <h3>Acciones</h3>
        <button onClick={() => updateEstado('EN_PROCESO')}>Marcar en Proceso</button>
        <button onClick={() => updateEstado('RESUELTO')}>Marcar Resuelto</button>
        <button onClick={openAssignModal}>Asignar a...</button>
      </section>
    </div>
  );
}
```

#### 5. /reclamos/new
**Descripción**: Crear nuevo reclamo manualmente  
**Acceso**: operator+  

---

## 🔐 Autenticación y Cookies

### Login Flow
```
1. Usuario ingresa credenciales
2. Dashboard envía POST /api/auth/login
3. Gateway valida con MS-Auth
4. Retorna JWT + RefreshToken
5. Dashboard guarda:
   - accessToken en memory
   - refreshToken en cookie HTTP-only
6. Incluye accessToken en Authorization header para requests subsecuentes
```

### Gestión de Tokens

```typescript
// lib/auth.ts
export async function login(username: string, password: string) {
  const res = await fetch('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ username, password })
  });

  const { accessToken, refreshToken, user } = await res.json();

  // Guardar tokens
  localStorage.setItem('accessToken', accessToken);
  setCookie('refreshToken', refreshToken, { httpOnly: true, path: '/' });

  return user;
}

export function getAccessToken(): string | null {
  return localStorage.getItem('accessToken');
}

export async function refreshToken() {
  const refresh = getCookie('refreshToken');
  const res = await fetch('/api/auth/refresh', {
    method: 'POST',
    body: JSON.stringify({ refreshToken: refresh })
  });

  const { accessToken } = await res.json();
  localStorage.setItem('accessToken', accessToken);

  return accessToken;
}

export function logout() {
  localStorage.removeItem('accessToken');
  deleteCookie('refreshToken');
}
```

### Request Interceptor

```typescript
// lib/apiClient.ts
export async function apiRequest(
  url: string,
  options: RequestInit = {}
): Promise<Response> {
  const token = getAccessToken();

  const headers = {
    ...options.headers,
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  };

  let res = await fetch(url, { ...options, headers });

  // Si 401, intentar refresh
  if (res.status === 401) {
    const newToken = await refreshToken();
    headers['Authorization'] = `Bearer ${newToken}`;
    res = await fetch(url, { ...options, headers });
  }

  return res;
}
```

---

## 🔗 API Integration

### Endpoints Consumidos

#### GET /api/reclamos
```typescript
interface ListReclamosParams {
  estado?: string;
  categoria?: string;
  prioridad?: string;
  page?: number;
  limit?: number;
}

async function listReclamos(params: ListReclamosParams) {
  const query = new URLSearchParams(Object.entries(params).map(([k, v]) => [k, String(v)]));
  const res = await apiRequest(`/api/reclamos?${query}`);
  return res.json();
}
```

#### GET /api/reclamos/[id]
```typescript
async function getReclamo(id: string) {
  const res = await apiRequest(`/api/reclamos/${id}`);
  return res.json();
}
```

#### PATCH /api/reclamos/[id]
```typescript
interface UpdateReclamoPayload {
  estado?: string;
  asignado_a?: string;
  observaciones?: string;
}

async function updateReclamo(id: string, payload: UpdateReclamoPayload) {
  const res = await apiRequest(`/api/reclamos/${id}`, {
    method: 'PATCH',
    body: JSON.stringify(payload)
  });
  return res.json();
}
```

#### POST /api/inbound/messages
```typescript
async function createReclamoFromWeb(data: {
  descripcion: string;
  categoria: string;
  direccion: string;
  email: string;
}) {
  const res = await fetch('/api/inbound/messages', {
    method: 'POST',
    body: JSON.stringify({
      channel: 'web',
      channelIdentity: { type: 'email', value: data.email },
      messageText: `${data.descripcion}\nDirección: ${data.direccion}`,
      externalMessageId: generateUUID()
    })
  });
  return res.json();
}
```

---

## 🎨 Componentes Principales

### ReclamoCard
```typescript
// components/ReclamoCard.tsx
interface ReclamoCardProps {
  reclamo: Reclamo;
  onClick?: (id: string) => void;
}

export default function ReclamoCard({ reclamo, onClick }: ReclamoCardProps) {
  return (
    <div className="card" onClick={() => onClick?.(reclamo.id)}>
      <h4>{reclamo.numeroReclamo}</h4>
      <p className="descripcion">{reclamo.descripcion}</p>
      <div className="badges">
        <Badge label={reclamo.estado} type="estado" />
        <Badge label={reclamo.prioridad} type="prioridad" />
      </div>
    </div>
  );
}
```

### ReclamoFilters
```typescript
// components/ReclamoFilters.tsx
export default function ReclamoFilters({ onFilter }) {
  return (
    <div className="filters">
      <select onChange={e => onFilter({ estado: e.target.value })}>
        <option value="">-- Estado --</option>
        <option value="PENDIENTE">Pendiente</option>
        <option value="EN_PROCESO">En Proceso</option>
        <option value="RESUELTO">Resuelto</option>
      </select>
      {/* Más filtros */}
    </div>
  );
}
```

### MapComponent (Google Maps)
```typescript
// components/MapComponent.tsx
export default function MapComponent({ lat, lng }) {
  return (
    <iframe
      width="100%"
      height="400"
      src={`https://www.google.com/maps/embed/v1/place?key=YOUR_KEY&q=${lat},${lng}`}
    />
  );
}
```

---

## 🔗 Dependencias Frontend

```json
{
  "next": "^16.2.6",
  "react": "^19.2.4",
  "react-dom": "^19.2.4",
  "typescript": "^5.x.x",
  "@tanstack/react-query": "^5.x.x",
  "axios": "^1.7.7",
  "nookies": "^2.5.2",
  "tailwindcss": "^3.x.x"
}
```

---

## ⚠️ Limitaciones Actuales

- ❌ Sin mapas interactivos (solo embed)
- ❌ Sin upload de archivos
- ⚠️ Sin offline mode
- ⚠️ Sin reportes/exportación
- ⚠️ Sin búsqueda avanzada
- ⚠️ Sin sistema de notificaciones en tiempo real (WebSocket)

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
