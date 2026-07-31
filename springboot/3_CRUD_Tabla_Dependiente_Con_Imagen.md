# 3. Programar CRUD para una tabla independiente con gestión de imagen

En esta guía vamos a desarrollar los componentes para crear un CRUD para la tabla `repuestos_servicios`, entidad **RepuestoServicio** que ya fue creada. A diferencia del CRUD de Marca, este recurso permite adjuntar una imagen: se sube a disco dentro de la carpeta `images` (subcarpeta de `uploads`), y en la tabla **solo se guarda el nombre del archivo**, nunca la ruta completa.

## 3.1 Confirmar la propiedad de la carpeta de subida

En `application.properties` ya existe:

```properties
app.upload.dir=${UPLOAD_DIR:uploads}
```

Las imágenes se guardarán dentro de `${app.upload.dir}/images/`. No hace falta agregar nada nuevo aquí — solo la vamos a leer con `@Value` en las clases nuevas.

## 3.2 Crear la clase DTO

```java
package com.devsv.autofix_api.dto;

import com.devsv.autofix_api.enums.TipoItem;
import lombok.Getter;
import lombok.Setter;

import java.math.BigDecimal;

@Getter
@Setter
public class RepuestoServicioDTO {
    private Integer id;

    private String nombre;

    private String descripcion;

    private BigDecimal precio;

    private Integer stock;

    private String foto; // solo el nombre del archivo

    private TipoItem tipo;
}
```

## 3.3 Crear el mapper para convertir de entity a dto y viceversa

```java
package com.devsv.autofix_api.mappers;

import com.devsv.autofix_api.dto.RepuestoServicioDTO;
import com.devsv.autofix_api.entities.RepuestoServicio;
import org.mapstruct.Mapper;
import org.mapstruct.ReportingPolicy;

import java.util.List;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface RepuestoServicioMapper {
    RepuestoServicioDTO toDTO(RepuestoServicio entity);

    RepuestoServicio toEntity(RepuestoServicioDTO dto);

    List<RepuestoServicioDTO> toDtoList(List<RepuestoServicio> entities);
}
```

## 3.4 Crear los repository

Necesitamos dos: el propio de RepuestoServicio, y uno de DetalleOrden para poder validar, antes de eliminar, si el repuesto/servicio ya fue usado en alguna orden de trabajo.

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.RepuestoServicio;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface RepuestoServicioRepository extends JpaRepository<RepuestoServicio, Integer> {
}
```

```java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.DetalleOrden;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface DetalleOrdenRepository extends JpaRepository<DetalleOrden, Long> {

    // Se usa para bloquear la eliminación de un RepuestoServicio
    // que ya fue usado en al menos una orden de trabajo.
    boolean existsByRepuestoServicioId(Integer repuestoServicioId);
}
```

## 3.5 Crear la utilidad para guardar y eliminar imágenes en disco

```java
package com.devsv.autofix_api.utils;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.util.UUID;

@Component
public class FileStorageUtil {

    private static final String SUBCARPETA_IMAGENES = "images";

    @Value("${app.upload.dir}")
    private String uploadDir;

    public String guardarImagen(MultipartFile archivo) {
        try {
            Path carpetaImagenes = Paths.get(uploadDir, SUBCARPETA_IMAGENES);
            Files.createDirectories(carpetaImagenes);

            String extension = obtenerExtension(archivo.getOriginalFilename());
            String nombreGenerado = UUID.randomUUID() + extension;

            Path destino = carpetaImagenes.resolve(nombreGenerado);
            Files.copy(archivo.getInputStream(), destino, StandardCopyOption.REPLACE_EXISTING);

            return nombreGenerado;
        } catch (IOException e) {
            throw new UncheckedIOException("No se pudo guardar la imagen: " + e.getMessage(), e);
        }
    }

    public void eliminarImagen(String nombreArchivo) {
        if (nombreArchivo == null || nombreArchivo.isBlank()) return;
        try {
            Path ruta = Paths.get(uploadDir, SUBCARPETA_IMAGENES, nombreArchivo);
            Files.deleteIfExists(ruta);
        } catch (IOException e) {
            // que falle borrar el archivo físico no debe impedir
            // que la operación de negocio continúe
        }
    }

    private String obtenerExtension(String nombreOriginal) {
        if (nombreOriginal == null || !nombreOriginal.contains(".")) return "";
        return nombreOriginal.substring(nombreOriginal.lastIndexOf('.'));
    }
}
```

**Por qué el nombre se genera con `UUID`:** si dos productos distintos suben una imagen llamada `foto.jpg`, sin esto la segunda subida sobrescribiría el archivo de la primera. El UUID garantiza que cada archivo en disco sea único, sin importar el nombre original.

## 3.6 Crear el WebConfig para exponer las imágenes por URL

```java
package com.devsv.autofix_api.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.nio.file.Paths;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Value("${app.upload.dir}")
    private String uploadDir;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        String ubicacionAbsoluta = Paths.get(uploadDir, "images")
                .toAbsolutePath()
                .toUri()
                .toString();

        registry.addResourceHandler("/images/**")
                .addResourceLocations(ubicacionAbsoluta);
    }
}
```

**Por qué es necesaria esta clase:** Spring Boot, por defecto, solo sirve archivos estáticos que están dentro de `src/main/resources/static` (y quedan empaquetados dentro del `.jar`). Las imágenes que suben los usuarios se guardan **fuera** de ese classpath, en una carpeta del sistema de archivos — sin este `WebConfig`, el archivo se guarda correctamente pero no hay ninguna URL que lo sirva. Con esto, una imagen guardada como `3f2a1c9e-frenos.jpg` queda accesible en `http://localhost:8080/images/3f2a1c9e-frenos.jpg`.

## 3.7 Definir la interfaz del Service

```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.RepuestoServicioDTO;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

public interface IRepuestoServicioService {
    List<RepuestoServicioDTO> findAll();

    RepuestoServicioDTO findById(Integer id);
    
    RepuestoServicioDTO save(RepuestoServicioDTO dto, MultipartFile imagen);

    void delete(Integer id);
}
```

## 3.8 Programar el Service

```java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.RepuestoServicioDTO;
import com.devsv.autofix_api.entities.RepuestoServicio;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IRepuestoServicioService;
import com.devsv.autofix_api.mappers.RepuestoServicioMapper;
import com.devsv.autofix_api.repository.DetalleOrdenRepository;
import com.devsv.autofix_api.repository.RepuestoServicioRepository;
import com.devsv.autofix_api.utils.FileStorageUtil;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

@Service
@RequiredArgsConstructor
public class RepuestoServicioService implements IRepuestoServicioService {

    private final RepuestoServicioRepository repository;
    private final DetalleOrdenRepository detalleOrdenRepository;
    private final RepuestoServicioMapper mapper;
    private final FileStorageUtil fileStorageUtil;

    @Override
    @Transactional(readOnly = true)
    public List<RepuestoServicioDTO> findAll() {
        return mapper.toDtoList(repository.findAll());
    }

    @Override
    @Transactional(readOnly = true)
    public RepuestoServicioDTO findById(Integer id) {
        RepuestoServicio entity = buscarEntidad(id);
        return mapper.toDTO(entity);
    }

    @Override
    @Transactional
    public RepuestoServicioDTO save(RepuestoServicioDTO dto, MultipartFile imagen) {
        // 1. si trae id, es una actualización: recuperamos la entidad actual
        RepuestoServicio entity;
        String fotoAnterior = null;

        if (dto.getId() != null) {
            entity = buscarEntidad(dto.getId());
            fotoAnterior = entity.getFoto();
        } else {
            entity = new RepuestoServicio();
        }

        // 2. copiamos los campos del DTO a la entidad a mano
        //    (no usamos mapper.toEntity aquí: generaría un objeto NUEVO
        //    y perderíamos el id/foto ya cargados en una actualización)
        entity.setNombre(dto.getNombre());
        entity.setDescripcion(dto.getDescripcion());
        entity.setPrecio(dto.getPrecio());
        entity.setStock(dto.getStock());
        entity.setTipo(dto.getTipo());

        // 3. si llega una imagen nueva, la guardamos y actualizamos el nombre.
        //    si NO llega, la entidad conserva la foto que ya tenía.
        if (imagen != null && !imagen.isEmpty()) {
            String nuevoNombreArchivo = fileStorageUtil.guardarImagen(imagen);
            entity.setFoto(nuevoNombreArchivo);
        }

        // 4. persistimos
        RepuestoServicio guardado = repository.save(entity);

        // 5. si reemplazamos la imagen, borramos el archivo físico anterior
        if (imagen != null && !imagen.isEmpty() && fotoAnterior != null) {
            fileStorageUtil.eliminarImagen(fotoAnterior);
        }

        return mapper.toDTO(guardado);
    }

    @Override
    @Transactional
    public void delete(Integer id) {
        RepuestoServicio entity = buscarEntidad(id);

        // No se puede eliminar un repuesto/servicio que ya fue usado
        // en alguna orden de trabajo (protege el historial de detalle_ordenes).
        if (detalleOrdenRepository.existsByRepuestoServicioId(id)) {
            throw new BadRequestException(
                    "No se puede eliminar '" + entity.getNombre() +
                            "' porque ya está registrado en una o más órdenes de trabajo");
        }

        repository.delete(entity);
        fileStorageUtil.eliminarImagen(entity.getFoto());
    }

    private RepuestoServicio buscarEntidad(Integer id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException(
                        "No existe el repuesto/servicio con ID: " + id));
    }
}
```

## 3.9 Programar el Controller

```java
package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.dto.RepuestoServicioDTO;
import com.devsv.autofix_api.enums.TipoItem;
import com.devsv.autofix_api.interfaces.IRepuestoServicioService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@CrossOrigin
@RequestMapping("/api")
@RequiredArgsConstructor
public class RepuestoServicioController {
    private final IRepuestoServicioService repuestoServicioService;

    @GetMapping("/repuestos-servicios")
    public ResponseEntity<List<RepuestoServicioDTO>> getAll() {
        return ResponseEntity.ok(repuestoServicioService.findAll());
    }

    @GetMapping("/repuestos-servicios/{id}")
    public ResponseEntity<RepuestoServicioDTO> getById(@PathVariable Integer id) {
        return ResponseEntity.ok(repuestoServicioService.findById(id));
    }

    // multipart/form-data: va acompañado del archivo de imagen,
    // por eso NO puede ser @RequestBody JSON puro como en MarcaController
    @PostMapping(value = "/repuestos-servicios", consumes = "multipart/form-data")
    public ResponseEntity<?> create(
            @RequestParam String nombre,
            @RequestParam(required = false) String descripcion,
            @RequestParam BigDecimal precio,
            @RequestParam Integer stock,
            @RequestParam TipoItem tipo,
            @RequestParam(required = false) MultipartFile imagen) {

        Map<String, Object> response = new HashMap<>();

        RepuestoServicioDTO dto = new RepuestoServicioDTO();
        dto.setNombre(nombre);
        dto.setDescripcion(descripcion);
        dto.setPrecio(precio);
        dto.setStock(stock);
        dto.setTipo(tipo);

        RepuestoServicioDTO guardado = repuestoServicioService.save(dto, imagen);

        response.put("message", "Repuesto/Servicio registrado correctamente...!");
        response.put("repuestoServicio", guardado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    // la imagen es opcional en la actualización: si no llega, se conserva la anterior
    @PutMapping(value = "/repuestos-servicios/{id}", consumes = "multipart/form-data")
    public ResponseEntity<?> update(
            @PathVariable Integer id,
            @RequestParam String nombre,
            @RequestParam(required = false) String descripcion,
            @RequestParam BigDecimal precio,
            @RequestParam Integer stock,
            @RequestParam TipoItem tipo,
            @RequestParam(required = false) MultipartFile imagen) {

        Map<String, Object> response = new HashMap<>();

        RepuestoServicioDTO dto = new RepuestoServicioDTO();
        dto.setId(id);
        dto.setNombre(nombre);
        dto.setDescripcion(descripcion);
        dto.setPrecio(precio);
        dto.setStock(stock);
        dto.setTipo(tipo);

        RepuestoServicioDTO actualizado = repuestoServicioService.save(dto, imagen);

        response.put("message", "Repuesto/Servicio actualizado correctamente");
        response.put("repuestoServicio", actualizado);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @DeleteMapping("/repuestos-servicios/{id}")
    public ResponseEntity<?> delete(@PathVariable Integer id) {
        repuestoServicioService.delete(id);

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Repuesto/Servicio eliminado con éxito");

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

## 3.10 Probar en Postman

