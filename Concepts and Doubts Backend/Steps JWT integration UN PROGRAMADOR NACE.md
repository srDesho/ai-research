Steps JWT integration UN PROGRAMADOR NACE Method:

¡Claro\! Aquí tienes de nuevo toda la información y el código para implementar JWT en tu backend de Spring Boot, pero esta vez con todos los códigos directamente en pantalla, sin usar las etiquetas `immersive`. Mantendré los comentarios técnicos en inglés dentro del código y la explicación ("cátedra") en español.

-----

## La Autenticación con JWT en Spring Boot: Una Perspectiva de Entrevista 🧑‍💻

Cuando hablamos de autenticación en aplicaciones modernas, especialmente con frontends desacoplados (como tu aplicación React), **JWT** (JSON Web Tokens) es la solución predominante. La razón principal es su naturaleza **`stateless` (sin estado)**. Esto significa que el servidor no necesita mantener un registro de las sesiones de los usuarios. Cada token contiene toda la información necesaria para que el servidor lo valide de forma independiente.

### El Flujo de Autenticación con JWT (el "Cómo" y el "Por qué")

1.  **Petición de Credenciales (Login):**

      * El usuario envía su `username` y `password` al backend (normalmente a un endpoint `/auth/login`).
      * **Backend:** Spring Security valida estas credenciales. Si son correctas, en lugar de crear una sesión en el servidor, **genera un JWT**.
      * **JWT Contenido:** Este token incluye información útil (los "claims"), como el ID del usuario, sus roles/permisos, y una fecha de expiración. Además, está firmado digitalmente con una clave secreta del servidor.
      * **Respuesta:** El servidor devuelve este JWT al cliente (tu aplicación React).

2.  **Almacenamiento del Token (Frontend):**

      * La aplicación React recibe el JWT y lo almacena (comúnmente en `localStorage` o `sessionStorage`).

3.  **Peticiones Protegidas (Acceso a Recursos):**

      * Para acceder a recursos protegidos (ej., `/api/v1/books`), la aplicación React **adjunta el JWT en el encabezado `Authorization`** de cada solicitud, típicamente en formato `Bearer <token>`.
      * **Backend:** Antes de que la solicitud llegue a tu controlador, un **filtro de seguridad de Spring Security** intercepta la petición:
          * Extrae el JWT del encabezado.
          * Valida la firma del token (usando la misma clave secreta). Si la firma es inválida o el token ha sido alterado, se rechaza la petición.
          * Verifica la fecha de expiración.
          * Decodifica el token para extraer los claims (`username`, `roles/permisos`).
          * Crea un objeto de autenticación de Spring Security y lo establece en el **`SecurityContext`**. Esto hace que el usuario sea "autenticado" para el resto de la cadena de filtros y tu lógica de negocio.
      * **Autorización:** Una vez autenticado, Spring Security (y tus anotaciones `@PreAuthorize`, `@Secured`, etc.) verifican si el usuario tiene los roles/permisos necesarios para acceder a ese recurso.
      * **Respuesta:** Si todo es válido, la petición procede a tu controlador y se devuelve la respuesta.

### Ventajas Clave para una Entrevista:

  * **Scalability (Escalabilidad):** Al ser `stateless`, no hay necesidad de sincronizar sesiones entre múltiples instancias de servidores (si tuvieras un clúster). Cualquier servidor puede validar cualquier token, lo que facilita la escalabilidad horizontal.
  * **Decoupling (Desacoplamiento):** El frontend y el backend están completamente separados. No dependen de cookies de sesión ni de la infraestructura de sesiones del servidor.
  * **Mobile-Friendly:** JWT es ideal para aplicaciones móviles, ya que el token se puede almacenar fácilmente y enviar en los encabezados HTTP.
  * **Security:** Los tokens son firmados, lo que garantiza su **integridad** (no han sido alterados) y **autenticidad** (fueron emitidos por tu servidor).
  * **Performance:** No hay consultas a la base de datos para buscar sesiones en cada petición protegida.

-----

## Implementación Paso a Paso con tu Código 📚

Vamos a integrar tus fragmentos de código y llenaremos los huecos necesarios en una aplicación típica de Spring Boot.

### 1\. `pom.xml`: Las Dependencias JWT

Tu `pom.xml` ya es perfecto. La dependencia `com.auth0:java-jwt` es la biblioteca que nos permite crear, firmar y verificar los JWT.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.2.3</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.app</groupId>
	<artifactId>SpringSeecurityApp</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>SpringSeecurityApp</name>
	<description>Spring Security App</description>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>com.mysql</groupId>
			<artifactId>mysql-connector-j</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!-- JWT Dependency: Auth0's java-jwt library for JWT creation, signing, and verification -->
		<dependency>
			<groupId>com.auth0</groupId>
			<artifactId>java-jwt</artifactId>
			<version>4.4.0</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<excludes>
						<exclude>
							<groupId>org.projectlombok</groupId>
							<artifactId>lombok</artifactId>
						</exclude>
					</excludes>
				</configuration>
			</plugin>
		</plugins>
	</build>

</project>
```

-----

### 2\. `src/main/java/com/app/util/JwtUtils.java`: El Cerebro del Token

Esta clase es fundamental. Se encarga de la **creación**, **validación** y **extracción de información** de los JWT. Es donde definimos la clave secreta y el emisor (issuer) del token.

  * `@Component`: La marca como un bean de Spring, lo que permite inyectarla en otras clases.
  * `@Value("${security.jwt.key.private}")`: Inyecta el valor de la clave secreta desde `application.properties` (o `application.yml`). **¡Esta clave es crítica\!** Debe ser secreta y solo conocida por tu servidor.
  * `@Value("${security.jwt.user.generator}")`: Inyecta el emisor del token (quién lo generó, que en este caso es tu backend).

<!-- end list -->

```java
package com.app.util;

import com.auth0.jwt.JWT;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTVerificationException;
import com.auth0.jwt.interfaces.DecodedJWT;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

/**
 * Utility class for JWT (JSON Web Token) operations.
 * Handles token creation, validation, and claim extraction.
 */
@Component
public class JwtUtils {

    // Injects the private key from application.properties for signing/verifying JWTs.
    // This key must be strong and kept secret on the server-side.
    @Value("${security.jwt.key.private}")
    private String privateKey;

    // Injects the token generator identifier from application.properties.
    // This identifies who issued the JWT (e.g., your backend application name).
    @Value("${security.jwt.user.generator}")
    private String userGenerator;

    /**
     * Creates a new JWT for the authenticated user.
     * @param username The user's username.
     * @param authorities The user's granted authorities (roles and permissions).
     * @return A signed JWT string.
     */
    public String createToken(String username, List<GrantedAuthority> authorities) {
        // Define the algorithm for signing the token using the private key.
        Algorithm algorithm = Algorithm.HMAC256(privateKey); 
        
        // Map GrantedAuthority objects to String representation for inclusion in token claims.
        List<String> authoritiesClaims = authorities.stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList());

        return JWT.create()
                .withIssuer(this.userGenerator) // Issuer of the token (your backend)
                .withSubject(username) // Subject of the token (the authenticated user's username)
                .withClaim("authorities", authoritiesClaims) // Custom claim for user authorities/permissions
                .withIssuedAt(new Date()) // Timestamp when the token was issued
                // Token expiration time (e.g., 24 hours from now)
                .withExpiresAt(new Date(System.currentTimeMillis() + TimeUnit.HOURS.toMillis(24)))
                .sign(algorithm); // Sign the token with the defined algorithm and private key
    }

    /**
     * Validates and decodes a given JWT.
     * @param token The JWT string to validate.
     * @return A DecodedJWT object if the token is valid.
     * @throws JWTVerificationException If the token is invalid, expired, or tampered with.
     */
    public DecodedJWT validateToken(String token) {
        try {
            // Re-create the algorithm using the same private key for verification.
            Algorithm algorithm = Algorithm.HMAC256(privateKey);
            return JWT.require(algorithm)
                    .withIssuer(this.userGenerator) // Must match the issuer set during token creation
                    .build()
                    .verify(token); // Verify the signature and validity of the token
        } catch (JWTVerificationException e) {
            // Catch specific JWT exceptions and re-throw with a generic message for security.
            throw new JWTVerificationException("Invalid or expired token. Message: " + e.getMessage());
        }
    }

    /**
     * Extracts the subject (username) from a decoded JWT.
     * @param decodedJWT The decoded JWT object.
     * @return The username as a String.
     */
    public String extractUsername(DecodedJWT decodedJWT) {
        return decodedJWT.getSubject();
    }

    /**
     * Extracts authorities (roles/permissions) from a decoded JWT.
     * Assumes authorities are stored as a List<String> in the "authorities" claim.
     * @param decodedJWT The decoded JWT object.
     * @return A list of GrantedAuthority objects.
     */
    public List<GrantedAuthority> extractAuthorities(DecodedJWT decodedJWT) {
        // Retrieve the list of authority strings from the "authorities" claim.
        return decodedJWT.getClaim("authorities").asList(String.class)
                .stream()
                .map(SimpleGrantedAuthority::new) // Convert each string to SimpleGrantedAuthority
                .collect(Collectors.toList());
    }
}
```

**`application.properties` (or `application.yml`):**

Necesitas agregar estas propiedades para que `JwtUtils` funcione. El `privateKey` debe ser una cadena aleatoria y **secreta**. Puedes generar una buena clave usando un generador de contraseñas seguras, o incluso una cadena base64 larga.

```properties
# JWT Configuration
security.jwt.key.private=your-super-secret-jwt-key-that-should-be-very-long-and-random-and-kept-securely
security.jwt.user.generator=MyBookAppBackend
```

-----

### 3\. `src/main/java/com/app/service/UserDetailServiceImpl.java`: El Servicio de Usuario y Generador de Tokens

Esta clase es el corazón de la integración de Spring Security con tus usuarios. Hereda de `UserDetailsService` (para cargar usuarios para Spring Security) y contiene la lógica para registrar y autenticar usuarios, y **generar el JWT** en el proceso de login y registro.

  * `loadUserByUsername`: Método requerido por `UserDetailsService`. Carga un `UserEntity` desde tu repositorio y lo transforma en un `UserDetails` de Spring Security, incluyendo sus roles y permisos como `GrantedAuthority`.
  * `loginUser`: Este es el método que tu `AuthController` llamará para el login. Es donde se realiza la autenticación (usando el `authenticate` de Spring Security) y, si es exitosa, se **llama a `jwtUtils.createToken` para generar el JWT**.
  * `authenticate`: Un método auxiliar que verifica las credenciales del usuario con `PasswordEncoder`.
  * `createUser`: Similar a `loginUser`, pero para el registro. Después de guardar el nuevo usuario, también **genera un JWT** para él.

<!-- end list -->

```java
package com.app.service;

import com.app.controller.dto.AuthCreateUserRequest;
import com.app.controller.dto.AuthLoginRequest;
import com.app.controller.dto.AuthResponse;
import com.app.persistence.entity.RoleEntity;
import com.app.persistence.entity.UserEntity;
import com.app.persistence.repository.UserRepository;
import com.app.util.JwtUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * Service implementation for Spring Security's UserDetailsService.
 * Handles loading user details, authentication logic, and JWT generation for login/registration.
 */
@Service
public class UserDetailServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    private PasswordEncoder passwordEncoder;

    // Assuming you have a RoleRepository if roles are mapped via IDs/Enums
    // @Autowired
    // private RoleRepository roleRepository;

    /**
     * Loads user details by username for Spring Security.
     * This method is called by Spring Security's authentication provider.
     * @param username The username to load.
     * @return UserDetails object containing user information and authorities.
     * @throws UsernameNotFoundException If the user does not exist.
     */
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // Retrieve UserEntity from the database. Throws UsernameNotFoundException if not found.
        UserEntity userEntity = userRepository.findUserEntityByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("El usuario " + username + " no fue encontrado."));

        List<SimpleGrantedAuthority> authorityList = new ArrayList<>();

        // Add user roles as GrantedAuthorities (prefixed with "ROLE_").
        userEntity.getRoles()
                .forEach(role -> authorityList.add(new SimpleGrantedAuthority("ROLE_" + role.getRoleEnum().name())));

        // Add user permissions from roles as GrantedAuthorities.
        userEntity.getRoles().stream()
                .flatMap(role -> role.getPermissionList().stream())
                .forEach(permission -> authorityList.add(new SimpleGrantedAuthority(permission.getName())));

        // Return a Spring Security User object.
        return new User(userEntity.getUsername(),
                userEntity.getPassword(),
                userEntity.isEnabled(),
                userEntity.isAccountNonExpired(),
                userEntity.isCredentialsNonExpired(),
                userEntity.isAccountNonLocked(),
                authorityList);
    }

    /**
     * Handles user login and generates an access token (JWT).
     * This method is typically called from the AuthenticationController.
     * @param authLoginRequest DTO containing username and password.
     * @return AuthResponse containing username, message, JWT, and status.
     * @throws BadCredentialsException If credentials are invalid.
     */
    public AuthResponse loginUser(AuthLoginRequest authLoginRequest) {
        String username = authLoginRequest.username();
        String password = authLoginRequest.password();

        // Authenticate user credentials. This will throw an exception if credentials are bad.
        Authentication authentication = this.authenticate(username, password);
        
        // Set the authenticated user in Spring Security's context.
        SecurityContextHolder.getContext().setAuthentication(authentication);

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        // Generate the JWT access token using the user's username and authorities.
        String accessToken = jwtUtils.createToken(userDetails.getUsername(), userDetails.getAuthorities().stream().collect(Collectors.toList()));
        
        // Construct and return the authentication response.
        return new AuthResponse(username, "Login exitoso", accessToken, true);
    }

    /**
     * Authenticates user credentials.
     * @param username The username provided by the user.
     * @param password The raw password provided by the user.
     * @return An Authentication object if credentials are valid.
     * @throws BadCredentialsException If username not found or password does not match.
     */
    public Authentication authenticate(String username, String password) {
        // Load user details by username. Throws UsernameNotFoundException if user doesn't exist.
        UserDetails userDetails = loadUserByUsername(username);

        // Validate if the raw password matches the encoded password.
        if (!passwordEncoder.matches(password, userDetails.getPassword())) {
            throw new BadCredentialsException("Contraseña incorrecta.");
        }

        // Return an authenticated token without exposing the password.
        return new UsernamePasswordAuthenticationToken(username, userDetails.getPassword(), userDetails.getAuthorities());
    }

    /**
     * Creates a new user in the database and generates an access token (JWT) for them.
     * @param authCreateUserRequest DTO containing new user details and desired roles.
     * @return AuthResponse containing new username, message, JWT, and status.
     */
    public AuthResponse createUser(AuthCreateUserRequest authCreateUserRequest) {
        String username = authCreateUserRequest.username();
        String password = authCreateUserRequest.password();
        List<String> roleRequest = authCreateUserRequest.roleRequest().roleListName();

        // Map role names from request to actual RoleEntity objects.
        // Assumes RoleEntity has a constructor/builder that accepts RoleEnum or string name.
        // You might need to fetch RoleEntity from a repository based on role names if they are database entities.
        Set<RoleEntity> roles = roleRequest.stream()
                .map(role -> RoleEntity.builder().roleEnum(Enum.valueOf(RoleEntity.RoleEnum.class, role)).build()) // Example: assuming RoleEntity has a builder with RoleEnum
                .collect(Collectors.toSet());

        // Build and save the new user entity with encoded password and assigned roles.
        UserEntity userEntity = UserEntity.builder()
                .username(username)
                .password(passwordEncoder.encode(password)) // IMPORTANT: Encode password using BCrypt
                .roles(roles)
                .isEnabled(true)
                .accountNonLocked(true)
                .accountNonExpired(true)
                .credentialsNonExpired(true)
                .build();

        userRepository.save(userEntity);

        // Collect authorities (roles and permissions) for the newly created user.
        List<SimpleGrantedAuthority> authorities = new ArrayList<>();
        userEntity.getRoles().forEach(role -> authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getRoleEnum().name())));
        userEntity.getRoles().stream()
                .flatMap(role -> role.getPermissionList().stream())
                .forEach(permission -> authorities.add(new SimpleGrantedAuthority(permission.getName())));

        // Generate a JWT for the newly created user.
        String accessToken = jwtUtils.createToken(username, authorities);

        // Define the response.
        return new AuthResponse(username, "Usuario creado exitosamente", accessToken, true);
    }
}
```

-----

### 4\. `src/main/java/com/app/controller/AuthenticationController.java`: Los Endpoints de Autenticación

Este controlador expone los endpoints para que los clientes puedan iniciar sesión (`/log-in`) y registrarse (`/sign-up`). Ambos devuelven un `AuthResponse` que contendrá el JWT.

  * `@RestController` y `@RequestMapping("/auth")`: Define este como un controlador REST para las rutas bajo `/auth`.
  * `login(@RequestBody @Valid AuthLoginRequest userRequest)`: Maneja las solicitudes de inicio de sesión. Llama al `UserDetailServiceImpl` para autenticar y obtener el token.
  * `register(@RequestBody @Valid AuthCreateUserRequest authCreateUser)`: Maneja el registro de nuevos usuarios, también delegando en `UserDetailServiceImpl` y devolviendo un token.

<!-- end list -->

```java
package com.app.controller;

import com.app.controller.dto.AuthCreateUserRequest;
import com.app.controller.dto.AuthLoginRequest;
import com.app.controller.dto.AuthResponse;
import com.app.service.UserDetailServiceImpl;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * REST Controller for user authentication and registration.
 * Provides endpoints for logging in and signing up, returning JWTs upon success.
 */
@RestController
@RequestMapping("/auth")
public class AuthenticationController {

    @Autowired
    private UserDetailServiceImpl userDetailService;

    /**
     * Handles user login requests.
     * Authenticates the user and returns an AuthResponse containing a JWT.
     * @param userRequest DTO containing username and password for login.
     * @return ResponseEntity with AuthResponse (including JWT) and HTTP status OK.
     */
    @PostMapping("/log-in")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid AuthLoginRequest userRequest) {
        AuthResponse authResponse = userDetailService.loginUser(userRequest);
        return new ResponseEntity<>(authResponse, HttpStatus.OK);
    }

    /**
     * Handles new user registration requests.
     * Creates a new user in the system and returns an AuthResponse containing a JWT.
     * @param authCreateUser DTO containing new user details for registration.
     * @return ResponseEntity with AuthResponse (including JWT) and HTTP status CREATED.
     */
    @PostMapping("/sign-up")
    public ResponseEntity<AuthResponse> register(@RequestBody @Valid AuthCreateUserRequest authCreateUser) {
        AuthResponse authResponse = userDetailService.createUser(authCreateUser);
        return new ResponseEntity<>(authResponse, HttpStatus.CREATED);
    }
}
```

**Necesitarás definir los DTOs `AuthLoginRequest`, `AuthCreateUserRequest`, `AuthResponse` y `RoleRequest`** si aún no los tienes. Por ejemplo:

```java
// AuthLoginRequest.java
package com.app.controller.dto;
public record AuthLoginRequest(String username, String password) {}

// AuthCreateUserRequest.java
package com.app.controller.dto;
import java.util.List;
public record AuthCreateUserRequest(String username, String password, RoleRequest roleRequest) {}

// RoleRequest.java
package com.app.controller.dto;
import java.util.List;
public record RoleRequest(List<String> roleListName) {} // Assuming roles are passed as a list of strings

// AuthResponse.java
package com.app.controller.dto;
public record AuthResponse(String username, String message, String jwt, boolean status) {}
```

-----

### 5\. `src/main/java/com/app/config/filter/JwtTokenValidator.java`: El Filtro de Validación del Token

Este es un **filtro personalizado** de Spring Security que intercepta cada solicitud para validar el JWT antes de que la petición llegue a los controladores.

  * `OncePerRequestFilter`: Garantiza que el filtro se ejecute una sola vez por cada solicitud HTTP, evitando duplicidades.
  * `doFilterInternal`: El método principal donde se implementa la lógica del filtro.
      * **Extrae el Token**: Busca el token en el encabezado `Authorization` (`Bearer <token>`).
      * **Valida con `JwtUtils`**: Llama a `jwtUtils.validateToken()` para verificar la firma y la validez del token. Si falla, se lanza una excepción y la cadena de filtros se detiene.
      * **Extrae Claims**: Obtiene el `username` y las `authorities` (roles y permisos) del token decodificado.
      * **Crea `Authentication` y establece `SecurityContext`**: Este es el paso clave. Si el token es válido, se crea un objeto `UsernamePasswordAuthenticationToken` (sin contraseña, ya que el token ya verificó la identidad) y se establece en el **`SecurityContextHolder`**. Esto le dice a Spring Security que la solicitud está autenticada y autorizada para el resto de la ejecución.

<!-- end list -->

    ```java
    package com.cristianml.TomeVault.security.config.filter;

    import com.auth0.jwt.exceptions.JWTVerificationException;
    import com.auth0.jwt.interfaces.DecodedJWT;
    import com.cristianml.TomeVault.utilities.JwtUtils;
    import jakarta.servlet.FilterChain;
    import jakarta.servlet.ServletException;
    import jakarta.servlet.http.HttpServletRequest;
    import jakarta.servlet.http.HttpServletResponse;
    import lombok.RequiredArgsConstructor;
    import org.springframework.lang.NonNull;
    import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
    import org.springframework.security.core.Authentication;
    import org.springframework.security.core.GrantedAuthority;
    import org.springframework.security.core.authority.AuthorityUtils;
    import org.springframework.security.core.context.SecurityContext;
    import org.springframework.security.core.context.SecurityContextHolder;
    import org.springframework.stereotype.Component;
    import org.springframework.web.filter.OncePerRequestFilter;

    import java.io.IOException;
    import java.util.Collection;
    import java.util.List;

    /**
    * Custom Spring Security filter to validate JWTs in incoming requests.
    * Extends OncePerRequestFilter to ensure the filter runs only once per request.
    */
    @Component
    @RequiredArgsConstructor
    public class JwtTokenValidator extends OncePerRequestFilter {

        private final JwtUtils jwtUtils; // Injects JwtUtils for token operations

        /**
        * This method is executed once per HTTP request.
        * It intercepts the request to validate the JWT in the Authorization header.
        */
        @Override
        protected void doFilterInternal(@NonNull HttpServletRequest request,
                                        @NonNull HttpServletResponse response,
                                        @NonNull FilterChain filterChain) throws ServletException, IOException {
            // 1. Get the Authorization header.
            String authHeader = request.getHeader("Authorization");

            // 2. Check if the header exists and starts with "Bearer ".
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                filterChain.doFilter(request, response); // Continue filter chain if no Bearer token is found
                return;
            }

            try {
                // 3. Extract the token (remove "Bearer " prefix).
                String token = authHeader.substring(7);

                // 4. Validate and decode the token using JwtUtils.
                DecodedJWT decodedJWT = this.jwtUtils.validateToken(token);

                // 5. Extract username (subject) and authorities (claims) from the decoded token.
                String username = jwtUtils.extractUsername(decodedJWT);
                // Ensure the "authorities" claim is stored as a comma-separated string in the JWT.
                String srtAuthorities = jwtUtils.getSpecificClaim(decodedJWT, "authorities").asString();

                // Convert the comma-separated authorities string into a Collection of GrantedAuthority.
                // AuthorityUtils.commaSeparatedStringToAuthorityList is a convenient Spring Security utility.
                Collection<? extends GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList(srtAuthorities);

                // 6. Set the authenticated user in Spring Security's context.
                // SecurityContextHolder holds the SecurityContext for the current thread of execution.
                SecurityContext context = SecurityContextHolder.getContext();

                // Create an Authentication object. Password is 'null' as authentication happened via JWT.
                Authentication authentication = new UsernamePasswordAuthenticationToken(username, null, authorities);

                // Set the authentication object in the security context.
                context.setAuthentication(authentication);
                SecurityContextHolder.setContext(context);  // Update the SecurityContextHolder explicitly

            } catch (JWTVerificationException e) {
                // Handle token validation failures (e.g., malformed, expired, invalid signature).
                // Set HTTP status to Unauthorized and write an error message to the response.
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("Authentication failed: " + e.getMessage());
                return; // Stop the filter chain and return response immediately
            } catch (Exception e) {
                // Catch any other unexpected exceptions during token processing.
                response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
                response.getWriter().write("An unexpected error occurred during authentication: " + e.getMessage());
                return; // Stop the filter chain
            }

            // 7. Continue the filter chain. If authentication was successful, the request
            // will now proceed with an authenticated principal in the SecurityContext.
            filterChain.doFilter(request, response);
        }
    }
```

-----

### 6\. `src/main/java/com/app/config/SecurityConfig.java`: La Configuración de Seguridad

Esta es la configuración central de Spring Security. Aquí le decimos a Spring cómo proteger tus endpoints, cómo manejar las sesiones y dónde encaja nuestro `JwtTokenValidator`.

  * `@Configuration` y `@EnableWebSecurity`: Habilitan la configuración de seguridad de Spring.
  * `@EnableMethodSecurity`: Habilita la seguridad a nivel de método (para `@PreAuthorize`, `@Secured`, etc.).
  * `securityFilterChain(HttpSecurity httpSecurity)`: El bean principal para configurar la cadena de filtros de seguridad.
      * `.csrf(csrf -> csrf.disable())`: Deshabilita CSRF. Esto es común en APIs REST sin estado que no dependen de cookies de sesión, ya que los JWT mitigan muchos de los ataques de CSRF.
      * `.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))`: **¡Crucial\!** Configura Spring Security para que no cree ni gestione sesiones HTTP. Esto es lo que hace que tu aplicación sea "sin estado" a nivel del servidor.
      * `.authorizeHttpRequests(...)`: Define las reglas de autorización para tus URLs:
          * `authorize.requestMatchers(HttpMethod.POST, "/auth/**").permitAll();`: Permite el acceso a tus endpoints de login y registro sin autenticación.
          * `authorize.requestMatchers("/api/v1/books/**").authenticated();`: **EJEMPLO:** Protege los endpoints de tu API de libros, requiriendo autenticación.
          * `authorize.anyRequest().authenticated();`: Cualquier otra petición no especificada requiere autenticación por defecto.
      * `.addFilterBefore(jwtTokenValidator, UsernamePasswordAuthenticationFilter.class)`: **¡Esencial\!** Aquí le decimos a Spring que nuestro `JwtTokenValidator` debe ejecutarse *antes* que los filtros de autenticación estándar de Spring. Esto significa que si un JWT válido está presente, nuestro filtro autenticará al usuario y los filtros posteriores pueden saltarse su trabajo.
  * `authenticationManager`: Expone el `AuthenticationManager`, necesario para la autenticación manual (como en tu `UserDetailServiceImpl`).
  * `authenticationProvider`: Configura cómo Spring Security autentica a los usuarios, conectando `UserDetailServiceImpl` y el `PasswordEncoder`.
  * `passwordEncoder`: Define el codificador de contraseñas (`BCrypt`, altamente recomendado). **¡Nunca uses `NoOpPasswordEncoder` en producción\!**

<!-- end list -->

```java
package com.app.config;

import com.app.config.filter.JwtTokenValidator;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

/**
 * Main Spring Security configuration class.
 * Configures security filters, authorization rules, session management, and JWT integration.
 */
@Configuration
@EnableWebSecurity // Enables Spring Security's web security features
@EnableMethodSecurity(prePostEnabled = true) // Enables method-level security (e.g., @PreAuthorize)
public class SecurityConfig {

    @Autowired
    private JwtTokenValidator jwtTokenValidator; // Injects our custom JWT filter

    @Autowired
    private UserDetailsService userDetailsService; // Injects our custom UserDetailsService implementation

    /**
     * Configures the Security Filter Chain. This is the core of Spring Security setup.
     * @param httpSecurity HttpSecurity object to configure security settings.
     * @return The configured SecurityFilterChain.
     * @throws Exception If an error occurs during configuration.
     */
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
        return httpSecurity
                // Disable CSRF (Cross-Site Request Forgery) protection.
                // Common for stateless REST APIs that use token-based authentication (like JWT).
                .csrf(csrf -> csrf.disable())
                // Configure CORS (Cross-Origin Resource Sharing).
                // This is crucial for allowing your frontend (e.g., React on localhost:3000)
                // to make requests to your backend.
                .cors(cors -> cors.configurationSource(corsConfigurationSource())) // Use custom CORS config

                // Configure session management to be stateless.
                // This means Spring Security will not create or use HTTP sessions to store user state,
                // relying entirely on JWT for authentication.
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                
                // Configure authorization rules for HTTP requests.
                .authorizeHttpRequests(authorize -> {
                    // Public endpoints: allow access to /auth/** for POST requests (login, signup) without authentication.
                    authorize.requestMatchers(HttpMethod.POST, "/auth/**").permitAll();
                    
                    // Allow OPTIONS requests (preflight requests for CORS) for all paths.
                    // This is important for browser-based clients.
                    authorize.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll();

                    // Specific private endpoints (example from your course)
                    authorize.requestMatchers(HttpMethod.POST, "/method/post").hasAnyRole("ADMIN", "DEVELOPER");
                    authorize.requestMatchers(HttpMethod.PATCH, "/method/patch").hasAnyAuthority("REFACTOR");
                    authorize.requestMatchers(HttpMethod.GET, "/method/get").hasAnyRole("INVITED");

                    // Protect your /books API endpoints. Only authenticated users can access.
                    // Adjust roles/authorities as needed for your book management logic.
                    authorize.requestMatchers("/api/v1/books/**").authenticated();
                    
                    // Configure other unspecified endpoints: require authentication by default.
                    // If you have more public endpoints, ensure to add them with permitAll()
                    authorize.anyRequest().authenticated();
                })
                // Add our custom JwtTokenValidator filter before Spring Security's UsernamePasswordAuthenticationFilter.
                // This ensures our JWT validation runs early in the filter chain.
                .addFilterBefore(jwtTokenValidator, UsernamePasswordAuthenticationFilter.class)
                .build();
    }

    /**
     * Provides the AuthenticationManager.
     * This bean is necessary for authenticating users, especially in the login process.
     * @param authenticationConfiguration The AuthenticationConfiguration.
     * @return The AuthenticationManager.
     * @throws Exception If an error occurs during configuration.
     */
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    /**
     * Configures the AuthenticationProvider for UserDetailsService.
     * This tells Spring Security how to load user details and how to encode/decode passwords.
     * @return The AuthenticationProvider.
     */
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService); // Sets our custom UserDetailsService
        provider.setPasswordEncoder(passwordEncoder()); // Sets the password encoder
        return provider;
    }

    /**
     * Defines the PasswordEncoder bean.
     * BCryptPasswordEncoder is recommended for strong password hashing.
     * @return The PasswordEncoder.
     */
    @Bean
    public PasswordEncoder passwordEncoder(){
        // NEVER use NoOpPasswordEncoder in production! It's for development/testing unencoded passwords only.
        return new BCryptPasswordEncoder(); 
    }

    /**
     * Configures CORS (Cross-Origin Resource Sharing) for the application.
     * This allows your frontend (e.g., React app on a different port) to communicate with your backend.
     * @return CorsConfigurationSource bean.
     */
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        // Allow requests from your React frontend URL(s). Adjust if your React app runs on a different port/domain.
        configuration.setAllowedOrigins(List.of("http://localhost:3000", "https://your-frontend-domain.com")); // Add your production frontend domain here!
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")); // Allowed HTTP methods
        configuration.setAllowedHeaders(Arrays.asList("Authorization", "Content-Type", "X-Requested-With", "Accept")); // Allowed request headers
        configuration.setAllowCredentials(true); // Allow sending credentials (like JWT in Authorization header)
        configuration.setExposedHeaders(List.of("Authorization")); // Expose Authorization header so frontend can read it if needed

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration); // Apply this CORS config to all paths
        return source;
    }
}
```

-----

### Consideraciones Adicionales:

  * **CORS (Cross-Origin Resource Sharing):** He incluido la configuración CORS en `SecurityConfig`. Esto es **esencial** si tu frontend React se ejecuta en un dominio/puerto diferente al de tu backend (por ejemplo, React en `localhost:3000` y Spring Boot en `localhost:8080`). Ajusta `setAllowedOrigins` en el bean `corsConfigurationSource()` según sea necesario para incluir el dominio de tu frontend en producción.
  * **Seguridad de Endpoints (`.anyRequest().authenticated()`):** He cambiado la línea `http.anyRequests().denyAll()` (que es muy restrictiva) a `http.anyRequest().authenticated()`. Esto significa que cualquier solicitud a un endpoint no especificado con `permitAll()` requerirá autenticación. Asegúrate de que las reglas de `requestMatchers` para tus endpoints de libros (`/api/v1/books/**`) sean correctas y que estén antes de `anyRequest().authenticated()`.
  * **Estructura de DTOs:** Asegúrate de que tus DTOs (`AuthLoginRequest`, `AuthCreateUserRequest`, `AuthResponse`, `RoleRequest`) estén definidos correctamente en tu paquete `com.app.controller.dto` para que coincidan con lo que espera tu `AuthController` y `UserDetailServiceImpl`.
  * **Base de Datos y Entidades:** Este código asume que ya tienes las entidades `UserEntity` y `RoleEntity` (con `RoleEnum` y `PermissionList`), y sus repositorios (`UserRepository`, `RoleRepository`) correctamente configurados con JPA.

-----

### Pasos para tu Proyecto:

1.  **Asegúrate de tener todas las Entidades y Repositorios:** `UserEntity`, `RoleEntity`, `PermissionEntity`, `UserRepository`, `RoleRepository`, y sus respectivas enumeraciones (como `RoleEnum`) si no las tienes.
2.  **Crea los DTOs:** Define `AuthLoginRequest`, `AuthCreateUserRequest`, `AuthResponse`, `RoleRequest` en tu paquete `com.app.controller.dto`.
3.  **Añade las Propiedades en `application.properties`:** Copia las dos líneas de `security.jwt.key.private` y `security.jwt.user.generator` a tu archivo de propiedades y **asegúrate de usar una clave privada fuerte y única para desarrollo**. Recuerda que en producción usarás una **variable de entorno** para esta clave.
4.  **Implementa los Archivos:** Copia el código proporcionado a las rutas de archivo correspondientes en tu proyecto (crea las carpetas si no existen, como `config/filter` y `util`).
5.  **Ajusta `SecurityConfig`:** Revisa la sección `authorizeHttpRequests` y ajusta las reglas de `requestMatchers` para tus endpoints de libros y cualquier otro endpoint que necesite protección. **No olvides el CORS si tu frontend está en un dominio/puerto diferente.**
6.  **Inicia tu Aplicación:** Inicia tu aplicación Spring Boot.

Con estos cambios, tu backend estará configurado para emitir y validar JWTs, protegiendo tus recursos de manera robusta. ¡Mucha suerte con tus entrevistas\!