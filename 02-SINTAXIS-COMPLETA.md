# SAC Advanced Formulas - Sintaxis Completa

## 1. CONFIGURACION (CONFIG)

Se pone al inicio del script. Controla el comportamiento global.

```
// Define la jerarquia temporal (calendario o fiscal)
CONFIG.TIME_HIERARCHY = CALENDARYEAR
CONFIG.TIME_HIERARCHY = FISCALYEAR

// Trata celdas vacias (unbooked) como 0
// Por defecto OFF: las celdas vacias se ignoran
CONFIG.UNBOOKED = ON

// Respetar reglas de signo segun tipo de cuenta
// Por defecto OFF: todo se calcula en valor absoluto
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON
```

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
```

---

## 3. VARIABLES

Almacenamiento temporal durante la ejecucion. No se persisten en el modelo.

```
// Variable miembro (referencia a un miembro virtual)
VARIABLEMEMBER #TempRevenue OF [d/Account]

// Variable numerica entera (por defecto = 0)
VARIABLEINTEGER @counter

// Variable numerica decimal (por defecto = 0.0)
VARIABLEFLOAT @ratio
```

**Reglas:**
- Variables miembro empiezan con `#`
- Variables numericas empiezan con `@`
- La dimension debe existir en el modelo
- No se soporta la dimension Version

### Uso de variables miembro

```
VARIABLEMEMBER #Backup OF [d/Account]

// Guardar datos temporalmente
DATA([d/Account] = #Backup) = RESULTLOOKUP([d/Account] = "Revenue")

// Usar los datos guardados despues
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = #Backup) * 1.1
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
```

**Operadores de comparacion:** `=`, `<>`, `>`, `<`, `>=`, `<=`
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

// Nested: FOREACH dentro de IF (patron recomendado para rendimiento)
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
VARIABLEINTEGER @i

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

---

## 10. Funciones de agregacion

```
// Agregar resultado a un miembro especifico
AGGREGATE_WRITETO [d/Product] = "TOTAL_PRODUCT"

// Sin AGGREGATE_WRITETO, los resultados se escriben en celdas hoja (leaf)
```

---

## 11. Operadores aritmeticos

```
DATA() = RESULTLOOKUP() + 100          // Suma
DATA() = RESULTLOOKUP() - 50           // Resta
DATA() = RESULTLOOKUP() * 1.1          // Multiplicacion
DATA() = RESULTLOOKUP() / 12           // Division
DATA() = RESULTLOOKUP() % 3            // Modulo

// Combinar
DATA([d/Account] = "AvgMonthly") =
    RESULTLOOKUP([d/Account] = "AnnualTotal") / 12

// Funciones matematicas
ABS(value)                              // Valor absoluto
ROUND(value, decimals)                  // Redondear
TRUNCATE(value, decimals)              // Truncar
POWER(base, exponent)                  // Potencia
LOG(value)                             // Logaritmo natural
EXP(value)                             // e^value
```

---

## 12. Funciones de tiempo

```
PREVIOUS(n)     // n periodos hacia atras
NEXT(n)         // n periodos hacia adelante
PARENT()        // nivel padre en jerarquia temporal (mes → trimestre)

// Ejemplos
RESULTLOOKUP([d/Date] = PREVIOUS(1))   // Mes anterior
RESULTLOOKUP([d/Date] = PREVIOUS(12))  // Mismo mes, año anterior
RESULTLOOKUP([d/Date] = NEXT(1))       // Mes siguiente
```

---

## 13. Parametros

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
