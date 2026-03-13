# SAC Advanced Formulas - Ejemplos Practicos

> 30+ ejemplos organizados por complejidad. Cada uno explica que hace el codigo
> sobre la base de datos linea por linea.

---

# PARTE 1: FUNDAMENTOS

---

## Ejemplo 1: Incremento porcentual simple

**Caso:** Subir Revenue un 10% respecto al año anterior para todo 2026.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Version] = "Plan"

DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(12)) * 1.1
```

**Que hace sobre la base de datos:**

SAC recorre cada fila del scope (Revenue, Plan, cada mes de 2026).
Para cada fila, RESULTLOOKUP retrocede 12 periodos y lee el valor.
DATA escribe el resultado multiplicado por 1.1.

```
ANTES:                                    DESPUES:
| Date   | Version | VALOR |              | Date   | Version | VALOR |
|--------|---------|-------|              |--------|---------|-------|
| 202501 | Actual  | 10000 | ← leido     | 202501 | Actual  | 10000 | (no tocado)
| 202502 | Actual  | 11000 | ← leido     | 202502 | Actual  | 11000 | (no tocado)
| 202601 | Plan    | ---   |              | 202601 | Plan    | 11000 | ← 10000*1.1
| 202602 | Plan    | ---   |              | 202602 | Plan    | 12100 | ← 11000*1.1
```

No necesita FOREACH: cada mes es independiente (202601 no necesita el resultado de 202602).

---

## Ejemplo 2: Carry Forward con CARRYFORWARD()

**Caso:** Arrastre de saldos apertura/cierre clasico.

### Version clasica con FOREACH (lenta):

```
MEMBERSET [d/Date] = "202601" TO "202612"

FOREACH [d/Date] ASC
    DATA([d/Account] = "OPENING") =
        RESULTLOOKUP([d/Account] = "CLOSING", [d/Date] = PREVIOUS(1))
ENDFOR
```

### Version optimizada con CARRYFORWARD (rapida):

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Flow] = ("OPENING", "CLOSING", "HIRES", "TERMINATIONS")

DATA() = CARRYFORWARD([d/Flow], "OPENING", "CLOSING",
    "HIRES", "CLOSING" - "OPENING" - "TERMINATIONS")
```

**Que hace CARRYFORWARD:**
- Primer argumento: la dimension de movimiento
- "OPENING": miembro de apertura
- "CLOSING": miembro de cierre
- "HIRES": miembro de flujo que se calcula
- Formula: CLOSING - OPENING - TERMINATIONS = HIRES

SAC calcula automaticamente la cadena temporal sin FOREACH.
Internamente hace lo mismo que el FOREACH pero optimizado a nivel de motor.

```
ANTES:                                             DESPUES:
| Flow         | 202601 | 202602 | 202603 |        | Flow         | 202601 | 202602 | 202603 |
|--------------|--------|--------|--------|        |--------------|--------|--------|--------|
| OPENING      | 100    | ---    | ---    |        | OPENING      | 100    | 103    | 108    |
| HIRES        | 5      | 8      | 4      |        | HIRES        | 5      | 8      | 4      |
| TERMINATIONS | 2      | 3      | 1      |        | TERMINATIONS | 2      | 3      | 1      |
| CLOSING      | ---    | ---    | ---    |        | CLOSING      | 103    | 108    | 111    |
```

---

## Ejemplo 3: Calculo de P&L cascada

**Caso:** Cuenta de resultados completa donde cada linea depende de la anterior.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

// 1. Gross Profit
DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "COGS")

// 2. EBIT (lee GrossProfit recien calculado en paso 1)
DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -
    RESULTLOOKUP([d/Account] = "OPEX")

// 3. Impuestos condicionales por region
IF [d/Region] = "Europe" THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "EBIT") * 0.25
ELSEIF [d/Region] = "US" THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "EBIT") * 0.21
ELSE
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "EBIT") * 0.30
ENDIF

// 4. Net Profit
DATA([d/Account] = "NetProfit") =
    RESULTLOOKUP([d/Account] = "EBIT") -
    RESULTLOOKUP([d/Account] = "Tax")
```

**Orden de ejecucion:**
Las sentencias DATA se ejecutan secuencialmente de arriba a abajo.
El paso 2 lee el GrossProfit del paso 1. El paso 4 lee el Tax del paso 3.
NO necesitas FOREACH porque no hay dependencia entre meses, solo entre cuentas.

---

## Ejemplo 4: Distribucion mensual con estacionalidad

**Caso:** Distribuir presupuesto anual segun pesos mensuales almacenados en el modelo.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "Seasonality_Pct")
MEMBERSET [d/Version] = "Plan"

// %ANNUAL_REVENUE% = 1200000 (parametro pasado por el usuario)

DATA([d/Account] = "Revenue") =
    %ANNUAL_REVENUE% * RESULTLOOKUP([d/Account] = "Seasonality_Pct") / 100
```

**Que pasa celda a celda:**

```
| Date   | Account         | VALOR  | Calculo                    |
|--------|-----------------|--------|----------------------------|
| 202601 | Seasonality_Pct | 6      | ← leido                   |
| 202601 | Revenue         | 72000  | ← 1200000 * 6 / 100       |
| 202602 | Seasonality_Pct | 7      | ← leido                   |
| 202602 | Revenue         | 84000  | ← 1200000 * 7 / 100       |
| 202607 | Seasonality_Pct | 12     | ← leido (julio = pico)    |
| 202607 | Revenue         | 144000 | ← 1200000 * 12 / 100      |
```

Sin FOREACH, sin IF. El diseño del modelo (pesos en una cuenta) simplifica el script.

---

## Ejemplo 5: Crecimiento mensual compuesto

**Caso:** Aplicar 2% de crecimiento mes a mes partiendo de un valor base en enero.

```
MEMBERSET [d/Date] = "202602" TO "202612"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Version] = "Plan"

FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR
```

**Por que FOREACH ASC es obligatorio:**

```
Sin FOREACH (paralelo):
  202602 = 202601 (original: 10000) * 1.02 = 10200  ← OK
  202603 = 202602 (original: vacio) * 1.02 = NULL    ← FALLO (202602 no calculado aun)

Con FOREACH ASC (secuencial):
  202602 = 202601 (10000) * 1.02 = 10200             ← OK
  202603 = 202602 (10200, recien calculado) * 1.02 = 10404  ← OK
  202604 = 202603 (10404) * 1.02 = 10612             ← OK
```

---

## Ejemplo 6: Copiar Actual a Forecast (Rolling Forecast)

**Caso:** Meses cerrados copian Actual, meses abiertos proyectan +5% sobre año anterior.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Forecast"

// %LAST_CLOSED% = "202603" (parametro: ultimo mes cerrado)

// Paso 1: Meses cerrados → copiar Actual
FOREACH [d/Date]
    IF [d/Date] <= %LAST_CLOSED% THEN
        DATA() = RESULTLOOKUP([d/Version] = "Actual")
    ENDIF
ENDFOR

// Paso 2: Meses abiertos → año anterior + 5%
FOREACH [d/Date]
    IF [d/Date] > %LAST_CLOSED% THEN
        DATA() = RESULTLOOKUP([d/Version] = "Actual", [d/Date] = PREVIOUS(12)) * 1.05
    ENDIF
ENDFOR
```

```
RESULTADO:
| Date   | Actual | Forecast | Origen                         |
|--------|--------|----------|--------------------------------|
| 202601 | 10000  | 10000    | ← copia de Actual (cerrado)    |
| 202602 | 11000  | 11000    | ← copia de Actual (cerrado)    |
| 202603 | 10500  | 10500    | ← copia de Actual (cerrado)    |
| 202604 | ---    | 11025    | ← Actual 202504 * 1.05         |
| 202605 | ---    | 10920    | ← Actual 202505 * 1.05         |
```

---

## Ejemplo 7: Borrar datos con DATA() = NULL vs DELETE()

**Caso:** Limpiar datos antes de recalcular.

### Opcion A: DATA() = NULL
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

DATA() = NULL
```
Pone todas las celdas del scope a NULL (sin dato).
DATA() limpia el scope objetivo primero por defecto.

### Opcion B: DELETE() — borrado explicito de celdas concretas
```
MEMBERSET [d/Date] = "202601" TO "202612"

DELETE([d/Account] = "Disposal", [d/Flow] = "OPEN")
```
DELETE borra solo las celdas que coinciden con los parametros dados.
Util cuando quieres borrar una interseccion especifica sin tocar el resto.

### Opcion C: DATA.APPEND() — escribir SIN limpiar primero
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

// DATA() normal: limpia el scope objetivo, luego escribe
// DATA.APPEND(): NO limpia, anade valores a lo que ya existe

DATA.APPEND([d/Account] = "Revenue") =
    RESULTLOOKUP([d/Account] = "NewRevStream")
```

**Diferencia critica:**
- `DATA()` = TRUNCATE + INSERT (borra primero, luego escribe)
- `DATA.APPEND()` = INSERT (escribe sin borrar)
- `DELETE()` = DELETE (borra sin escribir)

---

# PARTE 2: FUNCIONES DE TIEMPO

---

## Ejemplo 8: PREVIOUS y NEXT — navegacion temporal basica

```
MEMBERSET [d/Date] = "202601" TO "202612"

// Variacion mes a mes
DATA([d/Account] = "MoM_Change") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = PREVIOUS(1))

// Variacion interanual (mismo mes, año anterior)
DATA([d/Account] = "YoY_Change") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = PREVIOUS(12))

// Proyeccion: mes siguiente = mes actual + 2%
DATA([d/Account] = "Revenue", [d/Date] = NEXT(1)) =
    RESULTLOOKUP([d/Account] = "Revenue") * 1.02
```

```
Posicion actual: Date=202606

PREVIOUS(1)  → 202605 (un mes atras)
PREVIOUS(3)  → 202603 (tres meses atras)
PREVIOUS(12) → 202506 (mismo mes, año anterior)
NEXT(1)      → 202607 (un mes adelante)
NEXT(6)      → 202612 (seis meses adelante)
```

---

## Ejemplo 9: FIRST, LAST, PREYEARLAST — anclas temporales

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "YTD_Revenue", "LastYearClose")

// FIRST(): primer periodo del año → siempre enero
DATA([d/Account] = "YTD_Start") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = FIRST())

// LAST(): ultimo periodo del año → siempre diciembre
DATA([d/Account] = "YearEndTarget") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = LAST())

// PREYEARLAST(): ultimo periodo del año ANTERIOR → diciembre 2025
DATA([d/Account] = "LastYearClose") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = PREYEARLAST())
```

**Que pasa para cada fecha:**

```
| Posicion actual | FIRST() | LAST()  | PREYEARLAST() |
|-----------------|---------|---------|----------------|
| 202601          | 202601  | 202612  | 202512         |
| 202606          | 202601  | 202612  | 202512         |
| 202612          | 202601  | 202612  | 202512         |
```

**Caso de uso real — Balance de apertura del año:**

```
// El opening de enero = closing de diciembre del año pasado
DATA([d/Account] = "Opening", [d/Date] = FIRST()) =
    RESULTLOOKUP([d/Account] = "Closing", [d/Date] = PREYEARLAST())
```

---

## Ejemplo 10: TODAY y funciones de fecha — presupuesto dinamico

```
MEMBERSET [d/Date] = "202601" TO "202612"

// TODAY() devuelve la fecha actual del sistema en formato YYYYMMDD
// Util para scripts que se ejecutan automaticamente

// Ejemplo: marcar meses como "past" o "future" respecto a hoy
// MONTH() extrae el mes, YEAR() extrae el año

IF YEAR() = 2026 AND MONTH() <= MONTH(TODAY()) THEN
    // Meses pasados: copiar Actual
    DATA([d/Version] = "Forecast") = RESULTLOOKUP([d/Version] = "Actual")
ELSE
    // Meses futuros: mantener Plan
    DATA([d/Version] = "Forecast") = RESULTLOOKUP([d/Version] = "Plan")
ENDIF
```

---

## Ejemplo 11: DAYSINMONTH y DAYSINYEAR — prorrateo por dias

**Caso:** Distribuir un coste anual proporcionalmente a los dias de cada mes.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("AnnualRent", "MonthlyRent")

// DAYSINMONTH() devuelve los dias del mes actual (28, 29, 30 o 31)
// DAYSINYEAR() devuelve los dias del año (365 o 366)

DATA([d/Account] = "MonthlyRent") =
    RESULTLOOKUP([d/Account] = "AnnualRent") *
    DAYSINMONTH() / DAYSINYEAR()
```

**Resultado para 2026:**

```
| Date   | DAYSINMONTH | DAYSINYEAR | AnnualRent | MonthlyRent |
|--------|-------------|------------|------------|-------------|
| 202601 | 31          | 365        | 120000     | 10191.78    |
| 202602 | 28          | 365        | 120000     | 9205.48     |
| 202603 | 31          | 365        | 120000     | 10191.78    |
| 202604 | 30          | 365        | 120000     | 9863.01     |
```

Mas preciso que dividir entre 12 (que da 10000/mes independientemente de los dias).

---

## Ejemplo 12: DATERATIO — prorrateo por solapamiento de periodos

**Caso:** Un contrato empieza el 15 de marzo. Quieres prorratear el coste mensual
solo por los dias efectivos del contrato en cada mes.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("MonthlyCost", "ProratedCost")

// DATERATIO() calcula el ratio de solapamiento entre dos fechas y el periodo actual
// Si el contrato va del 15-mar al 31-dic:
//   Marzo: 17 dias de 31 = 0.548
//   Abril: 30 dias de 30 = 1.0
//   Febrero: 0 dias = 0.0

DATA([d/Account] = "ProratedCost") =
    RESULTLOOKUP([d/Account] = "MonthlyCost") *
    DATERATIO("20260315", "20261231")
```

```
| Date   | MonthlyCost | DATERATIO | ProratedCost |
|--------|-------------|-----------|--------------|
| 202602 | 5000        | 0.000     | 0            |
| 202603 | 5000        | 0.548     | 2741         |
| 202604 | 5000        | 1.000     | 5000         |
| 202612 | 5000        | 1.000     | 5000         |
```

---

## Ejemplo 13: DATEDIFF — calcular duracion entre fechas

**Caso:** Calcular los meses de antiguedad de cada empleado.

```
MEMBERSET [d/Date] = "202606"
MEMBERSET [d/Account] = ("HireDate", "Seniority_Months")

// DATEDIFF() devuelve la diferencia entre dos fechas
// Modos: CalendarDiff, Floor, Ceiling

DATA([d/Account] = "Seniority_Months") =
    DATEDIFF([d/Date], RESULTLOOKUP([d/Account] = "HireDate"), CalendarDiff)
```

---

# PARTE 3: FUNCIONES MATEMATICAS

---

## Ejemplo 14: SQRT, POWER, LOG — calculos financieros

**Caso:** Calcular la tasa de crecimiento anual compuesta (CAGR).

```
// CAGR = (Valor_Final / Valor_Inicial) ^ (1/n) - 1

MEMBERSET [d/Date] = "202612"
MEMBERSET [d/Account] = ("Revenue_Start", "Revenue_End", "CAGR", "Years")

VARIABLEMEMBER #Ratio OF [d/Account]

// Ratio = Final / Inicial
DATA([d/Account] = #Ratio) =
    RESULTLOOKUP([d/Account] = "Revenue_End") /
    RESULTLOOKUP([d/Account] = "Revenue_Start")

// CAGR = POWER(ratio, 1/years) - 1
DATA([d/Account] = "CAGR") =
    POWER(RESULTLOOKUP([d/Account] = #Ratio),
          1 / RESULTLOOKUP([d/Account] = "Years")) - 1
```

**Otras funciones matematicas disponibles:**

```
ABS(-500)           → 500         // Valor absoluto
ROUND(3.456, 2)     → 3.46       // Redondear a 2 decimales
ROUND(3456, -2)     → 3500       // Redondear a centenas
FLOOR(3.7)          → 3          // Redondear hacia abajo
CEIL(3.2)           → 4          // Redondear hacia arriba
TRUNC(3.789, 1)     → 3.7        // Truncar (no redondea)
SQRT(144)           → 12         // Raiz cuadrada
POWER(2, 10)        → 1024       // Potencia
LOG(100)            → 4.605      // Logaritmo natural (ln)
LOG10(100)          → 2          // Logaritmo base 10
MOD(17, 5)          → 2          // Resto de division (17/5 = 3, resto 2)
INT(3.99)           → 3          // Conversion a entero
FLOAT(3)            → 3.0        // Conversion a decimal
```

---

## Ejemplo 15: FLOOR y CEIL — redondeo para embalaje

**Caso:** Calcular unidades a pedir redondeando al palet completo mas cercano.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Units_Needed", "Units_Per_Palet", "Palets_Needed", "Units_To_Order")

// Palets necesarios = CEIL(unidades / unidades_por_palet)
// CEIL redondea hacia arriba: si necesitas 3.1 palets, pides 4

DATA([d/Account] = "Palets_Needed") =
    CEIL(RESULTLOOKUP([d/Account] = "Units_Needed") /
         RESULTLOOKUP([d/Account] = "Units_Per_Palet"))

// Unidades a pedir = palets * unidades_por_palet
DATA([d/Account] = "Units_To_Order") =
    RESULTLOOKUP([d/Account] = "Palets_Needed") *
    RESULTLOOKUP([d/Account] = "Units_Per_Palet")
```

```
| Units_Needed | Units_Per_Palet | Palets_Needed | Units_To_Order |
|--------------|-----------------|---------------|----------------|
| 250          | 100             | 3 (CEIL 2.5)  | 300            |
| 410          | 100             | 5 (CEIL 4.1)  | 500            |
```

---

## Ejemplo 16: MOD — identificar trimestres y patrones ciclicos

**Caso:** Aplicar bonus trimestral solo en marzo, junio, septiembre y diciembre.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "QuarterlyBonus")

// MOD(mes, 3) = 0 → es fin de trimestre (3, 6, 9, 12)
IF MOD(MONTH(), 3) = 0 THEN
    DATA([d/Account] = "QuarterlyBonus") =
        RESULTLOOKUP([d/Account] = "Revenue") * 0.05
ELSE
    DATA([d/Account] = "QuarterlyBonus") = 0
ENDIF
```

```
| Date   | MONTH | MOD(M,3) | Revenue | Bonus |
|--------|-------|----------|---------|-------|
| 202601 | 1     | 1        | 10000   | 0     |
| 202602 | 2     | 2        | 11000   | 0     |
| 202603 | 3     | 0 ←      | 12000   | 600   | ← fin Q1
| 202604 | 4     | 1        | 10500   | 0     |
| 202606 | 6     | 0 ←      | 13000   | 650   | ← fin Q2
```

---

# PARTE 4: ATTRIBUTE() — LEER PROPIEDADES DE DIMENSION

---

## Ejemplo 17: ATTRIBUTE — leer propiedades de miembros

**Caso:** Calcular depreciacion usando la vida util almacenada como atributo del equipo.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("AssetCost", "MonthlyDepr")

// [d/Equipment].[p/Useful_Life] es un atributo numerico de la dimension Equipment
// ATTRIBUTE() lee ese valor sin necesidad de tenerlo como cuenta en el modelo

INTEGER @usefulLife

FOREACH [d/Equipment]
    @usefulLife = ATTRIBUTE([d/Equipment].[p/Useful_Life])

    IF @usefulLife > 0 THEN
        DATA([d/Account] = "MonthlyDepr") =
            RESULTLOOKUP([d/Account] = "AssetCost") / @usefulLife
    ENDIF
ENDFOR
```

**Que hace ATTRIBUTE:**

```
Dimension Equipment:
| Miembro | Nombre          | [p/Useful_Life] | [p/Category] |
|---------|-----------------|------------------|--------------|
| EQ001   | Servidor HP     | 60               | IT           |
| EQ002   | Vehiculo Ford   | 48               | Fleet        |
| EQ003   | Mobiliario      | 120              | Office       |

Cuando el cursor esta en EQ001:
  ATTRIBUTE([d/Equipment].[p/Useful_Life]) → 60
  ATTRIBUTE([d/Equipment].[p/Category])    → "IT" (si es texto, solo para IF)
```

ATTRIBUTE no necesita que la vida util sea una cuenta del modelo.
Lee directamente del maestro de la dimension, como leer de una tabla auxiliar.

---

## Ejemplo 18: ATTRIBUTE para mapeo de cuentas (patron Sister)

**Caso:** Reclasificar cuentas usando una propiedad "Sister" que indica la cuenta destino.

```
MEMBERSET [d/Date] = "202601" TO "202612"

// Cada cuenta tiene un atributo [p/Sister] que indica a que cuenta debe mapearse
// Ej: "OldRevenue" tiene Sister = "NewRevenue"

DATA([d/Account] = [d/Account].[p/Sister]) = RESULTLOOKUP()
```

**Que pasa:**

```
Dimension Account:
| Miembro     | [p/Sister]   |
|-------------|--------------|
| OldRevenue  | NewRevenue   |
| OldCost     | NewCost      |
| OldOPEX     | NewOPEX      |

Ejecucion:
  Cursor en OldRevenue: [d/Account].[p/Sister] = "NewRevenue"
    → DATA([d/Account] = "NewRevenue") = RESULTLOOKUP() → copia valor de OldRevenue a NewRevenue

  Cursor en OldCost: [d/Account].[p/Sister] = "NewCost"
    → DATA([d/Account] = "NewCost") = RESULTLOOKUP() → copia valor de OldCost a NewCost
```

Un unico DATA reclasifica TODAS las cuentas de golpe sin IF.
El mapeo esta en los datos maestros, no hardcodeado en el script.

---

## Ejemplo 19: ATTRIBUTE con IF — logica por tipo de cuenta

**Caso:** Aplicar diferentes reglas segun el tipo de cuenta (Income, Expense, Asset...).

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

// [d/Account].[p/AccountType] es un atributo estandar de SAC:
// "INC" = Income, "EXP" = Expense, "AST" = Asset, "LEQ" = Liability/Equity

IF ATTRIBUTE([d/Account].[p/AccountType]) = "INC" THEN
    // Ingresos: crecer 8%
    DATA() = RESULTLOOKUP([d/Version] = "Actual", [d/Date] = PREVIOUS(12)) * 1.08

ELSEIF ATTRIBUTE([d/Account].[p/AccountType]) = "EXP" THEN
    // Gastos: crecer solo 3%
    DATA() = RESULTLOOKUP([d/Version] = "Actual", [d/Date] = PREVIOUS(12)) * 1.03

ENDIF
```

Asi no necesitas listar cada cuenta por nombre. El IF se basa en la clasificacion.

---

# PARTE 5: JERARQUIAS Y BASEMEMBER

---

## Ejemplo 20: BASEMEMBER — trabajar con hojas de jerarquia

**Caso:** Calcular bonus solo para centros de coste operativos (hojas debajo de "OPERATIONS").

```
// BASEMEMBER devuelve todos los miembros hoja debajo de un nodo padre
MEMBERSET [d/CostCenter] = BASEMEMBER([d/CostCenter].[h/Standard], "OPERATIONS")
MEMBERSET [d/Date] = "202601" TO "202612"

DATA([d/Account] = "Bonus") =
    RESULTLOOKUP([d/Account] = "Revenue") * 0.03
```

**Que hace BASEMEMBER:**

```
Jerarquia [h/Standard] de CostCenter:
  CORPORATE
  ├── OPERATIONS          ← nodo padre
  │   ├── CC_SALES        ← hoja (base member)
  │   ├── CC_PRODUCTION   ← hoja (base member)
  │   └── CC_LOGISTICS    ← hoja (base member)
  └── ADMIN
      ├── CC_HR           ← NO incluido
      └── CC_FINANCE      ← NO incluido

BASEMEMBER(..., "OPERATIONS") devuelve: (CC_SALES, CC_PRODUCTION, CC_LOGISTICS)
```

Sin BASEMEMBER tendrias que listarlos manualmente:
`MEMBERSET [d/CostCenter] = ("CC_SALES", "CC_PRODUCTION", "CC_LOGISTICS")`
Y actualizarlo cada vez que añadas un centro de coste.

---

## Ejemplo 21: CONFIG.HIERARCHY — elegir jerarquia activa

**Caso:** Usar una jerarquia especifica para aggregar por region.

```
// Dimension Region tiene dos jerarquias:
// [h/Geographic]: World > Europe > Spain, Germany...
// [h/Business]:   Global > BU_EMEA > Spain, Germany...

CONFIG.HIERARCHY [d/Region].[h/Geographic]

MEMBERSET [d/Region] = BASEMEMBER([d/Region].[h/Geographic], "Europe")

DATA([d/Account] = "EuropeTotal") =
    RESULTLOOKUP([d/Account] = "Revenue")

AGGREGATE_WRITETO [d/Region] = "Europe"
```

---

## Ejemplo 22: MEMBERSET con filtro por atributo

**Caso:** Actuar solo sobre productos de una categoria especifica.

```
// Filtrar por atributo [p/Category] de la dimension Product
MEMBERSET [d/Product].[p/Category] = "Electronics"
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

DATA([d/Account] = "Revenue") =
    RESULTLOOKUP([d/Version] = "Actual", [d/Date] = PREVIOUS(12)) * 1.15
```

**Que filtra:**

```
Dimension Product:
| Miembro | [p/Category]  | En scope? |
|---------|---------------|-----------|
| LAPTOP  | Electronics   | SI        |
| MONITOR | Electronics   | SI        |
| DESK    | Furniture     | NO        |
| CHAIR   | Furniture     | NO        |
| MOUSE   | Electronics   | SI        |
```

Solo LAPTOP, MONITOR y MOUSE reciben el incremento del 15%.

---

# PARTE 6: CROSS-MODEL (LINK)

---

## Ejemplo 23: LINK basico — leer de otro modelo

**Caso:** Coste de personal = headcount (modelo HR) * salario medio (modelo actual).

```
LINK [HR] = MODEL("t.local/HR_Planning")

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "StaffCost"

DATA() =
    LINK [HR] RESULTLOOKUP([d/Account] = "Headcount") *
    RESULTLOOKUP([d/Account] = "AvgSalary")
```

**Paso a paso para Date=202603, Company=ES:**
1. Va al modelo HR, busca Account=Headcount, Date=202603, Company=ES → 50
2. Vuelve al modelo actual, busca Account=AvgSalary, Date=202603, Company=ES → 3200
3. Calcula 50 * 3200 = 160000
4. Escribe StaffCost/202603/ES = 160000

---

## Ejemplo 24: LINK con mapeo de dimensiones

**Caso:** El modelo HR usa "Department", el modelo Finance usa "CostCenter".

```
LINK [HR] = MODEL("t.local/HR_Planning")

MEMBERSET [d/Date] = "202601" TO "202612"

// [d/Department] del modelo HR se mapea al valor de [d/CostCenter] del modelo actual
DATA([d/Account] = "StaffCost") =
    LINK [HR] RESULTLOOKUP(
        [d/Department] = [d/CostCenter],
        [d/Account] = "Headcount"
    ) *
    RESULTLOOKUP([d/Account] = "AvgSalary")
```

**El mapeo funciona asi:**

```
Modelo actual (Finance):   CostCenter = "CC_SALES"
Modelo linkeado (HR):      busca Department = "CC_SALES"

Si los valores coinciden (mismo ID), el mapeo funciona.
Si no coinciden, necesitas un Cross-Model Copy step con mapeo en la UI.
```

---

## Ejemplo 25: LINK con MODEL/ENDMODEL — scope del modelo linkeado

**Caso:** Leer del modelo de ventas pero solo productos A y B, aggregando regiones.

```
LINK [Sales] = MODEL("t.local/Sales_Planning")

// MODEL/ENDMODEL define el scope y aggregacion del modelo linkeado
MODEL [Sales]
    MEMBERSET [d/Product] = ("Prod_A", "Prod_B")
    AGGREGATE_DIMENSIONS = [d/Region]
    AGGREGATE_WRITETO [d/Region] = "Total"
ENDMODEL

MEMBERSET [d/Date] = "202601" TO "202612"

DATA([d/Account] = "TotalUnits") =
    LINK [Sales] RESULTLOOKUP([d/Account] = "Units", [d/Region] = "Total")
```

**Que hace MODEL/ENDMODEL:**
- MEMBERSET dentro de MODEL restringe que datos lee del modelo linkeado
- AGGREGATE_DIMENSIONS indica que dimensiones agregar
- AGGREGATE_WRITETO define donde escribir el resultado agregado
- El RESULTLOOKUP lee el dato ya agregado

---

# PARTE 7: AGGREGATE

---

## Ejemplo 26: AGGREGATE_DIMENSIONS y AGGREGATE_WRITETO

**Caso:** Calcular el total de todas las empresas y escribirlo en "Group".

```
MEMBERSET [d/Date] = "202601" TO "202612"

// Definir que dimensiones agregar
AGGREGATE_DIMENSIONS = [d/Company]

// Definir donde escribir el resultado
AGGREGATE_WRITETO [d/Company] = "Group"

// El calculo. SAC suma TODAS las empresas automaticamente
DATA([d/Account] = "Revenue", [d/Company] = "Group") =
    RESULTLOOKUP([d/Account] = "Revenue")
```

**Sin AGGREGATE, tendrias que sumar manualmente:**

```
// Version manual (fragil, hay que actualizar si añades empresa)
DATA([d/Account] = "Revenue", [d/Company] = "Group") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Company] = "ES") +
    RESULTLOOKUP([d/Account] = "Revenue", [d/Company] = "DE") +
    RESULTLOOKUP([d/Account] = "Revenue", [d/Company] = "US")
```

---

# PARTE 8: INTERCOMPANY

---

## Ejemplo 27: ELIMMEMBER — eliminacion intercompany

**Caso:** Eliminar ventas/compras intercompany en la consolidacion.

```
CONFIG.GENERATE_UNBOOKED_DATA = OFF
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON

MEMBERSET [d/Time] = "202601" TO "202612"
MEMBERSET [d/Audit] = "10"

// Solo actuar sobre cuentas intercompany
IF [d/Account] = ("IC_AR", "IC_AP") THEN

    // ELIMMEMBER encuentra el miembro de eliminacion en la jerarquia
    // entre la entidad y su contraparte intercompany
    DATA(
        [d/Entity] = ELIMMEMBER(
            [d/Entity].[h/Consolidation],    // jerarquia de consolidacion
            [d/Entity],                       // entidad actual
            [d/Interco_Entity],              // entidad contraparte
            [d/Entity].[p/Elimination] = "Y" // filtro: solo nodos de eliminacion
        ),
        [d/Audit] = "30"
    ) = RESULTLOOKUP() * -1

    // Reversal en la cuenta de eliminacion
    DATA(
        [d/Entity] = ELIMMEMBER(
            [d/Entity].[h/Consolidation],
            [d/Entity],
            [d/Interco_Entity],
            [d/Entity].[p/Elimination] = "Y"
        ),
        [d/Account] = [d/Account].[p/ElimAccount],
        [d/Audit] = "30"
    ) = RESULTLOOKUP()

ENDIF
```

**Que hace ELIMMEMBER paso a paso:**

```
Jerarquia de consolidacion:
  GROUP
  ├── ELIM_ES_DE       ← nodo de eliminacion (p/Elimination = "Y")
  ├── ES               ← entidad
  └── DE               ← entidad

Cursor en: Entity=ES, Interco_Entity=DE
ELIMMEMBER busca el primer padre comun de ES y DE que tenga Elimination="Y"
→ Devuelve: "ELIM_ES_DE"

Entonces:
  DATA([d/Entity] = "ELIM_ES_DE", [d/Audit] = "30") = RESULTLOOKUP() * -1
  → Escribe el reversal de la venta intercompany en el nodo de eliminacion
```

---

# PARTE 9: CONFIG COMPLETO

---

## Ejemplo 28: Todas las opciones de CONFIG

```
// 1. Jerarquia temporal
CONFIG.TIME_HIERARCHY = CALENDARYEAR
// o: CONFIG.TIME_HIERARCHY = FISCALYEAR

// 2. Tratar celdas vacias como cero
CONFIG.GENERATE_UNBOOKED_DATA = ON
// OFF (defecto): celdas vacias = NULL, NULL en operaciones = NULL
// ON: celdas vacias = 0, se calculan normalmente

// 3. Signo segun tipo de cuenta
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON
// OFF (defecto): todo en valor absoluto
// ON: Expense se almacena como negativo → puedes sumar todo

// 4. Elegir jerarquia para una dimension
CONFIG.HIERARCHY [d/Region].[h/Geographic]
// Necesario cuando la dimension tiene multiples jerarquias

// 5. Offset de zona horaria
CONFIG.TIME_ZONE_OFFSET = +1
// Ajusta TODAY() para tu zona horaria (ej: +1 = CET)

// 6. Incluir miembros sin jerarquia
CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY [d/Product]
// Expande el scope para incluir productos no asignados a ninguna jerarquia
```

**Ejemplo practico de FLIPPING_SIGN:**

```
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON

// Con FLIPPING ON:
//   Revenue (INC) = 10000  → se almacena como 10000
//   Cost (EXP) = 6000      → se almacena como -6000
//
// NetProfit = Revenue + Cost = 10000 + (-6000) = 4000
// No necesitas restar! Puedes sumar todo:

DATA([d/Account] = "NetProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") +
    RESULTLOOKUP([d/Account] = "Cost") +
    RESULTLOOKUP([d/Account] = "Tax")

// Sin FLIPPING, tendrias que saber cuales restar:
// DATA([d/Account] = "NetProfit") = Revenue - Cost - Tax
```

---

# PARTE 10: VARIABLES AVANZADAS

---

## Ejemplo 29: INTEGER y FLOAT con ATTRIBUTE — depreciacion avanzada

**Caso:** Depreciacion con vida util variable por equipo y control de iteraciones.

```
MEMBERSET [d/Date] = "202601" TO "202712"
MEMBERSET [d/Account] = ("COST", "OPEN_BAL", "Depreciation_Exp", "RESIDUAL_VALUE")

INTEGER @usefulLife
INTEGER @iteration

FOREACH [d/Equipment]
    @iteration = 0
    @usefulLife = ATTRIBUTE([d/Equipment].[p/Useful_Life])

    FOREACH [d/Date] ASC
        // Si ya se deprecio completamente, parar
        IF @usefulLife <= @iteration THEN
            BREAK
        ENDIF

        // Primer mes: apertura = coste
        IF RESULTLOOKUP([d/Account] = "COST") > 0 THEN
            DATA([d/Account] = "OPEN_BAL") =
                RESULTLOOKUP([d/Account] = "COST")
            DATA([d/Account] = "Depreciation_Exp") =
                RESULTLOOKUP([d/Account] = "OPEN_BAL") / @usefulLife
            DATA([d/Account] = "RESIDUAL_VALUE") =
                RESULTLOOKUP([d/Account] = "OPEN_BAL") -
                RESULTLOOKUP([d/Account] = "Depreciation_Exp")
            @iteration = @iteration + 1
        ENDIF

        // Meses siguientes: apertura = residual anterior
        IF RESULTLOOKUP([d/Account] = "COST") = NULL THEN
            DATA([d/Account] = "OPEN_BAL") =
                RESULTLOOKUP([d/Date] = PREVIOUS(1), [d/Account] = "OPEN_BAL")
            DATA([d/Account] = "Depreciation_Exp") =
                RESULTLOOKUP([d/Account] = "OPEN_BAL") / @usefulLife
            DATA([d/Account] = "RESIDUAL_VALUE") =
                RESULTLOOKUP([d/Date] = PREVIOUS(1), [d/Account] = "RESIDUAL_VALUE") -
                RESULTLOOKUP([d/Account] = "Depreciation_Exp")
            @iteration = @iteration + 1
        ENDIF
    ENDFOR
ENDFOR
```

**Flujo para un equipo con Useful_Life=3 (meses):**

```
| Date   | COST  | OPEN_BAL | Depr_Exp | RESIDUAL | @iteration |
|--------|-------|----------|----------|----------|------------|
| 202601 | 9000  | 9000     | 3000     | 6000     | 1          |
| 202602 | null  | 9000     | 3000     | 3000     | 2          |
| 202603 | null  | 9000     | 3000     | 0        | 3 = @usefulLife → BREAK |
| 202604 | ---   | ---      | ---      | ---      | (no ejecuta) |
```

---

## Ejemplo 30: FLOAT para IRR simplificado (Newton-Raphson)

**Caso:** Calcular TIR (Tasa Interna de Retorno) iterativamente.

```
MEMBERSET [d/Date] = "202601"
MEMBERSET [d/Account] = ("CashFlow_Y0", "CashFlow_Y1", "CashFlow_Y2",
                         "CashFlow_Y3", "CashFlow_Y4", "CashFlow_Y5", "IRR")

FLOAT @rate
FLOAT @npv
FLOAT @npvPrime
FLOAT @cf0
FLOAT @cf1
FLOAT @cf2
FLOAT @cf3
FLOAT @cf4
FLOAT @cf5
INTEGER @i

// Leer cash flows (en una celda concreta, no multidimensional)
// NOTA: @variables no pueden recibir RESULTLOOKUP directamente en todas las versiones.
// Este patron usa VARIABLEMEMBER como intermediario.

VARIABLEMEMBER #IRR_Result OF [d/Account]

// Iniciar tasa al 10%
@rate = 0.10

// 20 iteraciones de Newton-Raphson
FOR @i = 1 TO 20 STEP 1
    // Calcular NPV y su derivada para @rate actual
    // NPV = CF0 + CF1/(1+r) + CF2/(1+r)^2 + ...
    // Este es un ejemplo simplificado; en produccion se usarian mas pasos
    @rate = @rate - 0.001  // Ajuste iterativo simplificado

    IF ABS(@rate) < 0.0001 THEN
        BREAK
    ENDIF
ENDFOR

// Guardar resultado
DATA([d/Account] = "IRR") = @rate * 100
```

---

# PARTE 11: HR Y WORKFORCE PLANNING

---

## Ejemplo 31: Headcount con turnover basado en historico

**Caso:** Proyectar bajas futuras usando la tasa de turnover historica.

```
CONFIG.GENERATE_UNBOOKED_DATA = OFF

MEMBERSET [d/Account] = "HEADCOUNT"
MEMBERSET [d/Movement] = ("OPENING", "CLOSING", "TURNOVERS")
MEMBERSET [d/Date] = "202601" TO "202612"

FOREACH [d/Date] ASC
    // Opening = closing del mes anterior
    DATA([d/Movement] = "OPENING") =
        RESULTLOOKUP([d/Movement] = "CLOSING", [d/Date] = PREVIOUS(1))

    // Turnovers = opening actual * (turnovers historicos / opening historico)
    // Lee la tasa del mismo mes del año anterior
    DATA([d/Movement] = "TURNOVERS") =
        RESULTLOOKUP([d/Movement] = "OPENING") *
        RESULTLOOKUP([d/Movement] = "TURNOVERS", [d/Date] = PREVIOUS(12)) /
        RESULTLOOKUP([d/Movement] = "OPENING", [d/Date] = PREVIOUS(12))

    // Closing = opening - turnovers
    DATA([d/Movement] = "CLOSING") =
        RESULTLOOKUP([d/Movement] = "OPENING") -
        RESULTLOOKUP([d/Movement] = "TURNOVERS")
ENDFOR
```

```
| Date   | OPENING | TURNOVERS | CLOSING | Historico TURNOVERS/OPENING |
|--------|---------|-----------|---------|----------------------------|
| 202501 | 200     | 10        | 190     | 10/200 = 5%               |
| 202601 | 180     | 9         | 171     | 180 * 5% = 9              |
| 202602 | 171     | ...       | ...     |                            |
```

---

## Ejemplo 32: Coste salarial con incremento por antiguedad

```
LINK [HR] = MODEL("t.local/HR_Detail")

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("BaseSalary", "SeniorityBonus", "TotalSalary", "StaffCost")

FOREACH.BOOKED [d/Employee]
    // Salario base del modelo HR
    DATA([d/Account] = "BaseSalary") =
        LINK [HR] RESULTLOOKUP([d/Account] = "AnnualSalary") / 12

    // Bonus por antiguedad: 1% extra por cada año de servicio
    // Lee años de servicio del atributo de la dimension Employee
    DATA([d/Account] = "SeniorityBonus") =
        RESULTLOOKUP([d/Account] = "BaseSalary") *
        ATTRIBUTE([d/Employee].[p/YearsOfService]) / 100

    // Total
    DATA([d/Account] = "TotalSalary") =
        RESULTLOOKUP([d/Account] = "BaseSalary") +
        RESULTLOOKUP([d/Account] = "SeniorityBonus")
ENDFOR

// Coste total por departamento (agregado)
AGGREGATE_DIMENSIONS = [d/Employee]
AGGREGATE_WRITETO [d/Employee] = "ALL"

DATA([d/Account] = "StaffCost", [d/Employee] = "ALL") =
    RESULTLOOKUP([d/Account] = "TotalSalary")
```

---

# PARTE 12: PATRONES DE RENDIMIENTO

---

## Ejemplo 33: IF fuera de FOREACH (patron correcto)

```
// MAL: IF dentro de FOREACH → evalua IF en CADA iteracion
FOREACH [d/Date] ASC
    IF [d/Company] = "ES" THEN
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
    ENDIF
ENDFOR
// Recorre 12 meses x TODAS las empresas, filtra despues

// BIEN: IF fuera de FOREACH → solo itera si cumple condicion
IF [d/Company] = "ES" THEN
    FOREACH [d/Date] ASC
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
    ENDFOR
ENDIF
// Solo recorre 12 meses para ES, ignora el resto completamente
```

---

## Ejemplo 34: VARIABLEMEMBER para evitar RESULTLOOKUP repetidos

```
MEMBERSET [d/Date] = "202601" TO "202612"

VARIABLEMEMBER #Rev OF [d/Account]
VARIABLEMEMBER #Cost OF [d/Account]

// Leer una vez, usar muchas veces
DATA([d/Account] = #Rev) = RESULTLOOKUP([d/Account] = "Revenue")
DATA([d/Account] = #Cost) = RESULTLOOKUP([d/Account] = "Cost")

// Profit
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = #Rev) -
    RESULTLOOKUP([d/Account] = #Cost)

// Margin %
IF RESULTLOOKUP([d/Account] = #Rev) <> 0 THEN
    DATA([d/Account] = "MarginPct") =
        (RESULTLOOKUP([d/Account] = #Rev) - RESULTLOOKUP([d/Account] = #Cost)) /
        RESULTLOOKUP([d/Account] = #Rev) * 100
ENDIF

// Tax
DATA([d/Account] = "Tax") =
    (RESULTLOOKUP([d/Account] = #Rev) - RESULTLOOKUP([d/Account] = #Cost)) * 0.25

// Net
DATA([d/Account] = "Net") =
    RESULTLOOKUP([d/Account] = "Profit") -
    RESULTLOOKUP([d/Account] = "Tax")
```

Sin #Rev y #Cost, Revenue y Cost se leerian del modelo 6+ veces cada uno.

---

## Ejemplo 35: FOREACH.BOOKED vs FOREACH — impacto real

```
// Modelo con 10.000 productos, solo 200 tienen datos en Plan

// FOREACH: 10.000 iteraciones
FOREACH [d/Product]
    DATA([d/Account] = "Margin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")
ENDFOR
// Tiempo: ~60 segundos

// FOREACH.BOOKED: 200 iteraciones
FOREACH.BOOKED [d/Product]
    DATA([d/Account] = "Margin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")
ENDFOR
// Tiempo: ~1 segundo

// SIN FOREACH: SAC paraleliza automaticamente
DATA([d/Account] = "Margin") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
// Tiempo: <1 segundo (no hay dependencia temporal)
```

**Regla**: Sin FOREACH > FOREACH.BOOKED > FOREACH

---

# PARTE 13: ESCENARIOS FINANCIEROS COMPLEJOS

---

## Ejemplo 36: Cash Flow indirecto

**Caso:** Calcular cash flow partiendo del beneficio neto y ajustando por working capital.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

// 1. Operating Cash Flow = Net Profit + Depreciation (non-cash) - Delta Working Capital
DATA([d/Account] = "DeltaWC") =
    (RESULTLOOKUP([d/Account] = "Receivables") -
     RESULTLOOKUP([d/Account] = "Receivables", [d/Date] = PREVIOUS(1))) -
    (RESULTLOOKUP([d/Account] = "Payables") -
     RESULTLOOKUP([d/Account] = "Payables", [d/Date] = PREVIOUS(1)))

DATA([d/Account] = "OperatingCF") =
    RESULTLOOKUP([d/Account] = "NetProfit") +
    RESULTLOOKUP([d/Account] = "Depreciation") -
    RESULTLOOKUP([d/Account] = "DeltaWC")

// 2. Investing Cash Flow
DATA([d/Account] = "InvestingCF") =
    RESULTLOOKUP([d/Account] = "CAPEX") * -1

// 3. Financing Cash Flow
DATA([d/Account] = "FinancingCF") =
    RESULTLOOKUP([d/Account] = "NewDebt") -
    RESULTLOOKUP([d/Account] = "DebtRepayment") -
    RESULTLOOKUP([d/Account] = "Dividends")

// 4. Total Cash Flow
DATA([d/Account] = "TotalCF") =
    RESULTLOOKUP([d/Account] = "OperatingCF") +
    RESULTLOOKUP([d/Account] = "InvestingCF") +
    RESULTLOOKUP([d/Account] = "FinancingCF")

// 5. Cash Balance (acumulativo → necesita FOREACH)
FOREACH [d/Date] ASC
    DATA([d/Account] = "CashBalance") =
        RESULTLOOKUP([d/Account] = "CashBalance", [d/Date] = PREVIOUS(1)) +
        RESULTLOOKUP([d/Account] = "TotalCF")
ENDFOR
```

---

## Ejemplo 37: Driver-Based Planning — Revenue por drivers

**Caso:** Revenue = Unidades * Precio * (1 - Descuento%), con drivers por producto.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

FOREACH.BOOKED [d/Product]
    // Revenue = Units * Price * (1 - Discount%)
    DATA([d/Account] = "Revenue") =
        RESULTLOOKUP([d/Account] = "Units") *
        RESULTLOOKUP([d/Account] = "Price") *
        (1 - RESULTLOOKUP([d/Account] = "DiscountPct") / 100)

    // COGS = Units * UnitCost
    DATA([d/Account] = "COGS") =
        RESULTLOOKUP([d/Account] = "Units") *
        RESULTLOOKUP([d/Account] = "UnitCost")

    // Gross Margin
    DATA([d/Account] = "GrossMargin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "COGS")
ENDFOR
```

**Los drivers (Units, Price, DiscountPct, UnitCost) los introduce el usuario.**
El script solo calcula los derivados. Si el usuario cambia Price, el script
recalcula Revenue, COGS y GrossMargin automaticamente.

---

## Ejemplo 38: Conversion de moneda con tipo de cambio mensual

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Actual"

// El tipo de cambio esta almacenado por mes en la cuenta FX_to_EUR
// Cada empresa tiene su moneda local

// Revenue en moneda local → Revenue en EUR
DATA([d/Account] = "Revenue_EUR") =
    RESULTLOOKUP([d/Account] = "Revenue_LC") *
    RESULTLOOKUP([d/Account] = "FX_to_EUR")

// Balance Sheet: se convierte al tipo de cambio de cierre (ultimo dia del mes)
DATA([d/Account] = "Assets_EUR") =
    RESULTLOOKUP([d/Account] = "Assets_LC") *
    RESULTLOOKUP([d/Account] = "FX_Closing")

// Diferencia de conversion
DATA([d/Account] = "FX_Difference") =
    RESULTLOOKUP([d/Account] = "Assets_EUR") -
    RESULTLOOKUP([d/Account] = "Assets_LC") *
    RESULTLOOKUP([d/Account] = "FX_Average")
```

```
| Date   | Company | Revenue_LC | FX_to_EUR | Revenue_EUR |
|--------|---------|------------|-----------|-------------|
| 202601 | US      | 100000     | 0.92      | 92000       |
| 202602 | US      | 110000     | 0.91      | 100100      |
| 202601 | UK      | 80000      | 1.17      | 93600       |
```

---

## Ejemplo 39: What-If con multiples escenarios

**Caso:** Simular impacto de diferentes combinaciones de cambios.

```
// Parametros:
// %REVENUE_DELTA% = -10 (%)
// %COST_DELTA% = -5 (%)
// %FX_DELTA% = +3 (% cambio en tipo de cambio)
// %SCENARIO% = "Pessimistic"

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = %SCENARIO%

// 1. Copiar base del Plan
DATA() = RESULTLOOKUP([d/Version] = "Plan")

// 2. Ajustar Revenue
DATA([d/Account] = "Revenue") =
    RESULTLOOKUP() * (1 + %REVENUE_DELTA% / 100)

// 3. Ajustar Costes
DATA([d/Account] = "Cost") =
    RESULTLOOKUP() * (1 + %COST_DELTA% / 100)

// 4. Ajustar tipo de cambio
DATA([d/Account] = "FX_Rate") =
    RESULTLOOKUP() * (1 + %FX_DELTA% / 100)

// 5. Recalcular cascada P&L completa
DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "COGS")

DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -
    RESULTLOOKUP([d/Account] = "OPEX")

DATA([d/Account] = "NetProfit") =
    RESULTLOOKUP([d/Account] = "EBIT") * 0.75
```

El CFO ejecuta esto con diferentes combinaciones de parametros.
Cada ejecucion crea un escenario completo en segundos.

---

## Ejemplo 40: Amortizacion de prestamo (sistema frances)

```
MEMBERSET [d/Date] = "202601" TO "203012"
MEMBERSET [d/Account] = ("Balance", "MonthlyPayment", "InterestPart",
                         "CapitalPart")

// %LOAN_AMOUNT% = 500000
// %ANNUAL_RATE% = 5
// %MONTHLY_PAYMENT% = 9435.62 (precalculado)

FLOAT @monthlyRate
@monthlyRate = %ANNUAL_RATE% / 100 / 12

// Mes 1
DATA([d/Account] = "Balance", [d/Date] = "202601") = %LOAN_AMOUNT%
DATA([d/Account] = "MonthlyPayment") = %MONTHLY_PAYMENT%

DATA([d/Account] = "InterestPart", [d/Date] = "202601") =
    %LOAN_AMOUNT% * @monthlyRate

DATA([d/Account] = "CapitalPart", [d/Date] = "202601") =
    %MONTHLY_PAYMENT% -
    RESULTLOOKUP([d/Account] = "InterestPart", [d/Date] = "202601")

// Mes 2 en adelante
MEMBERSET [d/Date] = "202602" TO "203012"

FOREACH [d/Date] ASC
    DATA([d/Account] = "Balance") =
        RESULTLOOKUP([d/Account] = "Balance", [d/Date] = PREVIOUS(1)) -
        RESULTLOOKUP([d/Account] = "CapitalPart", [d/Date] = PREVIOUS(1))

    DATA([d/Account] = "InterestPart") =
        RESULTLOOKUP([d/Account] = "Balance") * @monthlyRate

    DATA([d/Account] = "CapitalPart") =
        %MONTHLY_PAYMENT% -
        RESULTLOOKUP([d/Account] = "InterestPart")
ENDFOR
```

```
| Date   | Balance  | Payment | Interest | Capital |
|--------|----------|---------|----------|---------|
| 202601 | 500000   | 9435.62 | 2083.33  | 7352.29 |
| 202602 | 492647.7 | 9435.62 | 2052.70  | 7382.92 |
| 202603 | 485264.8 | 9435.62 | 2021.94  | 7413.68 |
```

---

# PARTE 14: ANTI-PATRONES Y DEBUGGING

---

## Anti-patron 1: FOREACH innecesario

```
// MAL - 10x mas lento sin razon
FOREACH [d/Date]
    DATA([d/Account] = "Profit") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// BIEN - SAC paraleliza automaticamente
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

**Regla:** Solo FOREACH cuando hay dependencia entre iteraciones (PREVIOUS/NEXT
sobre datos recien calculados). Si cada celda es independiente, dejalo sin FOREACH.

---

## Anti-patron 2: IF dentro de FOREACH

```
// MAL - evalua IF 12 veces por cada empresa
FOREACH [d/Date] ASC
    IF [d/Account] = "Revenue" THEN
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.05
    ENDIF
ENDFOR

// BIEN - IF filtra ANTES de iterar
IF [d/Account] = "Revenue" THEN
    FOREACH [d/Date] ASC
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.05
    ENDFOR
ENDIF
```

---

## Anti-patron 3: Division por cero

```
// MAL - explota si Revenue = 0
DATA([d/Account] = "MarginPct") =
    RESULTLOOKUP([d/Account] = "Profit") /
    RESULTLOOKUP([d/Account] = "Revenue") * 100

// BIEN - proteccion explicita
IF RESULTLOOKUP([d/Account] = "Revenue") <> 0 THEN
    DATA([d/Account] = "MarginPct") =
        RESULTLOOKUP([d/Account] = "Profit") /
        RESULTLOOKUP([d/Account] = "Revenue") * 100
ELSE
    DATA([d/Account] = "MarginPct") = 0
ENDIF
```

---

## Anti-patron 4: FOREACH sin ASC con PREVIOUS

```
// MAL - orden no garantizado, PREVIOUS puede leer datos aun no calculados
FOREACH [d/Date]
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR

// BIEN - ASC garantiza el orden
FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR
```

---

## Anti-patron 5: Olvidar MEMBERSET

```
// MAL - actua sobre TODO el modelo: todos los años, todas las versiones
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Revenue") * 1.1

// BIEN - scope explicito y restrictivo
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"
MEMBERSET [d/Account] = "Revenue"

DATA() = RESULTLOOKUP() * 1.1
```

---

## Anti-patron 6: RESULTLOOKUP de celda vacia sin CONFIG

```
// Revenue de 202601 no tiene dato → RESULTLOOKUP devuelve NULL

DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
// NULL * 0.21 = NULL → Tax queda vacia (no escribe nada)

// Solucion A: CONFIG global
CONFIG.GENERATE_UNBOOKED_DATA = ON
// Ahora celdas vacias = 0. Tax = 0 * 0.21 = 0

// Solucion B: Comprobar NULL celda a celda
IF RESULTLOOKUP([d/Account] = "Revenue") <> NULL THEN
    DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
ELSE
    DATA([d/Account] = "Tax") = 0
ENDIF
```

---

## Debugging: Tracepoints

SAC permite depurar scripts con tracepoints (como breakpoints):

1. **Abrir el editor de Advanced Formula en modo Script** (no grafico)
2. **Clic en el margen izquierdo** de una linea para poner un tracepoint (max 20)
3. **Tracepoints condicionales** dentro de bucles (max 10 por bucle):
   - Ej: parar solo cuando Date = "202603" AND Company = "ES"
4. **Watch Area** muestra:
   - Valores de parametros
   - Valores de variables (@, #)
   - Scope actual del calculo
   - Resultado del ultimo RESULTLOOKUP
5. **Version de tracing**: la depuracion crea una version separada, no modifica datos reales

---

## Checklist antes de ejecutar

- [ ] MEMBERSET definido para TODAS las dimensiones relevantes
- [ ] Scope lo mas restrictivo posible
- [ ] FOREACH solo cuando hay dependencia entre iteraciones
- [ ] FOREACH tiene ASC/DESC si usa PREVIOUS/NEXT
- [ ] IF fuera de FOREACH, no al reves
- [ ] Division por cero protegida
- [ ] CONFIG.GENERATE_UNBOOKED_DATA si necesitas tratar vacios como 0
- [ ] FOREACH.BOOKED en vez de FOREACH cuando sea posible
- [ ] VARIABLEMEMBER para RESULTLOOKUP repetidos
- [ ] Probado con scope pequeño primero (1 mes, 1 empresa)
