# 2. Programar CRUD para una tabla independiente
En esta guía vamos a desarrollar los componentes para crear un CRUD para la tabla mascas, entidad **Marca** que ya fue creada, en la clase y guía anterior

## 2.1 Agregar dependencia de **MapStruct** para hacer conversiones de entidades a DTO y viseversa
Editamos el archivo pom.xml, en la sección de **properties**, justo después de la versión de java, agregamos esta linea
```xml
<org.mapstruct.version>1.6.3</org.mapstruct.version>
```

luego en la sección de **dependencies** justamente antes de </dependencies>, agregamos la dependencia de MapStruct

```xml
<dependency>
			<groupId>org.mapstruct</groupId>
			<artifactId>mapstruct</artifactId>
			<version>${org.mapstruct.version}</version>
		</dependency>
```
Luego compilamos el proyecto con maven

```bash
.\mvnw.cmd clean install -U
```

## 2.2 Crear paquete de **exceptions** y crear las siguientes clases
* GlobalExceptionHadler
* BadRequestException
* ResourceNotFoundException

## 2.3 Crear la clase DTO

## 2.4 Crear repository 

## 2.5 Definir crear interface y definir métodos a implementar para crear las funcionalidades
## 2.6 Programar el Service
## 2.7 Programar el Controller
## 2.8 Probar en Postman
