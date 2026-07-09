# 0. Introducción a Java, la JVM y el entorno de desarrollo

## 0.1 ¿Qué es Java?

Java es un lenguaje de programación de propósito general, orientado a objetos, fuertemente tipado y **multiplataforma**, creado por Sun Microsystems en 1995 (hoy propiedad de Oracle). Su lema histórico es *"Write Once, Run Anywhere"* (WORA): el código fuente se compila a un formato intermedio llamado **bytecode**, que puede ejecutarse en cualquier sistema operativo que tenga instalada una JVM.

Características clave que el estudiante debe poder explicar con sus propias palabras:

- **Compilado e interpretado a la vez**: el compilador `javac` traduce `.java` a `.class` (bytecode); luego la JVM interpreta/compila ese bytecode a código nativo en tiempo de ejecución (JIT — *Just In Time Compiler*).
- **Orientado a objetos**: todo (excepto los tipos primitivos) es un objeto.
- **Gestión automática de memoria**: el *Garbage Collector* libera memoria de objetos que ya no se usan.
- **Fuertemente tipado**: cada variable tiene un tipo conocido en tiempo de compilación.
- **Multihilo** de forma nativa.
- **Ecosistema empresarial masivo**: Spring, Hibernate, Maven/Gradle — la base de lo que verán más adelante en el módulo.

Java 21 (LTS, septiembre 2023) es la versión que usaremos porque introduce mejoras que simplifican mucho el código de consola que escribiremos: `records`, `pattern matching` para `switch`, `text blocks`, y *virtual threads* (aunque este último tema es más avanzado y se retoma en el desarrollo backend).

## 0.2 JVM, JDK y JRE — la relación que todos confunden

```
                ┌───────────────────────────────────────────┐
                │                    JDK                    │
                │   (Java Development Kit - para programar) │
                │                                           │
                │   ┌───────────────────────────────────┐   │
                │   │               JRE                 │   │
                │   │  (Java Runtime Environment -      │   │
                │   │   para ejecutar programas Java)   │   │
                │   │                                   │   │
                │   │   ┌───────────────────────────┐   │   │
                │   │   │            JVM            │   │   │
                │   │   │  (Java Virtual Machine -  │   │   │
                │   │   │  ejecuta el bytecode)     │   │   │
                │   │   └───────────────────────────┘   │   │
                │   │   + Librerías estándar (java.*)   │   │
                │   └───────────────────────────────────┘   │
                │   + javac, jar, javadoc, jdb, jshell...   │
                └───────────────────────────────────────────┘
```

- **JVM (Java Virtual Machine)**: la "máquina" abstracta que ejecuta bytecode. Es específica por sistema operativo/arquitectura, pero el bytecode que ejecuta es siempre el mismo. Se encarga de: cargar clases (*Class Loader*), verificar bytecode, ejecutar (intérprete + JIT) y gestionar memoria (Garbage Collector, *heap*, *stack*).
- **JRE (Java Runtime Environment)**: JVM + librerías estándar (`java.lang`, `java.util`, `java.io`, etc.) necesarias para **ejecutar** una aplicación Java ya compilada. Si solo necesitas correr un `.jar`, con el JRE bastaría (en la práctica, desde Java 11 Oracle dejó de distribuir el JRE por separado; se instala el JDK completo aunque solo se necesite ejecutar).
- **JDK (Java Development Kit)**: JRE + herramientas de desarrollo (`javac` el compilador, `jar` el empaquetador, `javadoc` el generador de documentación, `jshell` el REPL interactivo, `jdb` el depurador). Es lo que instalamos para **programar**.

**Analogía para el aula:** la JVM es el motor de un carro, el JRE es el carro completo listo para manejar, y el JDK es el taller mecánico completo (incluye el carro + las herramientas para construir/reparar carros).

### Ciclo de vida de un programa Java

```
MiPrograma.java  --(javac)-->  MiPrograma.class (bytecode)  --(java, dentro de la JVM)-->  Ejecución
```

Demo rápida en terminal (sin IDE, para que entiendan qué hace IntelliJ "por debajo"):

```bash
javac HolaMundo.java   # genera HolaMundo.class
java HolaMundo         # la JVM carga y ejecuta el bytecode
```

## 0.3 ¿Qué es JDBC? (visión general — se profundiza en el archivo 06)

**JDBC (Java Database Connectivity)** es una API estándar de Java que permite a una aplicación conectarse y ejecutar operaciones (SELECT, INSERT, UPDATE, DELETE) contra una base de datos relacional, sin importar el motor (PostgreSQL, MySQL, SQL Server, Oracle...). Funciona así:

1. Tu código Java usa interfaces estándar: `Connection`, `Statement`, `PreparedStatement`, `ResultSet`.
2. Un **driver JDBC** específico del motor (ej. `postgresql-42.x.jar`) traduce esas llamadas al protocolo real de la base de datos.
3. Esto es exactamente lo que hace **Spring Data JPA/Hibernate** por debajo — entender JDBC "a detalle" primero hace que entender el ORM después sea mucho más fácil, en lugar de una caja negra mágica.

No profundizamos aquí porque requiere haber visto POO, excepciones y colecciones primero.

## 0.4 Instalación del JDK 21

### Windows

1. Descargar el zip desde el sitio oficial de **https://jdk.java.net/archive/** → seleccionar **21.0.2 (build 21.0.2+13) (LTS)**, sistema operativo Windows, arquitectura x64
2.Crear una carpeta Java en su unidad o ruta de su preferencia, por ejemplo **C:\Java** y descomprimir archivo, como se muestra en la siguiente imágen
<img width="971" height="218" alt="image" src="https://github.com/user-attachments/assets/d16d2302-8450-4aaa-aaf6-878ec47cbf81" />

3. Ejecutar el instalador. Marcar la opción **"Set JAVA_HOME variable"** y **"Add to PATH"** si el instalador las ofrece (Temurin las incluye como checkbox).
4. Verificar instalación abriendo `cmd` o PowerShell:
   ```powershell
   java -version
   javac -version
   ```
   Ambos deben responder `21.x.x`.
5. Si `java -version` no es reconocido, configurar manualmente las variables de entorno:
   - `JAVA_HOME` → ruta de instalación, ej. `C:\Program Files\Eclipse Adoptium\jdk-21.0.x`
   - Agregar `%JAVA_HOME%\bin` al `PATH`.

### macOS

```bash
brew install --cask temurin21
java -version
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install openjdk-21-jdk
java -version
javac -version
```

## 0.5 Instalación de IntelliJ IDEA Community Edition

1. Descargar desde `https://www.jetbrains.com/idea/download/` → pestaña **Community Edition** (gratuita, open source; suficiente para todo este módulo — no necesitamos la Ultimate con soporte Spring integrado, ya que instalaremos Spring Boot manualmente vía Maven más adelante).
2. Instalar con las opciones por defecto. En Windows, marcar "Add launchers dir to PATH" y ".java" para asociar archivos.
3. Al abrir por primera vez:
   - **New Project** → **New Project (Java)**.
   - En **JDK**, IntelliJ suele detectar automáticamente el JDK 21 instalado; si no aparece, clic en **Add JDK** → navegar hasta la carpeta de instalación del JDK.
   - Marcar **Add sample code** para verificar que corre un `HolaMundo` de inmediato.
4. Configuración recomendada para el curso:
   - `Settings → Build, Execution, Deployment → Build Tools → Maven` (lo usaremos más adelante para las dependencias JDBC/Spring).
   - `Settings → Appearance` → activar tema oscuro/claro según preferencia (irrelevante para el aprendizaje, pero mejora la adopción del IDE).
5. Verificación: crear clase `HolaMundo.java`, escribir:
   ```java
   public class HolaMundo {
       public static void main(String[] args) {
           System.out.println("Entorno configurado correctamente");
       }
   }
   ```
   Ejecutar con `Shift + F10` (Windows/Linux) o `Ctrl + R` (macOS).

## 0.6 Anatomía de un programa Java mínimo

```java
public class HolaMundo {                 // El nombre de la clase debe coincidir con el nombre del archivo
    public static void main(String[] args) {  // Punto de entrada: la JVM busca exactamente esta firma
        System.out.println("Hola, mundo"); // Imprime y salta de línea
    }
}
```

- `public class HolaMundo`: una clase pública, el archivo **debe** llamarse `HolaMundo.java`.
- `public static void main(String[] args)`: método especial que la JVM invoca automáticamente. `static` porque se ejecuta sin necesidad de crear un objeto; `String[] args` recibe argumentos de línea de comandos.
- `System.out` es un objeto `PrintStream` estático de la clase `System` que representa la salida estándar (consola).

## 0.7 Laboratorio 0 — "Diagnóstico de entorno"

Crear una clase `DiagnosticoEntorno` que, usando `System.getProperty(...)`, imprima en consola (con formato de tabla simple usando `String.format`):

- Versión de Java en ejecución (`java.version`)
- Sistema operativo (`os.name`) y arquitectura (`os.arch`)
- Directorio home del usuario (`user.home`)
- Directorio de trabajo actual (`user.dir`)

```java
public class DiagnosticoEntorno {
    public static void main(String[] args) {
        System.out.println("=== Diagnóstico de entorno Java ===");
        System.out.printf("%-20s: %s%n", "Versión de Java", System.getProperty("java.version"));
        // Completar el resto...
    }
}
```

## 0.8 Reto 0 (no tan común) — "Detective de bytecode"

1. Compilar `HolaMundo.java` desde la terminal con `javac`.
2. Abrir el archivo `.class` generado con un editor hexadecimal o con `xxd HolaMundo.class | head` (Linux/macOS) o el comando `javap -c HolaMundo` (viene con el JDK).
3. Pedir a los estudiantes que identifiquen en la salida de `javap -c` las instrucciones de bytecode correspondientes a la llamada `System.out.println`.
4. Preguntas de reflexión para entregar por escrito (media página): ¿por qué el archivo `.class` no es legible como texto plano? ¿Qué significa que el bytecode sea "independiente de la plataforma" a partir de lo que observaron?

> Objetivo pedagógico: que la JVM deje de ser un concepto abstracto de diapositiva y se vuelva algo que "tocaron" con sus manos antes de empezar a programar en serio.
