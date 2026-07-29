# 2. Programar CRUD para una tabla independiente
En esta guía vamos a desarrollar los componentes para crear un CRUD para la tabla mascas, entidad **Marca** que ya fue creada, en la clase y guía anterior

## 2.1 Agregar dependencia de **MapStruct** para hacer conversiones de entidades a DTO y viseversa
Editamos el archivo pom.xml, en la sección de **properties**, justo después de la versión de java, agregamos esta linea
```xml
<org.mapstruct.version>1.6.3</org.mapstruct.version>
```

luego en la sección de **dependencies** justamente antes de </dependencies>, agregamos la dependencia de MapStruct

```xml
<dependency>
			<groupId>org.mapstruct</groupId>
			<artifactId>mapstruct</artifactId>
			<version>${org.mapstruct.version}</version>
		</dependency>
```
Luego compilamos el proyecto con maven

```bash
.\mvnw.cmd clean install -U
```

## 2.2 Crear paquete de **exceptions** y crear las siguientes clases
* BadRequestException

```java
package com.devsv.autofix_api.exceptions;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value = HttpStatus.BAD_REQUEST)
public class BadRequestException extends RuntimeException{
    public BadRequestException(String message){ super(message);}
}

```
* 
```java
package com.devsv.autofix_api.exceptions;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value = HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException{
    private static final long serialVersionUID = 1L;

    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```
* GlobalExceptionHadler
```java
package com.devsv.autofix_api.exceptions;

import org.springframework.dao.DataAccessException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // 1. Maneja errores de validacion de los campos, en caso de usar
    // @NotBlank, @Size, entre otras
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleValidationError(MethodArgumentNotValidException ex) {
        Map<String, Object> response = new HashMap<>();
        Map<String, String> errors = new HashMap<>();

        ex.getBindingResult().getFieldErrors().forEach(err ->{
           errors.put(err.getField(), err.getDefaultMessage());
        });
        response.put("message", "Error de validación en los campos enviados");
        response.put("errors", errors);
        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }

    // 2. para recursos no encontrados
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<?> handleResourceNotFound(ResourceNotFoundException ex){
        Map<String, Object> response = new HashMap<>();
        response.put("message", ex.getMessage());
        return new ResponseEntity<>(response, HttpStatus.NOT_FOUND);
    }

    // 3. Manejo de peticiones incorrectas o lógica de negocios fallida
    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<?> handleBadRequest(BadRequestException ex){
        Map<String, Object> response = new HashMap<>();
        response.put("message", ex.getMessage());
        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }
    
    // 4. Manejo de errores provenientes de la base de datos
    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<?> handleDbError(DataAccessException ex){
        Map<String, Object> response = new HashMap<>();
        response.put("message", "Error al realizar la operación en la base datos");
        response.put("error", ex.getMessage());
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```
## 2.3 Crear la clase DTO

## 2.4 Crear repository 

## 2.5 Definir crear interface y definir métodos a implementar para crear las funcionalidades
## 2.6 Programar el Service
## 2.7 Programar el Controller
## 2.8 Probar en Postman
