# SAC Advanced Formulas - Cheatsheet Rapido

## Estructura basica
```
CONFIG.xxx = valor
MEMBERSET [d/Dim] = ("member1", "member2")
VARIABLEMEMBER #temp OF [d/Dim]
DATA([d/Dim] = "target") = RESULTLOOKUP([d/Dim] = "source") * factor
```

## Leer datos
```
RESULTLOOKUP()                              celda actual
RESULTLOOKUP([d/Dim] = "member")            miembro especifico
RESULTLOOKUP([d/Date] = PREVIOUS(1))        periodo anterior
RESULTLOOKUP([d/Date] = PREVIOUS(12))       año anterior
RESULTLOOKUP([d/Date] = NEXT(1))            periodo siguiente
```

## Escribir datos
```
DATA() = valor                              celda actual
DATA([d/Dim] = "member") = valor            miembro especifico
DATA() = NULL                               borrar celda
```

## Scope
```
MEMBERSET [d/Dim] = ("A", "B", "C")         lista
MEMBERSET [d/Date] = "202601" TO "202612"    rango
MEMBERSET [d/Dim].[p/Attr] = "valor"         por atributo
MEMBERSET [d/Dim] = NOT ("X")               excluir
MEMBERSET [d/Dim] = %PARAM%                 parametro
```

## Variables
```
VARIABLEMEMBER #nombre OF [d/Dim]            miembro virtual (prefijo #)
VARIABLEINTEGER @nombre                      entero (prefijo @)
VARIABLEFLOAT @nombre                        decimal (prefijo @)
```

## Condicionales
```
IF [d/Dim] = "member" THEN ... ENDIF
IF valor > 100 THEN ... ELSEIF ... ELSE ... ENDIF
```

## Bucles
```
FOREACH [d/Dim] ASC ... ENDFOR              todos los miembros
FOREACH.BOOKED [d/Dim] ... ENDFOR           solo con datos (rapido)
FOR @i = 1 TO 10 STEP 1 ... ENDFOR         numerico
BREAK                                        salir del bucle
```

## Cross-model
```
LINK [alias] = MODEL("t.local/NombreModelo")
LINK [alias] RESULTLOOKUP([d/Dim] = "member")
```

## Config
```
CONFIG.TIME_HIERARCHY = CALENDARYEAR | FISCALYEAR
CONFIG.UNBOOKED = ON | OFF
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON | OFF
```

## Operadores
```
+  -  *  /  %                               aritmeticos
=  <>  >  <  >=  <=                         comparacion
AND  OR  NOT                                logicos
```

## Funciones
```
ABS(x)  ROUND(x,n)  TRUNCATE(x,n)          matematicas
POWER(x,n)  LOG(x)  EXP(x)
PREVIOUS(n)  NEXT(n)  PARENT()              tiempo
```

## Parametros
```
%NOMBRE_PARAMETRO%                          se define en la UI de Data Action
```

## Reglas de oro
1. Evita FOREACH si el calculo no depende de iteraciones anteriores
2. Pon IF fuera de FOREACH, no al reves
3. Usa FOREACH.BOOKED en vez de FOREACH cuando puedas
4. Usa variables para no repetir RESULTLOOKUP
5. Define MEMBERSET lo mas restrictivo posible
