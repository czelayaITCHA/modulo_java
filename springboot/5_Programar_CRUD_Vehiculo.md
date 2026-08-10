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

    VehiculoDTO save(VehiculoDTO dto);

    void delete(Integer id);
}
```
