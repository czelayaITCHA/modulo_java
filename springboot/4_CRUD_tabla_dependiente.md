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
    // El nombre se valida ÚNICO DE FORMA GLOBAL, sin importar la marca -
    // dos marcas distintas NO pueden tener un modelo con el mismo nombre.
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
