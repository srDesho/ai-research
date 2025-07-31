# Fix: HTML Frontend Not Connecting to Spring Boot Backend

## Problema
Cuando se abre el archivo `index.html` directamente desde el sistema de archivos o desde el preview de IntelliJ IDEA (`localhost:63342`), las llamadas a la API fallan porque apuntan al puerto incorrecto.

## Causa
El frontend intenta conectarse al puerto 63342 (servidor de IntelliJ) en lugar de al puerto 8085 donde Spring Boot está ejecutándose.

## Solución
Usar URLs relativas en lugar de URLs absolutas con lógica condicional compleja.

### Código Incorrecto ❌
```javascript
// Función complicada que intenta determinar la URL base
function getApiBase() {
    return window.location.port === '8085' ? '' : '/api/v1';
}

// Uso complicado
const response = await fetch(`${getApiBase()}/ssds`);
```

### Código Correcto ✅
```javascript
// Simple - usar URLs relativas
const response = await fetch('/ssds');
```

## ¿Por qué funciona?
1. **Spring Boot sirve contenido estático** desde `src/main/resources/static/`
2. Cuando accedes a `http://localhost:8085/`, Spring Boot sirve automáticamente `index.html`
3. Las URLs relativas (`/ssds`) apuntan al mismo servidor (`localhost:8085`)
4. No se necesita lógica condicional porque HTML y API están en el mismo origen

## Estructura del Proyecto
```
src/main/resources/static/
└── index.html          # Frontend HTML/JS
```

## Configuración de Spring Boot
Spring Boot configura automáticamente:
- `GET /` → sirve `index.html`
- `GET /ssds` → llama al controlador `SSDController`
- `GET /ssds/all` → llama al endpoint de registro automático

## Pasos para Implementar
1. Colocar `index.html` en `src/main/resources/static/`
2. Usar URLs relativas en todas las llamadas fetch
3. Acceder siempre a `http://localhost:8085/` (no usar preview de IntelliJ)
4. Las rutas relativas se resolverán automáticamente al origen correcto

## Ejemplo Completo
```javascript
// Todas las llamadas API usan rutas relativas
async function loadSSDs() {
    const response = await fetch('/ssds');
    // ...
}

async function refreshData() {
    const response = await fetch('/ssds/all');
    // ...
}
```

## Notas Importantes
- **No abrir el HTML directamente** desde el sistema de archivos
- **No usar el preview de IntelliJ** (`localhost:63342`)
- **Siempre acceder** a `http://localhost:8085/`
- Las URLs relativas funcionan porque Spring Boot maneja el routing

Esta solución elimina complejidad innecesaria y funciona correctamente en el entorno de desarrollo Spring Boot.





# Handling Interview Questions About This Fix

## Para entrevistas técnicas - Cómo explicar la solución:

### Si preguntan: "¿Has tenido problemas con conexiones frontend-backend?"

**Respuesta estructurada:**

"Sí, tuve un caso donde el frontend no se conectaba al backend Spring Boot. El problema era que el archivo HTML se abría desde diferentes orígenes: directamente del sistema de archivos, desde el preview del IDE, y desde Spring Boot. Esto causaba que las URLs de API apuntaran a puertos incorrectos."

### Si preguntan: "¿Cómo lo solucionaste?"

**Respuesta técnica:**

"Implementé una solución basada en URLs relativas en lugar de lógica condicional compleja. En lugar de tener una función `getApiBase()` que intentaba determinar el puerto, simplifiqué todo usando rutas relativas (`/endpoint`) que Spring Boot resuelve automáticamente cuando sirve contenido estático."

### Si preguntan detalles técnicos:

**Explicación concisa:**

"Spring Boot, cuando sirve contenido estático desde `src/main/resources/static/`, configura automáticamente el routing. Las URLs relativas apuntan al mismo origen, eliminando problemas de CORS y asegurando que todas las peticiones vayan al servidor correcto (puerto 8085)."

### Si quieren un ejemplo de código:

**Demo rápida:**

```javascript
// ANTES - Complicado y propenso a errores
function getApiBase() {
    if (window.location.port === '63342') return 'http://localhost:8085/api';
    if (window.location.port === '8085') return '/api';
    return '/api/v1';
}

// DESPUÉS - Simple y robusto
// Todas las llamadas usan rutas relativas
const response = await fetch('/ssds');
```

### Si preguntan sobre lecciones aprendidas:

**Reflexión profesional:**

"Aprendí que a veces las soluciones más simples son las mejores. Estaba sobre-ingenierizando con lógica condicional cuando Spring Boot ya proporcionaba una solución elegante mediante el serving de contenido estático. También entendí la importancia de comprender cómo el entorno de desarrollo afecta el comportamiento del código."

### Puntos clave para mencionar:
1. **Problema**: Múltiples orígenes causando conflictos en las URLs
2. **Solución**: URLs relativas + Spring Boot static serving
3. **Beneficio**: Eliminó complejidad, mejoró mantenibilidad
4. **Lección**: No sobre-complicar soluciones cuando hay patrones establecidos

### Frases útiles para la entrevista:
- "Opté por la simplicidad sobre la complejidad innecesaria"
- "Aproveché las capacidades nativas de Spring Boot"
- "La solución fue más elegante y menos propensa a errores"
- "Entendí mejor el ciclo de request-response en aplicaciones web"

Esta explicación muestra tanto competencia técnica como capacidad de reflexión y aprendizaje, cualidades valiosas en cualquier desarrollador.