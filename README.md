# SAC Scripting - Guia de Referencia

Guia completa y practica de scripting en **SAP Analytics Cloud** (SAC). Cubre los dos lenguajes de scripting de la plataforma: Advanced Formulas para transformacion de datos en Data Actions, y JavaScript para control de la interfaz en Analytic Applications.

Todo el contenido esta en espanol.

---

## Contenido

### Seccion 1: Advanced Formulas (Data Actions)

| # | Archivo | Descripcion |
|---|---------|-------------|
| 01 | [Conceptos Base](01-CONCEPTOS-BASE.md) | Que es una Data Action, tipos de pasos, el lenguaje |
| 02 | [Sintaxis Completa](02-SINTAXIS-COMPLETA.md) | Referencia de todas las instrucciones: CONFIG, MEMBERSET, DATA, RESULTLOOKUP, FOREACH, IF, LINK... |
| 03 | [Ejemplos Practicos](03-EJEMPLOS-PRACTICOS.md) | +30 ejemplos ordenados por complejidad con explicacion linea a linea |
| 04 | [Cheatsheet](04-CHEATSHEET.md) | Referencia rapida de sintaxis en una pagina |
| 05 | [Recursos](05-RECURSOS.md) | Documentacion oficial, cursos SAP Learning, blogs y tutoriales |
| 06 | [Aprendizaje Guiado](06-APRENDIZAJE-GUIADO.md) | Ruta de cero a escribir scripts reales, con modelo mental progresivo |
| 07 | [Casos de Negocio Reales](07-CASOS-NEGOCIO-REALES.md) | Forecast rolling, top-down allocation, currency conversion y mas |
| 08 | [Errores Comunes](08-ERRORES-COMUNES.md) | Errores frecuentes con ejemplo malo, correccion y regla |
| 09 | [Ejercicios](09-EJERCICIOS.md) | Ejercicios con solucion oculta, ordenados de basico a avanzado |
| 10 | [Mecanica Interna](10-MECANICA-INTERNA.md) | Que pasa en la base de datos con cada instruccion (tablas antes/despues) |

### Seccion 2: JavaScript Scripting (Analytic Applications)

| # | Archivo | Descripcion |
|---|---------|-------------|
| 01 | [Introduccion](javascript-scripting/01-INTRODUCCION.md) | Diferencias con Advanced Formulas, eventos, acceso a datos |
| 02 | [Version Management](javascript-scripting/02-VERSION-MANAGEMENT.md) | Crear, copiar, publicar y eliminar versiones via script |
| 03 | [DataSource API](javascript-scripting/03-DATASOURCE-API.md) | Filtros, getResultSet, getData, refreshData |
| 04 | [Dropdowns y Filtros](javascript-scripting/04-DROPDOWNS-Y-FILTROS.md) | Poblar dropdowns, cascading filters, filtros dinamicos |
| 05 | [Master Data CRUD](javascript-scripting/05-MASTER-DATA-CRUD.md) | Crear, leer, actualizar y eliminar miembros de dimensiones |
| 06 | [Popups y Navegacion](javascript-scripting/06-POPUPS-Y-NAVEGACION.md) | Dialogos modales, navegacion entre paginas, busy indicator |
| 07 | [Data Locking](javascript-scripting/07-DATA-LOCKING.md) | Bloqueo y desbloqueo de datos para workflows de planning |
| 08 | [Script Variables y Objetos](javascript-scripting/08-SCRIPT-VARIABLES-Y-OBJETOS.md) | Variables globales, ScriptObjects, buenas practicas |
| 09 | [Temas Avanzados](javascript-scripting/09-TEMAS-AVANZADOS.md) | Composites, Export, Custom Widgets, seguridad, nuevas APIs 2025 |
| 10 | [Table Widget API](javascript-scripting/10-TABLE-WIDGET-API.md) | getSelections, sort, ranking, comentarios, Planning API en tablas |
| 11 | [Chart Widget API](javascript-scripting/11-CHART-WIDGET-API.md) | getSelections, addMeasure/removeMeasure, feeds, charts dinamicos |
| 12 | [Data Entry y Planning](javascript-scripting/12-DATA-ENTRY-PLANNING.md) | setUserInput, submitData, validacion, Planning Sequences |
| 13 | [Error Handling y Debugging](javascript-scripting/13-ERROR-HANDLING-DEBUGGING.md) | try/catch, console.log, breakpoints, debug mode, errores comunes |
| 14 | [Data Actions Avanzado](javascript-scripting/14-DATA-ACTIONS-AVANZADO.md) | Parametros, executeInBackground, encadenar, contexto de filtro |
| 15 | [Application Lifecycle](javascript-scripting/15-APPLICATION-LIFECYCLE.md) | onInitialization, Timer, Pause Refresh, estado, sesion |
| 16 | [Widget Manipulation Condicional](javascript-scripting/16-WIDGET-MANIPULATION-CONDICIONAL.md) | show/hide por datos, roles, CSS dinamico, formularios |
| 17 | [Integration Patterns](javascript-scripting/17-INTEGRATION-PATTERNS.md) | NavigationUtils, URL API, embedding, BW/BPC, cross-app |
| 18 | [Dashboards Practicos](javascript-scripting/18-DASHBOARDS-PRACTICOS.md) | 6 dashboards completos: ejecutivo, plan vs actual, planning, multi-pagina, auto-refresh, export |
| 19 | [Botones Practicos](javascript-scripting/19-BOTONES-PRACTICOS.md) | 12 patrones de botones: guardar, toggle, confirmar, filtros, export, Data Actions, undo, toolbar |

---

## Orden de lectura sugerido

**Si empiezas desde cero con SAC Planning:**

1. `01-CONCEPTOS-BASE` - Entender que es una Data Action
2. `06-APRENDIZAJE-GUIADO` - Modelo mental progresivo
3. `02-SINTAXIS-COMPLETA` - Referencia del lenguaje
4. `03-EJEMPLOS-PRACTICOS` - Ver codigo real
5. `04-CHEATSHEET` - Tener a mano como referencia rapida
6. `08-ERRORES-COMUNES` - Evitar trampas tipicas
7. `09-EJERCICIOS` - Practicar
8. `07-CASOS-NEGOCIO-REALES` - Aplicar a escenarios reales
9. `10-MECANICA-INTERNA` - Entender que pasa por debajo

**Para la parte de JavaScript**, empieza por `javascript-scripting/01-INTRODUCCION` y sigue el orden numerico.

---

## Que NO es esta guia

- No es un curso de SAC completo (solo cubre scripting)
- No cubre administracion, conexiones de datos ni modelado
- No sustituye la documentacion oficial de SAP (enlazada en [05-RECURSOS](05-RECURSOS.md))

---

## Contribuir

Las contribuciones son bienvenidas. Si encuentras errores, tienes ejemplos adicionales o quieres mejorar alguna explicacion, abre un issue o un pull request.

## Licencia

Este trabajo esta licenciado bajo [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).
