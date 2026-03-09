# Aprendizaje Guiado - De cero a escribir scripts reales

## Como pensar en Advanced Formulas

Olvida la programacion clasica. Aqui no hay objetos, arrays ni funciones que
tu defines. Piensa en una HOJA DE CALCULO MULTIDIMENSIONAL.

Imagina un cubo de datos con estas dimensiones:
- Account (Revenue, Cost, Profit...)
- Date (202601, 202602...)
- Company (ES, DE, US...)
- Version (Actual, Plan, Forecast...)
- Product (Prod_A, Prod_B...)

Cada celda es UNA INTERSECCION de todas las dimensiones.
Por ejemplo: Revenue + 202603 + ES + Plan + Prod_A = 15000

Tu script hace 3 cosas:
1. Decide DONDE trabajar (scope/MEMBERSET)
2. LEE celdas (RESULTLOOKUP)
3. ESCRIBE en celdas (DATA)

Eso es todo. Todo lo demas (IF, FOREACH, LINK...) es para controlar
el flujo de lectura/escritura.

---

## Nivel 0: Entender la celda

```
┌─────────────────────────────────────────────────────┐
│              Date: 202601    Date: 202602            │
│            ┌──────────────┬──────────────┐           │
│ Revenue    │    10000     │    12000     │           │
│ Cost       │     6000     │     7000     │           │
│ Profit     │    (vacio)   │   (vacio)    │           │
│            └──────────────┴──────────────┘           │
│  Company = ES, Version = Plan, Product = Prod_A      │
└─────────────────────────────────────────────────────┘

Queremos rellenar Profit = Revenue - Cost
```

El script mas simple posible:

```
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

Linea por linea:
- DATA([d/Account] = "Profit")     → "escribe en la celda donde Account = Profit"
- RESULTLOOKUP([d/Account] = "Revenue")  → "lee el valor donde Account = Revenue"
- RESULTLOOKUP([d/Account] = "Cost")     → "lee el valor donde Account = Cost"
- Las demas dimensiones (Date, Company...) se mantienen igual automaticamente

Resultado:
```
┌─────────────────────────────────────────────────────┐
│              Date: 202601    Date: 202602            │
│            ┌──────────────┬──────────────┐           │
│ Revenue    │    10000     │    12000     │           │
│ Cost       │     6000     │     7000     │           │
│ Profit     │     4000     │     5000     │  ← nuevo  │
│            └──────────────┴──────────────┘           │
└─────────────────────────────────────────────────────┘
```

PREGUNTA CLAVE: "¿Por que se calculo para los dos meses si no puse FOREACH?"
RESPUESTA: Porque SAC ejecuta el calculo para TODAS las combinaciones dentro del
scope automaticamente. Solo necesitas FOREACH cuando un calculo DEPENDE del
resultado anterior (ej: mes 2 depende de mes 1).

---

## Nivel 1: MEMBERSET - Controlar el scope

Sin MEMBERSET, el script actua sobre TODO el modelo. Eso es lento y peligroso.

### Ejemplo: Solo quiero calcular Q1 2026

```
MEMBERSET [d/Date] = "202601" TO "202603"
MEMBERSET [d/Account] = ("Revenue", "Cost", "Profit")
MEMBERSET [d/Version] = "Plan"

DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

Que pasa aqui:
- Solo toca enero, febrero y marzo de 2026
- Solo toca Revenue, Cost y Profit (no otras cuentas)
- Solo toca la version Plan (no Actual ni Forecast)
- El calculo se ejecuta para cada combinacion de Company x Product x Date
  que exista dentro de ese scope

### Regla: Se lo mas restrictivo posible con MEMBERSET
Menos datos en scope = script mas rapido = menos riesgo de sobreescribir algo.

---

## Nivel 2: RESULTLOOKUP - Leer de sitios distintos

RESULTLOOKUP sin parametros lee "la misma celda". Con parametros, lee OTRA celda
cambiando una o mas dimensiones.

### Visualizacion mental

```
Estas parado en: Account=Revenue, Date=202603, Company=ES

RESULTLOOKUP()                        → Lee Revenue, 202603, ES = celda actual
RESULTLOOKUP([d/Account] = "Cost")    → Lee Cost, 202603, ES    = cambio Account
RESULTLOOKUP([d/Date] = PREVIOUS(1))  → Lee Revenue, 202602, ES = cambio Date
RESULTLOOKUP([d/Date] = PREVIOUS(12)) → Lee Revenue, 202503, ES = cambio Date

Puedes cambiar varias dimensiones a la vez:
RESULTLOOKUP([d/Account] = "Cost", [d/Version] = "Actual")
→ Lee Cost, 202603, ES, Actual = cambio Account Y Version
```

### Analogia Excel

Si estas en la celda B5:
- RESULTLOOKUP() = leer B5 (la misma celda)
- RESULTLOOKUP([d/Account] = "Cost") = leer C5 (otra fila, misma columna)
- RESULTLOOKUP([d/Date] = PREVIOUS(1)) = leer A5 (misma fila, columna anterior)

La diferencia es que aqui tienes 5+ dimensiones, no solo filas y columnas.

### Que pasa si RESULTLOOKUP lee una celda vacia?

Devuelve NULL. Y cuando DATA() recibe NULL, NO escribe nada.
Esto es importante: no sobreescribe con 0, simplemente no hace nada.

Si quieres tratar vacios como 0:
```
CONFIG.UNBOOKED = ON
```
Ahora las celdas vacias valen 0 en vez de NULL.

---

## Nivel 3: FOREACH - Cuando y por que

### REGLA DE ORO:
Solo usa FOREACH cuando el calculo de una iteracion DEPENDE del resultado
de la iteracion anterior. Si no depende, NO lo uses.

### Ejemplo donde NO necesitas FOREACH

Calcular IVA del 21% para cada mes:
```
MEMBERSET [d/Date] = "202601" TO "202612"

DATA([d/Account] = "IVA") = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
```
Cada mes es independiente. Enero no necesita saber el IVA de diciembre.
SAC lo paraleliza internamente → rapido.

### Ejemplo donde SI necesitas FOREACH

Saldo acumulado mes a mes:
```
┌──────────────────────────────────────────────────────┐
│  Mes:      Ene     Feb     Mar     Abr     May       │
│  Ventas:   100     150     120     180     200       │
│  Acumul:   100     250     370     550     750       │
│                     ↑       ↑                        │
│                  100+150  250+120                     │
│              (depende del mes anterior)               │
└──────────────────────────────────────────────────────┘
```

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Sales", "Cumulative")

// Enero: el acumulado es igual a las ventas
DATA([d/Account] = "Cumulative", [d/Date] = "202601") =
    RESULTLOOKUP([d/Account] = "Sales", [d/Date] = "202601")

// Desde febrero: acumulado anterior + ventas del mes
MEMBERSET [d/Date] = "202602" TO "202612"

FOREACH [d/Date] ASC
    DATA([d/Account] = "Cumulative") =
        RESULTLOOKUP([d/Account] = "Cumulative", [d/Date] = PREVIOUS(1)) +
        RESULTLOOKUP([d/Account] = "Sales")
ENDFOR
```

Linea por linea:
- FOREACH [d/Date] ASC → recorre los meses en orden (febrero, marzo, abril...)
- Para febrero: lee Cumulative de enero (ya calculado) + Sales de febrero
- Para marzo: lee Cumulative de febrero (recien calculado) + Sales de marzo
- ASC es critico: garantiza el orden cronologico

Sin FOREACH, SAC intentaria calcular todos los meses a la vez, y el
RESULTLOOKUP([d/Date] = PREVIOUS(1)) leeria datos ORIGINALES (vacios),
no los recien calculados.

### FOREACH.BOOKED - La version rapida

```
// FOREACH: itera 1000 productos (aunque 950 no tengan datos)
FOREACH [d/Product]
    DATA([d/Account] = "Margin") = ...
ENDFOR

// FOREACH.BOOKED: itera solo los 50 productos con datos
FOREACH.BOOKED [d/Product]
    DATA([d/Account] = "Margin") = ...
ENDFOR
```

Usa FOREACH.BOOKED siempre que puedas. Unica excepcion: cuando necesitas
escribir datos en miembros que actualmente estan vacios.

---

## Nivel 4: IF - Logica condicional

### Comparar dimension con miembro
```
IF [d/Company] = "ES" THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "EBIT") * 0.25
ENDIF
```
"Si la empresa es ES, el impuesto es 25% del EBIT"

### Comparar valor numerico
```
IF RESULTLOOKUP([d/Account] = "Revenue") > 0 THEN
    DATA([d/Account] = "MarginPct") =
        RESULTLOOKUP([d/Account] = "Profit") /
        RESULTLOOKUP([d/Account] = "Revenue") * 100
ENDIF
```
"Solo calcula el % de margen si Revenue > 0 (evita division por cero)"

### Combinar condiciones
```
IF [d/Company] = "ES" AND [d/Account] = "Revenue" THEN
    DATA() = RESULTLOOKUP() * 1.1
ENDIF
```

### IF + FOREACH (el orden importa para rendimiento)

```
// BIEN: IF fuera, FOREACH dentro
// Si la empresa no es ES, ni siquiera entra al bucle
IF [d/Company] = "ES" THEN
    FOREACH [d/Date] ASC
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
    ENDFOR
ENDIF

// MAL: FOREACH fuera, IF dentro
// Recorre TODOS los meses para TODAS las empresas, y luego filtra
FOREACH [d/Date] ASC
    IF [d/Company] = "ES" THEN
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
    ENDIF
ENDFOR
```

---

## Nivel 5: Variables - Almacenamiento temporal

### Problema sin variables

```
// Lee Revenue 3 veces del modelo → lento
DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
DATA([d/Account] = "Net") = RESULTLOOKUP([d/Account] = "Revenue") * 0.79
DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = "Revenue") * 0.02
```

### Solucion con variable miembro

```
VARIABLEMEMBER #Rev OF [d/Account]

// Lee Revenue UNA vez y lo guarda en #Rev
DATA([d/Account] = #Rev) = RESULTLOOKUP([d/Account] = "Revenue")

// Ahora lee de #Rev (que esta en memoria, no en el modelo)
DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = #Rev) * 0.21
DATA([d/Account] = "Net") = RESULTLOOKUP([d/Account] = #Rev) * 0.79
DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = #Rev) * 0.02
```

### Variables numericas

```
VARIABLEINTEGER @monthCount
VARIABLEFLOAT @avgRevenue

// No se pueden usar directamente con DATA/RESULTLOOKUP
// Se usan en bucles FOR y en condiciones
FOR @monthCount = 1 TO 12 STEP 1
    // ...
ENDFOR
```

### Cuando usar cada tipo

| Tipo | Prefijo | Para que |
|------|---------|----------|
| VARIABLEMEMBER | # | Guardar datos de celdas temporalmente |
| VARIABLEINTEGER | @ | Contadores en bucles FOR |
| VARIABLEFLOAT | @ | Calculos intermedios con decimales |

---

## Nivel 6: Parametros - Scripts reutilizables

Los parametros se definen en la UI de la Data Action (no en el script).
En el script solo los usas con %NOMBRE%.

### Ejemplo: Script generico de copia entre versiones

```
// Parametros definidos en la UI:
//   SOURCE_VERSION (tipo: miembro de dimension Version)
//   TARGET_VERSION (tipo: miembro de dimension Version)
//   DATE_FROM (tipo: miembro de dimension Date)
//   DATE_TO (tipo: miembro de dimension Date)

MEMBERSET [d/Date] = %DATE_FROM% TO %DATE_TO%
MEMBERSET [d/Version] = %TARGET_VERSION%

DATA() = RESULTLOOKUP([d/Version] = %SOURCE_VERSION%)
```

Cuando un usuario ejecuta esta Data Action:
- Le aparece un prompt para elegir SOURCE_VERSION, TARGET_VERSION, fechas
- El script usa esos valores
- Un unico script sirve para: Actual→Plan, Actual→Forecast, Plan→Forecast...

### Pasar parametros desde JavaScript (Analytic Application)

En el JavaScript de una Analytic App:
```javascript
// JavaScript (Analytic Application)
var selectedVersion = Dropdown_1.getSelectedKey();
var selectedDate = DatePicker_1.getValue();

Planning.triggerDataAction("DA_CopyVersion", {
    "SOURCE_VERSION": "Actual",
    "TARGET_VERSION": selectedVersion,
    "DATE_FROM": "202601",
    "DATE_TO": selectedDate
});
```

Esto conecta la UI interactiva (JavaScript) con la logica de datos
(Advanced Formula). Son dos mundos distintos que se comunican via parametros.

---

## Nivel 7: LINK - Trabajar con multiples modelos

### El problema
Tienes datos en modelos separados:
- Modelo Finance: Revenue, Cost, Profit
- Modelo HR: Headcount, AvgSalary, Turnover
- Modelo Sales: Units, Price, Discount

Necesitas combinar datos de varios modelos en un calculo.

### La solucion

```
// Definir los links (como un "import")
LINK [HR] = MODEL("t.local/HR_Planning")
LINK [Sales] = MODEL("t.local/Sales_Planning")

MEMBERSET [d/Date] = "202601" TO "202612"

// Coste de personal = headcount del modelo HR * salario medio del modelo HR
DATA([d/Account] = "StaffCost") =
    LINK [HR] RESULTLOOKUP([d/Account] = "Headcount") *
    LINK [HR] RESULTLOOKUP([d/Account] = "AvgSalary")

// Revenue = unidades del modelo Sales * precio del modelo Sales
DATA([d/Account] = "Revenue") =
    LINK [Sales] RESULTLOOKUP([d/Account] = "Units") *
    LINK [Sales] RESULTLOOKUP([d/Account] = "Price")
```

### Reglas de LINK
1. Solo puedes LEER del modelo linkeado (RESULTLOOKUP)
2. Solo puedes ESCRIBIR en el modelo principal (DATA)
3. Las dimensiones se mapean automaticamente por nombre
4. Si los nombres no coinciden, mapeas manualmente:
   ```
   LINK [HR] RESULTLOOKUP([d/CostCenter] = [d/Department])
   ```
   "Lee CostCenter del modelo HR usando el valor de Department del modelo actual"
