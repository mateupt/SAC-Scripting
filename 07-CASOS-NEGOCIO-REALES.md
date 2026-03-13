# Casos de Negocio Reales - De principio a fin

Cada caso incluye: contexto de negocio, datos de ejemplo, script completo,
y explicacion paso a paso.

---

## CASO 1: Forecast Rolling

### Contexto
Tu empresa cierra los libros cada mes. El controller quiere un Forecast que
use datos reales para los meses cerrados y proyecciones para el resto.
Marzo acaba de cerrar. Abril a diciembre se proyectan con un +5% sobre el
mismo mes del año anterior.

### Datos antes del script
```
           Ene    Feb    Mar    Abr    May    Jun  ...  Dic
Actual:    100    110    105    ---    ---    ---       ---
Plan:      95     108    100    120    115    130       140
Forecast:  ---    ---    ---    ---    ---    ---       ---
```

### Script
```
// Parametro: %LAST_CLOSED% = "202603" (marzo cerrado)

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Forecast"

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

### Resultado
```
           Ene    Feb    Mar    Abr    May    Jun  ...
Forecast:  100    110    105    ←real  ←real  ←real
                               aprAnt mayAnt junAnt
                               *1.05  *1.05  *1.05
```

### Por que dos FOREACH separados
Podriamos usar un solo FOREACH con IF/ELSE. Pero separandolos:
- El codigo es mas legible (un bloque por logica)
- Si necesitas cambiar la regla de proyeccion, solo tocas el paso 2
- En scripts largos, la separacion evita errores

---

## CASO 2: Distribucion de top-down a bottom-up

### Contexto
El director financiero ha puesto un objetivo de Revenue total de 1.000.000 EUR
para 2026. Quieres distribuirlo entre productos proporcionalmente a lo que
vendio cada uno en 2025.

### Datos
```
                    2025 Actual    Peso      2026 Plan (objetivo: 1M)
Producto_A:         400.000        40%       400.000
Producto_B:         350.000        35%       350.000
Producto_C:         250.000        25%       250.000
Total:            1.000.000       100%     1.000.000
```

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Version] = "Plan"

// Variable para guardar el total del año pasado
VARIABLEMEMBER #LastYearTotal OF [d/Account]

// Paso 1: Calcular total de 2025 (suma de todos los productos)
// Usamos AGGREGATE_WRITETO para obtener el total agregado
AGGREGATE_WRITETO [d/Product] = "ALL_PRODUCTS"

DATA([d/Account] = #LastYearTotal, [d/Product] = "ALL_PRODUCTS") =
    RESULTLOOKUP([d/Version] = "Actual", [d/Date] = PREVIOUS(12),
                 [d/Product] = "ALL_PRODUCTS")

// Paso 2: Para cada producto, su peso = ventas2025 / total2025
// Paso 3: Aplicar peso al objetivo
// %TARGET_REVENUE% = 1000000 (parametro numerico)

FOREACH.BOOKED [d/Product]
    DATA() =
        RESULTLOOKUP([d/Version] = "Actual", [d/Date] = PREVIOUS(12)) /
        RESULTLOOKUP([d/Account] = #LastYearTotal, [d/Product] = "ALL_PRODUCTS") *
        %TARGET_REVENUE% / 12
ENDFOR
```

### Que hace cada linea
1. Lee las ventas de cada producto en el mismo mes de 2025
2. Divide entre el total 2025 → obtiene el peso (%)
3. Multiplica el objetivo anual por ese peso
4. Divide entre 12 para distribuir mensualmente

---

## CASO 3: Calculo de P&L completo con cascada

### Contexto
Necesitas calcular una cuenta de resultados completa donde cada linea
depende de las anteriores.

### Estructura de la P&L
```
Revenue                    ← dato base (ya existe)
- COGS                     ← dato base (ya existe)
= Gross Profit             ← calculado
- OPEX                     ← dato base (ya existe)
= EBIT                     ← calculado
- Interest                 ← dato base (ya existe)
= EBT                      ← calculado
- Tax (25%)                ← calculado sobre EBT
= Net Profit               ← calculado
```

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"
MEMBERSET [d/Account] = ("Revenue", "COGS", "GrossProfit", "OPEX",
                         "EBIT", "Interest", "EBT", "Tax", "NetProfit")

// Gross Profit = Revenue - COGS
DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "COGS")

// EBIT = Gross Profit - OPEX
DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -
    RESULTLOOKUP([d/Account] = "OPEX")

// EBT = EBIT - Interest
DATA([d/Account] = "EBT") =
    RESULTLOOKUP([d/Account] = "EBIT") -
    RESULTLOOKUP([d/Account] = "Interest")

// Tax = 25% de EBT (solo si EBT > 0)
IF RESULTLOOKUP([d/Account] = "EBT") > 0 THEN
    DATA([d/Account] = "Tax") =
        RESULTLOOKUP([d/Account] = "EBT") * 0.25
ELSE
    DATA([d/Account] = "Tax") = 0
ENDIF

// Net Profit = EBT - Tax
DATA([d/Account] = "NetProfit") =
    RESULTLOOKUP([d/Account] = "EBT") -
    RESULTLOOKUP([d/Account] = "Tax")
```

### Nota importante
Los RESULTLOOKUP leen los datos YA ESCRITOS por los DATA anteriores
en el mismo script. El orden importa:
- GrossProfit se calcula primero
- EBIT lee GrossProfit (ya calculado)
- EBT lee EBIT (ya calculado)
- Y asi sucesivamente

NO necesitas FOREACH aqui porque no hay dependencia entre meses.
Cada mes se calcula de forma independiente.

---

## CASO 4: Conversion de moneda simplificada

### Contexto
Tienes ventas en moneda local (EUR, USD, GBP) y necesitas convertir
todo a EUR para el reporting consolidado.

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue_LC", "Revenue_EUR")
MEMBERSET [d/Version] = "Actual"

// Tipo de cambio almacenado en otra cuenta del modelo
// FX_to_EUR: EUR=1, USD=0.92, GBP=1.17

IF [d/Company] = "ES" THEN
    // España ya esta en EUR
    DATA([d/Account] = "Revenue_EUR") =
        RESULTLOOKUP([d/Account] = "Revenue_LC")

ELSEIF [d/Company] = "US" THEN
    DATA([d/Account] = "Revenue_EUR") =
        RESULTLOOKUP([d/Account] = "Revenue_LC") *
        RESULTLOOKUP([d/Account] = "FX_to_EUR")

ELSEIF [d/Company] = "UK" THEN
    DATA([d/Account] = "Revenue_EUR") =
        RESULTLOOKUP([d/Account] = "Revenue_LC") *
        RESULTLOOKUP([d/Account] = "FX_to_EUR")
ENDIF
```

### Version mas limpia (sin repetir)
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Actual"

// FX_to_EUR tiene 1.0 para EUR, 0.92 para USD, 1.17 para GBP
// Asi funciona para TODAS las empresas sin IF
DATA([d/Account] = "Revenue_EUR") =
    RESULTLOOKUP([d/Account] = "Revenue_LC") *
    RESULTLOOKUP([d/Account] = "FX_to_EUR")
```

Leccion: muchas veces un buen diseño del modelo elimina la necesidad
de logica condicional en el script.

---

## CASO 5: Proyeccion de headcount con contrataciones y bajas

### Contexto
RRHH planifica headcount mes a mes. Parten de un headcount inicial
y cada mes se suman contrataciones y se restan bajas.

### Datos
```
           Ene    Feb    Mar    Abr    May    Jun
Hires:      5      3      4      2      6      3
Leavers:    2      1      3      1      2      4
HC_Start:  100    ---    ---    ---    ---    ---   ← solo enero tiene dato
HC_End:    ---    ---    ---    ---    ---    ---
```

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("HC_Start", "HC_End", "Hires", "Leavers")

// Enero: HC_End = HC_Start + Hires - Leavers
DATA([d/Account] = "HC_End", [d/Date] = "202601") =
    RESULTLOOKUP([d/Account] = "HC_Start", [d/Date] = "202601") +
    RESULTLOOKUP([d/Account] = "Hires", [d/Date] = "202601") -
    RESULTLOOKUP([d/Account] = "Leavers", [d/Date] = "202601")

// Feb en adelante: HC_Start = HC_End del mes anterior
MEMBERSET [d/Date] = "202602" TO "202612"

FOREACH [d/Date] ASC
    // El inicio de este mes = el final del mes pasado
    DATA([d/Account] = "HC_Start") =
        RESULTLOOKUP([d/Account] = "HC_End", [d/Date] = PREVIOUS(1))

    // El final = inicio + contrataciones - bajas
    DATA([d/Account] = "HC_End") =
        RESULTLOOKUP([d/Account] = "HC_Start") +
        RESULTLOOKUP([d/Account] = "Hires") -
        RESULTLOOKUP([d/Account] = "Leavers")
ENDFOR
```

### Resultado
```
           Ene    Feb    Mar    Abr    May    Jun
HC_Start:  100    103    105    106    107    111
Hires:       5      3      4      2      6      3
Leavers:     2      1      3      1      2      4
HC_End:    103    105    106    107    111    110
```

### Por que FOREACH es obligatorio aqui
Febrero necesita HC_End de enero. Marzo necesita HC_End de febrero.
Es una cadena de dependencias → FOREACH ASC.

---

## CASO 6: Escenario What-If con parametros

### Contexto
El CFO quiere simular: "¿Que pasa si Revenue cae un X% y reducimos OPEX un Y%?"

### Script
```
// Parametros:
// %REVENUE_CHANGE% = -10 (caida del 10%)
// %OPEX_CHANGE% = -15 (reduccion del 15%)
// %SCENARIO_VERSION% = "WhatIf_1"

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = %SCENARIO_VERSION%

// Copiar base del Plan
DATA() = RESULTLOOKUP([d/Version] = "Plan")

// Ajustar Revenue
DATA([d/Account] = "Revenue") =
    RESULTLOOKUP([d/Account] = "Revenue") *
    (1 + %REVENUE_CHANGE% / 100)

// Ajustar OPEX
DATA([d/Account] = "OPEX") =
    RESULTLOOKUP([d/Account] = "OPEX") *
    (1 + %OPEX_CHANGE% / 100)

// Recalcular P&L
DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "COGS")

DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -
    RESULTLOOKUP([d/Account] = "OPEX")

DATA([d/Account] = "NetProfit") =
    RESULTLOOKUP([d/Account] = "EBIT") * 0.75
```

### Flujo
1. Copia todo el Plan a la version WhatIf
2. Ajusta Revenue y OPEX con los porcentajes del usuario
3. Recalcula la cascada de P&L

El CFO puede ejecutar esto multiples veces con diferentes parametros
para comparar escenarios.

---

## CASO 7: Amortizacion lineal de activos

### Contexto
Tienes activos con un coste y una vida util. Necesitas calcular
la amortizacion mensual.

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("AssetCost", "UsefulLife_Months", "MonthlyDepr",
                         "AccumDepr", "NetBookValue")

// Amortizacion mensual = Coste / Vida util en meses
DATA([d/Account] = "MonthlyDepr") =
    RESULTLOOKUP([d/Account] = "AssetCost") /
    RESULTLOOKUP([d/Account] = "UsefulLife_Months")

// Amortizacion acumulada (necesita FOREACH por dependencia temporal)
// Enero: acumulado = solo la amortizacion de enero
DATA([d/Account] = "AccumDepr", [d/Date] = "202601") =
    RESULTLOOKUP([d/Account] = "MonthlyDepr", [d/Date] = "202601")

MEMBERSET [d/Date] = "202602" TO "202612"

FOREACH [d/Date] ASC
    DATA([d/Account] = "AccumDepr") =
        RESULTLOOKUP([d/Account] = "AccumDepr", [d/Date] = PREVIOUS(1)) +
        RESULTLOOKUP([d/Account] = "MonthlyDepr")
ENDFOR

// Valor neto contable
MEMBERSET [d/Date] = "202601" TO "202612"

DATA([d/Account] = "NetBookValue") =
    RESULTLOOKUP([d/Account] = "AssetCost") -
    RESULTLOOKUP([d/Account] = "AccumDepr")
```

### Estructura del calculo
```
Coste del activo:     120.000 EUR
Vida util:            60 meses
Amort mensual:        2.000 EUR

Ene    Feb    Mar    ...
MonthlyDepr:   2000   2000   2000
AccumDepr:     2000   4000   6000   ← acumulativo (FOREACH)
NetBookValue: 118000 116000 114000  ← coste - acumulado
```

---

## CASO 8: Consolidacion multi-empresa

### Contexto
Tres filiales reportan por separado. Necesitas una fila "Group" que sume
todo y elimine operaciones intercompany.

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Actual"

// Paso 1: Sumar todas las empresas en "Group"
AGGREGATE_WRITETO [d/Company] = "Group"

DATA([d/Company] = "Group") =
    RESULTLOOKUP([d/Company] = "ES") +
    RESULTLOOKUP([d/Company] = "DE") +
    RESULTLOOKUP([d/Company] = "US")

// Paso 2: Eliminar intercompany
// Las ventas intercompany estan en la cuenta "IC_Revenue"
// Las compras intercompany estan en "IC_Cost"
// En consolidacion se eliminan (suman cero entre empresas)

DATA([d/Account] = "IC_Revenue", [d/Company] = "Group") = 0
DATA([d/Account] = "IC_Cost", [d/Company] = "Group") = 0

// Paso 3: Recalcular P&L del grupo
DATA([d/Account] = "GrossProfit", [d/Company] = "Group") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Company] = "Group") -
    RESULTLOOKUP([d/Account] = "COGS", [d/Company] = "Group")
```

---

## CASO 9: Eliminacion intercompany con ELIMMEMBER

### Contexto
Tres filiales se venden entre si. El equipo de consolidacion necesita eliminar
las ventas/compras intercompany automaticamente usando la jerarquia de consolidacion.
Las cuentas IC_AR (cuentas a cobrar intercompany) e IC_AP (cuentas a pagar) deben
cancelarse en el nodo de eliminacion correspondiente.

### Jerarquia de consolidacion
```
GROUP
├── ELIM_ES_DE       ← nodo de eliminacion (atributo Elimination = "Y")
├── ELIM_ES_US       ← nodo de eliminacion
├── ELIM_DE_US       ← nodo de eliminacion
├── ES               ← filial España
├── DE               ← filial Alemania
└── US               ← filial USA
```

### Dimensiones del modelo
- `[d/Entity]` — las filiales (ES, DE, US) y nodos de eliminacion
- `[d/Interco_Entity]` — la contraparte intercompany
- `[d/Account]` — cuentas (IC_AR, IC_AP, y sus cuentas de eliminacion en atributo [p/ElimAccount])
- `[d/Audit]` — pista de auditoria ("10" = dato original, "30" = eliminacion)

### Script
```
CONFIG.GENERATE_UNBOOKED_DATA = OFF
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Audit] = "10"

// Solo actuar sobre cuentas intercompany
IF [d/Account] = ("IC_AR", "IC_AP") THEN

    // Paso 1: Reversal — escribir el negativo en el nodo de eliminacion
    // ELIMMEMBER() busca en la jerarquia el primer padre comun de Entity y
    // Interco_Entity que tenga el atributo Elimination = "Y"
    DATA(
        [d/Entity] = ELIMMEMBER(
            [d/Entity].[h/Consolidation],
            [d/Entity],
            [d/Interco_Entity],
            [d/Entity].[p/Elimination] = "Y"
        ),
        [d/Audit] = "30"
    ) = RESULTLOOKUP() * -1

    // Paso 2: Reclasificacion — mover a la cuenta de eliminacion
    // [d/Account].[p/ElimAccount] es un atributo que mapea IC_AR → ELIM_AR, IC_AP → ELIM_AP
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

### Que pasa paso a paso

Ejemplo: ES vende 100.000 a DE.

```
ANTES (dato original, Audit=10):
  Entity=ES, Interco=DE, Account=IC_AR, Valor=100000
  Entity=DE, Interco=ES, Account=IC_AP, Valor=100000

ELIMMEMBER busca en la jerarquia:
  Padre comun de ES y DE con Elimination="Y" → ELIM_ES_DE

DESPUES (Audit=30, eliminacion):
  Entity=ELIM_ES_DE, Account=IC_AR, Valor=-100000  ← reversal paso 1
  Entity=ELIM_ES_DE, Account=ELIM_AR, Valor=100000 ← reclasificacion paso 2
  Entity=ELIM_ES_DE, Account=IC_AP, Valor=-100000  ← reversal paso 1
  Entity=ELIM_ES_DE, Account=ELIM_AP, Valor=100000 ← reclasificacion paso 2

RESULTADO CONSOLIDADO (Group):
  IC_AR:  100000 (ES) + (-100000) (ELIM) = 0  ← eliminado
  IC_AP:  100000 (DE) + (-100000) (ELIM) = 0  ← eliminado
```

### Por que usar ELIMMEMBER en vez de hardcodear
- Si añades una filial nueva, ELIMMEMBER la detecta automaticamente via jerarquia
- No necesitas IF por cada par de filiales
- El auditor ve las eliminaciones separadas en Audit=30

---

## CASO 10: Arrastre de saldos con CARRYFORWARD

### Contexto
Modelo de balance con apertura, cierre y movimientos intermedios. El opening
de cada mes es el closing del anterior. Clasicamente se hace con FOREACH, pero
CARRYFORWARD es mas rapido y limpio.

### Datos
```
Dimension [d/Flow]: OPENING, HIRES, TERMINATIONS, TRANSFERS, CLOSING
Dimension [d/Date]: 202601 a 202612
Dimension [d/Account]: HEADCOUNT
```

### Script con FOREACH (clasico, lento)
```
MEMBERSET [d/Date] = "202601" TO "202612"

FOREACH [d/Date] ASC
    DATA([d/Flow] = "OPENING") =
        RESULTLOOKUP([d/Flow] = "CLOSING", [d/Date] = PREVIOUS(1))

    DATA([d/Flow] = "CLOSING") =
        RESULTLOOKUP([d/Flow] = "OPENING") +
        RESULTLOOKUP([d/Flow] = "HIRES") -
        RESULTLOOKUP([d/Flow] = "TERMINATIONS") +
        RESULTLOOKUP([d/Flow] = "TRANSFERS")
ENDFOR
```

### Script con CARRYFORWARD (optimizado)
```
MEMBERSET [d/Date] = "202601" TO "202612"

DATA() = CARRYFORWARD(
    [d/Flow],           // dimension de movimiento
    "OPENING",          // miembro de apertura
    "CLOSING",          // miembro de cierre (se calcula automaticamente)
    "HIRES",            // primer flujo
    "TERMINATIONS",     // segundo flujo
    "TRANSFERS"         // tercer flujo
)
```

CARRYFORWARD hace internamente lo que el FOREACH hace manualmente:
1. Opening del mes = Closing del mes anterior
2. Closing = Opening + todos los flujos
3. Repite para cada periodo en orden

### Ventajas de CARRYFORWARD
- Mas rapido (optimizado a nivel de motor SAP)
- Menos codigo (1 linea vs 8 lineas)
- Menos errores (no te olvidas de un flujo)
- SAP puede optimizar la ejecucion internamente

### Resultado
```
| Flow         | 202601 | 202602 | 202603 |
|--------------|--------|--------|--------|
| OPENING      | 100    | 108    | 113    |
| HIRES        | 12     | 8      | 5      |
| TERMINATIONS | 3      | 2      | 4      |
| TRANSFERS    | -1     | -1     | 0      |
| CLOSING      | 108    | 113    | 114    |
```

---

## CASO 11: Driver-Based Planning con ATTRIBUTE

### Contexto
Revenue se calcula a partir de drivers (Units, Price, Discount) que el usuario
introduce. Cada producto tiene un tipo de pricing diferente almacenado como
atributo de la dimension Product.

### Atributos de la dimension Product
```
| Miembro | [p/PricingType] | [p/MinMargin] |
|---------|-----------------|---------------|
| Prod_A  | standard        | 20            |
| Prod_B  | volume_discount | 15            |
| Prod_C  | premium         | 35            |
```

### Script
```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

FOREACH.BOOKED [d/Product]

    // Calculo de Revenue segun tipo de pricing
    IF ATTRIBUTE([d/Product].[p/PricingType]) = "standard" THEN
        DATA([d/Account] = "Revenue") =
            RESULTLOOKUP([d/Account] = "Units") *
            RESULTLOOKUP([d/Account] = "Price")

    ELSEIF ATTRIBUTE([d/Product].[p/PricingType]) = "volume_discount" THEN
        // Descuento escalonado: >1000 unidades = 10% dto
        IF RESULTLOOKUP([d/Account] = "Units") > 1000 THEN
            DATA([d/Account] = "Revenue") =
                RESULTLOOKUP([d/Account] = "Units") *
                RESULTLOOKUP([d/Account] = "Price") * 0.90
        ELSE
            DATA([d/Account] = "Revenue") =
                RESULTLOOKUP([d/Account] = "Units") *
                RESULTLOOKUP([d/Account] = "Price")
        ENDIF

    ELSEIF ATTRIBUTE([d/Product].[p/PricingType]) = "premium" THEN
        DATA([d/Account] = "Revenue") =
            RESULTLOOKUP([d/Account] = "Units") *
            RESULTLOOKUP([d/Account] = "Price") * 1.15

    ENDIF

    // COGS y Margen
    DATA([d/Account] = "COGS") =
        RESULTLOOKUP([d/Account] = "Units") *
        RESULTLOOKUP([d/Account] = "UnitCost")

    DATA([d/Account] = "Margin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "COGS")

    // Alerta: margen por debajo del minimo (leido del atributo)
    IF RESULTLOOKUP([d/Account] = "Revenue") > 0 THEN
        IF (RESULTLOOKUP([d/Account] = "Margin") /
            RESULTLOOKUP([d/Account] = "Revenue") * 100) <
            ATTRIBUTE([d/Product].[p/MinMargin]) THEN
            DATA([d/Account] = "MarginAlert") = 1
        ELSE
            DATA([d/Account] = "MarginAlert") = 0
        ENDIF
    ENDIF

ENDFOR
```

### Que aporta ATTRIBUTE aqui
- El tipo de pricing esta en los datos maestros, no hardcodeado
- Si añades un producto nuevo con su PricingType, el script lo maneja sin cambios
- El margen minimo es configurable por producto sin tocar el script
