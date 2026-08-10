# 4. Programar CRUD para una tabla dependiente
En esta guía vamos a desarrollar los componentes para crear un CRUD para la tabla `modelos`, entidad **Modelo** que ya fue creada. A diferencia de `Marca`, `Modelo` **depende** de que exista una `Marca` (llave foránea obligatoria) — cómo se resuelve esa relación al guardar es la parte nueva de este capítulo. El nombre del modelo sigue siendo único de forma **global** (igual que en `Marca`), sin importar a qué marca pertenezca. También agregamos un endpoint de listado filtrado por marca, pensado para un `<select>` dependiente en el frontend.

## 4.1 Crear la clase DTO

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class ModeloDTO {
    private Integer id;

    private String nombre;

    // Para CREAR/ACTUALIZAR: el frontend solo manda el id de la marca
    // (viene de un <select> cargado antes desde /api/marcas)
    private Integer marcaId;

    // Para LEER: objeto completo, reutilizando MarcaDTO ya existente
    private MarcaDTO marca;
}
```

## 4.2 Crear el mapper para convertir de entity a dto y viceversa

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.ModeloDTO;
import com.devsv.autofix_api.entities.Modelo;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.ReportingPolicy;

import java.util.List;

// "uses = MarcaMapper.class" reutiliza el mapper de Marca ya existente
// para convertir entity.getMarca() en el MarcaDTO anidado, sin duplicar esa lógica.
@Mapper(componentModel = "spring", uses = {MarcaMapper.class}, unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface ModeloMapper {

    @Mapping(target = "marcaId", source = "marca.id")
    ModeloDTO toDTO(Modelo entity);

    // "marca" queda en null a propósito: el mapper no puede consultar la BD
    // para resolver marcaId -> Marca. Esa resolución la hace el Service.
    @Mapping(target = "marca", ignore = true)
    Modelo toEntity(ModeloDTO dto);

    List<ModeloDTO> toDtoList(List<Modelo> entities);
}
```
## 4.3 Crear los repository

Necesitamos dos: el propio de `Modelo`, y uno mínimo de `Vehiculo` para poder bloquear la eliminación de un modelo que ya esté asignado a algún vehículo.

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Modelo;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface ModeloRepository extends JpaRepository<Modelo, Integer> {
    boolean existsByNombre(String nombre);
    boolean existsByNombreAndIdNot(String nombre, Integer id);

    // Para el <select> dependiente en el frontend: al elegir una marca,
    // se cargan solo los modelos de esa marca (ej. GET /api/modelos?marcaId=1)
    List<Modelo> findByMarcaId(Integer marcaId);
}
```

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Vehiculo;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface VehiculoRepository extends JpaRepository<Vehiculo, Integer> {
    // Se usa para bloquear la eliminación de un Modelo que ya está
    // asignado a algún vehículo registrado.
    boolean existsByModeloId(Integer modeloId);
}
```
## 4.4 Crear interface y definir métodos a implementar para crear las funcionalidades

```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.ModeloDTO;

import java.util.List;

public interface IModeloService {
    //definición del método para obtener todos los modelos
    List<ModeloDTO> findAll();

    //método para obtener un Modelo por su id
    ModeloDTO findById(Integer id);

    //método para listar los modelos de una marca específica
    List<ModeloDTO> findByMarcaId(Integer marcaId);

    //método para guardar/actualizar un modelo
    ModeloDTO save(ModeloDTO dto);

    //método para eliminar un modelo
    void delete(Integer id);
}
```
## 4.5 Crear clase de Excepción para gestionar mensajes de conflicto

Adicional a los otras clases de gestión de excepciones, vamos a crear la clase **ConflictException** para gestionar mensajes de conflictos y devolver un 409

```java
package com.devsv.autofix_api.exceptions;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value = HttpStatus.CONFLICT)
public class ConflictException extends RuntimeException{
    private static final long serialVersionUID = 1L;

    public ConflictException(String message) {
        super(message);
    }
}
```  
En la clase **GlobalExceptionHandler**, adicionar este método:

```Java
// 5. manejo de excepciones de conflicto
    @ExceptionHandler(ConflictException.class)
    public ResponseEntity<?> handleConflictRequest(ConflictException ex){
        Map<String, Object> response = new HashMap<>();
        response.put("message", ex.getMessage());
        return new ResponseEntity<>(response, HttpStatus.CONFLICT);
    }
```

## 4.6 Programar el Service

```java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.ModeloDTO;
import com.devsv.autofix_api.entities.Marca;
import com.devsv.autofix_api.entities.Modelo;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ConflictException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IModeloService;
import com.devsv.autofix_api.mappers.ModeloMapper;
import com.devsv.autofix_api.repository.MarcaRepository;
import com.devsv.autofix_api.repository.ModeloRepository;
import com.devsv.autofix_api.repository.VehiculoRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ModeloService implements IModeloService {

    private final ModeloRepository repository;
    private final VehiculoRepository vehiculoRepository;
    private final MarcaRepository marcaRepository;
    private final ModeloMapper mapper;

    @Override
    @Transactional(readOnly = true)
    public List<ModeloDTO> findAll() {
        return mapper.toDtoList(repository.findAll());
    }

    @Override
    @Transactional(readOnly = true)
    public ModeloDTO findById(Integer id) {
        Modelo entity = buscarEntidad(id);
        return mapper.toDTO(entity);
    }

    @Override
    @Transactional(readOnly = true)
    public List<ModeloDTO> findByMarcaId(Integer marcaId) {
        return mapper.toDtoList(repository.findByMarcaId(marcaId));
    }

    @Override
    @Transactional
    public ModeloDTO saveOrUpdate(ModeloDTO dto) {
        //validamos que la marca asociada al modelo exista
        Marca marca = buscarMarca(dto.getMarcaId());
        //declaramos un objeto modelo
        Modelo entity;
        //evaluamos si es una inserción o actualización del modelo
        if(dto.getId() == null){
            //validamos que no se duplique el nombre del modelo
            if(repository.existsByNombre(dto.getNombre())){
                throw new ConflictException("Ya existe un modelo con el nombre: " + dto.getNombre());
            }
            entity = new Modelo();
        }else{
            //evitamos que se duplique el nombre del modelo por actualización
            entity = buscarEntidad(dto.getId());
            if(repository.existsByNombreAndIdNot(dto.getNombre(), dto.getId())){
                throw new ConflictException("Ya existe otro modelo con el nombre: " + dto.getNombre());
            }
        }
        //seteamos los valores a entity
        entity.setNombre(dto.getNombre());
        entity.setMarca(marca);
        return mapper.toDTO(repository.save(entity));
    }

    @Override
    @Transactional
    public void delete(Integer id) {
        //obtenemos el modelo
        Modelo entity = buscarEntidad(id);
        //Comprobamos si existen vehiculos asociados para no eliminar
        if(vehiculoRepository.existsByModeloId(id)){
            throw new BadRequestException("No se puede eliminar el modelo "
                    + entity.getNombre() + " porque ya esta asignado a uno o mas vehículos");
        }
        repository.delete(entity);
    }

    //métodos auxiliares para buscar entidades
    private Modelo buscarEntidad(Integer id){
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el modelo con ID: " + id));
    }

    private Marca buscarMarca(Integer marcaId){
        if(marcaId == null){
            throw new BadRequestException("Debe especificar una marca del modelo");
        }
        return marcaRepository.findById(marcaId)
                .orElseThrow(() -> new ResourceNotFoundException("No existe la marca con ID: " + marcaId));
    }
}
```

**Nota sobre el uso de `Optional`:** cada `findById` de un repository (`ModeloRepository`, `MarcaRepository`) devuelve `Optional<T>`. En todo el archivo se desempaca siempre con `.orElseThrow(...)`, nunca con `.get()` a ciegas — así cualquier id inválido termina en una `ResourceNotFoundException` con un mensaje claro, en vez de un `NoSuchElementException` genérico o, peor, un `null` silencioso más adelante en el código.

## 4.7 Programar el Controller

`Modelo` vuelve a ser JSON normal — mismo patrón que `MarcaController`, con el agregado del filtro opcional por marca.

```java
package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.dto.ModeloDTO;
import com.devsv.autofix_api.interfaces.IModeloService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@CrossOrigin
@RequestMapping("/api")
@RequiredArgsConstructor
public class ModeloController {
    private final IModeloService modeloService;

    //creamos el endpoint para obtener todos los modelos, o filtrados por marca
    @GetMapping("/modelos")
    public ResponseEntity<List<ModeloDTO>> getAll(@RequestParam(required = false) Integer marcaId) {
        if (marcaId != null) {
            return ResponseEntity.ok(modeloService.findByMarcaId(marcaId));
        }
        return ResponseEntity.ok(modeloService.findAll());
    }

    //endpoint para obtener un modelo
    @GetMapping("/modelos/{id}")
    public ResponseEntity<ModeloDTO> getById(@PathVariable Integer id) {
        return ResponseEntity.ok(modeloService.findById(id));
    }

    //endpoint para crear o registrar un modelo 
    @PostMapping("/modelos")
    public ResponseEntity<?> create(@RequestBody ModeloDTO dto) {
        Map<String, Object> response = new HashMap<>();

        ModeloDTO guardado = modeloService.save(dto);

        response.put("message", "Modelo registrado correctamente...!");
        response.put("modelo", guardado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    @PutMapping("/modelos/{id}")
    public ResponseEntity<?> update(@PathVariable Integer id, @RequestBody ModeloDTO dto) {
        Map<String, Object> response = new HashMap<>();

        dto.setId(id);
        ModeloDTO actualizado = modeloService.save(dto);

        response.put("message", "Modelo actualizado correctamente");
        response.put("modelo", actualizado);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @DeleteMapping("/modelos/{id}")
    public ResponseEntity<?> delete(@PathVariable Integer id) {
        modeloService.delete(id);

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Modelo eliminado con éxito");

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

## 4.8 Probar en Postman

Antes de empezar, crear al menos dos marcas por su propio endpoint (ej. `Toyota` id=1, `Nissan` id=2) y ajusta los `marcaId` de abajo según lo que devuelva su base.

**1. Listar todos**

`GET http://localhost:8080/api/modelos` — sin body.

**2. Filtrar por marca**

`GET http://localhost:8080/api/modelos?marcaId=1`

```json
[
    {
        "id": 1,
        "marca": {
            "id": 1,
            "nombre": "Toyota"
        },
        "marcaId": 1,
        "nombre": "Corolla"
    },
    {
        "id": 2,
        "marca": {
            "id": 1,
            "nombre": "Toyota"
        },
        "marcaId": 1,
        "nombre": "Yaris"
    },
    {
        "id": 3,
        "marca": {
            "id": 1,
            "nombre": "Toyota"
        },
        "marcaId": 1,
        "nombre": "4Runner"
    }
]
```

**3. Filtrar por marca inexistente**

`GET http://localhost:8080/api/modelos?marcaId=999` → `404 Not Found`:
```json
{ "message": "No existe la marca con ID: 999" }
```

**4. Obtener uno**

`GET http://localhost:8080/api/modelos/1` → `200 OK`.
`GET http://localhost:8080/api/modelos/999` → `404 Not Found`:
```json
{ "message": "No existe el modelo con ID: 999" }
```

**5. Crear (éxito)**

`POST http://localhost:8080/api/modelos`
```json
{ "nombre": "Corolla", "marcaId": 1 }
```
Respuesta (`201 Created`):
```json
{
  "message": "Modelo registrado correctamente...!",
  "modelo": {
    "id": 1,
    "nombre": "Corolla",
    "marcaId": 1,
    "marca": { "id": 1, "nombre": "Toyota" }
  }
}
```

**6. Crear — nombre duplicado (global)**

Repite el JSON del punto 5 → `400 Bad Request`:
```json
{ "message": "Ya existe un modelo con el nombre 'Corolla'" }
```

**7. Crear — `marcaId` inexistente**

```json
{ "nombre": "Rouge", "marcaId": 999 }
```
→ `404 Not Found`:
```json
{ "message": "No existe la marca con ID: 999" }
```

**9. Crear — sin `marcaId`**

```json
{ "nombre": "Rouge" }
```
→ `400 Bad Request`:
```json
{ "message": "Debe indicar la marca del modelo" }
```

**10. Actualizar (éxito)**

`PUT http://localhost:8080/api/modelos/1`
```json
{ "nombre": "Corolla Hybrid", "marcaId": 1 }
```
→ `200 OK` con el modelo actualizado.

**11. Actualizar generando duplicado con otro registro**

Si ya existe otro modelo llamado `Rouge` (marca 2) y tratas de renombrar el modelo 1 a ese mismo nombre:

`PUT http://localhost:8080/api/modelos/1`
```json
{ "nombre": "Rouge", "marcaId": 1 }
```
→ `400 Bad Request`:
```json
{ "message": "Ya existe otro modelo con el nombre 'Rouge'" }
```

**12. Eliminar (éxito, sin vehículos asignados)**

`DELETE http://localhost:8080/api/modelos/1` → `200 OK`:
```json
{ "message": "Modelo eliminado con éxito" }
```

**13. Eliminar — bloqueado por tener vehículos asignados**

1. Crea un `Vehiculo` con `modeloId: 1` (cuando tengas ese CRUD armado).
2. `DELETE http://localhost:8080/api/modelos/1` → `400 Bad Request`:
```json
{ "message": "No se puede eliminar el modelo 'Corolla Hybrid' porque ya está asignado a uno o más vehículos" }
```

