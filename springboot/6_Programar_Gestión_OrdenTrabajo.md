# 6. Programar el módulo de Órdenes de Trabajo (transacción maestro-detalle)

En esta guía vamos a desarrollar los componentes para crear una **OrdenTrabajo** junto con todos sus **DetalleOrden** en una sola operación transaccional. Es el módulo más completo del ejemplo hasta ahora: depende de `Vehiculo` y opcionalmente de `Empleado` (mecánico), controla y revierte stock de `RepuestoServicio` dentro de la misma transacción, valida transiciones de estado, y maneja bloqueo optimista (`@Version`) para evitar que dos usuarios sobrescriban la misma orden sin darse cuenta.

## 6.1 Piezas de apoyo que faltaban: Empleado

`OrdenTrabajo` necesita mostrar el mecánico asignado, y ese DTO aún no existía.

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class EmpleadoDTO {
    private Integer id;

    private String nombre;

    private String email;
}
```

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.EmpleadoDTO;
import com.devsv.autofix_api.entities.Empleado;
import org.mapstruct.Mapper;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface EmpleadoMapper {
    EmpleadoDTO toDTO(Empleado entity);

    Empleado toEntity(EmpleadoDTO dto);

    List<EmpleadoDTO> toDtoList(List<Empleado> entities);
}
```

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Empleado;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface EmpleadoRepository extends JpaRepository<Empleado, Integer> {
}
```

## 6.2 Crear el DTO de DetalleOrden

```java
package com.devsv.autofix_api.dto;

import lombok.Getter;
import lombok.Setter;

import java.math.BigDecimal;

@Getter
@Setter
public class DetalleOrdenDTO {
    private Long id;

    private Integer cantidad;

    // Para CREAR/ACTUALIZAR: el frontend solo manda el id
    private Integer repuestoServicioId;

    // Para LEER: objeto completo
    private RepuestoServicioDTO repuestoServicio;
}
```

## 6.3 Crear el mapper de DetalleOrden

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.DetalleOrdenDTO;
import com.devsv.autofix_api.entities.DetalleOrden;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", uses = {RepuestoServicioMapper.class}, unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface DetalleOrdenMapper {

    @Mapping(target = "repuestoServicioId", source = "repuestoServicio.id")
    DetalleOrdenDTO toDTO(DetalleOrden entity);

    List<DetalleOrdenDTO> toDtoList(List<DetalleOrden> entities);
    
}
```

## 6.4 Crear el DTO de OrdenTrabajo

```java
package com.devsv.autofix_api.dto;

import com.devsv.autofix_api.enums.EstadoOrden;
import lombok.Getter;
import lombok.Setter;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;

@Getter
@Setter
public class OrdenTrabajoDTO {
    private Long id;

    private String numero;

    private LocalDate fecha;

    private EstadoOrden estado;

    private String observaciones;

    // El total lo CALCULA el Service sumando los subtotales de los detalles 
    private BigDecimal total;

    // Para CREAR/ACTUALIZAR
    private Integer vehiculoId;
    private Integer mecanicoId; // opcional - puede asignarse después

    // Para LEER
    private VehiculoDTO vehiculo;
    private EmpleadoDTO mecanico;

    // Se envía completo tanto al crear como al actualizar
    private List<DetalleOrdenDTO> detalles;
}
```

## 6.5 Crear el mapper de OrdenTrabajo

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.OrdenTrabajoDTO;
import com.devsv.autofix_api.entities.OrdenTrabajo;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", uses = {VehiculoMapper.class, EmpleadoMapper.class, DetalleOrdenMapper.class},
        unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface OrdenTrabajoMapper {

    @Mapping(target = "vehiculoId", source = "vehiculo.id")
    @Mapping(target = "mecanicoId", source = "mecanico.id")
    @Mapping(target = "detalles", source = "detalleOrden")
    OrdenTrabajoDTO toDTO(OrdenTrabajo entity);

    List<OrdenTrabajoDTO> toDtoList(List<OrdenTrabajo> entities);

}
```

## 6.6 Actualizar el repository

`OrdenTrabajoRepository` ya existía (se creó para la protección de borrado de `Vehiculo`) — se completa con la validación de número duplicado.

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.OrdenTrabajo;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface OrdenTrabajoRepository extends JpaRepository<OrdenTrabajo, Long> {
    boolean existsByVehiculoId(Integer vehiculoId);

    boolean existsByNumero(String numero);
    boolean existsByNumeroAndIdNot(String numero, Long id);
}
```

## 6.7 Definir la interfaz del Service

```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.OrdenTrabajoDTO;
import com.devsv.autofix_api.enums.EstadoOrden;

import java.util.List;

public interface IOrdenTrabajoService {
    List<OrdenTrabajoDTO> findAll();

    OrdenTrabajoDTO findById(Long id);

    OrdenTrabajoDTO create(OrdenTrabajoDTO dto);

    OrdenTrabajoDTO update(Long id, OrdenTrabajoDTO dto);

    OrdenTrabajoDTO changeState(Long id, EstadoOrden newState);

    void delete(Long id);
}

```

## 6.8 Programar el Service (el corazón del módulo)

```java

```
