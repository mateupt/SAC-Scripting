# Ejercicios - Practica por tu cuenta

Intenta resolver cada ejercicio antes de mirar la solucion.
Estan ordenados de facil a dificil.

---

## Ejercicio 1: Incremento simple
**Nivel: Basico**

Incrementa un 8% todas las cuentas de Cost para todo 2026
en la version Plan, respecto a los datos de la version Actual.

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = "Cost"
MEMBERSET [d/Version] = "Plan"

DATA() = RESULTLOOKUP([d/Version] = "Actual") * 1.08
```

No necesitas FOREACH: cada mes es independiente.
</details>

---

## Ejercicio 2: Bonus condicional
**Nivel: Basico**

Si Revenue de un mes supera 50.000, escribe un Bonus de 2.000.
Si no, el Bonus es 0.

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "Bonus")
MEMBERSET [d/Version] = "Plan"

IF RESULTLOOKUP([d/Account] = "Revenue") > 50000 THEN
    DATA([d/Account] = "Bonus") = 2000
ELSE
    DATA([d/Account] = "Bonus") = 0
ENDIF
```
</details>

---

## Ejercicio 3: Copiar y ajustar
**Nivel: Basico**

Copia todos los datos de la version "Budget" a la version "Forecast".
Luego incrementa solo Revenue un 3%.

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Forecast"

// Paso 1: Copiar todo
DATA() = RESULTLOOKUP([d/Version] = "Budget")

// Paso 2: Ajustar solo Revenue
DATA([d/Account] = "Revenue") =
    RESULTLOOKUP([d/Account] = "Revenue") * 1.03
```

Nota: El segundo DATA sobreescribe el Revenue copiado en el paso 1.
Las demas cuentas quedan como estaban.
</details>

---

## Ejercicio 4: Media movil de 3 meses
**Nivel: Intermedio**

Calcula la media movil de 3 meses de Revenue y guardala en
la cuenta "Revenue_MA3". Para enero y febrero, que no tienen
3 meses de historia, deja el valor original.

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "Revenue_MA3")
MEMBERSET [d/Version] = "Actual"

// Enero y febrero: copiar valor original
DATA([d/Account] = "Revenue_MA3", [d/Date] = "202601") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = "202601")

DATA([d/Account] = "Revenue_MA3", [d/Date] = "202602") =
    RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = "202602")

// Marzo en adelante: media de los 3 ultimos meses
MEMBERSET [d/Date] = "202603" TO "202612"

DATA([d/Account] = "Revenue_MA3") =
    (RESULTLOOKUP([d/Account] = "Revenue") +
     RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = PREVIOUS(1)) +
     RESULTLOOKUP([d/Account] = "Revenue", [d/Date] = PREVIOUS(2))) / 3
```

**¿Por que NO lleva FOREACH?** Cada mes lee solo datos originales de Revenue
(no datos recien calculados por este script). Marzo lee Revenue de ene/feb/mar,
abril lee Revenue de feb/mar/abr — son lecturas independientes.
FOREACH solo se necesita cuando una iteracion depende del resultado de la anterior.
</details>

---

## Ejercicio 5: Estacionalidad
**Nivel: Intermedio**

Tienes un Revenue anual de 1.200.000 y quieres distribuirlo
con estacionalidad. Los pesos mensuales estan en la cuenta
"Seasonality_Pct" (ej: enero=6%, julio=12%, diciembre=10%...).

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("Revenue", "Seasonality_Pct")
MEMBERSET [d/Version] = "Plan"

// %ANNUAL_REVENUE% = 1200000 (parametro)

DATA([d/Account] = "Revenue") =
    %ANNUAL_REVENUE% * RESULTLOOKUP([d/Account] = "Seasonality_Pct") / 100
```

Sin FOREACH ni IF. Los pesos ya estan en el modelo, el script solo multiplica.
Diseñar bien el modelo simplifica los scripts enormemente.
</details>

---

## Ejercicio 6: Crecimiento compuesto con techo
**Nivel: Intermedio**

Aplica un crecimiento del 5% mensual a Revenue desde febrero,
pero con un techo (cap) de 100.000. Si el calculo supera
100.000, escribe 100.000.

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202602" TO "202612"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Version] = "Plan"

FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.05

    IF RESULTLOOKUP() > 100000 THEN
        DATA() = 100000
    ENDIF
ENDFOR
```

FOREACH es obligatorio: cada mes depende del anterior.
ASC es obligatorio: necesitamos orden cronologico.
El IF dentro del FOREACH es correcto aqui porque ambos operan sobre
la misma dimension [d/Date].
</details>

---

## Ejercicio 7: Imputacion de costes por headcount
**Nivel: Avanzado**

Los costes de la oficina central (Company="HQ") se reparten entre
las filiales proporcionalmente a su headcount. Lee headcount de
un modelo externo de RRHH.

<details>
<summary>Solucion</summary>

```
LINK [HR] = MODEL("t.local/HR_Model")

MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Account] = ("HQ_Allocated_Cost", "HQ_TotalCost")
MEMBERSET [d/Version] = "Actual"
MEMBERSET [d/Company] = NOT ("HQ")

VARIABLEMEMBER #TotalHC OF [d/Account]

// Leer headcount total (todas las filiales, sin HQ)
// Necesitamos saber el total para calcular el peso de cada filial
AGGREGATE_WRITETO [d/Company] = "ALL_COMPANIES"

DATA([d/Account] = #TotalHC, [d/Company] = "ALL_COMPANIES") =
    LINK [HR] RESULTLOOKUP([d/Account] = "Headcount",
                           [d/Company] = "ALL_COMPANIES")

// Para cada filial: su peso = su HC / HC total
// Coste asignado = coste HQ * peso
FOREACH.BOOKED [d/Company]
    DATA([d/Account] = "HQ_Allocated_Cost") =
        RESULTLOOKUP([d/Account] = "HQ_TotalCost", [d/Company] = "HQ") *
        LINK [HR] RESULTLOOKUP([d/Account] = "Headcount") /
        RESULTLOOKUP([d/Account] = #TotalHC, [d/Company] = "ALL_COMPANIES")
ENDFOR
```

Conceptos que combina:
- LINK para leer de otro modelo
- VARIABLEMEMBER para dato temporal
- AGGREGATE_WRITETO para totales
- FOREACH.BOOKED para eficiencia
- Division proporcional
</details>

---

## Ejercicio 8: Simulacion de prestamo
**Nivel: Avanzado**

Calcula las cuotas mensuales de un prestamo con sistema frances
(cuota constante). El principal es 500.000, el tipo de interes
anual es 5%, y el plazo es 60 meses. Calcula para cada mes:
interes, amortizacion de capital y saldo pendiente.

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "203012"
MEMBERSET [d/Account] = ("Principal", "MonthlyRate", "MonthlyPayment",
                         "InterestPart", "CapitalPart", "Balance")

// Parametros
// %LOAN_AMOUNT% = 500000
// %ANNUAL_RATE% = 5
// %TERM_MONTHS% = 60

FLOAT @monthlyRate
FLOAT @monthlyPayment

// Tipo mensual = tipo anual / 12 / 100
@monthlyRate = %ANNUAL_RATE% / 100 / 12

// Cuota mensual con sistema frances:
// Cuota = Principal * (r * (1+r)^n) / ((1+r)^n - 1)
// POWER() esta disponible en Advanced Formulas
@monthlyPayment = %LOAN_AMOUNT% *
    (@monthlyRate * POWER(1 + @monthlyRate, %TERM_MONTHS%)) /
    (POWER(1 + @monthlyRate, %TERM_MONTHS%) - 1)
// Con 500000, 5%/12, 60 meses = 9435.62

// Mes 1
DATA([d/Account] = "Balance", [d/Date] = "202601") = %LOAN_AMOUNT%

DATA([d/Account] = "MonthlyPayment") = @monthlyPayment

DATA([d/Account] = "InterestPart", [d/Date] = "202601") =
    %LOAN_AMOUNT% * @monthlyRate

DATA([d/Account] = "CapitalPart", [d/Date] = "202601") =
    @monthlyPayment -
    RESULTLOOKUP([d/Account] = "InterestPart", [d/Date] = "202601")

// Mes 2 en adelante
MEMBERSET [d/Date] = "202602" TO "203012"

FOREACH [d/Date] ASC
    // Saldo = saldo anterior - capital amortizado el mes pasado
    DATA([d/Account] = "Balance") =
        RESULTLOOKUP([d/Account] = "Balance", [d/Date] = PREVIOUS(1)) -
        RESULTLOOKUP([d/Account] = "CapitalPart", [d/Date] = PREVIOUS(1))

    // Interes = saldo pendiente * tipo mensual
    DATA([d/Account] = "InterestPart") =
        RESULTLOOKUP([d/Account] = "Balance") * @monthlyRate

    // Capital = cuota - interes
    DATA([d/Account] = "CapitalPart") =
        @monthlyPayment -
        RESULTLOOKUP([d/Account] = "InterestPart")
ENDFOR
```

Este es un caso real de tesoreria/finance. Cada mes depende del
anterior (el saldo va bajando) → FOREACH ASC obligatorio.
</details>

---

## Ejercicio 9: Reconciliacion automatica
**Nivel: Avanzado**

Compara datos entre dos versiones (Actual y Plan) y genera:
- Variacion absoluta (Actual - Plan)
- Variacion porcentual ((Actual - Plan) / Plan * 100)
- Flag: "Over" si Actual > Plan, "Under" si no

<details>
<summary>Solucion</summary>

```
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Analysis"
MEMBERSET [d/Account] = ("Revenue", "Cost", "Var_Abs", "Var_Pct")

// Copiar Actual como base
DATA() = RESULTLOOKUP([d/Version] = "Actual")

// Variacion absoluta
DATA([d/Account] = "Var_Abs") =
    RESULTLOOKUP([d/Version] = "Actual") -
    RESULTLOOKUP([d/Version] = "Plan")

// Variacion porcentual (proteger division por cero)
IF RESULTLOOKUP([d/Account] = "Revenue", [d/Version] = "Plan") <> 0 THEN
    DATA([d/Account] = "Var_Pct") =
        (RESULTLOOKUP([d/Account] = "Revenue", [d/Version] = "Actual") -
         RESULTLOOKUP([d/Account] = "Revenue", [d/Version] = "Plan")) /
        RESULTLOOKUP([d/Account] = "Revenue", [d/Version] = "Plan") * 100
ELSE
    DATA([d/Account] = "Var_Pct") = 0
ENDIF
```

Nota: Advanced Formulas no soporta escribir texto ("Over"/"Under")
en celdas numericas. Los flags de texto se gestionan con atributos
de dimension o en la capa de visualizacion (Story/App), no en el script.
</details>

---

## Ejercicio 10: Diseña tu propio caso
**Nivel: Experto**

Piensa en un caso real de tu trabajo con SAP:
1. ¿Que datos tienes?
2. ¿Que calculos necesitas?
3. ¿Hay dependencias entre periodos?
4. ¿Necesitas datos de otro modelo?

Escribe el script y consultame para revisarlo.
