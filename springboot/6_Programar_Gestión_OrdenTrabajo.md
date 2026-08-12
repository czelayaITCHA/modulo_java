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
    private List<DetalleOrdenDTO> detalleOrden;
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

@Mapper(componentModel = "spring", uses = {VehiculoMapper.class,
        EmpleadoMapper.class, DetalleOrdenMapper.class},
        unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface OrdenTrabajoMapper {

    @Mapping(target = "vehiculoId", source = "vehiculo.id")
    @Mapping(target = "mecanicoId", source = "mecanico.id")
    @Mapping(target = "detalleOrden", source = "detalleOrden")
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

import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

@Repository
public interface OrdenTrabajoRepository extends JpaRepository<OrdenTrabajo, Long> {
    boolean existsByVehiculoId(Integer vehiculoId);

    boolean existsByNumero(String numero);
    boolean existsByNumeroAndIdNot(String numero, Long id);

    Optional<OrdenTrabajo> findFirstByNumeroStartingWithOrderByNumeroDesc(String prefijo);

    //métodos para obtener listado de ordenes por rango de fechas
    List<OrdenTrabajo> findByFechaBetween(LocalDate fechaIncio, LocalDate FechaFin);
    //Obtener las órdenes por una fecha específica
    List<OrdenTrabajo> findByFecha(LocalDate fecha);

}
```

## 6.7 Definir la interfaz del Service

```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.OrdenTrabajoDTO;
import com.devsv.autofix_api.enums.EstadoOrden;

import java.util.List;

public interface IOrdenTrabajoService {
    List<OrdenTrabajoDTO> findAll(LocalDate fecha,
                                  LocalDate fechaInicio, LocalDate fechaFin);

    OrdenTrabajoDTO findById(Long id);

    OrdenTrabajoDTO create(OrdenTrabajoDTO dto);

    OrdenTrabajoDTO update(Long id, OrdenTrabajoDTO dto);

    OrdenTrabajoDTO changeState(Long id, EstadoOrden newState);

    void delete(Long id);
}

```

## 6.8 Programar el Service (el corazón del módulo)

```java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.DetalleOrdenDTO;
import com.devsv.autofix_api.dto.OrdenTrabajoDTO;
import com.devsv.autofix_api.entities.*;
import com.devsv.autofix_api.enums.EstadoOrden;
import com.devsv.autofix_api.enums.TipoItem;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ConflictException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IOrdenTrabajoService;
import com.devsv.autofix_api.mappers.OrdenTrabajoMapper;
import com.devsv.autofix_api.repository.EmpleadoRepository;
import com.devsv.autofix_api.repository.OrdenTrabajoRepository;
import com.devsv.autofix_api.repository.RepuestoServicioRepository;
import com.devsv.autofix_api.repository.VehiculoRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

@Service
@RequiredArgsConstructor
public class OrdenTrabajoService implements IOrdenTrabajoService {

    private final OrdenTrabajoRepository repository;
    private final VehiculoRepository vehiculoRepository;
    private final EmpleadoRepository empleadoRepository;
    private final RepuestoServicioRepository repuestoServicioRepository;
    private final OrdenTrabajoMapper mapper;

    // Estados a los que se puede pasar desde cada estado actual.
    // ENTREGADA y CANCELADA son estados finales - de ahí no se puede transicionar a nada.
    private static final Map<EstadoOrden, Set<EstadoOrden>> TRANSICIONES_VALIDAS = Map.of(
            EstadoOrden.PENDIENTE, Set.of(EstadoOrden.EN_PROCESO, EstadoOrden.CANCELADA),
            EstadoOrden.EN_PROCESO, Set.of(EstadoOrden.COMPLETADA, EstadoOrden.CANCELADA),
            EstadoOrden.COMPLETADA, Set.of(EstadoOrden.ENTREGADA),
            EstadoOrden.ENTREGADA, Set.of(),
            EstadoOrden.CANCELADA, Set.of()
    );

    @Override
    @Transactional(readOnly = true)
    public List<OrdenTrabajoDTO> findAll(LocalDate fecha,
                                         LocalDate fechaInicio, LocalDate fechaFin) {
        boolean tieneFechaExacta = fecha != null;
        boolean tieneRango = fechaInicio != null || fechaFin != null;
        //evaluamos que sean ambos casos
        if(tieneFechaExacta && tieneRango){
            throw new BadRequestException("Use 'fecha' para obtener órdenes de un día específico, o " +
                    "'fechaInicio' y 'fechaFin' para un rango - no ambas al mismo tiempo");
        }
        if(tieneFechaExacta){
            return mapper.toDtoList(repository.findByFecha(fecha));
        }
        if(tieneFechaExacta){
            if(fechaInicio == null || fechaFin == null){
                throw new BadRequestException("Debe especificar un rango de fechas");
            }
            if(fechaInicio.isAfter(fechaFin)){
                throw new BadRequestException("La fecha de inicio no puede ser mayor a la fecha final");
            }
            return mapper.toDtoList(repository.findByFechaBetween(fechaInicio, fechaFin));
        }
        return mapper.toDtoList(repository.findAll());
    }

    @Override
    @Transactional(readOnly = true)
    public OrdenTrabajoDTO findById(Long id) {
        return mapper.toDTO(buscarEntidad(id));
    }

    @Override
    @Transactional
    public OrdenTrabajoDTO create(OrdenTrabajoDTO dto) {
        // 1. validamos vehículo (obligatorio) y mecánico (opcional)
        Vehiculo vehiculo = buscarVehiculo(dto.getVehiculoId());
        Empleado mecanico = dto.getMecanicoId() != null ? buscarMecanico(dto.getMecanicoId()) : null;

        // 2. armamos el maestro - el número de orden se genera con el método privado generarNumeroOrden()
        OrdenTrabajo orden = new OrdenTrabajo();
        orden.setNumero(generarNumeroOrden());
        orden.setFecha(dto.getFecha() != null ? dto.getFecha() : LocalDate.now());
        orden.setEstado(EstadoOrden.PENDIENTE); // toda orden nueva inicia PENDIENTE, sin importar lo que venga en el DTO
        orden.setObservaciones(dto.getObservaciones());
        orden.setVehiculo(vehiculo);
        orden.setMecanico(mecanico);
        orden.setDetalleOrden(new ArrayList<>());

        // 3. procesamos los detalles: valida stock, descuenta, calcula precios y total.

        BigDecimal total = procesarDetalles(orden, dto.getDetalleOrden());
        orden.setTotal(total);

        OrdenTrabajo guardada = repository.save(orden);
        return mapper.toDTO(guardada);
    }

    @Override
    @Transactional
    public OrdenTrabajoDTO update(Long id, OrdenTrabajoDTO dto) {
        OrdenTrabajo orden = buscarEntidad(id);

        Vehiculo vehiculo = buscarVehiculo(dto.getVehiculoId());
        Empleado mecanico = dto.getMecanicoId() != null ? buscarMecanico(dto.getMecanicoId()) : null;

        // 1. devolvemos al inventario el stock reservado por los detalles ACTUALES
        revertirStock(orden);

        // 2. limpiamos la lista - orphanRemoval=true hace que Hibernate borre
        //    esos detalles viejos de la tabla al guardar
        orden.getDetalleOrden().clear();

        // 3. reconstruimos con los detalles nuevos (vuelve a validar y descontar stock).

        BigDecimal total = procesarDetalles(orden, dto.getDetalleOrden());

        orden.setFecha(dto.getFecha() != null ? dto.getFecha() : orden.getFecha());
        orden.setObservaciones(dto.getObservaciones());
        orden.setVehiculo(vehiculo);
        orden.setMecanico(mecanico);
        orden.setTotal(total);
        // "numero" y "estado" NO se tocan aquí a propósito, numero es inmutable, estado se cambia en changeState

        try {
            OrdenTrabajo actualizada = repository.save(orden);
            return mapper.toDTO(actualizada);
        } catch (ObjectOptimisticLockingFailureException e) {
            // @Version en la entidad: si otro usuario guardó esta misma orden
            // mientras la editabas, Hibernate lo detecta y lanza esta excepción
            throw new ConflictException(
                    "La orden fue modificada por otro usuario mientras la editabas; recarga e intenta de nuevo");
        }
    }

    @Override
    @Transactional
    public OrdenTrabajoDTO changeState(Long id, EstadoOrden newState) {
        OrdenTrabajo orden = buscarEntidad(id);

        Set<EstadoOrden> permitidos = TRANSICIONES_VALIDAS.get(orden.getEstado());
        if (permitidos == null || !permitidos.contains(newState)) {
            throw new BadRequestException(
                    "No se puede cambiar el estado de '" + orden.getEstado() + "' a '" + newState + "'");
        }

        // Si se cancela la orden, el stock reservado por sus repuestos se devuelve al inventario.
        if (newState == EstadoOrden.CANCELADA) {
            revertirStock(orden);
        }

        orden.setEstado(newState);

        try {
            return mapper.toDTO(repository.save(orden));
        } catch (ObjectOptimisticLockingFailureException e) {
            throw new ConflictException("La orden fue modificada por otro usuario; recarga e intenta de nuevo");
        }
    }

    @Override
    @Transactional
    public void delete(Long id) {
        OrdenTrabajo orden = buscarEntidad(id);

        // Solo se puede eliminar una orden que todavía no ha iniciado trabajo real.
        if (orden.getEstado() != EstadoOrden.PENDIENTE) {
            throw new ConflictException(
                    "Solo se pueden eliminar órdenes en estado PENDIENTE (estado actual: " + orden.getEstado() + ")");
        }

        // se devuelve el stock reservado antes de eliminar la orden
        revertirStock(orden);

        repository.delete(orden); // cascade + orphanRemoval eliminan los detalles automáticamente
    }

    // Métodos auxiliares

    /*
     * Valida y construye cada DetalleOrden a partir del DTO, descontando
     * stock cuando corresponde, y devuelve el total acumulado (suma de subtotales).
     * Lanza excepción ante el primer detalle inválido - eso corta toda la
     * transacción y revierte cualquier descuento de stock ya aplicado en este mismo método.
     */
    private BigDecimal procesarDetalles(OrdenTrabajo orden, List<DetalleOrdenDTO> detallesDto) {
        if (detallesDto == null || detallesDto.isEmpty()) {
            throw new BadRequestException("La orden debe tener al menos un detalle (repuesto o servicio)");
        }

        BigDecimal total = BigDecimal.ZERO;

        for (DetalleOrdenDTO detalleDto : detallesDto) {
            if (detalleDto.getRepuestoServicioId() == null) {
                throw new BadRequestException("Cada detalle debe indicar el repuesto/servicio");
            }
            if (detalleDto.getCantidad() == null || detalleDto.getCantidad() <= 0) {
                throw new BadRequestException("La cantidad de cada detalle debe ser mayor a cero");
            }

            RepuestoServicio repuestoServicio = repuestoServicioRepository.findById(detalleDto.getRepuestoServicioId())
                    .orElseThrow(() -> new ResourceNotFoundException(
                            "No existe el repuesto/servicio con ID: " + detalleDto.getRepuestoServicioId()));

            // El control de stock solo aplica a REPUESTOS - un SERVICIO no se "agota"
            if (repuestoServicio.getTipo() == TipoItem.REPUESTO) {
                if (repuestoServicio.getStock() < detalleDto.getCantidad()) {
                    throw new ConflictException(
                            "Stock insuficiente para '" + repuestoServicio.getNombre() + "': disponible " +
                                    repuestoServicio.getStock() + ", solicitado " + detalleDto.getCantidad());
                }
                repuestoServicio.setStock(repuestoServicio.getStock() - detalleDto.getCantidad());
                repuestoServicioRepository.save(repuestoServicio);
            }

            // El precio se toma del RepuestoServicio,
            BigDecimal precioUnitario = repuestoServicio.getPrecio();
            BigDecimal subtotal = precioUnitario.multiply(BigDecimal.valueOf(detalleDto.getCantidad()));

            DetalleOrden detalle = new DetalleOrden();
            detalle.setCantidad(detalleDto.getCantidad());
            detalle.setPrecioUnitario(precioUnitario);
            detalle.setSubtotal(subtotal);
            detalle.setRepuestoServicio(repuestoServicio);
            detalle.setOrdenTrabajo(orden); // lado dueño de la relación bidireccional

            orden.getDetalleOrden().add(detalle);
            total = total.add(subtotal);
        }

        return total;
    }

    /* Devuelve al inventario el stock reservado por cada detalle de tipo REPUESTO. */
    private void revertirStock(OrdenTrabajo orden) {
        for (DetalleOrden detalle : orden.getDetalleOrden()) {
            RepuestoServicio repuestoServicio = detalle.getRepuestoServicio();
            if (repuestoServicio.getTipo() == TipoItem.REPUESTO) {
                repuestoServicio.setStock(repuestoServicio.getStock() + detalle.getCantidad());
                repuestoServicioRepository.save(repuestoServicio);
            }
        }
    }

    /* Método para generar el número de orden con formato AAAA + MM + NNNN (10 caracteres) */

    private String generarNumeroOrden() {
        LocalDate hoy = LocalDate.now();
        String prefijo = String.format("%04d%02d", hoy.getYear(), hoy.getMonthValue());

        int siguienteCorrelativo = repository.findFirstByNumeroStartingWithOrderByNumeroDesc(prefijo)
                .map(ultimaOrden -> Integer.parseInt(ultimaOrden.getNumero().substring(6)) + 1)
                .orElse(1);

        if (siguienteCorrelativo > 9999) {
            throw new BadRequestException(
                    "Se alcanzó el máximo de órdenes de trabajo para " + prefijo + " (9999)");
        }

        return prefijo + String.format("%04d", siguienteCorrelativo);
    }

    private OrdenTrabajo buscarEntidad(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No existe la orden de trabajo con ID: " + id));
    }

    private Vehiculo buscarVehiculo(Integer vehiculoId) {
        if (vehiculoId == null) {
            throw new BadRequestException("Debe indicar el vehículo de la orden");
        }
        return vehiculoRepository.findById(vehiculoId)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el vehículo con ID: " + vehiculoId));
    }

    private Empleado buscarMecanico(Integer mecanicoId) {
        return empleadoRepository.findById(mecanicoId)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el empleado con ID: " + mecanicoId));
    }

}
```
## 6.9 Programar el Controller

```java
package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.dto.OrdenTrabajoDTO;
import com.devsv.autofix_api.enums.EstadoOrden;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.interfaces.IOrdenTrabajoService;
import lombok.RequiredArgsConstructor;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@CrossOrigin
@RequestMapping("/api")
@RequiredArgsConstructor
public class OrdenTrabajoController {
    private final IOrdenTrabajoService ordenTrabajoService;

    @GetMapping("/ordenes-trabajo")
    public ResponseEntity<List<OrdenTrabajoDTO>> getAll(
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fecha,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fechaInicio,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fechaFin
    ){
        return ResponseEntity.ok(ordenTrabajoService.findAll(fecha,fechaInicio,fechaFin));
    }

    @GetMapping("/ordenes-trabajo/{id}")
    public ResponseEntity<OrdenTrabajoDTO> getById(@PathVariable Long id){
        return ResponseEntity.ok(ordenTrabajoService.findById(id));
    }

    @PostMapping("/ordenes-trabajo")
    public ResponseEntity<?> create(@RequestBody OrdenTrabajoDTO dto){
        Map<String, Object> response = new HashMap<>();
        OrdenTrabajoDTO ordenCreated = ordenTrabajoService.create(dto);
        response.put("message", "Orden de trabajo registrada correctamente...!");
        response.put("ordenTrabajo", ordenCreated);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    //endpoint para actualizar la orden y reemplazar su lista de detalles
    @PutMapping("/ordenes-trabajo/{id}")
    public ResponseEntity<?> update(@PathVariable Long id, @RequestBody OrdenTrabajoDTO dto) {
        Map<String, Object> response = new HashMap<>();

        OrdenTrabajoDTO actualizada = ordenTrabajoService.update(id, dto);

        response.put("message", "Orden de trabajo actualizada correctamente");
        response.put("ordenTrabajo", actualizada);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    //endpoint dedicado para transiciones de estado (PENDIENTE -> EN_PROCESO -> ...)
    @PatchMapping("/ordenes-trabajo/{id}/estado")
    public ResponseEntity<?> cambiarEstado(@PathVariable Long id, @RequestBody Map<String, String> body) {
        Map<String, Object> response = new HashMap<>();

        EstadoOrden nuevoEstado;
        try {
            nuevoEstado = EstadoOrden.valueOf(body.get("estado"));
        } catch (IllegalArgumentException | NullPointerException e) {
            throw new BadRequestException(
                    "Estado inválido. Valores permitidos: PENDIENTE, EN_PROCESO, COMPLETADA, ENTREGADA, CANCELADA");
        }

        OrdenTrabajoDTO actualizada = ordenTrabajoService.changeState(id, nuevoEstado);

        response.put("message", "Estado actualizado a " + nuevoEstado);
        response.put("ordenTrabajo", actualizada);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @DeleteMapping("/ordenes-trabajo/{id}")
    public ResponseEntity<?> delete(@PathVariable Long id) {
        ordenTrabajoService.delete(id);

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Orden de trabajo eliminada con éxito");

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

}
```
## 6.10 Probar en Postman

### 1. Registrar una orden de trabajo
<img width="1934" height="1000" alt="image" src="https://github.com/user-attachments/assets/27e79d80-70a1-400c-a753-14aefdf93269" />

### 2. Forzar fallo por stock insuficiente de algún repuesto
<img width="1023" height="648" alt="image" src="https://github.com/user-attachments/assets/aef3cfda-4274-4923-a10d-208c6488a08c" />


