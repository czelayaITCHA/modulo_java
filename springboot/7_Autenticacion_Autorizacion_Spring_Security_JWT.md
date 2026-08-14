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
