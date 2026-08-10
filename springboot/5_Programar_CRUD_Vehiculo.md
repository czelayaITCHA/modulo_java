# 5. Programar CRUD para Vehiculo (depende de dos entidades)

En esta guía vamos a desarrollar los componentes para crear un CRUD para la tabla `vehiculos`, entidad **Vehiculo** que ya fue creada. Es el caso más completo hasta ahora: depende de **dos** llaves foráneas obligatorias (`Modelo` y `Cliente`), valida que la placa no se repita (única de forma global) usando `ConflictException` (409), y protege el borrado si el vehículo ya tiene historial de órdenes de trabajo.

## 5.1 Crear el DTO y mapper de Cliente (no existían todavía)

`Vehiculo` necesita mostrar el `Cliente` propietario anidado en las respuestas, y ese DTO aún no se había creado.

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class ClienteDTO {
    private Integer id;

    private String nombre;

    private String telefono;

    private String email;
}
```

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.ClienteDTO;
import com.devsv.autofix_api.entities.Cliente;
import org.mapstruct.Mapper;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface ClienteMapper {
    ClienteDTO toDTO(Cliente entity);

    Cliente toEntity(ClienteDTO dto);

    List<ClienteDTO> toDtoList(List<Cliente> entities);
}
```

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Cliente;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ClienteRepository extends JpaRepository<Cliente, Integer> {
}
```

*(Este `ClienteRepository` es el mínimo necesario para que `VehiculoService` pueda resolver `clienteId`. El CRUD completo de `Cliente` — su propio Service/Controller — se arma cuando lo necesites por separado, siguiendo este mismo patrón.)*

## 5.2 Crear el DTO de Vehiculo

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class VehiculoDTO {
    private Integer id;

    private String placa;

    private Integer anio;

    private String caracteristicas;

    // Para CREAR/ACTUALIZAR: el frontend solo manda los ids
    // (vienen de <select> cargados desde /api/modelos y /api/clientes)
    private Integer modeloId;
    private Integer clienteId;

    // Para LEER: objetos completos, reutilizando los DTO ya existentes
    private ModeloDTO modelo;
    private ClienteDTO cliente;
}
```

## 5.3 Crear el mapper de Vehiculo

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.VehiculoDTO;
import com.devsv.autofix_api.entities.Vehiculo;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", uses = {ModeloMapper.class, ClienteMapper.class},
        unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface VehiculoMapper {

    @Mapping(target = "modeloId", source = "modelo.id")
    @Mapping(target = "clienteId", source = "cliente.id")
    VehiculoDTO toDTO(Vehiculo entity);

    // "modelo" y "cliente" quedan en null a propósito: el mapper no puede
    // consultar la BD para resolver esos ids. Esa resolución la hace el Service.
    @Mapping(target = "modelo", ignore = true)
    @Mapping(target = "cliente", ignore = true)
    Vehiculo toEntity(VehiculoDTO dto);

    List<VehiculoDTO> toDtoList(List<Vehiculo> entities);
}
```

## 5.4 Actualizar/crear los repository

`VehiculoRepository` ya existía (se creó cuando armamos la protección de borrado de `Modelo`) — ahora se completa con la validación de placa y el filtro por cliente.

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Vehiculo;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface VehiculoRepository extends JpaRepository<Vehiculo, Integer> {
    boolean existsByModeloId(Integer modeloId);

    // Validación de placa duplicada (única de forma global)
    boolean existsByPlaca(String placa);
    boolean existsByPlacaAndIdNot(String placa, Integer id);

    // Para el portal del cliente / filtro en el frontend: "mis vehículos"
    List<Vehiculo> findByClienteId(Integer clienteId);
}
```

Y un `OrdenTrabajoRepository` mínimo, necesario para bloquear el borrado de un vehículo con historial:

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.OrdenTrabajo;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface OrdenTrabajoRepository extends JpaRepository<OrdenTrabajo, Long> {
    boolean existsByVehiculoId(Integer vehiculoId);
}
```

## 5.5 Definir la interfaz del Service

```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.VehiculoDTO;

import java.util.List;

public interface IVehiculoService {
    List<VehiculoDTO> findAll();

    VehiculoDTO findById(Integer id);

    List<VehiculoDTO> findByClienteId(Integer clienteId);

    VehiculoDTO saveOrUpdate(VehiculoDTO dto);

    void delete(Integer id);
}
```
## 5.6 Programar el Service
```java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.VehiculoDTO;
import com.devsv.autofix_api.entities.Cliente;
import com.devsv.autofix_api.entities.Modelo;
import com.devsv.autofix_api.entities.Vehiculo;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ConflictException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IVehiculoService;
import com.devsv.autofix_api.mappers.VehiculoMapper;
import com.devsv.autofix_api.repository.ClienteRepository;
import com.devsv.autofix_api.repository.ModeloRepository;
import com.devsv.autofix_api.repository.OrdenTrabajoRepository;
import com.devsv.autofix_api.repository.VehiculoRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class VehiculoService implements IVehiculoService {

    private final VehiculoRepository repository;
    private final ModeloRepository modeloRepository;
    private final ClienteRepository clienteRepository;
    private final OrdenTrabajoRepository ordenTrabajoRepository;
    private final VehiculoMapper mapper;

    @Override
    @Transactional(readOnly = true)
    public List<VehiculoDTO> findAll() {
        return mapper.toDtoList(repository.findAll());
    }

    @Override
    @Transactional(readOnly = true)
    public VehiculoDTO findById(Integer id) {
        Vehiculo entity = buscarEntidad(id);
        return mapper.toDTO(entity);
    }

    @Override
    @Transactional(readOnly = true)
    public List<VehiculoDTO> findByClienteId(Integer clienteId) {
        clienteRepository.findById(clienteId)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el cliente con ID: " + clienteId));

        return mapper.toDtoList(repository.findByClienteId(clienteId));
    }

    @Override
    @Transactional
    public VehiculoDTO saveOrUpdate(VehiculoDTO dto) {
        // 1. modelo y cliente son obligatorios - esto también valida que existan
        Modelo modelo = buscarModelo(dto.getModeloId());
        Cliente cliente = buscarCliente(dto.getClienteId());

        Vehiculo entity;

        if (dto.getId() == null) {
            // 2. creación: validamos que la placa no se repita (única global)
            if (repository.existsByPlaca(dto.getPlaca())) {
                throw new ConflictException(
                        "Ya existe un vehículo registrado con la placa '" + dto.getPlaca() + "'");
            }
            entity = new Vehiculo();
        } else {
            // 3. actualización: validamos que exista y que la placa no choque con OTRO registro
            entity = buscarEntidad(dto.getId());

            if (repository.existsByPlacaAndIdNot(dto.getPlaca(), dto.getId())) {
                throw new ConflictException(
                        "Ya existe otro vehículo registrado con la placa '" + dto.getPlaca() + "'");
            }
        }

        // 4. copiamos los campos y asignamos las relaciones ya resueltas
        entity.setPlaca(dto.getPlaca());
        entity.setAnio(dto.getAnio());
        entity.setCaracteristicas(dto.getCaracteristicas());
        entity.setModelo(modelo);
        entity.setCliente(cliente);

        // 5. persistimos
        return mapper.toDTO(repository.save(entity));
    }

    @Override
    @Transactional
    public void delete(Integer id) {
        Vehiculo entity = buscarEntidad(id);

        if (ordenTrabajoRepository.existsByVehiculoId(id)) {
            throw new ConflictException(
                    "No se puede eliminar el vehículo con placa '" + entity.getPlaca() +
                            "' porque ya tiene órdenes de trabajo registradas");
        }

        repository.delete(entity);
    }

    private Vehiculo buscarEntidad(Integer id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el vehículo con ID: " + id));
    }

    private Modelo buscarModelo(Integer modeloId) {
        if (modeloId == null) {
            throw new BadRequestException("Debe indicar el modelo del vehículo");
        }
        return modeloRepository.findById(modeloId)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el modelo con ID: " + modeloId));
    }

    private Cliente buscarCliente(Integer clienteId) {
        if (clienteId == null) {
            throw new BadRequestException("Debe indicar el cliente propietario del vehículo");
        }
        return clienteRepository.findById(clienteId)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el cliente con ID: " + clienteId));
    }
}
```

## 5.7 Programar el Controller
```java
package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.dto.VehiculoDTO;
import com.devsv.autofix_api.interfaces.IVehiculoService;
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
public class VehiculoController {
    private final IVehiculoService vehiculoService;

    @GetMapping("/vehiculos")
    public ResponseEntity<List<VehiculoDTO>> getAll(@RequestParam(required = false) Integer clienteId) {
        if (clienteId != null) {
            return ResponseEntity.ok(vehiculoService.findByClienteId(clienteId));
        }
        return ResponseEntity.ok(vehiculoService.findAll());
    }

    @GetMapping("/vehiculos/{id}")
    public ResponseEntity<VehiculoDTO> getById(@PathVariable Integer id) {
        return ResponseEntity.ok(vehiculoService.findById(id));
    }

    @PostMapping("/vehiculos")
    public ResponseEntity<?> create(@RequestBody VehiculoDTO dto) {
        Map<String, Object> response = new HashMap<>();

        VehiculoDTO guardado = vehiculoService.saveOrUpdate(dto);

        response.put("message", "Vehículo registrado correctamente...!");
        response.put("vehiculo", guardado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    @PutMapping("/vehiculos/{id}")
    public ResponseEntity<?> update(@PathVariable Integer id, @RequestBody VehiculoDTO dto) {
        Map<String, Object> response = new HashMap<>();

        dto.setId(id);
        VehiculoDTO actualizado = vehiculoService.save(dto);

        response.put("message", "Vehículo actualizado correctamente");
        response.put("vehiculo", actualizado);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @DeleteMapping("/vehiculos/{id}")
    public ResponseEntity<?> delete(@PathVariable Integer id) {
        vehiculoService.delete(id);

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Vehículo eliminado con éxito");

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```
## 5.8 Probar en Postman
