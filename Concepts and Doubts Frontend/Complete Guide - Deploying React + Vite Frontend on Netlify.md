# Guía Completa: Deploy Frontend React + Vite en Netlify

## Tabla de Contenidos
1. [Preparación del Proyecto Frontend](#1-preparación-del-proyecto-frontend)
2. [Configuración de Variables de Entorno para Vite](#2-configuración-de-variables-de-entorno-para-vite)
3. [Actualizar Servicios para Vite](#3-actualizar-servicios-para-vite)
4. [Configurar CORS en el Backend](#4-configurar-cors-en-el-backend)
5. [Verificar Repositorio en GitHub](#5-verificar-repositorio-en-github)
6. [Deploy en Netlify](#6-deploy-en-netlify)
7. [Solución: Page Not Found](#7-solución-page-not-found)
8. [Verificación Final](#8-verificación-final)

---

## 1. Preparación del Proyecto Frontend

### 1.1 Estructura del proyecto

Supongamos que tu repositorio tiene esta estructura:

```
my-fullstack-app/
├── backend-api/              # Backend (Spring Boot, Node.js, etc.)
│   ├── src/
│   ├── pom.xml o package.json
│   └── ...
└── frontend-web/             # Frontend React + Vite
    ├── src/
    ├── public/
    ├── package.json
    ├── vite.config.js
    └── ...
```

### 1.2 Actualizar `.gitignore`

En la carpeta de tu frontend, crea o actualiza `.gitignore`:

```gitignore
# Dependencies
node_modules/

# Production build
build/
dist/

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# IDE
.vscode/
.idea/
*.swp
*.swo
.DS_Store

# Vite
.cache/
```

---

## 2. Configuración de Variables de Entorno para Vite

### 2.1 Diferencia importante: Create React App vs Vite

| Aspecto | Create React App | Vite |
|---------|------------------|------|
| **Prefijo de variables** | `REACT_APP_` | `VITE_` |
| **Acceso en código** | `process.env.REACT_APP_VAR` | `import.meta.env.VITE_VAR` |
| **Directorio de build** | `build/` | `dist/` |

### 2.2 Crear archivo `.env.production`

En la raíz de tu carpeta frontend, crea `.env.production`:

```env
VITE_BACKEND_URL=https://your-backend.onrender.com/api/v1
```

**Ejemplo real:**
```env
VITE_BACKEND_URL=https://my-api-production.onrender.com/api/v1
```

### 2.3 Crear archivo `.env.development` (para desarrollo local)

```env
VITE_BACKEND_URL=http://localhost:8080/api/v1
```

### 2.4 Crear `.env.example` (plantilla pública)

Este archivo SÍ se sube a GitHub como referencia:

```env
VITE_BACKEND_URL=your_backend_url_here
```

---

## 3. Actualizar Servicios para Vite

### 3.1 Actualizar `AuthService.js` (o tu archivo de servicios principal)

**❌ ANTES (Create React App):**

```javascript
export const BACKEND_BASE_URL = process.env.REACT_APP_BACKEND_URL || 'http://localhost:8080/api/v1';
```

**✅ DESPUÉS (Vite):**

```javascript
export const BACKEND_BASE_URL = import.meta.env.VITE_BACKEND_URL || 'http://localhost:8080/api/v1';
```

**Archivo completo de ejemplo:**

```javascript
// Base URL for the backend API - uses Vite environment variable
export const BACKEND_BASE_URL = import.meta.env.VITE_BACKEND_URL || 'http://localhost:8080/api/v1';

// Key for storing the JWT in localStorage
export const TOKEN_KEY = 'jwtToken';

export const getAuthHeader = () => {
  const token = localStorage.getItem(TOKEN_KEY);
  if (!token) return null;
  return `Bearer ${token}`;
};

export const login = async (username, password) => {
  const response = await fetch(`${BACKEND_BASE_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password }),
  });

  if (response.ok) {
    const data = await response.json();
    if (data.jwt) {
      localStorage.setItem(TOKEN_KEY, data.jwt);
      return true;
    }
  }
  
  throw new Error('Login failed');
};

// ... resto de tus funciones
```

### 3.2 Los demás servicios NO necesitan cambios

Si otros servicios importan `BACKEND_BASE_URL` de `AuthService.js`, automáticamente usarán la variable de Vite.

### 3.3 Probar localmente

```bash
cd frontend-web
npm run dev
```

En la consola del navegador (F12):
```javascript
console.log('Backend URL:', import.meta.env.VITE_BACKEND_URL);
// Debería mostrar: http://localhost:8080/api/v1
```

---

## 4. Configurar CORS en el Backend

### 4.1 Actualizar CORS en Spring Boot

**Si usas `WebConfig.java`:**

```java
package com.example.myapp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/v1/**")
                .allowedOrigins(
                    "http://localhost:5173",           // Vite dev server
                    "http://localhost:3000",           // Alternative dev port
                    "https://your-app.netlify.app",    // Your Netlify URL
                    "https://*.netlify.app"            // Preview deployments
                )
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

**Si usas Spring Security (`SecurityConfig.java`):**

```java
package com.example.myapp.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            // ... resto de tu configuración
            
        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        
        configuration.setAllowedOrigins(Arrays.asList(
            "http://localhost:5173",           // Vite dev server
            "http://localhost:3000",           // Alternative port
            "https://your-app.netlify.app",    // Production
            "https://*.netlify.app"            // Preview deployments
        ));
        
        configuration.setAllowedMethods(Arrays.asList(
            "GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"
        ));
        
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### 4.2 Commit y push cambios del backend

```bash
cd backend-api
git add .
git commit -m "chore(cors): Add Netlify domains to CORS configuration"
cd ..
git push
```

Render redesplegará automáticamente.

---

## 5. Verificar Repositorio en GitHub

### 5.1 Commit cambios del frontend

```bash
cd frontend-web
git add .
git commit -m "chore(config): Update environment variables for Vite deployment"
cd ..
git push
```

---

## 6. Deploy en Netlify

### 6.1 Crear cuenta en Netlify

1. Ve a [https://www.netlify.com](https://www.netlify.com)
2. Click en **Sign up**
3. Selecciona **Continue with GitHub**
4. Autoriza a Netlify

### 6.2 Crear nuevo sitio

1. En Dashboard de Netlify, click **Add new site** → **Import an existing project**
2. Click en **Deploy with GitHub**
3. Busca y selecciona tu repositorio

### 6.3 Configurar Build Settings

#### Si tu frontend está en una subcarpeta (monorepo):

| Campo | Valor |
|-------|-------|
| **Branch to deploy** | `main` |
| **Base directory** | `frontend-web` |
| **Build command** | `npm run build` |
| **Publish directory** | `frontend-web/dist` |

#### Si tu frontend está en la raíz:

| Campo | Valor |
|-------|-------|
| **Branch to deploy** | `main` |
| **Base directory** | *(déjalo vacío)* |
| **Build command** | `npm run build` |
| **Publish directory** | `dist` |

### 6.4 Configurar Variables de Entorno

**ANTES de hacer deploy:**

1. Expande **Advanced build settings**
2. Click en **Add environment variable**
3. Agrega:

```
Key: VITE_BACKEND_URL
Value: https://your-backend.onrender.com/api/v1
```

**IMPORTANTE:**
- El nombre DEBE empezar con `VITE_`
- NO uses comillas
- NO incluyas barra final `/`

### 6.5 Iniciar deploy

1. Click en **Deploy [nombre-del-sitio]**
2. Espera 2-5 minutos

**Logs que verás:**

```bash
12:00:00 PM: Build ready to start
12:00:05 PM: Cloning repository
12:00:10 PM: Installing dependencies
12:00:10 PM: $ cd frontend-web && npm ci
12:01:00 PM: Running build command
12:01:00 PM: $ npm run build
12:01:05 PM: > vite build
12:01:30 PM: ✓ Build complete
12:02:00 PM: Site is live ✨
```

### 6.6 Verificar el deploy

Netlify te asignará una URL como:
```
https://random-name-123456.netlify.app
```

---

## 7. Solución: Page Not Found

### 7.1 El problema

Al intentar acceder o recargar rutas como `/login` o `/profile`, aparece:

```
Page not found

Looks like you've followed a broken link or entered a URL that doesn't exist on this site.
```

**¿Por qué pasa?**

Cuando recargas `/login`, Netlify busca un archivo físico llamado `login`, pero ese archivo no existe. React Router maneja las rutas en el cliente, no en el servidor.

### 7.2 La solución: Crear `netlify.toml`

En la raíz de tu carpeta frontend, crea el archivo `netlify.toml`:

```toml
# Build configuration
[build]
  publish = "dist"
  command = "npm run build"

# Redirect all routes to index.html for React Router (Vite)
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

# Security headers
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
    Permissions-Policy = "geolocation=(), microphone=(), camera=()"

# Cache static assets
[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

**¿Qué hace cada sección?**

- **`[build]`**: Define dónde está el build (`dist/`) y el comando
- **`[[redirects]]`**: Redirige todas las rutas a `index.html` para que React Router funcione
- **`[[headers]]`**: Agrega seguridad y cacheo de assets

### 7.3 Commit y push

```bash
cd frontend-web
git add netlify.toml
git commit -m "fix: Add Netlify redirects for client-side routing"
cd ..
git push
```

Netlify detectará el cambio y redesplegará automáticamente (1-2 minutos).

---

## 8. Verificación Final

### 8.1 Cambiar nombre del sitio (opcional)

1. Ve a **Site configuration** → **General** → **Site details**
2. Click en **Change site name**
3. Escribe: `my-app`
4. Tu URL será: `https://my-app.netlify.app`

### 8.2 Probar todo funciona

**✅ Checklist:**

- [ ] Página principal carga: `https://my-app.netlify.app`
- [ ] Login funciona y conecta con backend
- [ ] Navegación interna funciona (click en links)
- [ ] **Recarga de página (F5)** en `/login` funciona
- [ ] **Navegación directa** a `https://my-app.netlify.app/profile` funciona
- [ ] No hay errores CORS en consola
- [ ] No aparece "Page Not Found"

**Probar recarga:**
```
1. Ve a: https://my-app.netlify.app
2. Navega a cualquier ruta: /login, /profile, etc.
3. Presiona F5
4. ✅ Debería funcionar
5. ❌ Si muestra "404", verifica que netlify.toml esté correcto
```

**Probar navegación directa:**
```
1. Copia: https://my-app.netlify.app/profile
2. Pega en nueva pestaña del navegador
3. ✅ Debería cargar la página correctamente
4. ❌ Si muestra "404", verifica netlify.toml
```

### 8.3 Verificar variables de entorno

En consola del navegador (F12):

```javascript
console.log('Backend URL:', import.meta.env.VITE_BACKEND_URL);
console.log('Mode:', import.meta.env.MODE);
console.log('Production:', import.meta.env.PROD);

// Debería mostrar:
// Backend URL: https://your-backend.onrender.com/api/v1
// Mode: production
// Production: true
```

### 8.4 Verificar conexión con backend

```javascript
// En consola del navegador
fetch(import.meta.env.VITE_BACKEND_URL + '/health', {
  method: 'GET',
  headers: { 'Content-Type': 'application/json' }
})
  .then(res => res.json())
  .then(data => console.log('✅ Backend connected!', data))
  .catch(err => console.error('❌ Connection failed:', err));
```

---

## Resumen

### Archivos creados/modificados:

```
frontend-web/
├── .env.production          ← Creado (NO subir a Git)
├── .env.development         ← Creado (NO subir a Git)
├── .env.example             ← Creado (SÍ subir a Git)
├── netlify.toml             ← Creado (SÍ subir a Git)
├── .gitignore               ← Actualizado
└── src/
    └── services/
        └── AuthService.js   ← Actualizado (process.env → import.meta.env)
```

### Cambios en backend:

```
backend-api/
└── src/
    └── config/
        └── SecurityConfig.java  ← Actualizado (agregado Netlify a CORS)
```

### URLs finales:

| Componente | URL |
|-----------|-----|
| **Frontend** | `https://my-app.netlify.app` |
| **Backend** | `https://my-backend.onrender.com` |
| **API** | `https://my-backend.onrender.com/api/v1` |

---

**¡Listo!** Tu aplicación React + Vite está desplegada en Netlify y conectada con tu backend en Render. 🚀