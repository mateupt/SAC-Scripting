# SAC Data Actions - Conceptos Base

## Que es una Data Action

Una Data Action es un conjunto de pasos (steps) que transforman datos dentro de
modelos de planificacion en SAP Analytics Cloud. Es el mecanismo principal para
hacer calculos, copias y transformaciones de datos en planning.

## Tipos de pasos en una Data Action

| Tipo | Que hace | Complejidad |
|------|----------|-------------|
| **Copy** | Copia datos dentro del mismo modelo | Baja (UI) |
| **Cross-Model Copy** | Copia datos entre dos modelos distintos | Media (UI + mapeo) |
| **Advanced Formula** | Script con logica completa (calculos, loops, condicionales) | Alta (scripting) |
| **Allocation** | Distribuye valores de una dimension a otra | Media |
| **Conversion** | Copia entre medidas con conversion de moneda | Media |
| **Embedded** | Ejecuta otra Data Action como sub-paso | Baja |

## El lenguaje de Advanced Formulas

**No es JavaScript.** Es un lenguaje propio de SAP, declarativo, con sintaxis
que recuerda a SQL y a las formulas del modelador de SAC.

Caracteristicas:
- Orientado a operaciones sobre celdas de datos multidimensionales
- Trabaja sobre dimensiones ([d/NombreDimension]) y miembros
- No tiene funciones genericas, objetos ni clases
- No tiene arrays ni estructuras de datos complejas
- El flujo es: definir scope → leer datos → calcular → escribir datos

### Advanced Formulas vs JavaScript en SAC

| | Advanced Formulas (Data Actions) | JavaScript (Analytic Apps) |
|--|----------------------------------|---------------------------|
| **Donde se usa** | Dentro de Data Actions | En Analytic Applications / Stories |
| **Para que** | Calcular y transformar datos del modelo | Controlar UI, widgets, eventos |
| **Accede a datos** | Directamente (RESULTLOOKUP, DATA) | Indirectamente (APIs de widgets) |
| **Sintaxis** | Propia de SAP (tipo SQL/formulas) | JavaScript estandar (TypeScript-like) |
| **Ejecucion** | En el backend sobre el modelo | En el frontend del navegador |

Son complementarios: el JavaScript de Analytic Apps puede **disparar** Data
Actions y pasarles parametros, pero la logica pesada de datos va en Advanced
Formulas.

## Estructura de un script Advanced Formula

```
// 1. CONFIGURACION (opcional)
CONFIG.TIME_HIERARCHY = CALENDARYEAR

// 2. SCOPE - definir sobre que datos trabajas
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "Cost")

// 3. VARIABLES (opcional)
VARIABLEMEMBER #TempValue OF [d/Account]

// 4. LOGICA - leer, calcular, escribir
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

## Conceptos clave

### Dimensiones
Se referencian con la notacion `[d/NombreDimension]`.
Ejemplo: `[d/Account]`, `[d/Date]`, `[d/Company]`, `[d/Product]`

### Miembros
Son los valores concretos de una dimension.
Ejemplo: `[d/Account] = "Revenue"` → el miembro "Revenue" de la dimension Account

### Scope
El scope define SOBRE QUE DATOS actua tu script. Se define con:
- La tabla de scope de la Data Action (en la UI)
- MEMBERSET dentro del script (sobreescribe la tabla de scope)

Si no defines MEMBERSET, el script actua sobre TODOS los miembros del modelo.

### Celdas
Una celda es la interseccion de un miembro de cada dimension.
Ejemplo: Account=Revenue, Date=202601, Company=ES → una celda concreta.
DATA() escribe en celdas. RESULTLOOKUP() lee de celdas.
