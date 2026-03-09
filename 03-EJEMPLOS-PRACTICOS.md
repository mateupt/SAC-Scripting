# SAC Advanced Formulas - Ejemplos Practicos

## Ejemplo 1: Incremento porcentual simple

**Caso:** Subir Revenue un 10% respecto al año anterior para todo 2026.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Version] = "Plan"

DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(12)) * 1.1
```

**Que hace:**
- Para cada mes de 2026, lee el mismo mes de 2025 y le suma un 10%.
- Escribe el resultado en la version Plan.

---

## Ejemplo 2: Carry Forward (arrastre de saldos)

**Caso:** Copiar el cierre del periodo anterior como apertura del periodo actual.

```
MEMBERSET [d/Date] = "202601" TO "202612"

FOREACH [d/Date] ASC
    DATA([d/Account] = "OPENING") =
        RESULTLOOKUP([d/Account] = "CLOSING", [d/Date] = PREVIOUS(1))
ENDFOR
```

**Que hace:**
- Recorre cada mes en orden ascendente.
- El opening de enero = closing de diciembre, opening de febrero = closing de enero, etc.
- FOREACH es necesario aqui porque cada calculo depende del anterior.

---

## Ejemplo 3: Calculo de P&L (Profit & Loss)

**Caso:** Calcular beneficio, impuestos y beneficio neto.

```
MEMBERSET [d/Date] = "202601" TO "202612"

// Beneficio bruto
DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "COGS")

// Beneficio operativo
DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -
    RESULTLOOKUP([d/Account] = "OPEX")

// Impuestos segun region
IF [d/Region] = "Europe" THEN
    DATA([d/Account] = "Tax") =
        RESULTLOOKUP([d/Account] = "EBIT") * 0.25
ELSEIF [d/Region] = "US" THEN
    DATA([d/Account] = "Tax") =
        RESULTLOOKUP([d/Account] = "EBIT") * 0.21
ENDIF

// Beneficio neto
DATA([d/Account] = "NetProfit") =
    RESULTLOOKUP([d/Account] = "EBIT") -
    RESULTLOOKUP([d/Account] = "Tax")
```

---

## Ejemplo 4: Distribucion mensual de presupuesto anual

**Caso:** Tienes un presupuesto anual y quieres distribuirlo en 12 meses iguales.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "Budget"

VARIABLEMEMBER #AnnualBudget OF [d/Account]

// Guardar el total anual
DATA([d/Account] = #AnnualBudget) = RESULTLOOKUP([d/Account] = "AnnualBudget")

// Distribuir en 12 meses
DATA() = RESULTLOOKUP([d/Account] = #AnnualBudget) / 12
```

---

## Ejemplo 5: Crecimiento mensual compuesto

**Caso:** Aplicar un 2% de crecimiento mes a mes partiendo de un valor base.

```
MEMBERSET [d/Date] = "202602" TO "202612"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Version] = "Plan"

// Enero ya tiene valor base. Desde febrero, crecer 2% mensual.
FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR
```

**Nota:** FOREACH es obligatorio aqui porque febrero depende de enero,
marzo depende de febrero, etc. Sin FOREACH todos leerian de los datos
originales, no de los recien calculados.

---

## Ejemplo 6: Copiar Actual a Forecast

**Caso:** Los meses ya cerrados copian Actual, los futuros mantienen Forecast.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Forecast"

// Parametro: ultimo mes cerrado (ej: "202603" = marzo cerrado)
// Se pasa como parametro de la Data Action

FOREACH [d/Date]
    IF [d/Date] <= %LAST_ACTUAL_MONTH% THEN
        DATA() = RESULTLOOKUP([d/Version] = "Actual")
    ENDIF
ENDFOR
```

---

## Ejemplo 7: Calculo con datos de otro modelo (LINK)

**Caso:** Leer headcount de un modelo de RRHH y calcular coste de personal.

```
LINK [HR] = MODEL("t.local/HR_Planning_Model")

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "StaffCost"

DATA() =
    LINK [HR] RESULTLOOKUP([d/Account] = "Headcount") *
    RESULTLOOKUP([d/Account] = "AvgSalary")
```

**Que hace:**
- Lee headcount del modelo de RRHH.
- Lee salario medio del modelo actual.
- Multiplica para obtener coste de personal.

---

## Ejemplo 8: Calculo condicional con FOREACH.BOOKED

**Caso:** Calcular margen solo para productos que tienen datos (rendimiento).

```
MEMBERSET [d/Date] = "202601" TO "202612"

FOREACH.BOOKED [d/Product]
    DATA([d/Account] = "Margin") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")

    DATA([d/Account] = "MarginPct") =
        RESULTLOOKUP([d/Account] = "Margin") /
        RESULTLOOKUP([d/Account] = "Revenue") * 100
ENDFOR
```

**FOREACH.BOOKED** vs **FOREACH:**
- FOREACH itera sobre TODOS los miembros de la dimension (aunque no tengan datos).
- FOREACH.BOOKED solo itera sobre miembros que tienen datos → mucho mas rapido.

---

## Ejemplo 9: Escenario con parametros dinamicos

**Caso:** Script generico que aplica un % de crecimiento pasado como parametro.

```
// Parametros definidos en la Data Action UI:
// %SOURCE_VERSION% = "Actual"
// %TARGET_VERSION% = "Plan"
// %GROWTH_PCT% = 10  (numero)

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = %TARGET_VERSION%

DATA() =
    RESULTLOOKUP([d/Version] = %SOURCE_VERSION%, [d/Date] = PREVIOUS(12)) *
    (1 + %GROWTH_PCT% / 100)
```

**Ventaja:** Un unico script reutilizable. El usuario elige version y % al ejecutar.

---

## Ejemplo 10: Borrar datos (reset)

**Caso:** Limpiar datos de plan antes de recalcular.

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"

// Poner todo a NULL (borrar)
DATA() = NULL
```

---

## Anti-patrones (que NO hacer)

### NO: FOREACH innecesario
```
// MAL - FOREACH no necesario, todos los meses son independientes
FOREACH [d/Date]
    DATA([d/Account] = "Profit") =
        RESULTLOOKUP([d/Account] = "Revenue") -
        RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// BIEN - Sin FOREACH, SAC lo paraleliza automaticamente
DATA([d/Account] = "Profit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")
```

### NO: IF fuera de FOREACH
```
// MAL - El IF se evalua en cada iteracion del FOREACH
FOREACH [d/Date] ASC
    IF [d/Account] = "Revenue" THEN
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.05
    ENDIF
ENDFOR

// BIEN - IF fuera, FOREACH solo para los que cumplen
IF [d/Account] = "Revenue" THEN
    FOREACH [d/Date] ASC
        DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.05
    ENDFOR
ENDIF
```

### NO: Muchos RESULTLOOKUP redundantes
```
// MAL - Lee Revenue 3 veces
DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = "Revenue") * 0.21
DATA([d/Account] = "Net") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Revenue") * 0.21

// BIEN - Usa variable temporal
VARIABLEMEMBER #Rev OF [d/Account]
DATA([d/Account] = #Rev) = RESULTLOOKUP([d/Account] = "Revenue")
DATA([d/Account] = "Tax") = RESULTLOOKUP([d/Account] = #Rev) * 0.21
DATA([d/Account] = "Net") = RESULTLOOKUP([d/Account] = #Rev) - RESULTLOOKUP([d/Account] = "Tax")
```
