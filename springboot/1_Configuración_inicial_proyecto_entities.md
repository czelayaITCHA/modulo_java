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




