# Como Actua Cada Instruccion Sobre la Base de Datos

> Este documento explica QUE PASA EN EL MODELO (la base de datos multidimensional)
> cuando ejecutas cada instruccion. Con tablas antes/despues para cada operacion.

---

## EL MODELO COMO BASE DE DATOS

Un modelo de SAC Planning es una **base de datos multidimensional** (un cubo OLAP).
Piensa en una tabla gigante donde cada fila es una celda unica definida por la
combinacion de todas sus dimensiones:

```
| Account  | Date   | Company | Version | Product | VALOR   |
|----------|--------|---------|---------|---------|---------|
| Revenue  | 202601 | ES      | Plan    | Prod_A  | 10000   |
| Revenue  | 202601 | ES      | Plan    | Prod_B  | 8000    |
| Cost     | 202601 | ES      | Plan    | Prod_A  | 6000    |
| Cost     | 202601 | ES      | Plan    | Prod_B  | 5000    |
| Profit   | 202601 | ES      | Plan    | Prod_A  | (vacio) |
| Profit   | 202601 | ES      | Plan    | Prod_B  | (vacio) |
| Revenue  | 202602 | ES      | Plan    | Prod_A  | 12000   |
| ...      | ...    | ...     | ...     | ...     | ...     |
```

Cada instruccion de Advanced Formulas opera sobre esta tabla:
- **MEMBERSET** filtra filas (reduce el scope)
- **DATA()** escribe/modifica el campo VALOR de una o mas filas
- **RESULTLOOKUP()** lee el campo VALOR de una fila concreta
- **FOREACH** controla el orden en que se recorren las filas
- **Variables** crean columnas/filas temporales que no se guardan

---

## 1. MEMBERSET — Filtrar el scope

### Que hace en la base de datos

MEMBERSET es un **WHERE** de SQL. Reduce las filas sobre las que opera el script.

```
MEMBERSET [d/Date] = "202601" TO "202603"
MEMBERSET [d/Account] = ("Revenue", "Cost", "Profit")
MEMBERSET [d/Version] = "Plan"
```

**Equivalente mental en SQL:**
```sql
WHERE Date BETWEEN '202601' AND '202603'
  AND Account IN ('Revenue', 'Cost', 'Profit')
  AND Version = 'Plan'
```

### Efecto sobre la tabla

**Sin MEMBERSET** — el script ve TODAS las filas del modelo (peligroso, lento):
```
| Account  | Date   | Company | Version  | Product | VALOR |
|----------|--------|---------|----------|---------|-------|
| Revenue  | 202601 | ES      | Plan     | Prod_A  | 10000 |  ← en scope
| Revenue  | 202601 | ES      | Actual   | Prod_A  | 9500  |  ← en scope (!)
| Revenue  | 202601 | DE      | Plan     | Prod_A  | 7000  |  ← en scope
| Revenue  | 202507 | ES      | Plan     | Prod_A  | 8000  |  ← en scope (!)
| Tax      | 202601 | ES      | Plan     | Prod_A  | 2100  |  ← en scope (!)
  ... miles de filas mas ...
```

**Con MEMBERSET** — el script solo ve las filas que cumplen el filtro:
```
| Account  | Date   | Company | Version | Product | VALOR   |
|----------|--------|---------|---------|---------|---------|
| Revenue  | 202601 | ES      | Plan    | Prod_A  | 10000   |  ← en scope
| Revenue  | 202601 | ES      | Plan    | Prod_B  | 8000    |  ← en scope
| Cost     | 202601 | ES      | Plan    | Prod_A  | 6000    |  ← en scope
| Cost     | 202601 | ES      | Plan    | Prod_B  | 5000    |  ← en scope
| Profit   | 202601 | ES      | Plan    | Prod_A  | (vacio) |  ← en scope
| Profit   | 202601 | ES      | Plan    | Prod_B  | (vacio) |  ← en scope
| Revenue  | 202602 | ES      | Plan    | Prod_A  | 12000   |  ← en scope
| Revenue  | 202602 | ES      | Plan    | Prod_B  | 9000    |  ← en scope
  ... solo filas de Ene-Mar, Revenue/Cost/Profit, Plan ...
```

### Variantes de MEMBERSET

| Sintaxis | Que filtra | Equivalente SQL |
|----------|-----------|-----------------|
| `MEMBERSET [d/Account] = ("Revenue", "Cost")` | Solo esos miembros | `WHERE Account IN ('Revenue','Cost')` |
| `MEMBERSET [d/Date] = "202601" TO "202612"` | Rango inclusivo | `WHERE Date BETWEEN '202601' AND '202612'` |
| `MEMBERSET [d/Company] = NOT ("HQ")` | Todos menos HQ | `WHERE Company <> 'HQ'` |
| `MEMBERSET [d/Account].[p/AccType] = "INC"` | Filtro por atributo | `WHERE Account.Type = 'INC'` |
| `MEMBERSET [d/Date] = %PARAM%` | Valor dinamico | `WHERE Date = @param` |

### IMPORTANTE: MEMBERSET solo restringe el scope de ESCRITURA (DATA)

RESULTLOOKUP puede leer FUERA del MEMBERSET. Por ejemplo:

```
MEMBERSET [d/Version] = "Plan"

// Esto FUNCIONA: lee de Actual aunque el MEMBERSET dice "Plan"
DATA() = RESULTLOOKUP([d/Version] = "Actual")
```

El MEMBERSET limita DONDE escribes, no de donde lees.

---

## 2. DATA() — Escribir en la base de datos

### Que hace exactamente

DATA() es un **UPDATE** de SQL. Modifica el campo VALOR de las filas que coincidan.

### Caso 1: DATA() sin parametros — escribe en la celda actual

```
DATA() = 100
```

**Que pasa**: Para CADA fila dentro del scope, pone VALOR = 100.

```
ANTES:                                          DESPUES:
| Account | Date   | VALOR |                    | Account | Date   | VALOR |
|---------|--------|-------|                    |---------|--------|-------|
| Revenue | 202601 | 10000 |   ──DATA()=100──>  | Revenue | 202601 | 100   |
| Revenue | 202602 | 12000 |                    | Revenue | 202602 | 100   |
| Cost    | 202601 | 6000  |                    | Cost    | 202601 | 100   |
| Cost    | 202602 | 7000  |                    | Cost    | 202602 | 100   |
```

Cada fila del scope recibe el valor 100. No discrimina.

### Caso 2: DATA([d/Account] = "Profit") — escribe solo donde Account = Profit

```
DATA([d/Account] = "Profit") = 999
```

**Que pasa**: Para cada combinacion de las OTRAS dimensiones (Date, Company, Product...),
busca la fila donde Account = "Profit" y pone VALOR = 999.

```
ANTES:                                                  DESPUES:
| Account | Date   | Company | VALOR   |                | Account | Date   | Company | VALOR |
|---------|--------|---------|---------|                |---------|--------|---------|-------|
| Revenue | 202601 | ES      | 10000   |                | Revenue | 202601 | ES      | 10000 | (no tocada)
| Cost    | 202601 | ES      | 6000    |                | Cost    | 202601 | ES      | 6000  | (no tocada)
| Profit  | 202601 | ES      | (vacio) | ──999──>       | Profit  | 202601 | ES      | 999   | (escrita)
| Revenue | 202602 | ES      | 12000   |                | Revenue | 202602 | ES      | 12000 | (no tocada)
| Cost    | 202602 | ES      | 7000    |                | Cost    | 202602 | ES      | 7000  | (no tocada)
| Profit  | 202602 | ES      | (vacio) | ──999──>       | Profit  | 202602 | ES      | 999   | (escrita)
```

**La clave**: el parametro `[d/Account] = "Profit"` actua como un filtro adicional
dentro del scope. Las filas de Revenue y Cost no se tocan.

### Caso 3: DATA con calculo — donde todo se junta

```
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

**Que pasa paso a paso para la fila Profit/202601/ES**:

1. SAC esta "parado" en la interseccion: Date=202601, Company=ES
2. `RESULTLOOKUP([d/Account] = "Revenue")` → busca la fila Revenue/202601/ES → lee 10000
3. `RESULTLOOKUP([d/Account] = "Cost")` → busca la fila Cost/202601/ES → lee 6000
4. Calcula: 10000 - 6000 = 4000
5. `DATA([d/Account] = "Profit")` → busca la fila Profit/202601/ES → escribe 4000

```
ANTES:                                               DESPUES:
| Account | Date   | VALOR   |                       | Account | Date   | VALOR |
|---------|--------|---------|                       |---------|--------|-------|
| Revenue | 202601 | 10000   | ←── RESULTLOOKUP lee  | Revenue | 202601 | 10000 |
| Cost    | 202601 | 6000    | ←── RESULTLOOKUP lee  | Cost    | 202601 | 6000  |
| Profit  | 202601 | (vacio) | ←── DATA escribe      | Profit  | 202601 | 4000  |
| Revenue | 202602 | 12000   | ←── RESULTLOOKUP lee  | Revenue | 202602 | 12000 |
| Cost    | 202602 | 7000    | ←── RESULTLOOKUP lee  | Cost    | 202602 | 7000  |
| Profit  | 202602 | (vacio) | ←── DATA escribe      | Profit  | 202602 | 5000  |
```

SAC repite este proceso para CADA combinacion de Date x Company x Product.

### Caso 4: DATA() = NULL — borrar datos

```
DATA() = NULL
```

**Que pasa**: Elimina el valor de todas las filas del scope. La fila vuelve a
estado "unbooked" (no tiene dato). Es el equivalente de `DELETE FROM` en SQL.

```
ANTES:                          DESPUES:
| Account | Date   | VALOR |    | Account | Date   | VALOR   |
|---------|--------|-------|    |---------|--------|---------|
| Revenue | 202601 | 10000 |    | Revenue | 202601 | (vacio) |
| Cost    | 202601 | 6000  |    | Cost    | 202601 | (vacio) |
```

### Caso 5: DATA con multiples dimensiones fijadas

```
DATA([d/Account] = "Revenue", [d/Date] = "202601") = 50000
```

**Que pasa**: Fija Account=Revenue Y Date=202601. Solo escribe en las filas
que cumplan ambas condiciones dentro del scope.

```
ANTES:                                               DESPUES:
| Account | Date   | Company | VALOR |                | Account | Date   | Company | VALOR |
|---------|--------|---------|-------|                |---------|--------|---------|-------|
| Revenue | 202601 | ES      | 10000 | ──50000──>     | Revenue | 202601 | ES      | 50000 |
| Revenue | 202601 | DE      | 7000  | ──50000──>     | Revenue | 202601 | DE      | 50000 |
| Revenue | 202602 | ES      | 12000 | (no tocada)    | Revenue | 202602 | ES      | 12000 |
| Cost    | 202601 | ES      | 6000  | (no tocada)    | Cost    | 202601 | ES      | 6000  |
```

Solo cambian las filas donde Account=Revenue AND Date=202601.
Se repite para cada Company y Product del scope.

---

## 3. RESULTLOOKUP() — Leer de la base de datos

### Que hace exactamente

RESULTLOOKUP() es un **SELECT** de SQL. Lee el VALOR de una celda especifica.

### Mecanica interna: el "cursor"

SAC recorre todas las filas del scope una a una (como un cursor de base de datos).
En cada fila, tu estas "parado" en una interseccion concreta:

```
Iteracion actual: Account=Revenue, Date=202601, Company=ES, Version=Plan, Product=Prod_A
                  ↑ aqui estas "parado"
```

### RESULTLOOKUP() sin parametros — lee la celda donde estas parado

```
RESULTLOOKUP()
```

Equivale a: "dame el VALOR de la fila en la que estoy ahora mismo."

```
Estoy en: Account=Revenue, Date=202601, Company=ES
RESULTLOOKUP() → devuelve 10000
```

### RESULTLOOKUP([d/Account] = "Cost") — cambia UNA dimension, mantiene el resto

```
RESULTLOOKUP([d/Account] = "Cost")
```

Equivale a: "desde donde estoy, cambia Account a Cost y dame ese valor."

```
Estoy en: Account=Revenue, Date=202601, Company=ES
                    ↓ cambia solo Account
Busca:    Account=Cost,    Date=202601, Company=ES
                                                    → devuelve 6000
```

**Equivalente SQL:**
```sql
SELECT VALOR FROM modelo
WHERE Account = 'Cost'      -- dimension cambiada
  AND Date = '202601'       -- heredada de la posicion actual
  AND Company = 'ES'        -- heredada de la posicion actual
  AND Version = 'Plan'      -- heredada de la posicion actual
  AND Product = 'Prod_A'    -- heredada de la posicion actual
```

### RESULTLOOKUP con PREVIOUS/NEXT — navegar en el tiempo

```
RESULTLOOKUP([d/Date] = PREVIOUS(1))
```

Equivale a: "desde donde estoy, retrocede 1 periodo en Date y dame ese valor."

```
Estoy en: Account=Revenue, Date=202603, Company=ES
                                 ↓ retrocede 1
Busca:    Account=Revenue, Date=202602, Company=ES
                                                    → devuelve 12000
```

```
RESULTLOOKUP([d/Date] = PREVIOUS(12))
```

Equivale a: "retrocede 12 periodos (mismo mes, año anterior)."

```
Estoy en: Account=Revenue, Date=202603, Company=ES
                                 ↓ retrocede 12
Busca:    Account=Revenue, Date=202503, Company=ES
                                                    → devuelve 9000
```

### RESULTLOOKUP con multiples dimensiones cambiadas

```
RESULTLOOKUP([d/Account] = "Cost", [d/Version] = "Actual")
```

Equivale a: "cambia Account a Cost Y Version a Actual, mantiene Date y Company."

```
Estoy en: Account=Revenue, Date=202601, Company=ES, Version=Plan
                    ↓                                    ↓
Busca:    Account=Cost,    Date=202601, Company=ES, Version=Actual
                                                    → devuelve 5800
```

### Que devuelve si la celda esta vacia

| Situacion | CONFIG.GENERATE_UNBOOKED_DATA | Devuelve | Efecto en DATA() |
|-----------|-----------------|----------|------------------|
| Celda vacia | OFF (defecto) | NULL | DATA no escribe nada |
| Celda vacia | ON | 0 | DATA escribe 0 (o el resultado del calculo) |
| Celda con valor | cualquiera | el valor | DATA escribe normalmente |

**Ejemplo del problema con NULL:**

```
// Revenue de 202601 esta vacio (no hay dato)
DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
//                          NULL * 0.21 = NULL
//                          DATA recibe NULL → NO escribe nada
//                          Tax queda vacio
```

---

## 4. VARIABLEMEMBER (#) — Crear filas temporales

### Que hace en la base de datos

Crea un **miembro virtual** en una dimension. Es como anadir una fila temporal
a la tabla que solo existe durante la ejecucion del script. No se guarda.

```
VARIABLEMEMBER #TempRevenue OF [d/Account]
```

**Que pasa en la tabla:**

```
ANTES (tabla real):
| Account  | Date   | VALOR |
|----------|--------|-------|
| Revenue  | 202601 | 10000 |
| Cost     | 202601 | 6000  |
| Profit   | 202601 | 4000  |

DURANTE la ejecucion (tabla virtual):
| Account      | Date   | VALOR   |
|--------------|--------|---------|
| Revenue      | 202601 | 10000   |
| Cost         | 202601 | 6000    |
| Profit       | 202601 | 4000    |
| #TempRevenue | 202601 | (vacio) |  ← fila temporal, no existe en el modelo

DESPUES de la ejecucion:
| Account  | Date   | VALOR |
|----------|--------|-------|
| Revenue  | 202601 | 10000 |
| Cost     | 202601 | 6000  |
| Profit   | 202601 | 4000  |
                                    ← #TempRevenue desaparece
```

### Para que sirve: evitar lecturas multiples

**Sin variable** — lee Revenue del modelo 3 veces (3 consultas a la base de datos):

```
DATA([d/Account] = "Tax")   = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
DATA([d/Account] = "Net")   = RESULTLOOKUP([d/Account] = "Revenue") * 0.79
DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = "Revenue") * 0.02
```

```
Ejecucion interna:
  Paso 1: SELECT VALOR FROM modelo WHERE Account='Revenue' AND ... → 10000
          UPDATE modelo SET VALOR=2100 WHERE Account='Tax' AND ...
  Paso 2: SELECT VALOR FROM modelo WHERE Account='Revenue' AND ... → 10000 (otra vez)
          UPDATE modelo SET VALOR=7900 WHERE Account='Net' AND ...
  Paso 3: SELECT VALOR FROM modelo WHERE Account='Revenue' AND ... → 10000 (otra vez)
          UPDATE modelo SET VALOR=200 WHERE Account='Bonus' AND ...
```

**Con variable** — lee Revenue 1 vez, guarda en memoria, lee de memoria 3 veces:

```
VARIABLEMEMBER #Rev OF [d/Account]

DATA([d/Account] = #Rev) = RESULTLOOKUP([d/Account] = "Revenue")

DATA([d/Account] = "Tax")   = RESULTLOOKUP([d/Account] = #Rev) * 0.21
DATA([d/Account] = "Net")   = RESULTLOOKUP([d/Account] = #Rev) * 0.79
DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = #Rev) * 0.02
```

```
Ejecucion interna:
  Paso 1: SELECT VALOR FROM modelo WHERE Account='Revenue' AND ... → 10000
          INSERT INTO memoria_temporal (Account=#Rev, ..., VALOR=10000)
  Paso 2: SELECT VALOR FROM memoria_temporal WHERE Account=#Rev → 10000 (de memoria, rapido)
          UPDATE modelo SET VALOR=2100 WHERE Account='Tax' AND ...
  Paso 3: SELECT VALOR FROM memoria_temporal WHERE Account=#Rev → 10000 (de memoria)
          UPDATE modelo SET VALOR=7900 WHERE Account='Net' AND ...
  Paso 4: SELECT VALOR FROM memoria_temporal WHERE Account=#Rev → 10000 (de memoria)
          UPDATE modelo SET VALOR=200 WHERE Account='Bonus' AND ...
```

### Caso de uso avanzado: backup antes de sobreescribir

```
VARIABLEMEMBER #Backup OF [d/Account]

// 1. Guardo Revenue original en #Backup
DATA([d/Account] = #Backup) = RESULTLOOKUP([d/Account] = "Revenue")

// 2. Sobreescribo Revenue con un nuevo calculo
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = #Backup) * 1.1
```

```
Paso 1 (backup):
| Account  | Date   | VALOR |
|----------|--------|-------|
| Revenue  | 202601 | 10000 | ──lectura──┐
| #Backup  | 202601 | 10000 | ←──────────┘  copia a temporal

Paso 2 (sobreescribir):
| Account  | Date   | VALOR |
|----------|--------|-------|
| Revenue  | 202601 | 11000 | ← 10000 * 1.1 (lee de #Backup, no del Revenue ya cambiado)
| #Backup  | 202601 | 10000 | ──lectura──> 10000 * 1.1 = 11000
```

Sin #Backup, si intentas `DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Revenue") * 1.1`,
leerias el Revenue que acabas de sobreescribir, no el original.

---

## 5. INTEGER (@) y FLOAT (@) — Contadores y calculos intermedios

### Que hace en la base de datos

**Nada.** Las variables numericas NO existen en la tabla del modelo. Son variables
de memoria pura que solo viven en el script.

```
INTEGER @counter       // Crea una variable entera = 0
FLOAT @ratio           // Crea una variable decimal = 0.0
```

### Diferencia clave con VARIABLEMEMBER

| | VARIABLEMEMBER (#) | INTEGER/FLOAT (@) |
|--|---------------------|--------------------------|
| Donde vive | Fila temporal en la tabla del modelo | Solo en memoria del script |
| Tiene dimensiones | Si (Date, Company, etc.) | No, es un escalar |
| Se usa con DATA/RESULTLOOKUP | Si | No |
| Para que sirve | Guardar datos multidimensionales | Contadores, ratios, flags |

### Ejemplo: @counter como iterador

```
INTEGER @i

FOR @i = 1 TO 12 STEP 1
    // @i vale 1, luego 2, luego 3... hasta 12
    // Puedes usar @i en condiciones pero NO en DATA/RESULTLOOKUP directamente
ENDFOR
```

**Que pasa en memoria:**

```
Iteracion 1: @i = 1  → ejecuta bloque
Iteracion 2: @i = 2  → ejecuta bloque
Iteracion 3: @i = 3  → ejecuta bloque
...
Iteracion 12: @i = 12 → ejecuta bloque
Fin: @i se destruye
```

### Ejemplo: @ratio para calculo intermedio

```
FLOAT @growthRate

// Supongamos que calculas un ratio complejo
// @growthRate NO puede almacenar valores por celda, es UN SOLO NUMERO

// Uso tipico: como constante calculada
// NOTA: no puedes asignarle el resultado de RESULTLOOKUP directamente
// Los @variables se usan principalmente en bucles FOR y condiciones
```

---

## 6. FOREACH — Controlar el orden de recorrido

### Que hace en la base de datos

FOREACH NO modifica datos. Controla el **orden** en que SAC recorre las filas.

### Sin FOREACH — ejecucion paralela

```
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

**Que pasa**: SAC calcula TODAS las celdas de Profit a la vez (en paralelo).
El orden no importa porque ningun calculo depende de otro.

```
Ejecucion (conceptual):
  Thread 1: Profit/202601/ES = Revenue/202601/ES - Cost/202601/ES
  Thread 2: Profit/202602/ES = Revenue/202602/ES - Cost/202602/ES
  Thread 3: Profit/202601/DE = Revenue/202601/DE - Cost/202601/DE
  ... todos a la vez ...
```

### Con FOREACH — ejecucion secuencial

```
FOREACH [d/Date] ASC
    DATA([d/Account] = "Cumulative") =
        RESULTLOOKUP([d/Account] = "Cumulative", [d/Date] = PREVIOUS(1)) +
        RESULTLOOKUP([d/Account] = "Sales")
ENDFOR
```

**Que pasa**: SAC procesa las filas una dimension a la vez, en orden.

```
Iteracion 1 — Date=202601:
  Para CADA Company y Product del scope:
    Cumulative/202601 = Cumulative/202512 + Sales/202601

  | Account    | Date   | VALOR |
  |------------|--------|-------|
  | Sales      | 202601 | 100   | ← leido
  | Cumulative | 202512 | 950   | ← leido (mes anterior, ya existia)
  | Cumulative | 202601 | 1050  | ← ESCRITO

Iteracion 2 — Date=202602:
  Para CADA Company y Product del scope:
    Cumulative/202602 = Cumulative/202601 + Sales/202602
                        ↑ lee el 1050 que ACABA de escribir en iteracion 1

  | Account    | Date   | VALOR |
  |------------|--------|-------|
  | Sales      | 202602 | 150   | ← leido
  | Cumulative | 202601 | 1050  | ← leido (recien escrito!)
  | Cumulative | 202602 | 1200  | ← ESCRITO

Iteracion 3 — Date=202603:
  Cumulative/202603 = Cumulative/202602 (1200) + Sales/202603
  ... y asi sucesivamente ...
```

**SIN FOREACH** este calculo fallaria:

```
Ejecucion paralela (INCORRECTA):
  Cumulative/202601 = Cumulative/202512 + Sales/202601  → OK (lee dato existente)
  Cumulative/202602 = Cumulative/202601 + Sales/202602  → FALLO (202601 aun no calculado)
  Cumulative/202603 = Cumulative/202602 + Sales/202603  → FALLO (202602 aun no calculado)
```

### FOREACH.BOOKED — solo filas con datos

```
FOREACH.BOOKED [d/Product]
    DATA([d/Account] = "Margin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")
ENDFOR
```

**Diferencia en la base de datos:**

```
Tabla del modelo (500 productos, solo 3 con datos):
| Product | Account | VALOR |
|---------|---------|-------|
| Prod_A  | Revenue | 10000 |  ← tiene dato
| Prod_A  | Cost    | 6000  |
| Prod_B  | Revenue | 8000  |  ← tiene dato
| Prod_B  | Cost    | 5000  |
| Prod_C  | Revenue | 3000  |  ← tiene dato
| Prod_C  | Cost    | 2000  |
| Prod_D  | (nada)  |       |  ← sin datos
| Prod_E  | (nada)  |       |  ← sin datos
| ...     |         |       |
| Prod_ZZ | (nada)  |       |  ← sin datos

FOREACH [d/Product]       → itera 500 veces (497 iteraciones hacen nada)
FOREACH.BOOKED [d/Product] → itera 3 veces (solo Prod_A, Prod_B, Prod_C)
```

### FOREACH con ASC vs DESC

```
FOREACH [d/Date] ASC     → 202601, 202602, 202603, ... 202612
FOREACH [d/Date] DESC    → 202612, 202611, 202610, ... 202601
FOREACH [d/Date]         → ORDEN NO GARANTIZADO (peligroso con PREVIOUS/NEXT)
```

---

## 7. IF/ELSEIF/ELSE — Filtrar filas condicionalmente

### Que hace en la base de datos

IF NO modifica datos. Decide SI se ejecuta el DATA() o no para cada fila.

### Condicion sobre dimension — filtra filas

```
IF [d/Company] = "ES" THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "EBIT") * 0.25
ENDIF
```

**Que pasa**: Para cada fila del scope, SAC comprueba la dimension Company.
Solo ejecuta DATA para las filas donde Company = ES.

```
Fila actual: Account=EBIT, Date=202601, Company=ES
  → [d/Company] = "ES"? SI → ejecuta DATA

Fila actual: Account=EBIT, Date=202601, Company=DE
  → [d/Company] = "ES"? NO → salta, no hace nada

Fila actual: Account=EBIT, Date=202601, Company=US
  → [d/Company] = "ES"? NO → salta, no hace nada
```

```
ANTES:                                              DESPUES:
| Company | Account | VALOR |                       | Company | Account | VALOR |
|---------|---------|-------|                       |---------|---------|-------|
| ES      | EBIT    | 40000 |                       | ES      | EBIT    | 40000 |
| ES      | Tax     | ---   | ──10000──>            | ES      | Tax     | 10000 | (25% de 40000)
| DE      | EBIT    | 30000 |                       | DE      | EBIT    | 30000 |
| DE      | Tax     | ---   | (sin cambio)          | DE      | Tax     | ---   | (no tocada)
| US      | EBIT    | 50000 |                       | US      | EBIT    | 50000 |
| US      | Tax     | ---   | (sin cambio)          | US      | Tax     | ---   | (no tocada)
```

### Condicion sobre valor — compara el VALOR de una celda

```
IF RESULTLOOKUP([d/Account] = "Revenue") > 50000 THEN
    DATA([d/Account] = "Bonus") = 2000
ELSE
    DATA([d/Account] = "Bonus") = 0
ENDIF
```

**Que pasa**: Para cada fila del scope, SAC lee Revenue y compara:

```
Fila: Date=202601, Company=ES → Revenue=45000 → 45000 > 50000? NO → Bonus = 0
Fila: Date=202602, Company=ES → Revenue=62000 → 62000 > 50000? SI → Bonus = 2000
Fila: Date=202601, Company=DE → Revenue=55000 → 55000 > 50000? SI → Bonus = 2000
```

### IF con AND/OR — multiples condiciones

```
IF [d/Company] = "ES" AND RESULTLOOKUP([d/Account] = "Revenue") > 50000 THEN
    DATA([d/Account] = "SpecialBonus") = 5000
ENDIF
```

Ambas condiciones deben cumplirse para que DATA se ejecute.

---

## 8. CONFIG — Configuracion global del entorno de ejecucion

### CONFIG.GENERATE_UNBOOKED_DATA = ON

Cambia como la base de datos trata las celdas vacias.

```
CONFIG.GENERATE_UNBOOKED_DATA = OFF (defecto):
  Celda vacia → RESULTLOOKUP devuelve NULL
  NULL en cualquier operacion → NULL
  DATA(... ) = NULL → no escribe nada

CONFIG.GENERATE_UNBOOKED_DATA = ON:
  Celda vacia → RESULTLOOKUP devuelve 0
  0 en operaciones → se calcula normalmente
  DATA(... ) = 0 → escribe 0
```

**Impacto en la base de datos:**

```
Config OFF — Revenue de 202601 esta vacio:
  DATA([d/Account]="Tax") = RESULTLOOKUP([d/Account]="Revenue") * 0.21
  = NULL * 0.21 = NULL → Tax queda vacia

Config ON — Revenue de 202601 esta vacio:
  DATA([d/Account]="Tax") = RESULTLOOKUP([d/Account]="Revenue") * 0.21
  = 0 * 0.21 = 0 → Tax = 0 (se escribe un cero)
```

### CONFIG.TIME_HIERARCHY

Define como SAC interpreta la dimension de tiempo.

```
CONFIG.TIME_HIERARCHY = CALENDARYEAR    → enero=mes 1, diciembre=mes 12
CONFIG.TIME_HIERARCHY = FISCALYEAR      → respeta el año fiscal configurado
```

Afecta a como funcionan PREVIOUS(), NEXT() y PARENT().

### CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE

Respeta el tipo de cuenta (Income, Expense) para el signo.

```
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = OFF (defecto):
  Revenue (Income) = 10000    → se almacena como 10000
  Cost (Expense) = 6000       → se almacena como 6000
  Profit = Revenue - Cost     → 10000 - 6000 = 4000

CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON:
  Revenue (Income) = 10000    → se almacena como 10000
  Cost (Expense) = 6000       → se almacena como -6000 (signo invertido)
  Profit = Revenue + Cost     → 10000 + (-6000) = 4000
```

Cuando ON, las cuentas de tipo Expense se almacenan con signo negativo.
Esto permite sumar todo directamente sin restar manualmente.

---

## 9. LINK — Leer de otra base de datos (otro modelo)

### Que hace

Abre una conexion de SOLO LECTURA a otro modelo. Es como hacer un JOIN en SQL.

```
LINK [HR] = MODEL("t.local/HR_Planning")
```

### Mecanica interna

```
Modelo actual (Finance):
| Account   | Date   | Company | VALOR |
|-----------|--------|---------|-------|
| StaffCost | 202601 | ES      | ---   |  ← quiero escribir aqui
| AvgSalary | 202601 | ES      | 3000  |

Modelo linkeado (HR):
| Account   | Date   | Company | VALOR |
|-----------|--------|---------|-------|
| Headcount | 202601 | ES      | 50    |  ← quiero leer de aqui
| Headcount | 202601 | DE      | 30    |
```

```
DATA([d/Account] = "StaffCost") =
    LINK [HR] RESULTLOOKUP([d/Account] = "Headcount") *
    RESULTLOOKUP([d/Account] = "AvgSalary")
```

**Paso a paso para Date=202601, Company=ES:**

1. `LINK [HR] RESULTLOOKUP([d/Account] = "Headcount")`
   → Va al modelo HR
   → Busca: Account=Headcount, Date=202601, Company=ES (las dimensiones comunes se mapean automaticamente)
   → Devuelve: 50

2. `RESULTLOOKUP([d/Account] = "AvgSalary")`
   → Lee del modelo actual (Finance)
   → Busca: Account=AvgSalary, Date=202601, Company=ES
   → Devuelve: 3000

3. `50 * 3000 = 150000`

4. `DATA([d/Account] = "StaffCost")`
   → Escribe en modelo actual: StaffCost/202601/ES = 150000

```
DESPUES:
Modelo Finance (actualizado):
| Account   | Date   | Company | VALOR  |
|-----------|--------|---------|--------|
| StaffCost | 202601 | ES      | 150000 |  ← escrito
| AvgSalary | 202601 | ES      | 3000   |

Modelo HR (sin cambios — solo lectura):
| Account   | Date   | Company | VALOR |
|-----------|--------|---------|-------|
| Headcount | 202601 | ES      | 50    |  (no modificado)
```

### Mapeo de dimensiones con nombres distintos

Si el modelo HR usa "Department" en vez de "Company":

```
LINK [HR] RESULTLOOKUP([d/Department] = [d/Company], [d/Account] = "Headcount")
```

Esto dice: "en el modelo HR, busca en la dimension Department el valor que tiene
Company en el modelo actual."

```
Modelo actual: Company = "ES"
Modelo HR: busca Department = "ES"
```

---

## 10. AGGREGATE_WRITETO — Escribir totales agregados

### Que hace en la base de datos

Normalmente, DATA() solo escribe en miembros hoja (el nivel mas bajo).
AGGREGATE_WRITETO permite escribir en un miembro padre (total, grupo).

```
AGGREGATE_WRITETO [d/Product] = "TOTAL_PRODUCT"
```

```
ANTES:
| Product       | Account | VALOR |
|---------------|---------|-------|
| Prod_A        | Revenue | 10000 |
| Prod_B        | Revenue | 8000  |
| TOTAL_PRODUCT | Revenue | ---   |  ← nodo padre, normalmente SAC lo calcula automaticamente

CON AGGREGATE_WRITETO:
| Product       | Account | VALOR |
|---------------|---------|-------|
| Prod_A        | Revenue | 10000 |
| Prod_B        | Revenue | 8000  |
| TOTAL_PRODUCT | Revenue | 18000 |  ← ahora puedes escribir directamente aqui
```

---

## 11. PARAMETROS (%) — Valores inyectados desde fuera

### Que hace en la base de datos

Los parametros NO modifican la base de datos. Se sustituyen en el script
antes de la ejecucion, como variables de entorno.

```
// El usuario elige: %TARGET_VERSION% = "Forecast", %GROWTH% = 5

MEMBERSET [d/Version] = %TARGET_VERSION%
DATA() = RESULTLOOKUP([d/Version] = "Actual") * (1 + %GROWTH% / 100)
```

**Antes de ejecutar, SAC sustituye:**

```
MEMBERSET [d/Version] = "Forecast"
DATA() = RESULTLOOKUP([d/Version] = "Actual") * (1 + 5 / 100)
```

Es puro texto-reemplazo. El script resultante se ejecuta normalmente.

---

## 12. ORDEN DE EJECUCION — Como SAC procesa el script

### Regla fundamental

Dentro de un mismo step de Advanced Formula, las sentencias se ejecutan
**de arriba a abajo, secuencialmente**. Cada DATA() se completa para TODAS
las celdas del scope antes de pasar a la siguiente sentencia.

```
// Sentencia 1: se ejecuta para TODAS las celdas del scope
DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "COGS")

// Sentencia 2: se ejecuta DESPUES de que la sentencia 1 termine
// Por eso puede leer GrossProfit recien calculado
DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -   ← lee el valor de la sentencia 1
    RESULTLOOKUP([d/Account] = "OPEX")
```

### Diagrama de flujo completo

```
INICIO
  │
  ├── 1. Leer CONFIG (si existe)
  │
  ├── 2. Aplicar MEMBERSET → filtrar filas del modelo
  │
  ├── 3. Crear VARIABLES en memoria
  │
  ├── 4. Establecer LINK a otros modelos (si existe)
  │
  ├── 5. Ejecutar sentencias en orden:
  │     │
  │     ├── Sentencia 1 (DATA/IF/FOREACH):
  │     │   ├── Si es DATA sin FOREACH: ejecutar para TODAS las celdas en paralelo
  │     │   ├── Si es DATA con FOREACH: ejecutar celda a celda en orden
  │     │   └── Si es IF: evaluar condicion para cada celda
  │     │
  │     ├── Sentencia 2: (espera a que sentencia 1 termine completamente)
  │     │   └── ...
  │     │
  │     └── Sentencia N: ...
  │
  ├── 6. Destruir variables temporales (#)
  │
  ├── 7. Cerrar LINK a otros modelos
  │
  └── 8. Persistir cambios en el modelo
FIN
```

### Importante: los datos se persisten al final

Los cambios no se guardan fila a fila. Se acumulan en memoria durante la
ejecucion y se escriben todos juntos al modelo al final. Si el script falla
a mitad, NO se guarda nada (es transaccional).

---

## TABLA RESUMEN: Cada instruccion y su efecto

| Instruccion | Equivalente SQL | Lee datos | Escribe datos | Crea filas temp | Modifica scope |
|-------------|----------------|-----------|---------------|-----------------|----------------|
| `MEMBERSET` | `WHERE` | No | No | No | **Si** |
| `DATA()` | `UPDATE` / `INSERT` | No | **Si** | No | No |
| `DATA() = NULL` | `DELETE` | No | **Si** (borra) | No | No |
| `RESULTLOOKUP()` | `SELECT` | **Si** | No | No | No |
| `VARIABLEMEMBER #` | `CREATE TEMP TABLE` | No | No | **Si** | No |
| `INTEGER @` | `DECLARE @var INT` | No | No | No (solo memoria) | No |
| `FLOAT @` | `DECLARE @var FLOAT` | No | No | No (solo memoria) | No |
| `FOREACH` | `ORDER BY` + cursor | No | No | No | No (controla orden) |
| `FOREACH.BOOKED` | `ORDER BY` + `WHERE EXISTS` | No | No | No | No (filtra vacios) |
| `IF` | `CASE WHEN` | No | No | No | No (filtra filas) |
| `FOR @i` | `WHILE @i <= N` | No | No | No | No |
| `LINK` | `JOIN` (read-only) | No (abre conexion) | No | No | No |
| `CONFIG` | `SET` options | No | No | No | Cambia comportamiento |
| `%PARAM%` | `@parameter` | No | No | No | No (sustitucion texto) |
| `AGGREGATE_WRITETO` | Permite `UPDATE` en nodos padre | No | No | No | Habilita escritura en padres |
