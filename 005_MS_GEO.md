# Microservicio: MS-GEO

**Tipo**: Geolocation & Geocoding Service  
**Tecnología**: Google Maps API  
**Comunicación**: NATS (Request/Reply)  
**Framework**: NestJS  
**Base de Datos**: Ninguna (stateless)  

---

## 🎯 Propósito

Validar y geocodificar direcciones:
- ✅ Convertir dirección a coordenadas (lat/lng)
- ✅ Validar zona geográfica permitida
- ✅ Extraer componentes de dirección
- ✅ Detectar ubicaciones fuera de jurisdicción
- ✅ Manejar errores y ambigüedades

---

## 📡 NATS Interface

### Suscripción: geo.geocode-address (Request/Reply)
**Origen**: MS-AI  
**Descripción**: Geocodificar y validar dirección  
**Timeout**: 30 segundos  

**Request**:
```json
{
  "address": "Av. Principal 500, Funes",
  "allowedCountries": ["Argentina"],
  "allowedAreas": ["Funes", "Santa Fe"],
  "allowedZones": ["Centro", "Norte", "Sur", "Oeste"]
}
```

**Response (Válido)**:
```json
{
  "valid": true,
  "latitude": -32.9432,
  "longitude": -60.6537,
  "formattedAddress": "Av. Principal 500, Funes, Santa Fe, Argentina",
  "addressComponents": {
    "route": "Av. Principal",
    "streetNumber": "500",
    "locality": "Funes",
    "administrativeArea": "Santa Fe",
    "country": "Argentina",
    "postalCode": "S2669"
  },
  "placeId": "ChIJ..."
}
```

**Response (Fuera de Jurisdicción)**:
```json
{
  "valid": false,
  "reason": "OUTSIDE_ALLOWED_AREA",
  "latitude": -33.5,
  "longitude": -61.0,
  "formattedAddress": "Rosario, Santa Fe, Argentina",
  "message": "La ubicación está fuera de la zona permitida"
}
```

**Response (Error)**:
```json
{
  "valid": false,
  "reason": "INVALID_ADDRESS",
  "message": "No se encontró la dirección especificada"
}
```

---

## 🗺️ Validaciones Geográficas

### 1. País Permitido
```typescript
const allowedCountries = ['Argentina'];

function validateCountry(country: string): boolean {
  return allowedCountries.includes(country);
}
```

### 2. Provincia/Área Permitida
```typescript
const allowedAreas = ['Funes', 'Santa Fe'];

function validateArea(administrativeArea: string): boolean {
  return allowedAreas.includes(administrativeArea);
}
```

### 3. Zona Permitida (Opcional)
```typescript
const allowedZones = ['Centro', 'Norte', 'Sur', 'Oeste'];

function validateZone(zone: string): boolean {
  return !zone || allowedZones.includes(zone);
}
```

### 4. Coordenadas en Rango
```typescript
// Validar que coordenadas sean razonables
if (Math.abs(latitude) > 90 || Math.abs(longitude) > 180) {
  throw new Error('Coordenadas inválidas');
}

// Validar dentro de bounding box aproximado (Rosario/Funes)
const bbox = {
  minLat: -32.95,
  maxLat: -32.90,
  minLng: -60.75,
  maxLng: -60.50
};

if (!(latitude >= bbox.minLat && latitude <= bbox.maxLat &&
      longitude >= bbox.minLng && longitude <= bbox.maxLng)) {
  return { valid: false, reason: 'OUTSIDE_GEOGRAPHIC_AREA' };
}
```

---

## 🔧 Servicio Principal

### GeoService
```typescript
async geocodeAddress(
  address: string,
  constraints: GeoConstraints
): Promise<GeocodeResult> {
  try {
    // 1. Llamar a Google Maps Geocoding API
    const googleResponse = await this.callGoogleMapsApi(address);

    if (!googleResponse.results || googleResponse.results.length === 0) {
      return {
        valid: false,
        reason: 'INVALID_ADDRESS',
        message: 'Dirección no encontrada'
      };
    }

    // Obtener primer resultado (mejor coincidencia)
    const result = googleResponse.results[0];
    const { lat, lng } = result.geometry.location;

    // 2. Extraer componentes
    const components = this.extractAddressComponents(result.address_components);

    // 3. Validar restricciones
    if (!this.validateConstraints(components, constraints)) {
      return {
        valid: false,
        reason: 'OUTSIDE_ALLOWED_AREA',
        latitude: lat,
        longitude: lng,
        formattedAddress: result.formatted_address,
        message: `Ubicación fuera de zona permitida: ${constraints.allowedAreas.join(', ')}`
      };
    }

    // 4. Retornar resultado válido
    return {
      valid: true,
      latitude: lat,
      longitude: lng,
      formattedAddress: result.formatted_address,
      addressComponents: components,
      placeId: result.place_id
    };
  } catch (err) {
    logger.error('Geocoding error:', err);
    throw new InternalServerErrorException('Geocoding service error');
  }
}

private callGoogleMapsApi(address: string): Promise<GoogleMapsResponse> {
  const url = 'https://maps.googleapis.com/maps/api/geocode/json';
  const params = new URLSearchParams({
    address: address,
    key: this.configService.get('GOOGLE_MAPS_API_KEY'),
    language: 'es',
    region: 'ar'
  });

  return axios.get(`${url}?${params.toString()}`).then(res => res.data);
}

private extractAddressComponents(
  googleComponents: any[]
): AddressComponents {
  const components: AddressComponents = {};

  const typeMapping = {
    'route': 'route',
    'street_number': 'streetNumber',
    'postal_code': 'postalCode',
    'locality': 'locality',
    'administrative_area_level_1': 'administrativeArea',
    'country': 'country'
  };

  googleComponents.forEach(component => {
    component.types.forEach((type: string) => {
      if (typeMapping[type]) {
        components[typeMapping[type]] = component.long_name;
      }
    });
  });

  return components;
}

private validateConstraints(
  components: AddressComponents,
  constraints: GeoConstraints
): boolean {
  // Validar país
  if (constraints.allowedCountries &&
      !constraints.allowedCountries.includes(components.country)) {
    return false;
  }

  // Validar área/provincia
  if (constraints.allowedAreas &&
      !constraints.allowedAreas.includes(components.administrativeArea)) {
    return false;
  }

  // Validar zona (si se proporciona)
  if (constraints.allowedZones) {
    // Podría requerir reverse geocoding para determinar zona
    // Por ahora, skip
  }

  return true;
}
```

---

## 🔍 Extracción de Componentes

### Google Maps Address Components
```
Entrada: "Av. Principal 500, Funes, Santa Fe, Argentina"

Google retorna array:
[
  { long_name: "500", types: ["street_number"] },
  { long_name: "Av. Principal", types: ["route"] },
  { long_name: "Funes", types: ["locality", "political"] },
  { long_name: "Santa Fe", types: ["administrative_area_level_1", "political"] },
  { long_name: "Argentina", types: ["country", "political"] },
  { long_name: "S2669", types: ["postal_code"] }
]

Salida (nuestra estructura):
{
  "route": "Av. Principal",
  "streetNumber": "500",
  "locality": "Funes",
  "administrativeArea": "Santa Fe",
  "country": "Argentina",
  "postalCode": "S2669"
}
```

---

## 🛡️ Manejo de Errores

### Errores Google Maps

| Código | Significado | Acción |
|--------|-----------|--------|
| ZERO_RESULTS | No hay resultados | Retornar INVALID_ADDRESS |
| OVER_QUERY_LIMIT | Límite de API excedido | Reintentar o fallback |
| REQUEST_DENIED | API key inválida | Log error crítico |
| INVALID_REQUEST | Request malformado | Log error |
| UNKNOWN_ERROR | Error genérico | Retry con backoff |

### Retry Logic
```typescript
async callGoogleMapsApiWithRetry(
  address: string,
  maxRetries: number = 3
): Promise<GoogleMapsResponse> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await this.callGoogleMapsApi(address);
    } catch (err) {
      if (i < maxRetries - 1) {
        // Exponential backoff: 1s, 2s, 4s
        await sleep(Math.pow(2, i) * 1000);
      } else {
        throw err;
      }
    }
  }
}
```

---

## 📊 Limitaciones

### Google Maps API Limits
- **Rate Limit**: Por defecto 50 QPS (queries/segundo)
- **Billing**: $5 por 1000 requests (después de crédito gratuito)
- **Cache**: Implementar Redis para cachear resultados comunes

### Alcance Actual
- ✅ Geocodificación básica
- ✅ Validación de zona
- ❌ Reverse geocoding (coords → dirección)
- ❌ Optimización de rutas
- ❌ Cálculo de distancias
- ❌ Información de tráfico

---

## ⚙️ Configuración

### .env
```bash
GOOGLE_MAPS_API_KEY=AIzaSyD...
GEO_ALLOWED_COUNTRIES=Argentina
GEO_ALLOWED_AREAS=Funes,Santa Fe
GEO_ALLOWED_ZONES=Centro,Norte,Sur,Oeste
```

### Redis Caching (Planeado)
```typescript
// Cache dirección geocodificada
const cacheKey = `geocode:${address}`;
const cached = await redis.get(cacheKey);

if (cached) {
  return JSON.parse(cached);
}

const result = await this.geocodeAddress(address, constraints);
await redis.setex(cacheKey, 2592000, JSON.stringify(result)); // 30 días

return result;
```

---

## 🔗 Dependencias

```json
{
  "@nestjs/common": "^11.0.0",
  "axios": "^1.7.7"
}
```

---

*Documentación generada automáticamente - Última actualización: 11 de Mayo de 2026*
