# 7. Autenticación y Autorización con Spring Security 6 y JWT

En esta guía vamos a implementar login, registro y autorización por rol para autofix-api. La idea central: el token JWT carga toda la información que necesitamos (nombre, tipo de usuario, rol, y el id de empleado o cliente correspondiente), así que **ninguna petición autenticada vuelve a consultar la base de datos solo para saber quién es el usuario** - eso se resuelve leyendo los claims del propio token.

Regla de negocio confirmada para todo este módulo: **un usuario tiene un solo rol** (aunque la tabla `usuarios_roles` en el diagrama soporta muchos, en la práctica solo se asignará uno).

## 7.0 Prerrequisitos antes de escribir código

### Dependencias - agregar al `pom.xml`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT - jjwt (api en compile, impl/jackson en runtime) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

### Configuración - agregar a `application.properties`

```properties
# app.jwt.secret DEBE tener al menos 32 caracteres (256 bits) para HS256 -
# un secreto corto hace que jjwt lance una excepción al arrancar.
app.jwt.secret=CAMBIA_ESTO_por_una_cadena_aleatoria_de_al_menos_32_caracteres
# 4 horas en milisegundos. Ajusta según cuánto dure una jornada de uso típica.
app.jwt.expiration-ms=14400000
```

Genera un secreto real así:
```bash
openssl rand -base64 48
```
Copiar el hash generado como valor de la propiedad **app.jwt.secret** del archivo application.properties
### Datos - la tabla `roles` debe tener sus 4 filas antes de probar

```sql
INSERT INTO roles (nombre) VALUES ('ADMIN'), ('RECEPCIONISTA'), ('MECANICO'), ('CLIENTE');
```

Sin esto, `AuthService.registrarCliente` fallará con `404: No existe el rol 'CLIENTE'`.

## 7.1 Piezas de apoyo: repositorios que faltaban

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Usuario;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UsuarioRepository extends JpaRepository<Usuario, Integer> {
    Optional<Usuario> findByUsername(String username);

    boolean existsByUsername(String username);
}
```

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.UsuarioRole;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UsuarioRoleRepository extends JpaRepository<UsuarioRole, Integer> {
    //Un usuario -> un role
    Optional<UsuarioRole> findByUsuarioId(Integer usuarioId);

    boolean existsByUsuarioId(Integer usuarioId);
}
```

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Role;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface RoleRepository extends JpaRepository<Role, Integer> {
    Optional<Role> findByNombre(String nombre);
}
```

## 7.2 Nueva excepción: acceso a un recurso no autorizado (403)

```java
package com.devsv.autofix_api.exceptions;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

/**
 * Se lanza cuando el usuario SÍ tiene permiso para usar el endpoint (eso
 * ya lo filtró @PreAuthorize por rol), pero el recurso puntual que pide
 * no le pertenece - ej. un cliente pidiendo el vehículo de otro cliente.
 */
@ResponseStatus(value = HttpStatus.FORBIDDEN)
public class AccesoNoAutorizadoException extends RuntimeException {
    private static final long serialVersionUID = 1L;

    public AccesoNoAutorizadoException(String message) {
        super(message);
    }
}
```

Y se agrega su handler en `GlobalExceptionHandler` (junto a los que ya tienes):

```java
@ExceptionHandler(AccesoNoAutorizadoException.class)
public ResponseEntity<?> handleAccesoNoAutorizado(AccesoNoAutorizadoException ex){
    Map<String, Object> response = new HashMap<>();
    response.put("message", ex.getMessage());
    return new ResponseEntity<>(response, HttpStatus.FORBIDDEN);
}
```
## 7.3 UsuarioPrincipal - el adaptador hacia Spring Security

```java
package com.devsv.autofix_api.security;

import com.devsv.autofix_api.entities.Usuario;
import lombok.Getter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

/**
 * Envoltorio liviano sobre Usuario para satisfacer el contrato de Spring
 * Security. Se usa SOLO durante el login - @Getter de Lombok ya genera
 * getUsername()/getPassword().
 */
@Getter
public class UsuarioPrincipal implements UserDetails {

    private final Integer id;
    private final String username;
    private final String password;
    private final Boolean activo;
    private final String nombre;
    private final String tipo;        // "EMPLEADO" o "CLIENTE"
    private final Integer empleadoId;
    private final Integer clienteId;
    private final String rol;

    public UsuarioPrincipal(Usuario usuario, String nombre, String tipo, String rol) {
        this.id = usuario.getId();
        this.username = usuario.getUsername();
        this.password = usuario.getPassword();
        this.activo = usuario.getActivo();
        this.nombre = nombre;
        this.tipo = tipo;
        this.empleadoId = usuario.getEmpleado() != null ? usuario.getEmpleado().getId() : null;
        this.clienteId = usuario.getCliente() != null ? usuario.getCliente().getId() : null;
        this.rol = rol;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + rol));
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return Boolean.TRUE.equals(activo); }
}
```

## 7.4 CustomUserDetailsService

```java
package com.devsv.autofix_api.security;

import com.devsv.autofix_api.entities.Usuario;
import com.devsv.autofix_api.entities.UsuarioRole;
import com.devsv.autofix_api.repository.UsuarioRepository;
import com.devsv.autofix_api.repository.UsuarioRoleRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UsuarioRepository usuarioRepository;
    private final UsuarioRoleRepository usuarioRoleRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) {
        // Mensaje genérico a propósito: no revela si el problema fue el
        // username o el password.
        Usuario usuario = usuarioRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Usuario o contraseña incorrectos"));

        UsuarioRole usuarioRole = usuarioRoleRepository.findByUsuarioId(usuario.getId())
                .orElseThrow(() -> new UsernameNotFoundException("Usuario o contraseña incorrectos"));

        String tipo;
        String nombre;

        if (usuario.getEmpleado() != null) {
            tipo = "EMPLEADO";
            nombre = usuario.getEmpleado().getNombre();
        } else if (usuario.getCliente() != null) {
            tipo = "CLIENTE";
            nombre = usuario.getCliente().getNombre();
        } else {
            throw new UsernameNotFoundException("Usuario o contraseña incorrectos");
        }

        return new UsuarioPrincipal(usuario, nombre, tipo, usuarioRole.getRole().getNombre());
    }
}
```

## 7.5 JwtService - firmar y validar tokens

```java
package com.devsv.autofix_api.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Date;

@Service
public class JwtService {

    @Value("${app.jwt.secret}")
    private String secret;

    @Value("${app.jwt.expiration-ms}")
    private long expirationMs;

    private SecretKey obtenerLlave() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }

    public String generarToken(UsuarioPrincipal principal) {
        Date ahora = new Date();
        Date expiracion = new Date(ahora.getTime() + expirationMs);

        return Jwts.builder()
                .subject(principal.getUsername())
                .claim("nombre", principal.getNombre())
                .claim("tipo", principal.getTipo())
                .claim("empleadoId", principal.getEmpleadoId())
                .claim("clienteId", principal.getClienteId())
                .claim("rol", principal.getRol())
                .issuedAt(ahora)
                .expiration(expiracion)
                .signWith(obtenerLlave())
                .compact();
    }

    /** Lanza io.jsonwebtoken.JwtException si el token es inválido o expiró. */
    public Claims validarYExtraerClaims(String token) {
        return Jwts.parser()
                .verifyWith(obtenerLlave())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

## 7.6 AuthenticatedUser - lo que viaja en cada request ya autenticado

```java
package com.devsv.autofix_api.security;

/**
 * Se construye ÚNICAMENTE a partir de los claims del token
 * (JwtAuthenticationFilter), nunca consultando la base de datos.
 */
public record AuthenticatedUser(
        String username,
        String nombre,
        String tipo,
        Integer empleadoId,
        Integer clienteId,
        String rol
) {
}
```

## 7.7 JwtAuthenticationFilter

```java
package com.devsv.autofix_api.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                    @NonNull HttpServletResponse response,
                                    @NonNull FilterChain filterChain) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);

        try {
            Claims claims = jwtService.validarYExtraerClaims(token);

            AuthenticatedUser usuarioAutenticado = new AuthenticatedUser(
                    claims.getSubject(),
                    claims.get("nombre", String.class),
                    claims.get("tipo", String.class),
                    claims.get("empleadoId", Integer.class),
                    claims.get("clienteId", Integer.class),
                    claims.get("rol", String.class)
            );

            List<GrantedAuthority> authorities =
                    List.of(new SimpleGrantedAuthority("ROLE_" + usuarioAutenticado.rol()));

            var authentication = new UsernamePasswordAuthenticationToken(usuarioAutenticado, null, authorities);
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authentication);

        } catch (JwtException e) {
            SecurityContextHolder.clearContext();
        }

        filterChain.doFilter(request, response);
    }
}
```

## 7.8 Respuestas JSON consistentes para errores de seguridad (401 / 403)

Sin esto, Spring Security responde con su página de error genérica en vez de `{"message": ...}`.

```java
package com.devsv.autofix_api.security;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        String mensaje = "Debe iniciar sesión para acceder a este recurso";

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"message\":\"" + mensaje + "\"}");
    }
}
```

```java
package com.devsv.autofix_api.security;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                        AccessDeniedException accessDeniedException) throws IOException {
        String mensaje = "No tiene permisos suficientes para realizar esta acción";

        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"message\":\"" + mensaje + "\"}");
    }
}
```

## 7.9 SecurityConfig - donde se conecta todo

```java
package com.devsv.autofix_api.config;

import com.devsv.autofix_api.security.CustomUserDetailsService;
import com.devsv.autofix_api.security.JwtAccessDeniedHandler;
import com.devsv.autofix_api.security.JwtAuthenticationEntryPoint;
import com.devsv.autofix_api.security.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity // habilita @PreAuthorize en los Controllers
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAccessDeniedHandler jwtAccessDeniedHandler;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    // Lo usa AuthService para ejecutar authenticationManager.authenticate(...) en el login
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(csrf -> csrf.disable()) // API stateless con JWT - no hay sesión que un CSRF pueda secuestrar
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        // Solo estas dos rutas son públicas - /api/auth/registro-empleado
                        // NO se incluye aquí a propósito: además de @PreAuthorize("hasRole('ADMIN')")
                        // en el Controller, así también exige estar autenticado a nivel
                        // de filtro (defensa en profundidad).
                        .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
                        .anyRequest().authenticated()
                )
                .authenticationProvider(authenticationProvider())
                .exceptionHandling(ex -> ex
                        .authenticationEntryPoint(jwtAuthenticationEntryPoint) // 401
                        .accessDeniedHandler(jwtAccessDeniedHandler)           // 403
                )
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

## 7.10 DTOs de autenticación

```java
package com.devsv.autofix_api.dto.auth;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class LoginRequestDTO {
    private String username;

    private String password;
}
```

```java
package com.devsv.autofix_api.dto.auth;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@AllArgsConstructor
public class LoginResponseDTO {
    private String token;

    private String username;

    private String nombre;

    private String tipo;

    private String role;
}
```

```java
package com.devsv.autofix_api.dto.auth;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class RegistroClienteDTO {
    private String username;
    private String password;

    private String nombre;
    private String telefono;
    private String email;
}
```

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class RegistroEmpleadoDTO {
    // Datos de acceso
    private String username;
    private String password;
    private String role; // "ADMIN", "RECEPCIONISTA" o "MECANICO" - nunca "CLIENTE" aquí

    // Datos del Empleado - se crea junto con el Usuario, en la misma transacción
    private String nombre;
    private String email;
    private String telefono;
}
```

## 7.11 IAuthService y AuthService

```java
package com.devsv.autofix_api.interfaces;


import com.devsv.autofix_api.dto.RegistroEmpleadoDTO;
import com.devsv.autofix_api.dto.auth.LoginRequestDTO;
import com.devsv.autofix_api.dto.auth.LoginResponseDTO;
import com.devsv.autofix_api.dto.auth.RegistroClienteDTO;

public interface IAuthService {
    LoginResponseDTO login(LoginRequestDTO dto);

    LoginResponseDTO registrarCliente(RegistroClienteDTO dto);

    LoginResponseDTO registrarEmpleado(RegistroEmpleadoDTO dto);
}
```

```java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.RegistroEmpleadoDTO;
import com.devsv.autofix_api.dto.auth.LoginRequestDTO;
import com.devsv.autofix_api.dto.auth.LoginResponseDTO;
import com.devsv.autofix_api.dto.auth.RegistroClienteDTO;
import com.devsv.autofix_api.entities.Cliente;
import com.devsv.autofix_api.entities.Empleado;
import com.devsv.autofix_api.entities.Role;
import com.devsv.autofix_api.entities.Usuario;
import com.devsv.autofix_api.entities.UsuarioRole;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ConflictException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IAuthService;
import com.devsv.autofix_api.repository.ClienteRepository;
import com.devsv.autofix_api.repository.EmpleadoRepository;
import com.devsv.autofix_api.repository.RoleRepository;
import com.devsv.autofix_api.repository.UsuarioRepository;
import com.devsv.autofix_api.repository.UsuarioRoleRepository;
import com.devsv.autofix_api.security.JwtService;
import com.devsv.autofix_api.security.UsuarioPrincipal;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Set;

@Service
@RequiredArgsConstructor
public class AuthService implements IAuthService {

    private static final String ROLE_CLIENTE = "CLIENTE";
    private static final Set<String> ROLES_EMPLEADO = Set.of("ADMIN", "RECEPCIONISTA", "MECANICO");

    private final AuthenticationManager authenticationManager;
    private final UsuarioRepository usuarioRepository;
    private final UsuarioRoleRepository usuarioRoleRepository;
    private final ClienteRepository clienteRepository;
    private final EmpleadoRepository empleadoRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;

    @Override
    public LoginResponseDTO login(LoginRequestDTO dto) {
        Authentication authentication;
        try {
            authentication = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(dto.getUsername(), dto.getPassword()));
        } catch (BadCredentialsException e) {
            throw new BadRequestException("Usuario o contraseña incorrectos");
        }

        UsuarioPrincipal principal = (UsuarioPrincipal) authentication.getPrincipal();
        String token = jwtService.generarToken(principal);

        return new LoginResponseDTO(token, principal.getUsername(), principal.getNombre(),
                principal.getTipo(), principal.getRol());
    }

    @Override
    @Transactional
    public LoginResponseDTO registrarCliente(RegistroClienteDTO dto) {
        validarCredencialesNuevas(dto.getUsername(), dto.getPassword());

        // 1. creamos el Cliente
        Cliente cliente = new Cliente();
        cliente.setNombre(dto.getNombre());
        cliente.setTelefono(dto.getTelefono());
        cliente.setEmail(dto.getEmail());
        cliente = clienteRepository.save(cliente);

        // 2. creamos el Usuario vinculado a ESE Cliente - "empleado" queda en
        //    null a propósito: un usuario de autoregistro público NUNCA se
        //    vincula a un Empleado (esa es la regla "cliente XOR empleado").
        Usuario usuario = new Usuario();
        usuario.setUsername(dto.getUsername());
        usuario.setPassword(passwordEncoder.encode(dto.getPassword()));
        usuario.setActivo(true);
        usuario.setCliente(cliente);
        usuario = usuarioRepository.save(usuario);

        // 3. rol fijo: CLIENTE (no lo elige quien se autoregistra)
        asignarRole(usuario, ROLE_CLIENTE);

        return generarRespuestaConToken(usuario, cliente.getNombre(), "CLIENTE", ROLE_CLIENTE);
    }

    @Override
    @Transactional
    public LoginResponseDTO registrarEmpleado(RegistroEmpleadoDTO dto) {
        validarCredencialesNuevas(dto.getUsername(), dto.getPassword());

        if (dto.getRole() == null || dto.getRole().isBlank()) {
            throw new BadRequestException("Debe indicar el rol del empleado (ADMIN, RECEPCIONISTA o MECANICO)");
        }
        if (!ROLES_EMPLEADO.contains(dto.getRole())) {
            throw new BadRequestException(
                    "Rol inválido para un empleado. Valores permitidos: ADMIN, RECEPCIONISTA, MECANICO");
        }

        // 1. creamos el Empleado
        Empleado empleado = new Empleado();
        empleado.setNombre(dto.getNombre());
        empleado.setEmail(dto.getEmail());
        empleado.setTelefono(dto.getTelefono());
        empleado = empleadoRepository.save(empleado);

        // 2. creamos el Usuario vinculado a ESE Empleado - "cliente" queda en
        //    null a propósito, misma regla "cliente XOR empleado" que arriba.
        Usuario usuario = new Usuario();
        usuario.setUsername(dto.getUsername());
        usuario.setPassword(passwordEncoder.encode(dto.getPassword()));
        usuario.setActivo(true);
        usuario.setEmpleado(empleado);
        usuario = usuarioRepository.save(usuario);

        // 3. rol elegido por el ADMIN que está registrando (validado arriba)
        asignarRole(usuario, dto.getRole());

        return generarRespuestaConToken(usuario, empleado.getNombre(), "EMPLEADO", dto.getRole());
    }

    private void validarCredencialesNuevas(String username, String password) {
        if (username == null || username.isBlank()) {
            throw new BadRequestException("El username es obligatorio");
        }
        if (password == null || password.length() < 6) {
            throw new BadRequestException("La contraseña debe tener al menos 6 caracteres");
        }
        if (usuarioRepository.existsByUsername(username)) {
            throw new ConflictException("Ya existe un usuario con el username '" + username + "'");
        }
    }

    private void asignarRole(Usuario usuario, String nombreRole) {
        Role role = roleRepository.findByNombre(nombreRole)
                .orElseThrow(() -> new ResourceNotFoundException(
                        "No existe el rol '" + nombreRole + "' - debe crearse primero en la tabla roles"));

        UsuarioRole usuarioRole = new UsuarioRole();
        usuarioRole.setUsuario(usuario);
        usuarioRole.setRole(role);
        usuarioRoleRepository.save(usuarioRole);
    }

    /** Deja a la persona logueada de una vez, justo después de registrarse. */
    private LoginResponseDTO generarRespuestaConToken(Usuario usuario, String nombre, String tipo, String rol) {
        UsuarioPrincipal principal = new UsuarioPrincipal(usuario, nombre, tipo, rol);
        String token = jwtService.generarToken(principal);
        return new LoginResponseDTO(token, usuario.getUsername(), nombre, tipo, rol);
    }
}
```

## 7.12 AuthController

```java
package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.dto.RegistroEmpleadoDTO;
import com.devsv.autofix_api.dto.auth.LoginRequestDTO;
import com.devsv.autofix_api.dto.auth.LoginResponseDTO;
import com.devsv.autofix_api.dto.auth.RegistroClienteDTO;
import com.devsv.autofix_api.interfaces.IAuthService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@RestController
@CrossOrigin
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    private final IAuthService authService;

    //login para cualquier tipo de usuario (empleado o cliente) - público
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequestDTO dto) {
        Map<String, Object> response = new HashMap<>();

        LoginResponseDTO resultado = authService.login(dto);

        response.put("message", "Inicio de sesión exitoso");
        response.put("auth", resultado);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    //autoregistro público, exclusivo para clientes
    @PostMapping("/register")
    public ResponseEntity<?> registro(@RequestBody RegistroClienteDTO dto) {
        Map<String, Object> response = new HashMap<>();

        LoginResponseDTO resultado = authService.registrarCliente(dto);

        response.put("message", "Registro exitoso...!");
        response.put("auth", resultado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    //registro de cuentas de EMPLEADO - protegido, solo un ADMIN puede crearlas
    @PreAuthorize("hasRole('ADMIN')")
    @PostMapping("/registro-empleado")
    public ResponseEntity<?> registroEmpleado(@RequestBody RegistroEmpleadoDTO dto) {
        Map<String, Object> response = new HashMap<>();

        LoginResponseDTO resultado = authService.registrarEmpleado(dto);

        response.put("message", "Empleado registrado correctamente...!");
        response.put("auth", resultado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }
}
```

## 7.13 Aplicar autorización en un Controller existente (ejemplo con Vehiculo)

Dos capas de autorización trabajando juntas: `@PreAuthorize` por role, y una verificación manual para el caso del cliente.

```java
 @GetMapping("/vehiculos")
    public ResponseEntity<List<VehiculoDTO>> getAll(@RequestParam(required = false) Integer clienteId,
                                                    @AuthenticationPrincipal AuthenticatedUser usuarioAutenticado) {
        if ("CLIENTE".equals(usuarioAutenticado.tipo())) {
            if (clienteId == null || !clienteId.equals(usuarioAutenticado.clienteId())) {
                throw new AccesoNoAutorizadoException("Solo puede consultar sus propios vehículos");
            }
        }

        if (clienteId != null) {
            return ResponseEntity.ok(vehiculoService.findByClienteId(clienteId));
        }
        return ResponseEntity.ok(vehiculoService.findAll());
    }

    @GetMapping("/vehiculos/{id}")
    public ResponseEntity<VehiculoDTO> getById(@PathVariable Integer id) {
        return ResponseEntity.ok(vehiculoService.findById(id));
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'RECEPCIONISTA')")
    @PostMapping("/vehiculos")
    public ResponseEntity<?> create(@RequestBody VehiculoDTO dto) {
        Map<String, Object> response = new HashMap<>();

        VehiculoDTO guardado = vehiculoService.saveOrUpdate(dto);

        response.put("message", "Vehículo registrado correctamente...!");
        response.put("vehiculo", guardado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'RECEPCIONISTA')")
    @PutMapping("/vehiculos/{id}")
    public ResponseEntity<?> update(@PathVariable Integer id, @RequestBody VehiculoDTO dto) {
        Map<String, Object> response = new HashMap<>();

        dto.setId(id);
        VehiculoDTO actualizado = vehiculoService.saveOrUpdate(dto);

        response.put("message", "Vehículo actualizado correctamente");
        response.put("vehiculo", actualizado);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/vehiculos/{id}")
    public ResponseEntity<?> delete(@PathVariable Integer id) {
        vehiculoService.delete(id);

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Vehículo eliminado con éxito");

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
```

El mismo patrón aplica al resto de tus Controllers: `@PreAuthorize` para "¿tiene el role correcto?", y una comparación manual contra `AuthenticatedUser` cuando la pregunta es "¿es su propio dato?". Hágalo para los demás controladores de acuerdo a la siguiente tabla:

| Controller | GET | POST | PUT | PATCH estado | DELETE |
|---|---|---|---|---|---|
| `MarcaController` | sin restricción | `ADMIN`, `RECEPCIONISTA` | `ADMIN`, `RECEPCIONISTA` | — | `ADMIN` |
| `ModeloController` | sin restricción | `ADMIN`, `RECEPCIONISTA` | `ADMIN`, `RECEPCIONISTA` | — | `ADMIN` |
| `RepuestoServicioController` | sin restricción | `ADMIN`, `RECEPCIONISTA` | `ADMIN`, `RECEPCIONISTA` | — | `ADMIN` |
| `VehiculoController` | *(ya está definido, en este punto)* | `ADMIN`, `RECEPCIONISTA` | `ADMIN`, `RECEPCIONISTA` | — | `ADMIN` |
| `OrdenTrabajoController` | sin restricción | `ADMIN`, `RECEPCIONISTA` | `ADMIN`, `RECEPCIONISTA` | `ADMIN`, `RECEPCIONISTA`, `MECANICO` | `ADMIN` |

## 7.14 Registrar el primer ADMIN del sistema
### 7.14.1 Crear una clase de prueba para generar el Hash que se asignará al campo password
En el package test, package principal **autofix_api**, crear la siguiente clase:

```java
package com.devsv.autofix_api;

import org.junit.jupiter.api.Test;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

public class GenerarHashTest {

    @Test
    void generarHash(){
        String passwordPlano = "admin123";
        String hash = new BCryptPasswordEncoder().encode(passwordPlano);
        System.out.println("Hash generado: " + hash);

        boolean coincide = new BCryptPasswordEncoder().matches(passwordPlano, hash);
        System.out.println("Coincide? " +coincide);
    }
}
```
Corre la clase de prueba en el ícono de play que esta a la par, observa el hash generado
### 7.14.2 En una sesión de Query Tools de PgAdmin 4, inserta un empleado
```sql
INSERT INTO empleados(nombre, email, telefono) 
VALUES('Administrador del Sistema', 'admin@autofix.com','0000-0000');
```
### 7.14.3 Inserta el usuario, copia el Hash del punto 7.14.1 para el valor del password
```sql
INSERT INTO usuarios(username, password,activo,empleado_id) 
VALUES('admin','$2a$10$qogS/89f/5HKv1zMvrkf6eHDwToojpmTf6aBJmvbp.SV17ef6AEH6',true,2);
```
El valor del empleado_id, depende de que número le haya asignado cuando insertó al empleado

### 7.14.3 asignar el role al usuario
```sql
INSERT INTO usuarios_roles(usuario_id, role_id)
VALUES(
	(SELECT id FROM usuarios WHERE username='admin'),
	(SELECT id FROM roles WHERE nombre='ADMIN')
)
```

## 7.15 Probar en Postman

**1. Hacer login con el usuario admin**
`POST http://localhost:8080/api/auth/login`
```json
{
  "username": "admin",
  "password": "admin123"
}
```
Respuesta (`200 Ok`):
```json
{
    "auth": {
        "token": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsIm5vbWJyZSI6IkFkbWluaXN0cmFkb3IgZGVsIFNpc3RlbWEiLCJ0aXBvIjoiRU1QTEVBRE8iLCJlbXBsZWFkb0lkIjoyLCJyb2wiOiJBRE1JTiIsImlhdCI6MTc4Njk5MTM4MSwiZXhwIjoxNzg3MDA1NzgxfQ.ogoZbefvtC6E1ckAPlRL220ZKa3M-ox9iPSesWvXJXzesbGUwNmjheoo6fkficMWteKhWkbPscrdW4KAp3kLKw",
        "username": "admin",
        "nombre": "Administrador del Sistema",
        "tipo": "EMPLEADO",
        "role": "ADMIN"
    },
    "message": "Inicio de sesión exitoso"
}

**1. Registro de un cliente**

`POST http://localhost:8080/api/auth/register`
```json
{
  "username": "cperez",
  "password": "clave123",
  "nombre": "Carlos Pérez",
  "telefono": "7000-1234",
  "email": "carlos@correo.com"
}
```
Respuesta (`201 Created`):
```json
{
  "message": "Registro exitoso...!",
  "auth": {
    "token": "eyJhbGciOiJIUzI1NiJ9...",
    "username": "cperez",
    "nombre": "Carlos Pérez",
    "tipo": "CLIENTE",
    "role": "CLIENTE"
  }
}
```
Copie el `token` — lo necesitas para todo lo demás.
