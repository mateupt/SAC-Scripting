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
RESULTLOOKUP([d/Date] = FIRST())            primer periodo del año
RESULTLOOKUP([d/Date] = LAST())             ultimo periodo del año
RESULTLOOKUP([d/Date] = PREYEARLAST())      ultimo periodo año anterior
ATTRIBUTE([d/Dim].[p/Prop])                 leer propiedad de dimension
```

## Escribir datos
```
DATA() = valor                              celda actual
DATA([d/Dim] = "member") = valor            miembro especifico
DATA() = NULL                               borrar celda
DATA.APPEND() = valor                       escribir SIN limpiar scope
DELETE([d/Dim] = "member")                  borrar celdas especificas
```

## Scope
```
MEMBERSET [d/Dim] = ("A", "B", "C")                     lista
MEMBERSET [d/Date] = "202601" TO "202612"                rango
MEMBERSET [d/Dim].[p/Attr] = "valor"                     por atributo
MEMBERSET [d/Dim] = NOT ("X")                            excluir
MEMBERSET [d/Dim] = %PARAM%                              parametro
MEMBERSET [d/Dim] = BASEMEMBER([d/Dim].[h/Hier], "Node") hojas de jerarquia
```

## Variables
```
VARIABLEMEMBER #nombre OF [d/Dim]           miembro virtual (prefijo #)
INTEGER @nombre                              entero (prefijo @)
FLOAT @nombre                                decimal (prefijo @)
```

## Condicionales
```
IF [d/Dim] = "member" THEN ... ENDIF
IF valor > 100 THEN ... ELSEIF ... ELSE ... ENDIF
IF ATTRIBUTE([d/Dim].[p/Prop]) = "val" THEN ...
```

## Bucles
```
FOREACH [d/Dim] ASC ... ENDFOR              todos los miembros, en orden
FOREACH.BOOKED [d/Dim] ... ENDFOR           solo con datos (rapido)
FOR @i = 1 TO 10 STEP 1 ... ENDFOR         numerico
BREAK                                        salir del bucle
```

## Cross-model
```
LINK [alias] = MODEL("t.local/NombreModelo")
LINK [alias] RESULTLOOKUP([d/Dim] = "member")
MODEL [alias] ... ENDMODEL                  scope del modelo linkeado
```

## Agregacion
```
AGGREGATE_DIMENSIONS = [d/Dim1], [d/Dim2]
AGGREGATE_WRITETO [d/Dim] = "TotalMember"
BASEMEMBER([d/Dim].[h/Hier], "ParentNode")
```

## Arrastre de saldos
```
CARRYFORWARD([d/Flow], "OPENING", "CLOSING", "HIRES", formula)
```

## Intercompany
```
ELIMMEMBER([d/Entity].[h/Hier], [d/Entity], [d/Interco], filter)
```

## Config
```
CONFIG.TIME_HIERARCHY = CALENDARYEAR | FISCALYEAR
CONFIG.GENERATE_UNBOOKED_DATA = ON | OFF
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON | OFF
CONFIG.HIERARCHY [d/Dim].[h/Hierarchy]
CONFIG.TIME_ZONE_OFFSET = +1
CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY [d/Dim]
```

## Operadores
```
+  -  *  /  %                               aritmeticos
=  <>  !=  >  <  >=  <=                     comparacion
AND  OR  NOT                                logicos
```

## Funciones matematicas
```
ABS(x)  ROUND(x,n)  TRUNC(x,n)             redondeo
FLOOR(x)  CEIL(x)                           redondeo entero
SQRT(x)  POWER(x,n)                         raiz y potencia
LOG(x)  LOG10(x)                            logaritmos
MOD(x,y)  INT(x)  FLOAT(x)                 modulo y conversion
```

## Funciones de tiempo
```
PREVIOUS(n)  NEXT(n)                        navegar periodos
FIRST()  LAST()  PREYEARLAST()              anclas temporales
TODAY()                                      fecha actual (UTC)
DAY()  MONTH()  YEAR()  WEEK()              componentes de fecha
DAYSINMONTH()  DAYSINYEAR()                 dias en periodo
DATERATIO(start, end)                       solapamiento de fechas
DATEDIFF(date1, date2, mode)                diferencia entre fechas
```

## Referencia de dimensiones
```
[d/DimensionName]                           dimension
[d/DimensionName].[p/PropertyName]          atributo/propiedad
[d/DimensionName].[h/HierarchyName]         jerarquia
```

## Parametros
```
%NOMBRE_PARAMETRO%                          se define en la UI de Data Action
```

## Reglas de oro
1. Evita FOREACH si el calculo no depende de iteraciones anteriores
2. Pon IF fuera de FOREACH, no al reves
3. Usa FOREACH.BOOKED en vez de FOREACH cuando puedas
4. Usa VARIABLEMEMBER para no repetir RESULTLOOKUP
5. Define MEMBERSET lo mas restrictivo posible
6. Usa CARRYFORWARD en vez de FOREACH para opening/closing
7. Protege siempre contra division por cero
8. Usa ATTRIBUTE() para leer propiedades en vez de crear cuentas auxiliares
