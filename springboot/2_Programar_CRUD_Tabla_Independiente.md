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

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class MarcaDTO {
    private Integer id;

    private String nombre;
}
```

## 2.4 Crear repository 
```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Marca;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MarcaRepository extends JpaRepository<Marca, Integer> {
    boolean existsByNombre(String nombre);
    boolean existsByNombreAndIdNot(String nombre, Integer id);
}
```
## 2.5 Crear el mapper para convertir de entity a dto y viseversa
```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.MarcaDTO;
import com.devsv.autofix_api.entities.Marca;
import org.mapstruct.Mapper;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface MarcaMapper {
    MarcaDTO toDTO(Marca marca);
    
    Marca toEntity(MarcaDTO dto);
    
    List<MarcaDTO> toDtoList(List<Marca> entities);
}

```

## 2.6 Definir crear interface y definir métodos a implementar para crear las funcionalidades
```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.MarcaDTO;

import java.util.List;

public interface IMarcaService {
    //definición del método para obtener todas las marcas
    List<MarcaDTO> findAll();
    
    //método para obtener una Marca por su id
    MarcaDTO findById(Integer id);
    
    //método para guardar/actualizar una marca
    MarcaDTO save(MarcaDTO dto);
    
    //método para eliminar una marca
    void delete(Integer id);
}
```
## 2.7 Programar el Service
```java

```

## 2.8 Programar el Controller
## 2.9 Probar en Postman
