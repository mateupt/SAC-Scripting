# Errores Comunes y Como Evitarlos

## Error 1: Usar FOREACH cuando no hace falta

### Sintoma
El script tarda 10 minutos en vez de 10 segundos.

### Ejemplo malo
```
// MAL: Revenue * 1.1 no depende de otros meses
FOREACH [d/Date]
    DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Revenue") * 1.1
ENDFOR
```

### Correccion
```
// BIEN: Sin FOREACH, SAC paraleliza automaticamente
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Revenue") * 1.1
```

### Regla
Preguntate: "¿El calculo del mes 3 necesita el resultado del mes 2?"
- Si → FOREACH
- No → sin FOREACH

---

## Error 2: Olvidar MEMBERSET y sobreescribir todo el modelo

### Sintoma
Se borran o sobreescriben datos que no querias tocar.

### Ejemplo malo
```
// Sin MEMBERSET: actua sobre TODAS las versiones, TODOS los meses, TODO
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Revenue") * 1.1
```

### Correccion
```
// Siempre definir scope explicito
MEMBERSET [d/Date] = "202601" TO "202612"
MEMBERSET [d/Version] = "Plan"
MEMBERSET [d/Account] = "Revenue"

DATA() = RESULTLOOKUP() * 1.1
```

### Regla
SIEMPRE pon MEMBERSET. Aunque creas que no hace falta. Es tu cinturon
de seguridad.

---

## Error 3: Division por cero

### Sintoma
El script falla o produce resultados inesperados.

### Ejemplo malo
```
DATA([d/Account] = "MarginPct") =
    RESULTLOOKUP([d/Account] = "Profit") /
    RESULTLOOKUP([d/Account] = "Revenue") * 100
```
Si Revenue = 0, division por cero.

### Correccion
```
IF RESULTLOOKUP([d/Account] = "Revenue") <> 0 THEN
    DATA([d/Account] = "MarginPct") =
        RESULTLOOKUP([d/Account] = "Profit") /
        RESULTLOOKUP([d/Account] = "Revenue") * 100
ELSE
    DATA([d/Account] = "MarginPct") = 0
ENDIF
```

---

## Error 4: FOREACH sin ASC en calculos secuenciales

### Sintoma
Resultados incorrectos porque los meses se procesan en orden aleatorio.

### Ejemplo malo
```
// Sin ASC: SAC puede procesar marzo antes que febrero
FOREACH [d/Date]
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR
```

### Correccion
```
// Con ASC: garantiza orden cronologico
FOREACH [d/Date] ASC
    DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1)) * 1.02
ENDFOR
```

### Regla
Si usas PREVIOUS() o NEXT() dentro de FOREACH, pon SIEMPRE ASC o DESC.

---

## Error 5: Leer datos recien escritos sin FOREACH

### Sintoma
Los datos leidos son los originales, no los calculados.

### Ejemplo
```
// Quieres: calcular GrossProfit, luego usar GrossProfit para calcular EBIT
// El problema: sin FOREACH, SAC puede ejecutar ambos en paralelo

DATA([d/Account] = "GrossProfit") =
    RESULTLOOKUP([d/Account] = "Revenue") -
    RESULTLOOKUP([d/Account] = "Cost")

// Este RESULTLOOKUP de GrossProfit ¿lee el original o el recien calculado?
DATA([d/Account] = "EBIT") =
    RESULTLOOKUP([d/Account] = "GrossProfit") -
    RESULTLOOKUP([d/Account] = "OPEX")
```

### Aclaracion importante
Dentro de un mismo step de Advanced Formula, las sentencias DATA se ejecutan
en ORDEN SECUENCIAL (de arriba a abajo). Asi que el segundo DATA SI lee
el GrossProfit recien calculado.

PERO: si estan en steps SEPARADOS dentro de la misma Data Action, el orden
depende del orden de los steps.

### Regla
Pon calculos dependientes en el MISMO step, en el orden correcto.

---

## Error 6: RESULTLOOKUP devuelve NULL y no lo esperas

### Sintoma
Celdas que deberian tener valor quedan vacias.

### Explicacion
Si RESULTLOOKUP lee una celda vacia → devuelve NULL.
NULL * 1.1 = NULL. NULL + 100 = NULL.
DATA() = NULL → no escribe nada.

### Correccion (opcion A): CONFIG.GENERATE_UNBOOKED_DATA
```
CONFIG.GENERATE_UNBOOKED_DATA = ON
// Ahora las celdas vacias valen 0 en vez de NULL
DATA() = RESULTLOOKUP() * 1.1    // 0 * 1.1 = 0 (escribe 0)
```

### Correccion (opcion B): Comprobar NULL
```
IF RESULTLOOKUP() <> NULL THEN
    DATA() = RESULTLOOKUP() * 1.1
ELSE
    DATA() = 0
ENDIF
```

### Cuando usar cada opcion
- CONFIG.GENERATE_UNBOOKED_DATA = ON → cuando quieres tratar TODOS los vacios como 0
- Comprobar NULL → cuando solo algunos casos necesitan tratamiento especial

---

## Error 7: MEMBERSET demasiado amplio en FOREACH

### Sintoma
Script lentisimo porque itera sobre miles de combinaciones innecesarias.

### Ejemplo malo
```
// MEMBERSET incluye 500 productos, pero solo 20 tienen datos
MEMBERSET [d/Product] = ALL

FOREACH [d/Product]
    DATA([d/Account] = "Margin") = ...
ENDFOR
```

### Correccion
```
// FOREACH.BOOKED solo itera sobre productos con datos
FOREACH.BOOKED [d/Product]
    DATA([d/Account] = "Margin") = ...
ENDFOR
```

---

## Error 8: Confundir Advanced Formulas con JavaScript

### Sintoma
Escribir sintaxis JavaScript dentro del editor de Advanced Formulas.

### Ejemplos de confusion
```
// ESTO NO FUNCIONA en Advanced Formulas:
var revenue = 10000;               // No existe "var"
if (revenue > 0) { ... }          // Sintaxis JS, no de Advanced Formulas
console.log(revenue);              // No existe console
for (let i = 0; i < 12; i++) {}   // Sintaxis JS

// EN ADVANCED FORMULAS se escribe asi:
INTEGER @revenue
IF RESULTLOOKUP([d/Account] = "Revenue") > 0 THEN
    // ...
ENDIF
FOR @i = 0 TO 11 STEP 1
    // ...
ENDFOR
```

### Regla
- Advanced Formulas = lenguaje propio SAP (DATA, RESULTLOOKUP, FOREACH)
- JavaScript = Analytic Applications (controlar UI, disparar Data Actions)
- Son mundos separados. No mezclar sintaxis.

---

## Checklist antes de ejecutar un script

- [ ] ¿He puesto MEMBERSET para todas las dimensiones relevantes?
- [ ] ¿El scope es lo mas restrictivo posible?
- [ ] ¿Los FOREACH tienen ASC/DESC si uso PREVIOUS/NEXT?
- [ ] ¿He protegido contra division por cero?
- [ ] ¿Necesito CONFIG.GENERATE_UNBOOKED_DATA para tratar vacios?
- [ ] ¿Los calculos dependientes estan en orden correcto?
- [ ] ¿He usado FOREACH.BOOKED donde puedo?
- [ ] ¿He puesto IF fuera de FOREACH (no al reves)?
- [ ] ¿He probado primero con un scope pequeño (1 mes, 1 empresa)?
