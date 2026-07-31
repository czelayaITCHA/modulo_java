# Guía Explicativa de JPA Repository en Spring Boot

## Introducción
Spring Data JPA simplifica el acceso a bases de datos mediante repositorios, evitando escribir la mayoría del código SQL y JDBC.

## ¿Qué es un Repository?
Es una interfaz que administra una entidad.

```java
public interface ClienteRepository extends JpaRepository<Cliente, Long> {}
```

## Jerarquía
- `Repository`
- `CrudRepository`
- `PagingAndSortingRepository`
- `JpaRepository` (la más utilizada)

## Anotaciones comunes
- `@Entity`
- `@Id`
- `@GeneratedValue`
- `@Table`
- `@Column`
- `@Repository`
- `@Transactional`
- `@Query`
- `@Modifying`

## Métodos heredados
```java
findAll();
findById(id);
save(cliente);
deleteById(id);
existsById(id);
count();
```

## Construcción automática de métodos

Formato:

`findBy + Campo + Operador`

Ejemplos:

```java
findByNombre(String nombre);
findByApellidoAndEstado(String a, Boolean e);
findByEdadGreaterThan(int edad);
findByNombreContaining(String texto);
findByActivoTrue();
findByFechaBetween(LocalDate i, LocalDate f);
findTop5ByOrderByNombreAsc();
```

### Operadores frecuentes

| Operador | Ejemplo |
|----------|----------|
| And | findByNombreAndEstado |
| Or | findByNombreOrApellido |
| Between | findByFechaBetween |
| LessThan | findByEdadLessThan |
| GreaterThan | findByEdadGreaterThan |
| Like | findByNombreLike |
| Containing | findByNombreContaining |
| StartsWith | findByNombreStartingWith |
| EndsWith | findByNombreEndingWith |
| In | findByIdIn |
| OrderBy | findByActivoOrderByNombreAsc |

## JPQL

JPQL consulta entidades, no tablas.

```java
@Query("SELECT c FROM Cliente c WHERE c.activo=true")
List<Cliente> activos();
```

Con parámetros:

```java
@Query("SELECT c FROM Cliente c WHERE c.nombre=:nombre")
List<Cliente> buscar(@Param("nombre") String nombre);
```

## Consultas nativas

```java
@Query(value="SELECT * FROM clientes WHERE activo=1", nativeQuery=true)
List<Cliente> activos();
```

## Actualizar datos

```java
@Modifying
@Transactional
@Query("UPDATE Cliente c SET c.activo=false WHERE c.id=:id")
void desactivar(@Param("id") Long id);
```

## Paginación

```java
Page<Cliente> findByActivo(Boolean activo, Pageable pageable);
```

## Ordenamiento

```java
repository.findAll(Sort.by("nombre").ascending());
```

## Buenas prácticas

- Preferir métodos derivados.
- Usar JPQL para consultas complejas.
- Reservar SQL nativo para casos específicos.
- Evitar consultas dentro de bucles.
- Utilizar paginación para grandes volúmenes.
- Mantener nombres de métodos claros.

## Resumen

1. CRUD → métodos heredados.
2. Búsquedas simples → Query Methods.
3. Consultas complejas → JPQL.
4. SQL específico → Native Query.
5. Actualizaciones → `@Modifying`.
6. Grandes datos → `Pageable` y `Sort`.
