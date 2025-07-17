# Guía Completa: Deploy de Spring Boot en Render con Neon PostgreSQL

## Tabla de Contenidos
1. [Configuración de Neon Database](#1-configuración-de-neon-database)
2. [Preparación del Proyecto Spring Boot](#2-preparación-del-proyecto-spring-boot)
3. [Configuración de GitHub](#3-configuración-de-github)
4. [Deploy en Render](#4-deploy-en-render)
5. [Verificación y Troubleshooting](#5-verificación-y-troubleshooting)

---

## 1. Configuración de Neon Database

### 1.1 Crear cuenta en Neon

1. Ve a [https://neon.tech](https://neon.tech)
2. Click en **Sign Up**
3. Regístrate con:
   - GitHub (recomendado)
   - Google
   - Email

### 1.2 Crear proyecto y base de datos

1. Una vez dentro, click en **Create Project**
2. Configura:
   - **Project name**: `tomevault` (o tu nombre preferido)
   - **Region**: Selecciona `South America (São Paulo)` para mejor latencia
   - **Postgres version**: 16 (la más reciente)
3. Click en **Create Project**

### 1.3 Obtener credenciales de conexión

1. En tu proyecto, ve a **Dashboard**
2. En la sección **Connection Details**, verás algo como:
   ```
   postgresql://neondb_owner:npg_xxxxx@ep-small-art-xxxxx-pooler.sa-east-1.aws.neon.tech/neondb?sslmode=require
   ```
3. **Guarda esta información**, la necesitarás después:
   - **Host**: `ep-small-art-xxxxx-pooler.sa-east-1.aws.neon.tech`
   - **Database**: `neondb`
   - **Username**: `neondb_owner`
   - **Password**: `npg_xxxxx`

### 1.4 Configurar IP Allowlist (opcional pero recomendado)

1. Ve a **Settings** → **IP Allow**
2. Para desarrollo, puedes permitir todas las IPs: `0.0.0.0/0`
3. Para producción, restringe solo a las IPs de Render

---

## 2. Preparación del Proyecto Spring Boot

### 2.1 Crear archivo `.gitignore`

Crea o actualiza tu `.gitignore` en la raíz del proyecto:

```gitignore
# Maven
target/
!.mvn/wrapper/maven-wrapper.jar
!**/src/main/**/target/
!**/src/test/**/target/

# Gradle
.gradle
build/
!gradle/wrapper/gradle-wrapper.jar
!**/src/main/**/build/
!**/src/test/**/build/

# Environment variables
.env
.env.local
.env.*.local
application-local.properties
application-prod.properties

# IDE
.idea/
*.iml
*.ipr
*.iws
.vscode/
.DS_Store

# Logs
*.log
```

### 2.2 Configurar `application.properties`

En `src/main/resources/application.properties`:

```properties
spring.application.name=tomevault

server.servlet.context-path=/api/v1

# Database Configuration
spring.datasource.url=${DATABASE_URL}
spring.datasource.username=${DATABASE_USERNAME}
spring.datasource.password=${DATABASE_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# Hibernate Configuration
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true

# Google Books API
app.google-books.url=https://www.googleapis.com/books/v1/volumes
app.google-books.key=${GOOGLE_BOOKS_API_KEY}

# JWT Configuration
security.jwt.key.private=${JWT_PRIVATE_KEY_TOME}
security.jwt.user.generator=AUTH0JWT-Backend
```

### 2.3 Crear `application.properties.example`

Crea este archivo como plantilla (SÍ subirlo a Git):

```properties
spring.application.name=tomevault

server.servlet.context-path=/api/v1

# Database Configuration (Neon PostgreSQL)
spring.datasource.url=jdbc:postgresql://YOUR_NEON_HOST/YOUR_DATABASE?sslmode=require
spring.datasource.username=YOUR_USERNAME
spring.datasource.password=YOUR_PASSWORD
spring.datasource.driver-class-name=org.postgresql.Driver

# Hibernate Configuration
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true

# Google Books API
app.google-books.url=https://www.googleapis.com/books/v1/volumes
app.google-books.key=YOUR_GOOGLE_BOOKS_API_KEY

# JWT Configuration
security.jwt.key.private=YOUR_JWT_SECRET_KEY
security.jwt.user.generator=AUTH0JWT-Backend
```

### 2.4 Crear o actualizar `Dockerfile`

Crea un `Dockerfile` en la raíz del proyecto:

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN apk add --no-cache maven
RUN mvn clean package -DskipTests

# Stage 2: Run  
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar tomevault.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "tomevault.jar"]
```

**Explicación:**
- **Stage 1 (builder)**: Instala Maven y compila tu aplicación
- **Stage 2 (runtime)**: Usa solo JRE (más ligero) para ejecutar el JAR generado
- Esto permite que NO necesites subir `target/` a GitHub

### 2.5 Verificar dependencias en `pom.xml`

Asegúrate de tener estas dependencias:

```xml
<!-- PostgreSQL Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

---

## 3. Configuración de GitHub

### 3.1 Inicializar repositorio Git (si aún no lo has hecho)

```bash
git init
git add .
git commit -m "Initial commit: Spring Boot project setup"
```

### 3.2 Crear repositorio en GitHub

1. Ve a [https://github.com](https://github.com)
2. Click en el **+** (arriba derecha) → **New repository**
3. Configura:
   - **Repository name**: `tomevault`
   - **Description**: "Spring Boot REST API for book collection management"
   - **Visibility**: Private (recomendado) o Public
   - **NO** marques "Initialize with README" (ya tienes tu proyecto)
4. Click en **Create repository**

### 3.3 Conectar repositorio local con GitHub

```bash
git remote add origin https://github.com/TU_USUARIO/tomevault.git
git branch -M main
git push -u origin main
```

### 3.4 Verificar que `.gitignore` funcione

```bash
git status
```

**Asegúrate que NO aparezcan:**
- `target/` directory
- `.env` files
- `application-local.properties`

Si aparecen, significa que tu `.gitignore` no está funcionando. Elimínalos del tracking:

```bash
git rm -r --cached target/
git rm --cached .env
git commit -m "Remove ignored files from tracking"
git push
```

---

## 4. Deploy en Render

### 4.1 Crear cuenta en Render

1. Ve a [https://render.com](https://render.com)
2. Click en **Get Started**
3. Regístrate con **GitHub** (esto facilitará la conexión)

### 4.2 Conectar GitHub con Render

1. Una vez dentro de Render, ve a **Account Settings** (icono de usuario arriba derecha)
2. En el menú izquierdo, click en **GitHub**
3. Click en **Connect GitHub Account** o **Configure**
4. Autoriza a Render para acceder a tus repositorios
5. Selecciona:
   - **All repositories** (si confías en Render)
   - **Only select repositories** → Selecciona `tomevault`
6. Click en **Install & Authorize**

### 4.3 Crear nuevo Web Service

1. En el Dashboard de Render, click en **New +** → **Web Service**
2. Busca tu repositorio `tomevault` y click en **Connect**
3. Configura el servicio:

#### Basic Settings:
- **Name**: `tomevault-api` (este será parte de tu URL)
- **Region**: `Oregon (US West)` (gratis) o el más cercano a ti
- **Branch**: `main`
- **Root Directory**: (déjalo vacío si tu `Dockerfile` está en la raíz)
- **Runtime**: **Docker**

#### Build Settings:
- **Dockerfile Path**: `./Dockerfile`
- **Docker Command**: (déjalo vacío, el `ENTRYPOINT` en tu Dockerfile lo maneja)

#### Instance Type:
- **Free** (para pruebas) o **Starter** ($7/mes con más recursos)

4. **NO** hagas click en "Create Web Service" todavía

### 4.4 Configurar Variables de Entorno

Antes de crear el servicio, configura las variables de entorno:

1. Baja hasta la sección **Environment Variables**
2. Click en **Add Environment Variable** y agrega **una por una**:

```
Key: DATABASE_URL
Value: jdbc:postgresql://ep-small-art-xxxxx-pooler.sa-east-1.aws.neon.tech/neondb?sslmode=require
```

**IMPORTANTE**: 
- Cambia `postgresql://` por `jdbc:postgresql://`
- **ELIMINA** `channel_binding=require` de la URL de Neon
- Solo deja `sslmode=require` al final

```
Key: DATABASE_USERNAME
Value: neondb_owner
```

```
Key: DATABASE_PASSWORD
Value: npg_xxxxx
```

```
Key: GOOGLE_BOOKS_API_KEY
Value: tu_api_key_de_google_books
```

```
Key: JWT_PRIVATE_KEY_TOME
Value: tu_clave_secreta_jwt_generada
```

**Ejemplo completo de variables:**

| Key | Value |
|-----|-------|
| `DATABASE_URL` | `jdbc:postgresql://ep-small-art-a1b2c3d4-pooler.sa-east-1.aws.neon.tech/neondb?sslmode=require` |
| `DATABASE_USERNAME` | `neondb_owner` |
| `DATABASE_PASSWORD` | `npg_Abc123XyZ` |
| `GOOGLE_BOOKS_API_KEY` | `AIzaSyD_your_api_key_here` |
| `JWT_PRIVATE_KEY_TOME` | `my-super-secret-jwt-key-2024` |

### 4.5 Crear el servicio

1. Verifica que todas las variables estén correctas
2. Click en **Create Web Service**
3. Render comenzará a:
   - Clonar tu repositorio
   - Construir la imagen Docker (compilará tu código)
   - Iniciar el contenedor
   - Esto toma 5-10 minutos la primera vez

### 4.6 Monitorear el deploy

1. Verás logs en tiempo real
2. Busca estos mensajes de éxito:
   ```
   ==> Build successful
   ==> Deploying...
   ==> Starting service with 'java -jar tomevault.jar'
   Started TomevaultApplication in X.XXX seconds
   ```

3. Si todo está bien, verás:
   ```
   Your service is live 🎉
   ```

---

## 5. Verificación y Troubleshooting

### 5.1 Obtener la URL de tu API

1. En Render, tu servicio estará disponible en:
   ```
   https://tomevault-api.onrender.com
   ```

2. Tu API estará en:
   ```
   https://tomevault-api.onrender.com/api/v1
   ```

### 5.2 Probar endpoints

Prueba con Postman o curl:

```bash
# Health check (si tienes un endpoint de status)
curl https://tomevault-api.onrender.com/api/v1/health

# Login
curl -X POST https://tomevault-api.onrender.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}'
```

### 5.3 Errores Comunes y Soluciones

#### Error: "openjdk:21-jdk: not found"
**Causa**: Imagen Docker obsoleta  
**Solución**: Usa `eclipse-temurin:21-jdk-alpine` en tu Dockerfile

#### Error: "lstat /target: no such file or directory"
**Causa**: El directorio `target/` no existe porque no se compiló  
**Solución**: Usa el Dockerfile multi-stage que compila automáticamente

#### Error: "Connection refused" o "Connection timeout" a la base de datos
**Causa**: URL de base de datos incorrecta o falta `sslmode=require`  
**Solución**:
1. Verifica que `DATABASE_URL` comience con `jdbc:postgresql://`
2. Asegúrate de tener `?sslmode=require` al final
3. NO incluyas `channel_binding=require`
4. Verifica que las credenciales sean correctas

#### Error: "Unable to acquire JDBC Connection"
**Causa**: Credenciales incorrectas o problema de SSL  
**Solución**:
1. Verifica `DATABASE_USERNAME` y `DATABASE_PASSWORD`
2. En Neon, ve a **Settings** → **IP Allow** → Permite `0.0.0.0/0`

#### Error: "Application failed to start - Port 8080 is already in use"
**Causa**: Render espera que tu app use el puerto que él asigna  
**Solución**: Agrega esta variable de entorno en Render:
```
PORT=8080
```

Y en `application.properties`:
```properties
server.port=${PORT:8080}
```

#### App se inicia pero no responde
**Causa**: Render Free tier entra en "sleep" después de inactividad  
**Solución**:
- Primera petición tarda 50+ segundos en despertar
- Considera upgrade a plan Starter
- O usa un servicio de "keep-alive" que haga ping cada 10 minutos

### 5.4 Ver logs en tiempo real

En Render:
1. Ve a tu servicio
2. Click en **Logs** (menú izquierdo)
3. Verás logs de Spring Boot en tiempo real

### 5.5 Actualizar tu aplicación

Cada vez que hagas push a GitHub:

```bash
git add .
git commit -m "feat: Add new endpoint for book search"
git push
```

Render automáticamente:
1. Detecta el cambio
2. Reconstruye la imagen Docker
3. Despliega la nueva versión
4. Zero downtime si está configurado

---

## 6. Mejores Prácticas

### 6.1 Seguridad

✅ **NUNCA** subas a Git:
- Credenciales de base de datos
- API keys
- JWT secrets
- Archivos `.env`
- Directorio `target/`

✅ **SIEMPRE**:
- Usa variables de entorno
- Mantén `.gitignore` actualizado
- Usa repositorio privado para proyectos con datos sensibles
- Rota tus secrets periódicamente

### 6.2 Base de datos

- Usa `spring.jpa.hibernate.ddl-auto=update` solo en desarrollo
- En producción, usa `validate` y gestiona migraciones con Flyway o Liquibase
- Haz backups regulares de tu base de datos Neon

### 6.3 Monitoreo

- Revisa logs en Render regularmente
- Configura alertas para errores críticos
- Usa Render Metrics para monitorear uso de CPU/memoria

---

## 7. Resumen de Checklist

Antes de hacer deploy, verifica:

- [ ] `.gitignore` configurado correctamente
- [ ] `target/` NO está en Git
- [ ] Variables de entorno NO están hardcodeadas
- [ ] `Dockerfile` multi-stage creado
- [ ] Repositorio pusheado a GitHub
- [ ] Cuenta de Neon creada y DB configurada
- [ ] `DATABASE_URL` comienza con `jdbc:postgresql://`
- [ ] `DATABASE_URL` termina con `?sslmode=require` (sin `channel_binding`)
- [ ] Todas las variables de entorno configuradas en Render
- [ ] Render conectado con GitHub
- [ ] Deploy exitoso y logs sin errores

---

## 8. URLs de Referencia

- **Neon Docs**: https://neon.tech/docs/introduction
- **Render Docs**: https://render.com/docs
- **Spring Boot Docs**: https://docs.spring.io/spring-boot/docs/current/reference/html/
- **Eclipse Temurin**: https://adoptium.net/

---

**¡Listo!** Tu aplicación Spring Boot debería estar corriendo en producción con Render y Neon PostgreSQL. 🚀