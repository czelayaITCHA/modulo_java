# 1. Crear API Rest con Java Spring Boot 4 - Configuración inicial del proyecto
* Diagrama E-R para ejemplo de clase
  
  <img width="870" height="607" alt="image" src="https://github.com/user-attachments/assets/c97cc9d2-97ed-4acf-91ef-8cec2a92cf6b" />


## 1.1 Crear proyecto en Spring Initializr
En el navegador escribir **spring initializr**, haga click en el primer enlace, que lo lleva a la interfaz de la siguiente imágen, llenar los datos y agregar dependencias iniciales

<img width="851" height="654" alt="image" src="https://github.com/user-attachments/assets/002d2640-9ecb-4499-b7fd-d85a0eb8d818" />

Luego haga click en generar o Ctrl + Enter, va descargar un archivo .zip con el nombre que definió al proyecto

## 1.2 Abrir proyecto y hacer configuraciones iniciales

### 1.2.1 Descomprimir el proyecto en una carpeta que identifique o localice fácilmente

<img width="830" height="315" alt="image" src="https://github.com/user-attachments/assets/66f04e64-48ff-4519-a389-6727c87cb174" />

### 1.2.2 Abrir el proyecto desde IntelliJ IDEA

<img width="1014" height="416" alt="image" src="https://github.com/user-attachments/assets/701aed0f-a974-4ee1-b042-cd74f4da8f63" />

Buscarmos y seleccionamos el proyecto descomprimidor "autofix-api" (lo abrimos en una nueva ventana)

<img width="1021" height="486" alt="image" src="https://github.com/user-attachments/assets/31b5bf8a-6726-4244-8d36-d82126b3e1ea" />

### 1.2.3 Crear estructura de paquetes del proyecto
Para organizar el código por funcionalidades creamos la siguiente estructura de packages

<img width="411" height="691" alt="image" src="https://github.com/user-attachments/assets/d5c472cb-eab3-4458-bcf1-16b4fed83199" />


### 1.2.4 Personalizar el archivo application.properties

```xml
# 1. Configuración general de la aplicación
spring.application.name=autofix-api
server.address=0.0.0.0
server.port=${SERVER_PORT:8080}

# 2. Configuración de Seguridad y JWT


# 3. Conxión a la base de datos (PostgreSQL)
spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:autofix_db}
spring.datasource.username=${DB_USER:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}

# 4. Configuración de Persistencia (JPA / Hibernate)
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=${JPA_DDL_AUTO:update}
spring.jpa.show-sql=${JPA_SHOW_SQL:true}
spring.jpa.properties.hibernate.format_sql=true

# 5. Internacionalización y zona horaria (El Salvador)
spring.jpa.properties.hibernate.jdbc.time_zone=America/El_Salvador
spring.jackson.time-zone=America/El_Salvador
spring.jackson.locale=es_SV

# 6. Definir tamaños máximos de archivos
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
app.upload.dir=${UPLOAD_DIR:uploads}
server.compression.enabled=true
```

### 1.2.5 Ejecutar el proyecto para garantizar que funcione bien hasta el momento

* Nota: Es importante crear primero la base de datos autofix_db en PostgreSQL
Al ejecutar el proyecto, hará el proceso de compilación y ejecución, el resultado será similar al siguiente

 <img width="1008" height="763" alt="image" src="https://github.com/user-attachments/assets/fe050394-a95c-402e-b4f8-47441c8133e8" />

## 1.3 Programar las clases entidades

### 1.3.1 Programar la entidad Marca
Comenzamos a crear las entidades en un orden lógico, es decir aquellas que no dependen de otras o que ya estén creadas para relacionarlas

```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.*;

import java.io.Serializable;

@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@Entity
@Table(name = "marcas", schema = "public", catalog = "autofix_db")
public class Marca implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "nombre", nullable = false, length = 50, unique = true)
    private String nombre;

}

```
Re ejecute el proyecto y verifique que haya sido creada la tabla marcas en PostgreSQL

### 1.3.2 Programar la entidad Modelo

```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.io.Serializable;

@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "modelos", schema = "public", catalog = "autofix_db")
public class Modelo implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Integer id;

    @Column(name = "nombre", nullable = false, length = 50)
    private String nombre;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "marca_id", nullable = false)
    private Marca marca;
}
```
### 1.3.3 Crear la entidad Cliente

```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import java.io.Serializable;

@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "clientes", schema = "public", catalog = "autofix_db")
public class Cliente implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "nombre", nullable = false, length = 80)
    private String nombre;

    @Column(name = "telefono", nullable = false, length = 15)
    private String telefono;

    @Column(name = "email", nullable = true, length = 50)
    private String email;
}

```

### 1.3.4 Programar la entidad Vehiculo

```java
package com.devsv.autofix_api.entities;


import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(
        name = "vehiculos", schema = "public", catalog = "autofix_db",
        indexes = {
                @Index(name = "idx_vehiculos_cliente_id", columnList = "cliente_id"),
                @Index(name = "idx_vehiculos_modelo_id", columnList = "modelo_id")
        }
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Vehiculo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @EqualsAndHashCode.Include
    private Integer id;

    @Column(nullable = false, unique = true, length = 20)
    private String placa;

    @Column(nullable = false)
    private Integer anio;

    @Column(length = 200)
    private String caracteristicas;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "modelo_id", nullable = false)
    private Modelo modelo;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id", nullable = false)
    private Cliente cliente;
}
```
### 1.3.5 Crear la entidad Empleado de acuerdo al diagrama
```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import java.io.Serializable;

@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "empleados", schema = "public", catalog = "autofix_db")
public class Empleado implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, length = 80)
    private String nombre;

    @Column(length = 80)
    private String email;
}

```

### 1.3.6 Crear la entidad Role

```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.io.Serializable;

@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "roles", schema = "public", catalog = "autofix_db")
public class Role implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Integer id;

    @Column(name = "nombre", nullable = false, length = 50)
    private String nombre;
}

```
### 1.3.7 Crear la entidad Usuario

```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.*;

import java.io.Serializable;

@NoArgsConstructor
@AllArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "usuarios", schema = "public", catalog = "autofix_db")
public class Usuario implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, unique = true, length = 20)
    private String username;

    @ToString.Exclude //evitar que el password aparezca en el log
    @Column(nullable = false, length = 250)
    private String password;

    @Column(nullable = false)
    private Boolean activo;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "empleado_id")
    private Empleado empleado;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id")
    private Cliente cliente;

}
```

### 1.3.8 Crear la entidad UsuarioRole

```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.io.Serializable;

@AllArgsConstructor
@NoArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "usuarios_roles", schema = "public", catalog = "autofix_db")
public class UsuarioRole implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "usuario_id", nullable = false)
    private Usuario usuario;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "role_id", nullable = false)
    private Role role;
}
```

* Antes de crear las entidades de **RepuestoServicio** y **OrdenTrabajo** vamos a crear en el package de **enums** los enums que vamos a necesitar en ambas entidades
* enum TipoItem
  
```java
 package com.devsv.autofix_api.enums;

public enum TipoItem {
    REPUESTO,
    SERVICIO
}
```

* enum EstadoOrden

```java
package com.devsv.autofix_api.enums;

public enum EstadoOrden {
    PENDIENTE,
    EN_PROCESO,
    COMPLETADA,
    ENTREGADA,
    CANCELADA
}
```

### 1.3.9 Crear la entidad RepuestoServicio
```java
package com.devsv.autofix_api.entities;

import com.devsv.autofix_api.enums.TipoItem;
import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.io.Serializable;
import java.math.BigDecimal;

@NoArgsConstructor
@AllArgsConstructor
@Setter
@Getter
@Entity
@Table(name = "repuestos_servicios", schema = "public", catalog = "autofix_db")
public class RepuestoServicio implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(length = 250)
    private String descripcion;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal precio;

    @Column(nullable = false)
    private Integer stock;

    @Column(length = 80)
    private String foto;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 10)
    private TipoItem tipo;
}
```

### 1.3.10 Crear la entidad OrdenTrabajo
```java
package com.devsv.autofix_api.entities;

import com.devsv.autofix_api.enums.EstadoOrden;
import jakarta.persistence.*;
import lombok.*;

import java.io.Serializable;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(
        name = "ordenes_trabajos", schema = "public", catalog = "autofix_db",
        indexes = {
                @Index(name = "idx_ordenes_vehiculo_id", columnList = "vehiculo_id"),
                @Index(name = "idx_ordenes_mecanico_id", columnList = "mecanico_id"),
                @Index(name = "idx_ordenes_estado", columnList = "estado"),
                @Index(name = "idx_ordenes_fecha", columnList = "fecha")
        }
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrdenTrabajo implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 10)
    private String numero;

    @Column(nullable = false)
    private LocalDate fecha;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 10)
    private EstadoOrden estado;

    @Column(length = 250)
    private String observaciones;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal total;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "vehiculo_id", nullable = false)
    private Vehiculo vehiculo;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "mecanico_id")
    private Empleado mecanico;

    @Version
    private Long version; //evitar que dos mecanicos escriban en la misma orden

    // Relación Bidireccional, porque esta sí es una transacción maestro-detalle.
    // cascade = ALL   -> guardar/eliminar la orden guarda/elimina sus detalles automáticamente.
    // orphanRemoval   -> si un detalle se quita de esta lista, se borra de la BD (no queda huérfano).
    // fetch = LAZY    -> NUNCA se carga junto con la orden a menos que se pida explícitamente.
    @OneToMany(mappedBy = "ordenTrabajo", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @Builder.Default
    @ToString.Exclude   // evita recursión infinita con DetalleOrden.ordenTrabajo en toString()
    private List<DetalleOrden> detalleOrden = new ArrayList<>();
}
```

### 1.3.11 Crear la entidad DetalleOrden
```java
package com.devsv.autofix_api.entities;

import jakarta.persistence.*;
import lombok.*;

import java.io.Serializable;
import java.math.BigDecimal;


@Entity
@Table(
        name = "detalle_ordenes", schema = "public", catalog = "autofix_db",
        indexes = {
                @Index(name = "idx_detalle_orden_trabajo_id", columnList = "orden_trabajo_id"),
                @Index(name = "idx_detalle_repuesto_servicio_id", columnList = "repuesto_servicio_id")
        }
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class DetalleOrden implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Integer cantidad;

    @Column(name = "precio_unitario", nullable = false, precision = 12, scale = 2)
    private BigDecimal precioUnitario;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal subtotal;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "orden" +
            "_trabajo_id", nullable = false)
    private OrdenTrabajo ordenTrabajo;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "repuesto_servicio_id", nullable = false)
    private RepuestoServicio repuestoServicio;
}
```
