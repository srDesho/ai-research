# Solucionando NullPointerException en @AuthenticationPrincipal con JWT en Spring Security

## Problema

Cuando implementas autenticación JWT en Spring Boot y tratas de usar `@AuthenticationPrincipal` para obtener los detalles del usuario autenticado, obtienes un `NullPointerException`:

```java
@PostMapping("/increment-read/{bookId}")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<BookResponseDTO> incrementBookReadCount(
        @AuthenticationPrincipal CustomUserDetails customUserDetails, // ❌ Es null
        @PathVariable Long bookId) {
    // NullPointerException aquí ↓
    BookResponseDTO updatedBook = this.bookService.incrementBookReadCount(bookId, customUserDetails.getUserEntity());
    return ResponseEntity.ok(updatedBook);
}
```

**Error típico:**
```
java.lang.NullPointerException: Cannot invoke "CustomUserDetails.getUserEntity()" because "customUserDetails" is null
```

## Causa del Problema

El problema ocurre en el filtro JWT cuando estableces la autenticación en el `SecurityContext`. Si usas solo el `username` como principal:

```java
// ❌ INCORRECTO - Solo username como principal
Authentication authentication = new UsernamePasswordAuthenticationToken(
    username, // String, no UserDetails
    null, 
    authorities
);
```

Spring Security no puede inyectar un objeto `CustomUserDetails` en `@AuthenticationPrincipal` porque el principal es solo un `String`.

## Solución

### Paso 1: Modificar el filtro JWT

En tu `JwtTokenValidator`, cambia la creación del `Authentication` para incluir el objeto completo `UserDetails`:

```java
@Component
@RequiredArgsConstructor
public class JwtTokenValidator extends OncePerRequestFilter {

    private final JwtUtils jwtUtils;
    private final UserRepository userRepository; // ✅ Inyectar repositorio

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            // Extraer y validar token
            String token = authHeader.substring(7);
            DecodedJWT decodedJWT = this.jwtUtils.validateToken(token);
            String username = jwtUtils.extractUsername(decodedJWT);

            // ✅ SOLUCIÓN: Cargar UserEntity completa desde BD
            UserEntity userEntity = userRepository.findUserEntityByUsername(username)
                    .orElseThrow(() -> new RuntimeException("User not found: " + username));

            // ✅ Crear CustomUserDetails con UserEntity
            CustomUserDetails customUserDetails = new CustomUserDetails(userEntity);

            // ✅ CLAVE: Usar CustomUserDetails como principal (no solo username)
            Authentication authentication = new UsernamePasswordAuthenticationToken(
                    customUserDetails, // ✅ UserDetails completo como principal
                    null, 
                    customUserDetails.getAuthorities()
            );

            SecurityContextHolder.getContext().setAuthentication(authentication);

        } catch (Exception e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Authentication failed: " + e.getMessage());
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

### Paso 2: Asegurar que el repositorio tenga @Repository

El `UserRepository` debe estar anotado con `@Repository` para ser detectado por Spring:

```java
@Repository // ✅ IMPORTANTE: Sin esto, Spring no puede inyectar el repositorio
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findUserEntityByUsername(String username);
}
```

### Paso 3: Crear CustomUserDetails si no existe

Tu clase `CustomUserDetails` debe implementar `UserDetails` y contener la referencia a tu entidad:

```java
@RequiredArgsConstructor
public class CustomUserDetails implements UserDetails {

    private final UserEntity userEntity;

    // ✅ Método clave para acceder a la entidad desde controladores
    public UserEntity getUserEntity() {
        return this.userEntity;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<GrantedAuthority> authorities = new HashSet<>();
        
        // Agregar roles
        userEntity.getRoleList().forEach(role ->
                authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getRoleEnum().name())));
        
        // Agregar permisos
        userEntity.getRoleList().stream()
                .flatMap(roleEntity -> roleEntity.getPermissionList().stream())
                .forEach(permission -> 
                    authorities.add(new SimpleGrantedAuthority(permission.getPermissionEnum().name())));
        
        return authorities;
    }

    @Override
    public String getPassword() { return this.userEntity.getPassword(); }

    @Override
    public String getUsername() { return this.userEntity.getUsername(); }

    // Otros métodos de UserDetails...
}
```

## ¿Por qué funciona esta solución?

### Antes (❌ No funciona)
```java
// Principal es solo String
Authentication auth = new UsernamePasswordAuthenticationToken("username", null, authorities);

// En el controlador
@AuthenticationPrincipal CustomUserDetails user // ❌ null - no puede convertir String a CustomUserDetails
```

### Después (✅ Funciona)
```java
// Principal es CustomUserDetails completo
Authentication auth = new UsernamePasswordAuthenticationToken(customUserDetails, null, authorities);

// En el controlador
@AuthenticationPrincipal CustomUserDetails user // ✅ Recibe el objeto completo
```

## Ejemplo de uso en controladores

Una vez implementada la solución, puedes usar `@AuthenticationPrincipal` sin problemas:

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<List<Book>> getUserBooks(
            @AuthenticationPrincipal CustomUserDetails userDetails) {
        
        // ✅ Ahora funciona correctamente
        UserEntity user = userDetails.getUserEntity();
        List<Book> books = bookService.findByUser(user);
        return ResponseEntity.ok(books);
    }

    @PostMapping("/{id}/favorite")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> addToFavorites(
            @AuthenticationPrincipal CustomUserDetails userDetails,
            @PathVariable Long id) {
        
        // ✅ Acceso directo a la entidad del usuario
        UserEntity user = userDetails.getUserEntity();
        bookService.addToFavorites(id, user);
        return ResponseEntity.ok().build();
    }
}
```

## Dependencia Circular (Problema común)

Si encuentras un error de dependencia circular:

```
┌─────┐
|  jwtTokenValidator
↑     ↓
|  userDetailsServiceImpl  
↑     ↓
|  securityConfig
└─────┘
```

**Evita inyectar `UserDetailsService` en el filtro JWT.** Usa directamente el `UserRepository` como se muestra en la solución.

## ¿Por qué @Repository es necesario?

Sin `@Repository`, Spring Boot no detecta automáticamente tu repositorio:

1. **Detección de componentes**: Spring necesita `@Repository` para registrar el bean
2. **Inyección de dependencias**: Sin el bean, la inyección falla
3. **Traducción de excepciones**: Convierte excepciones de BD a excepciones de Spring
4. **Semántica clara**: Indica que es una clase de acceso a datos

## Casos de uso comunes

Esta solución es aplicable cuando necesitas:

- ✅ Acceder al usuario actual en controladores REST
- ✅ Implementar operaciones específicas por usuario (CRUD personalizado)
- ✅ Aplicar filtros de seguridad a nivel de datos
- ✅ Auditoría automática (quién modificó qué)
- ✅ Personalización de respuestas según el usuario

## Alternativas consideradas

1. **Usar solo String como principal**: ❌ Requiere consultas adicionales en cada endpoint
2. **Incluir más datos en JWT**: ❌ Tokens grandes, datos desactualizados
3. **ThreadLocal personalizado**: ❌ Más complejo, reinventa Spring Security
4. **Esta solución**: ✅ Simple, estándar, eficiente

## Conclusión

La clave está en entender que `@AuthenticationPrincipal` necesita que el `Authentication.getPrincipal()` sea del tipo esperado. Al establecer un objeto `CustomUserDetails` completo como principal en el filtro JWT, Spring Security puede inyectarlo correctamente en los controladores.

Esta solución mantiene la arquitectura estándar de Spring Security mientras proporciona acceso completo a los datos del usuario autenticado.