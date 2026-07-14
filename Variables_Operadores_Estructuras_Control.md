# 1. Variables, tipos de datos, operadores y estructuras de control

## 1.1 Variables y tipos de datos

Java es **fuertemente tipado**: toda variable se declara con un tipo que no cambia.

### Tipos primitivos (8 en total)

| Tipo | Tamaño | Rango / uso | Valor por defecto |
|------|--------|-------------|--------------------|
| `byte` | 8 bits | -128 a 127 | 0 |
| `short` | 16 bits | -32,768 a 32,767 | 0 |
| `int` | 32 bits | ±2,147,483,647 | 0 |
| `long` | 64 bits | enteros muy grandes (sufijo `L`) | 0L |
| `float` | 32 bits | decimales, poca precisión (sufijo `f`) | 0.0f |
| `double` | 64 bits | decimales, precisión estándar | 0.0 |
| `char` | 16 bits | un carácter Unicode (comillas simples) | '\u0000' |
| `boolean` | 1 bit (lógico) | `true` / `false` | false |

### Tipos de referencia

`String`, arreglos, clases, interfaces, enums — no almacenan el valor directamente sino una referencia al objeto en el *heap*. Su valor por defecto es `null`.

```java
int edad = 21;
double precio = 19.99;
char inicial = 'J';
boolean activo = true;
String nombre = "Carlos";           // String es un objeto, no primitivo
long poblacionMundial = 8_100_000_000L; // guion bajo como separador visual (Java 7+)
var contador = 0;                    // inferencia de tipo local (Java 10+), sigue siendo fuertemente tipado
```

**Punto importante para el aula:** `var` no es tipado dinámico. El compilador infiere `int` en tiempo de compilación y a partir de ahí `contador` es `int` para siempre.

### Conversión de tipos (casting)

```java
int entero = 10;
double decimal = entero;        // ensanchamiento (widening) - automático, sin pérdida
double pi = 3.14159;
int truncado = (int) pi;        // estrechamiento (narrowing) - explícito, con pérdida de datos -> 3
```

## 1.2 Operadores

| Categoría | Operadores | Ejemplo |
|-----------|-----------|---------|
| Aritméticos | `+ - * / % ++ --` | `int r = 7 % 2; // 1` |
| Relacionales | `== != > < >= <=` | `edad >= 18` |
| Lógicos | `&& \|\| !` | `activo && saldo > 0` |
| Asignación | `= += -= *= /= %=` | `total += 10;` |
| Ternario | `condicion ? valorSiTrue : valorSiFalse` | `String s = (edad >= 18) ? "Mayor" : "Menor";` |
| Bit a bit | `& \| ^ ~ << >>` | poco usado en apps de negocio, mencionar solo brevemente |

**Trampa clásica que hay que remarcar:** `%` con `double` funciona distinto a lo esperado en algunos casos, y la división entre `int` trunca:

```java
int a = 7, b = 2;
System.out.println(a / b);       // 3 (división entera)
System.out.println((double) a / b); // 3.5 (al menos un operando double)
```

## 1.3 Estructuras de control

### Condicionales

```java
if (nota >= 7.0) {
    System.out.println("Aprobado");
} else if (nota >= 4.0) {
    System.out.println("Recuperación");
} else {
    System.out.println("Reprobado");
}
```

`switch` clásico vs `switch` moderno (Java 14+, disponible y recomendado en Java 21):

```java
// Clásico
switch (diaSemana) {
    case 1: System.out.println("Lunes"); break;
    case 2: System.out.println("Martes"); break;
    default: System.out.println("Otro día");
}

// Moderno con expresión (arrow) - evita el "fallthrough" accidental
String nombreDia = switch (diaSemana) {
    case 1 -> "Lunes";
    case 2 -> "Martes";
    case 6, 7 -> "Fin de semana";
    default -> "Otro día";
};
```

### Bucles

```java
for (int i = 1; i <= 5; i++) { System.out.println(i); }

int i = 0;
while (i < 5) { System.out.println(i); i++; }

int j = 0;
do {
    System.out.println(j);
    j++;
} while (j < 5);            // se ejecuta al menos una vez -> ideal para menús

for (int valor : new int[]{10, 20, 30}) {  // for-each
    System.out.println(valor);
}
```

## 1.4 Patrón de menú de consola con Scanner (base para TODOS los laboratorios del curso)

Este patrón se usará constantemente durante el módulo, así que se enseña formalmente aquí:

```java
import java.util.Scanner;

public class MenuBase {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int opcion;

        do {
            System.out.println("\n===== MENÚ =====");
            System.out.println("1. Opción A");
            System.out.println("2. Opción B");
            System.out.println("0. Salir");
            System.out.print("Seleccione una opción: ");

            while (!sc.hasNextInt()) {          // validación de entrada no numérica
                System.out.print("Entrada inválida, ingrese un número: ");
                sc.next();
            }
            opcion = sc.nextInt();

            switch (opcion) {
                case 1 -> System.out.println("Ejecutando Opción A...");
                case 2 -> System.out.println("Ejecutando Opción B...");
                case 0 -> System.out.println("Saliendo...");
                default -> System.out.println("Opción no válida");
            }
        } while (opcion != 0);

        sc.close();
    }
}
```

**Detalle crítico que siempre falla en el aula:** mezclar `nextInt()` con `nextLine()` deja un `\n` pendiente en el buffer que rompe la siguiente lectura de texto. Se enseña la solución desde ya:

```java
int edad = sc.nextInt();
sc.nextLine();              // consumir el salto de línea pendiente
String nombre = sc.nextLine();
```

## 1.5 Laboratorio 1 — "Calculadora de nómina simple"

Construir un programa con menú (`do-while`) que:

1. Solicite nombre del empleado (`String`), horas trabajadas (`double`) y tarifa por hora (`double`).
2. Calcule salario bruto, aplique un descuento de ISSS (3%) y AFP (7.25%) — constantes declaradas con `final`.
3. Muestre un recibo formateado con `String.format` o `printf`, alineando montos a 2 decimales.
4. Repita el menú permitiendo calcular otro empleado o salir.

Validar que las horas y la tarifa sean números positivos (rechazar negativos con un mensaje, sin usar excepciones todavía — solo `if`).

## 1.6 Reto 1 (no tan común) — "Simulador de semáforo peatonal con temporizador manual"

Sin usar hilos (`Thread`) ni `Thread.sleep` (eso es prematuro), simular una máquina de estados de semáforo controlada por menú:

- Estado inicial: `ROJO`.
- El usuario "avanza el tiempo" presionando Enter (leyendo una línea con `Scanner`) — cada pulsación avanza un estado: `ROJO → VERDE → AMARILLO → ROJO...`
- El programa debe llevar un contador de cuántas veces ha estado en cada estado (`int` por estado) y mostrarlo al usuario en cada iteración.
- Usar `switch` con expresión (`->`) para decidir el siguiente estado a partir del estado actual (representado como `enum` es lo ideal, pero como los enums se ven en el archivo 4, aquí se resuelve con `String` o constantes `int` con nombres significativos — es intencional dejar sentir la "incomodidad" de no tener enums todavía, para que la clase de enums tenga más impacto).
- Debe rechazar cualquier entrada que no sea Enter/vacío o "salir".

> Objetivo pedagógico: practicar `switch`, control de flujo y conteo con estructuras simples, y sembrar la necesidad de un tipo mejor (enum) antes de enseñarlo.
