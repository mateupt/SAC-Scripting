# SAC Advanced Formulas - Sintaxis Completa

## 1. CONFIGURACION (CONFIG)

Se pone al inicio del script. Controla el comportamiento global.

```
// Define la jerarquia temporal (calendario o fiscal)
CONFIG.TIME_HIERARCHY = CALENDARYEAR
CONFIG.TIME_HIERARCHY = FISCALYEAR

// Trata celdas vacias (unbooked) como 0
// Por defecto OFF: las celdas vacias se ignoran
CONFIG.GENERATE_UNBOOKED_DATA = ON

// Respetar reglas de signo segun tipo de cuenta
// Por defecto OFF: todo se calcula en valor absoluto
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON

// Elegir jerarquia activa para una dimension
CONFIG.HIERARCHY [d/Region].[h/Geographic]

// Offset de zona horaria para TODAY()
CONFIG.TIME_ZONE_OFFSET = +1

// Incluir miembros no asignados a ninguna jerarquia
CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY [d/Product]
```

| CONFIG | Efecto | Defecto |
|--------|--------|---------|
| `TIME_HIERARCHY` | Calendario vs fiscal para PREVIOUS/NEXT | CALENDARYEAR |
| `GENERATE_UNBOOKED_DATA` | Celdas vacias = 0 en vez de NULL | OFF |
| `FLIPPING_SIGN_ACCORDING_ACCTYPE` | Expense como negativo (suma en vez de restar) | OFF |
| `HIERARCHY` | Selecciona jerarquia activa | Jerarquia por defecto |
| `TIME_ZONE_OFFSET` | Ajusta TODAY() a tu zona horaria | UTC (0) |
| `INCLUDE_MEMBERS_NOT_IN_HIERARCHY` | Expande scope a miembros sin jerarquia | No incluidos |

---

## 2. MEMBERSET - Definir Scope

Restringe sobre que miembros actua el script.
Si no se define, usa TODOS los miembros del modelo.

```
// Miembros especificos
MEMBERSET [d/Account] = ("Revenue", "Cost", "Profit")

// Rango (solo para fechas y jerarquias ordenadas)
MEMBERSET [d/Date] = "202601" TO "202612"

// Filtrar por atributo de dimension
MEMBERSET [d/Account].[p/AccountType] = "INC"

// Excluir miembros (todos menos los indicados)
MEMBERSET [d/Company] = NOT ("CORP_HQ")

// Usando parametro de la Data Action
MEMBERSET [d/Date] = %PARAM_DATE%

// Miembros hoja de un nodo de jerarquia
MEMBERSET [d/CostCenter] = BASEMEMBER([d/CostCenter].[h/Standard], "OPERATIONS")
```

| Sintaxis | Que filtra |
|----------|-----------|
| `= ("A", "B")` | Lista explicita de miembros |
| `= "202601" TO "202612"` | Rango inclusivo |
| `= NOT ("X")` | Todos excepto X |
| `.[p/Attr] = "val"` | Filtro por atributo/propiedad |
| `= %PARAM%` | Valor dinamico (parametro) |
| `= BASEMEMBER([d/Dim].[h/Hier], "Parent")` | Hojas debajo de un nodo padre |

**IMPORTANTE**: MEMBERSET restringe DONDE ESCRIBE DATA(), pero RESULTLOOKUP
puede leer FUERA del MEMBERSET. Ejemplo:
```
MEMBERSET [d/Version] = "Plan"
DATA() = RESULTLOOKUP([d/Version] = "Actual")  // lee de Actual, escribe en Plan
```

---

## 3. VARIABLES

Almacenamiento temporal durante la ejecucion. No se persisten en el modelo.

```
// Variable miembro (referencia a un miembro virtual de una dimension)
VARIABLEMEMBER #TempRevenue OF [d/Account]

// Variable numerica entera (por defecto = 0)
INTEGER @counter

// Variable numerica decimal (por defecto = 0.0)
FLOAT @ratio
```

**Reglas:**
- Variables miembro empiezan con `#`
- Variables numericas empiezan con `@`
- La dimension debe existir en el modelo
- No se soporta la dimension Version para VARIABLEMEMBER

### Diferencia entre tipos de variable

| Tipo | Prefijo | Vive en | Tiene dimensiones | Uso con DATA/RESULTLOOKUP |
|------|---------|---------|-------------------|---------------------------|
| `VARIABLEMEMBER` | `#` | Fila temporal del modelo | Si | Si |
| `INTEGER` | `@` | Memoria del script | No (escalar) | No |
| `FLOAT` | `@` | Memoria del script | No (escalar) | No |

### Uso de variables miembro

```
VARIABLEMEMBER #Backup OF [d/Account]

// Guardar datos temporalmente
DATA([d/Account] = #Backup) = RESULTLOOKUP([d/Account] = "Revenue")

// Usar los datos guardados despues
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = #Backup) * 1.1
```

### Uso de variables numericas

```
INTEGER @i
FLOAT @rate

@rate = 0.05

FOR @i = 1 TO 12 STEP 1
    @rate = @rate + 0.001
ENDFOR
```

---

## 4. DATA() - Escribir datos

Es la funcion para ASIGNAR valores a celdas.

```
// Escribir en la celda actual (misma interseccion de dimensiones)
DATA() = 100

// Escribir en un miembro especifico
DATA([d/Account] = "Profit") = 500

// Escribir en multiples dimensiones
DATA([d/Account] = "Revenue", [d/Date] = "202601") = 1000

// Borrar datos (poner a NULL)
DATA() = NULL

// Calcular y escribir
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

### DATA.APPEND() - Escribir sin limpiar scope

```
// DATA() normal: limpia el scope objetivo, luego escribe
// DATA.APPEND(): NO limpia, anade valores a lo que ya existe

DATA.APPEND([d/Account] = "Revenue") =
    RESULTLOOKUP([d/Account] = "NewRevStream")
```

### DELETE() - Borrar celdas especificas

```
// Borra solo las celdas que coinciden, sin tocar el resto
DELETE([d/Account] = "Disposal", [d/Flow] = "OPEN")
```

| Funcion | Equivalente | Limpia scope antes | Escribe |
|---------|------------|-------------------|---------|
| `DATA()` | TRUNCATE + INSERT | Si | Si |
| `DATA.APPEND()` | INSERT | No | Si |
| `DATA() = NULL` | UPDATE SET NULL | - | Si (NULL) |
| `DELETE()` | DELETE WHERE | - | No (borra) |

---

## 5. RESULTLOOKUP() - Leer datos

Lee el valor de una celda. Devuelve el valor numerico.
Si la celda esta vacia, devuelve NULL (y DATA no escribe nada).

```
// Leer celda actual
RESULTLOOKUP()

// Leer un miembro especifico
RESULTLOOKUP([d/Account] = "Revenue")

// Leer periodo anterior
RESULTLOOKUP([d/Date] = PREVIOUS(1))

// Leer 12 periodos atras (mismo mes, año anterior)
RESULTLOOKUP([d/Date] = PREVIOUS(12))

// Leer periodo siguiente
RESULTLOOKUP([d/Date] = NEXT(1))

// Leer de miembro no asignado
RESULTLOOKUP([d/Currency] = "#")

// Combinar varias dimensiones
RESULTLOOKUP([d/Account] = "Price", [d/Version] = "Actual")
```

### ATTRIBUTE() - Leer propiedades de dimension

```
// Lee el valor de una propiedad/atributo de un miembro de dimension
ATTRIBUTE([d/Product].[p/Volume])                    // propiedad del miembro actual
ATTRIBUTE([d/Equipment].[p/Useful_Life])              // vida util del equipo
ATTRIBUTE([d/Account].[p/AccountType])                // tipo de cuenta (INC, EXP, AST, LEQ)

// Usar en condiciones
IF ATTRIBUTE([d/Account].[p/AccountType]) = "INC" THEN ...

// Usar en calculos (si es numerico)
DATA() = RESULTLOOKUP() / ATTRIBUTE([d/Equipment].[p/Useful_Life])

// Mapeo de cuentas via propiedad (patron "Sister")
DATA([d/Account] = [d/Account].[p/Sister]) = RESULTLOOKUP()
```

### CARRYFORWARD() - Arrastre de saldos optimizado

```
// Reemplaza FOREACH loops para patrones opening/closing
DATA() = CARRYFORWARD([d/Flow], "OPENING", "CLOSING",
    "HIRES", "CLOSING" - "OPENING" - "TERMINATIONS")
```

### ELIMMEMBER() - Eliminacion intercompany

```
// Encuentra el miembro de eliminacion en la jerarquia de consolidacion
ELIMMEMBER(
    [d/Entity].[h/Consolidation],    // jerarquia
    [d/Entity],                       // entidad actual
    [d/Interco_Entity],              // contraparte
    [d/Entity].[p/Elimination] = "Y" // filtro
)
```

---

## 6. IF / ELSEIF / ELSE / ENDIF - Condicionales

```
// IF simple
IF [d/Company] = "ES" THEN
    DATA() = RESULTLOOKUP() * 1.21
ENDIF

// IF con ELSE
IF [d/Account] = "Revenue" THEN
    DATA() = RESULTLOOKUP() * 1.1
ELSE
    DATA() = RESULTLOOKUP() * 1.05
ENDIF

// IF con ELSEIF
IF [d/Region] = "Europe" THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Profit") * 0.25
ELSEIF [d/Region] = "US" THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Profit") * 0.21
ELSE
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Profit") * 0.30
ENDIF

// Comparar con valores numericos
IF RESULTLOOKUP([d/Account] = "Revenue") > 10000 THEN
    DATA([d/Account] = "Bonus") = 500
ENDIF

// Comparar con NULL (celda vacia)
IF RESULTLOOKUP() <> NULL THEN
    DATA() = RESULTLOOKUP() * 1.1
ENDIF

// Comparar atributo de dimension
IF ATTRIBUTE([d/Account].[p/AccountType]) = "EXP" THEN
    DATA() = RESULTLOOKUP() * 1.03
ENDIF
```

**Operadores de comparacion:** `=`, `<>` (o `!=`), `>`, `<`, `>=`, `<=`
**Operadores logicos:** `AND`, `OR`, `NOT`

---

## 7. FOREACH / ENDFOR - Bucles sobre dimensiones

Itera sobre los miembros de una dimension.

```
// Basico: iterar sobre todos los meses
MEMBERSET [d/Date] = "202601" TO "202612"

FOREACH [d/Date]
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR

// Con orden
FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) + 100
ENDFOR

// FOREACH.BOOKED: solo itera sobre celdas con datos (mejor rendimiento)
FOREACH.BOOKED [d/Product]
    DATA([d/Account] = "Margin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// Nested: IF fuera de FOREACH (patron recomendado para rendimiento)
IF [d/Account] = "Revenue" THEN
    FOREACH [d/Date] ASC
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.05
    ENDFOR
ENDIF

// FOREACH con BREAK
FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.1
    IF RESULTLOOKUP() > 50000 THEN
        BREAK
    ENDIF
ENDFOR
```

---

## 8. FOR numerico / ENDFOR

Bucle con contador numerico.

```
INTEGER @i

FOR @i = 1 TO 12 STEP 1
    // hace algo 12 veces
ENDFOR

// Con STEP
FOR @i = 0 TO 100 STEP 10
    // 0, 10, 20, 30 ... 100
ENDFOR
```

---

## 9. LINK() - Leer de otro modelo

Lee datos de un modelo distinto al modelo por defecto de la Data Action.
Solo lectura: no puedes ESCRIBIR en el modelo linkeado.

```
// Definir el link
LINK [Model2] = MODEL("t.local/ModeloFuente")

// Leer dato del otro modelo
DATA([d/Account] = "Headcount") =
    LINK [Model2] RESULTLOOKUP([d/Account] = "FTE_Count")

// Con mapeo de dimensiones
DATA([d/Account] = "ExtCost") =
    LINK [Model2] RESULTLOOKUP(
        [d/CostCenter] = [d/Department],
        [d/Account] = "TotalCost"
    )
```

### MODEL / ENDMODEL - Scope del modelo linkeado

```
LINK [Sales] = MODEL("t.local/Sales_Planning")

MODEL [Sales]
    MEMBERSET [d/Product] = ("Prod_A", "Prod_B")
    AGGREGATE_DIMENSIONS = [d/Region]
    AGGREGATE_WRITETO [d/Region] = "Total"
ENDMODEL

DATA([d/Account] = "TotalUnits") =
    LINK [Sales] RESULTLOOKUP([d/Account] = "Units", [d/Region] = "Total")
```

---

## 10. Funciones de agregacion

```
// Definir dimensiones a agregar
AGGREGATE_DIMENSIONS = [d/Company], [d/Product]

// Definir donde escribir el resultado agregado
AGGREGATE_WRITETO [d/Company] = "Group"
AGGREGATE_WRITETO [d/Product] = "#"

// Sin AGGREGATE_WRITETO, los resultados se escriben en celdas hoja (leaf)
```

### BASEMEMBER() - Miembros hoja de una jerarquia

```
// Devuelve todos los miembros hoja debajo de un nodo padre
MEMBERSET [d/CostCenter] = BASEMEMBER([d/CostCenter].[h/Standard], "OPERATIONS")

// Multiples padres
MEMBERSET [d/CostCenter] = BASEMEMBER([d/CostCenter].[h/Standard], "OPERATIONS", "ADMIN")
```

---

## 11. Operadores aritmeticos

```
DATA() = RESULTLOOKUP() + 100          // Suma
DATA() = RESULTLOOKUP() - 50           // Resta
DATA() = RESULTLOOKUP() * 1.1          // Multiplicacion
DATA() = RESULTLOOKUP() / 12           // Division
DATA() = RESULTLOOKUP() % 3            // Modulo (resto)

// Combinar
DATA([d/Account] = "AvgMonthly") =
    RESULTLOOKUP([d/Account] = "AnnualTotal") / 12
```

### Funciones matematicas

```
ABS(value)                    // Valor absoluto
ROUND(value, decimals)        // Redondear (decimals negativo = centenas, miles...)
TRUNC(value, decimals)        // Truncar (no redondea)
FLOOR(value)                  // Redondear hacia abajo al entero
CEIL(value)                   // Redondear hacia arriba al entero
SQRT(value)                   // Raiz cuadrada
POWER(base, exponent)         // Potencia
LOG(value)                    // Logaritmo natural (ln)
LOG10(value)                  // Logaritmo base 10
MOD(value, divisor)           // Modulo/resto (funcion, equivale a %)
INT(value)                    // Convertir a entero
FLOAT(value)                  // Convertir a decimal
```

---

## 12. Funciones de tiempo

### Navegacion

```
PREVIOUS(n)         // n periodos hacia atras
NEXT(n)             // n periodos hacia adelante
FIRST()             // Primer periodo del año (enero en calendario)
LAST()              // Ultimo periodo del año (diciembre en calendario)
PREYEARLAST()       // Ultimo periodo del año ANTERIOR
```

### Informacion de fecha

```
TODAY()             // Fecha actual del sistema (YYYYMMDD, UTC)
DAY()               // Dia del periodo actual
MONTH()             // Mes del periodo actual (1-12)
YEAR()              // Año del periodo actual
WEEK()              // Numero de semana (ISO o CALENDAR)
DAYSINMONTH()       // Dias en el mes actual (28-31)
DAYSINYEAR()        // Dias en el año actual (365 o 366)
```

### Calculos entre fechas

```
DATERATIO(start, end)           // Ratio de solapamiento entre fechas y periodo actual
DATEDIFF(date1, date2, mode)    // Diferencia entre fechas (CalendarDiff, Floor, Ceiling)
PERIOD(date)                    // Convierte fecha a miembro de tiempo
```

### Ejemplos

```
RESULTLOOKUP([d/Date] = PREVIOUS(1))    // Mes anterior
RESULTLOOKUP([d/Date] = PREVIOUS(12))   // Mismo mes, año anterior
RESULTLOOKUP([d/Date] = NEXT(1))        // Mes siguiente
RESULTLOOKUP([d/Date] = FIRST())        // Enero del mismo año
RESULTLOOKUP([d/Date] = LAST())         // Diciembre del mismo año
RESULTLOOKUP([d/Date] = PREYEARLAST())  // Diciembre del año pasado

// Prorrateo por dias
DATA() = RESULTLOOKUP([d/Account] = "Annual") * DAYSINMONTH() / DAYSINYEAR()

// Ratio de solapamiento (contrato del 15-mar al 31-dic)
DATA() = RESULTLOOKUP() * DATERATIO("20260315", "20261231")
```

---

## 13. Referencia de dimensiones

```
[d/DimensionName]                    // Dimension
[d/DimensionName].[p/PropertyName]   // Atributo/propiedad
[d/DimensionName].[h/HierarchyName]  // Jerarquia

// Ejemplos
[d/Account]                          // Dimension Account
[d/Account].[p/AccountType]          // Tipo de cuenta (INC, EXP, AST, LEQ)
[d/Account].[p/Sister]               // Atributo personalizado
[d/Region].[h/Geographic]            // Jerarquia geografica
[d/Version].[p/StartDate]            // Fecha inicio de la version
[d/Version].[p/EndDate]              // Fecha fin de la version
```

---

## 14. Parametros

Los parametros se definen en la UI de la Data Action y se referencian con `%`.

```
// Usar parametro en MEMBERSET
MEMBERSET [d/Date] = %TARGET_DATE%
MEMBERSET [d/Version] = %VERSION%

// Usar parametro en logica
DATA([d/Version] = %TARGET_VERSION%) =
    RESULTLOOKUP([d/Version] = %SOURCE_VERSION%)

// Parametro numerico
DATA() = RESULTLOOKUP() * %GROWTH_RATE%
```

Los parametros se pueden pasar desde:
- Prompt al usuario al ejecutar la Data Action
- JavaScript de Analytic Application (dinamico)
- Valor fijo configurado en la UI

---

## 15. Tabla resumen de todas las funciones

| Categoria | Funcion | Que hace |
|-----------|---------|----------|
| **Escritura** | `DATA()` | Escribe valores (limpia scope primero) |
| **Escritura** | `DATA.APPEND()` | Escribe valores (sin limpiar) |
| **Escritura** | `DELETE()` | Borra celdas especificas |
| **Lectura** | `RESULTLOOKUP()` | Lee valor de una celda |
| **Lectura** | `ATTRIBUTE()` | Lee propiedad de un miembro de dimension |
| **Lectura** | `CARRYFORWARD()` | Arrastre de saldos optimizado |
| **Lectura** | `ELIMMEMBER()` | Encuentra miembro de eliminacion IC |
| **Scope** | `MEMBERSET` | Filtra filas (WHERE) |
| **Scope** | `BASEMEMBER()` | Hojas de un nodo de jerarquia |
| **Agregacion** | `AGGREGATE_DIMENSIONS` | Dimensiones a agregar |
| **Agregacion** | `AGGREGATE_WRITETO` | Donde escribir el agregado |
| **Cross-model** | `LINK` / `MODEL` | Leer de otro modelo |
| **Tiempo** | `PREVIOUS(n)` / `NEXT(n)` | Navegar periodos |
| **Tiempo** | `FIRST()` / `LAST()` | Primer/ultimo periodo del año |
| **Tiempo** | `PREYEARLAST()` | Ultimo periodo del año anterior |
| **Tiempo** | `TODAY()` | Fecha actual UTC |
| **Tiempo** | `DAY/MONTH/YEAR/WEEK()` | Extraer componentes de fecha |
| **Tiempo** | `DAYSINMONTH/DAYSINYEAR()` | Dias en periodo |
| **Tiempo** | `DATERATIO()` | Solapamiento entre fechas |
| **Tiempo** | `DATEDIFF()` | Diferencia entre fechas |
| **Matematicas** | `ABS/ROUND/TRUNC/FLOOR/CEIL` | Redondeo |
| **Matematicas** | `SQRT/POWER/LOG/LOG10` | Potencias y logaritmos |
| **Matematicas** | `MOD/INT/FLOAT` | Modulo y conversion |
| **Variables** | `VARIABLEMEMBER #` | Fila temporal en el modelo |
| **Variables** | `INTEGER @` | Entero en memoria |
| **Variables** | `FLOAT @` | Decimal en memoria |
| **Control** | `IF/ELSEIF/ELSE/ENDIF` | Condicional |
| **Control** | `FOREACH/FOREACH.BOOKED` | Bucle sobre dimension |
| **Control** | `FOR @i/ENDFOR` | Bucle numerico |
| | `BREAK` | Salir del bucle |
| **Config** | `CONFIG.TIME_HIERARCHY` | Calendario vs fiscal |
| | `CONFIG.GENERATE_UNBOOKED_DATA` | Vacios = 0 |
| | `CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE` | Signo por tipo cuenta |
| | `CONFIG.HIERARCHY` | Jerarquia activa |
| | `CONFIG.TIME_ZONE_OFFSET` | Zona horaria |
| | `CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY` | Miembros sin jerarquia |
