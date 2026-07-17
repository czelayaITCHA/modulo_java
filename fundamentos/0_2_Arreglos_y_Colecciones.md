# Guía de trabajo autónomo — Arreglos y Colecciones en Java

**Modalidad:** Clase virtual asíncrona (sin sesión en vivo)
**Tiempo estimado de dedicación:** 2.5 – 3 horas
**Prerrequisito:** Haber completado los temas de Variables, Tipos de Datos, Operadores y Estructuras de Control

---

## Cómo usar esta guía

Esta sesión es **asíncrona**: no habrá clase en vivo, Está diseñada para que puedas avanzar solo/a sin quedarte atascado/a. Reglas de uso:

1. Sigue el orden exacto de las secciones — cada bloque se apoya en el anterior.
2. **No avances de bloque sin resolver la autoevaluación** correspondiente. Las respuestas están al final del documento, pero intenta responder sin verlas primero.
3. Cada ejemplo de código está pensado para que lo **transcribas tú mismo/a en IntelliJ** (no copies y pegues) — escribirlo a mano es lo que fija el aprendizaje de sintaxis.
4. Debes integrar el ejercicio al proyecto que se ha creado en clase, es decir, IntroJava
5. El ejercicio integrador final es la evidencia que debes entregar. Revisa la rúbrica antes de empezar a programarlo, no después.

---

## Objetivos de aprendizaje

Al finalizar esta guía serás capaz de:

- Explicar la diferencia entre un arreglo (tamaño fijo) y una colección (tamaño dinámico), y decidir cuál usar según el problema.
- Declarar, recorrer y manipular arreglos unidimensionales y bidimensionales.
- Elegir correctamente entre `List`, `Set` y `Map` según la necesidad de orden, duplicados y acceso por clave.
- Usar `ArrayList`, `HashSet` y `HashMap` en un programa de consola con menú interactivo (`Scanner`).
- Identificar por qué los genéricos (`<T>`) previenen errores en tiempo de compilación.

---

## Cronograma sugerido de la sesión

| Bloque | Contenido | Tiempo estimado |
|---|---|---|
| 1 | Arreglos: teoría + ejemplo transcrito | 35 min |
| — | Autoevaluación 1 | 10 min |
| 2 | Colecciones (List, Set, Map): teoría + ejemplos transcritos | 50 min |
| — | Autoevaluación 2 | 10 min |
| 3 | Ejercicio integrador (Agenda de contactos) | 60 min |
| — | Revisión con checklist + entrega | 15 min |

---

## Bloque 1 — Arreglos (Arrays)

### 1.1 ¿Qué es un arreglo?

Un arreglo es una estructura que almacena **varios valores del mismo tipo**, en **posiciones contiguas de memoria**, con un **tamaño fijo** definido al momento de crearlo. Cada posición se identifica con un índice que **empieza en 0**.

### Esquema — cómo se ve un arreglo en memoria

```
int[] numeros = {40, 15, 8, 23, 60};

 índice:      0     1     2     3     4
            ┌─────┬─────┬─────┬─────┬─────┐
   valor:   │ 40  │ 15  │  8  │ 23  │ 60  │
            └─────┴─────┴─────┴─────┴─────┘
              ▲                       ▲
        numeros[0]              numeros[4]
                              numeros.length = 5   (no length())

 Intentar acceder a numeros[5] -> ArrayIndexOutOfBoundsException
 (el último índice válido siempre es length - 1)
```

Este es el error más común entre quienes recién aprenden arreglos: **confundir `length` (cantidad de elementos) con el último índice válido (`length - 1`)**. Grábate esa resta.

### 1.2 Declaración, inicialización y recorrido

Transcribe este ejemplo completo en IntelliJ y ejecútalo:

```java
public class EjemploArreglos {
    public static void main(String[] args) {
        int[] numeros = new int[5];          // arreglo vacío, todos en 0
        numeros[0] = 40;
        numeros[1] = 15;
        numeros[2] = 8;
        numeros[3] = 23;
        numeros[4] = 60;

        // Forma abreviada equivalente:
        int[] otrosNumeros = {40, 15, 8, 23, 60};

        // Recorrido con índice (útil cuando necesitas saber la posición)
        for (int i = 0; i < numeros.length; i++) {
            System.out.println("Posición " + i + ": " + numeros[i]);
        }

        // Recorrido for-each (más limpio cuando NO necesitas el índice)
        int suma = 0;
        for (int valor : numeros) {
            suma += valor;
        }
        System.out.println("Suma total: " + suma);
    }
}
```

**Ejercicio corto (no entregable, solo práctica):** modifica el programa para que, en lugar de sumar, calcule el **promedio** y encuentre el **valor máximo** del arreglo sin usar `Arrays.stream` (hazlo con un bucle y una variable acumuladora, así entiendes la lógica antes de usar atajos).

### 1.3 Arreglos bidimensionales

```java
int[][] tablero = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

for (int fila = 0; fila < tablero.length; fila++) {
    for (int columna = 0; columna < tablero[fila].length; columna++) {
        System.out.print(tablero[fila][columna] + " ");
    }
    System.out.println();
}
```

### Esquema — arreglo bidimensional como "arreglo de arreglos"

```
tablero  ──▶  [ fila0 ] ──▶ [ 1 | 2 | 3 ]
              [ fila1 ] ──▶ [ 4 | 5 | 6 ]
              [ fila2 ] ──▶ [ 7 | 8 | 9 ]

  tablero[1][2]  →  fila índice 1, columna índice 2  →  valor 6
```

### 1.4 La limitación que motiva el siguiente bloque

Ejecuta mentalmente (sin escribir código) este escenario: tienes un arreglo `String[] tareas = new String[3];` completamente lleno, y necesitas agregar una cuarta tarea. **No puedes.** El tamaño de un arreglo es fijo desde su creación. Para resolver esto sin recrear el arreglo manualmente cada vez, Java ofrece las **colecciones** — el tema siguiente.

---

## Autoevaluación 1 (resuelve antes de continuar)

1. Si `int[] datos = new int[8];`, ¿cuál es el índice del último elemento válido?
2. ¿Qué excepción se lanza si intentas acceder a `datos[8]`?
3. ¿Cuál es la diferencia entre `datos.length` y `datos.length()`? (pista: una de las dos ni siquiera existe)
4. Escribe de memoria (sin ver el ejemplo) la línea de código para declarar un arreglo de 4 `String` inicializado con los valores "rojo", "verde", "azul", "amarillo".

---

## Bloque 2 — Colecciones (Java Collections Framework)

### 2.1 ¿Por qué colecciones en vez de arreglos?

Una colección puede **crecer y reducirse dinámicamente** en tiempo de ejecución, y ofrece métodos ya construidos (agregar, eliminar, buscar, ordenar) que en un arreglo tendrías que programar manualmente.

### Esquema — jerarquía de las colecciones que usaremos

```
                         Collection
                              │
        ┌─────────────────────┼─────────────────────┐
        │                                           │
       List                                        Set
  (permite duplicados,                        (NO permite duplicados,
   mantiene orden de                            sin orden garantizado)
   inserción, acceso                                   │
   por índice)                                     HashSet
        │
    ArrayList


                            Map
              (independiente de Collection)
                 pares clave → valor,
                 claves únicas
                        │
                    HashMap
```

| Estructura | ¿Duplicados? | ¿Orden? | ¿Acceso por índice? | Úsala cuando... |
|---|---|---|---|---|
| `ArrayList` | Sí | Mantiene inserción | Sí (`get(i)`) | Necesitas una lista ordenada que puede crecer, con posibles repetidos |
| `HashSet` | No | No garantizado | No | Necesitas garantizar que **no haya elementos repetidos** |
| `HashMap` | Claves no, valores sí | No garantizado | Por clave (`get(clave)`) | Necesitas asociar un dato con un identificador único (ej. teléfono → contacto) |

### 2.2 List / ArrayList — transcribe y ejecuta

```java
import java.util.ArrayList;
import java.util.List;

public class EjemploList {
    public static void main(String[] args) {
        List<String> tareas = new ArrayList<>();
        tareas.add("Comprar leche");
        tareas.add("Estudiar Java");
        tareas.add("Comprar leche");          // duplicado permitido

        System.out.println("Tamaño: " + tareas.size());
        System.out.println("Elemento en posición 0: " + tareas.get(0));

        tareas.remove("Comprar leche");        // elimina la PRIMERA coincidencia
        System.out.println("Después de eliminar: " + tareas);

        for (String tarea : tareas) {
            System.out.println("- " + tarea);
        }
    }
}
```

### 2.3 Set / HashSet — transcribe y ejecuta

```java
import java.util.HashSet;
import java.util.Set;

public class EjemploSet {
    public static void main(String[] args) {
        Set<String> correos = new HashSet<>();
        correos.add("carlos@correo.com");
        correos.add("ana@correo.com");
        correos.add("carlos@correo.com");     // intento de duplicado

        System.out.println("Total de correos únicos: " + correos.size()); // imprime 2, no 3
        System.out.println(correos.contains("ana@correo.com"));           // true
    }
}
```

### 2.4 Map / HashMap — transcribe y ejecuta

```java
import java.util.HashMap;
import java.util.Map;

public class EjemploMap {
    public static void main(String[] args) {
        Map<String, Double> inventario = new HashMap<>();
        inventario.put("Teclado", 15.50);
        inventario.put("Mouse", 8.75);
        inventario.put("Teclado", 16.00);      // sobrescribe el valor anterior de "Teclado"

        for (Map.Entry<String, Double> entrada : inventario.entrySet()) {
            System.out.printf("%s: $%.2f%n", entrada.getKey(), entrada.getValue());
        }

        double precio = inventario.getOrDefault("Monitor", 0.0); // evita un null si la clave no existe
        System.out.println("Precio de Monitor (no registrado): " + precio);
    }
}
```

### 2.5 Genéricos `<T>` — por qué siempre verás `List<String>` y no `List` a secas

```java
List sinTipo = new ArrayList();       // "raw type" - EVITAR
sinTipo.add("texto");
sinTipo.add(123);                     // compila, pero mezclar tipos es un error esperando ocurrir

List<String> conTipo = new ArrayList<>();
conTipo.add("texto");
// conTipo.add(123);                  // el compilador lo rechaza ANTES de ejecutar - esto es lo que queremos
```

Los genéricos hacen que el compilador te proteja de errores que, sin ellos, solo aparecerían en tiempo de ejecución (a veces en producción, con el cliente usando el sistema).

---

## Autoevaluación 2 (resuelve antes de continuar)

1. Tienes que guardar los correos electrónicos registrados de los usuarios de un sistema, garantizando que no se repita ninguno. ¿Usarías `List`, `Set` o `Map`? Justifica en una línea.
2. Tienes que guardar el historial de compras de un cliente, donde el mismo producto puede comprarse varias veces y el orden de compra importa. ¿`List`, `Set` o `Map`?
3. Tienes que guardar la relación DUI → nombre completo de una persona, donde necesitas buscar rápidamente el nombre a partir del DUI. ¿`List`, `Set` o `Map`?
4. ¿Qué pasa si haces `mapa.put("A", 1);` y luego `mapa.put("A", 2);`? ¿Cuántos elementos tiene el mapa al final y qué valor tiene la clave `"A"`?

*(Respuestas al final del documento)*

---

## Bloque 3 — Ejercicio integrador (entregable)

### "Agenda de contactos en memoria"

Construye un programa de consola con **menú repetitivo** (`do-while`) que gestione contactos usando un `ArrayList` para los contactos y un `HashSet` para evitar teléfonos duplicados.

**Requisitos funcionales:**

1. **Agregar contacto**: solicitar nombre y teléfono. Antes de agregar, verificar en el `HashSet<String>` de teléfonos si ya existe ese número — si existe, mostrar un mensaje de error y **no** agregar el contacto.
2. **Listar contactos**: recorrer el `ArrayList` con `for-each` e imprimir todos los contactos en formato `Nombre - Teléfono`.
3. **Buscar contacto por nombre**: recorrer la lista comparando con `equalsIgnoreCase` y mostrar el resultado (o un mensaje si no se encontró).
4. **Eliminar contacto por nombre**: eliminarlo del `ArrayList` **y** su teléfono del `HashSet` (ambas estructuras deben quedar sincronizadas).
5. **Salir**: termina el programa.

**Estructura mínima esperada** (puedes usar solo `String` para representar cada contacto con este esquema, ya que POO se ve en la siguiente sesión):

```java
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Scanner;
import java.util.Set;

public class AgendaContactos {

    static List<String> nombres = new ArrayList<>();
    static List<String> telefonos = new ArrayList<>();   // mismo índice que "nombres"
    static Set<String> telefonosRegistrados = new HashSet<>();

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int opcion;

        do {
            System.out.println("\n===== AGENDA DE CONTACTOS =====");
            System.out.println("1. Agregar contacto");
            System.out.println("2. Listar contactos");
            System.out.println("3. Buscar contacto por nombre");
            System.out.println("4. Eliminar contacto por nombre");
            System.out.println("0. Salir");
            System.out.print("Seleccione una opción: ");

            while (!sc.hasNextInt()) {
                System.out.print("Entrada inválida, ingrese un número: ");
                sc.next();
            }
            opcion = sc.nextInt();
            sc.nextLine();   // limpiar el salto de línea pendiente

            switch (opcion) {
                case 1 -> agregarContacto(sc);
                case 2 -> listarContactos();
                case 3 -> buscarContacto(sc);
                case 4 -> eliminarContacto(sc);
                case 0 -> System.out.println("Saliendo...");
                default -> System.out.println("Opción no válida");
            }
        } while (opcion != 0);

        sc.close();
    }

    static void agregarContacto(Scanner sc) {
        // TODO: completar tú - pedir nombre y teléfono,
        // validar contra telefonosRegistrados antes de agregar
    }

    static void listarContactos() {
        // TODO: completar tú
    }

    static void buscarContacto(Scanner sc) {
        // TODO: completar tú
    }

    static void eliminarContacto(Scanner sc) {
        // TODO: completar tú
    }
}
```

Se te da el esqueleto del menú a propósito (ya lo viste en la sesión anterior) — tu trabajo es completar los 4 métodos marcados con `TODO`.

### Salida esperada de ejemplo (para que verifiques tu propio resultado)

```
===== AGENDA DE CONTACTOS =====
1. Agregar contacto
2. Listar contactos
3. Buscar contacto por nombre
4. Eliminar contacto por nombre
0. Salir
Seleccione una opción: 1
Nombre: Carlos Zelaya
Teléfono: 7000-1234
Contacto agregado exitosamente.

Seleccione una opción: 1
Nombre: Ana Pérez
Teléfono: 7000-1234
Error: ese teléfono ya está registrado.

Seleccione una opción: 2
Carlos Zelaya - 7000-1234
```

---

## Checklist de autorrevisión (antes de entregar)

Marca cada punto antes de subir tu archivo `.java`:

- [ ] El programa compila sin errores (lo ejecuté al menos una vez de principio a fin).
- [ ] El menú se repite correctamente con `do-while` hasta elegir la opción `0`.
- [ ] No se puede agregar un teléfono duplicado (lo probé intencionalmente con un caso de prueba).
- [ ] Eliminar un contacto también elimina su teléfono del `HashSet` (verifiqué agregando de nuevo ese mismo teléfono después de eliminarlo, y el sistema lo permitió).
- [ ] Los nombres de variables y métodos siguen camelCase.
- [ ] El archivo se llama igual que la clase pública (`AgendaContactos.java`).
- [ ] Probé la opción "Buscar" tanto con un nombre que existe como con uno que no existe.

## Rúbrica de evaluación

| Criterio | Puntos |
|---|---|
| El programa compila y ejecuta sin errores | 20 |
| Menú funcional con las 5 opciones (`do-while`) | 15 |
| Agregar contacto valida teléfonos duplicados con `HashSet` | 20 |
| Listar y buscar contacto funcionan correctamente | 20 |
| Eliminar contacto sincroniza ambas estructuras (`ArrayList` y `HashSet`) | 15 |
| Convenciones de nombres y legibilidad del código | 10 |
| **Total** | **100** |

## Forma de entrega

Sube tu archivo `AgendaContactos.java` a la plataforma del curso, en la sección correspondiente a esta sesión, antes de la fecha límite indicada en el aula virtual. Si tienes dudas sobre el mecanismo exacto de entrega, revisa el anuncio fijado en el curso o escribe al foro de preguntas.

---

## Errores comunes al resolver este ejercicio (revisa aquí antes de pedir ayuda)

- **`ArrayIndexOutOfBoundsException` al eliminar por nombre:** ocurre cuando eliminas de `nombres` pero olvidas eliminar en la **misma posición** de `telefonos` — si eliminas primero de uno y luego buscas en el otro con el índice ya desfasado, se rompe la sincronización. Guarda el índice **antes** de eliminar de cualquiera de las dos listas.
- **El `Scanner` "se salta" la lectura del nombre después de leer la opción del menú:** revisa que estés usando `sc.nextLine()` inmediatamente después de `sc.nextInt()`, tal como está en el esqueleto.
- **El `HashSet` no detecta el duplicado:** revisa que estés comparando el `String` exactamente como se guardó (con los mismos guiones/espacios) — a diferencia de `equalsIgnoreCase`, el `HashSet` de `String` es sensible a mayúsculas/minúsculas y a cualquier diferencia de caracteres.

## Dudas para la próxima sesión

Usa este espacio para anotar cualquier cosa que no lograste resolver en los 10 minutos indicados al inicio de la guía, y tráelo a la siguiente sesión o al foro del curso:

```
1. ...
2. ...
```

## Recursos adicionales (opcionales)

- Documentación oficial de `ArrayList`: busca "java.util.ArrayList" en `docs.oracle.com`.
- Documentación oficial de `HashMap`: busca "java.util.HashMap" en `docs.oracle.com`.
- Practica adicional (opcional, no evaluada): en el mismo programa, intenta ordenar el listado de contactos alfabéticamente antes de mostrarlo usando `Collections.sort(nombres)`.

---

## Respuestas — Autoevaluación 1

1. Índice 7 (`length - 1` = `8 - 1`).
2. `ArrayIndexOutOfBoundsException`.
3. `datos.length` es una **propiedad** (sin paréntesis) y sí existe; `datos.length()` **no existe** en un arreglo — eso es lo que usan los `String` (`texto.length()`, con paréntesis, porque ahí sí es un método).
4. `String[] colores = {"rojo", "verde", "azul", "amarillo"};`

## Respuestas — Autoevaluación 2

1. `Set` — porque la restricción principal es "sin duplicados", y el orden no importa.
2. `List` — porque se permiten repetidos y el orden de inserción (orden cronológico de compra) sí importa.
3. `Map` — porque necesitas buscar rápidamente un valor (nombre) a partir de una clave única (DUI).
4. El mapa tiene **1 elemento** (la clave `"A"` es única). El valor final de `"A"` es `2`, porque el segundo `put` sobrescribe al primero.
