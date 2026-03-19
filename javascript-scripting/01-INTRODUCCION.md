# SAC Analytic Application Scripting - Introduccion

## Dos lenguajes, dos mundos

SAP Analytics Cloud tiene **dos lenguajes de scripting** completamente distintos:

| | Advanced Formulas | JavaScript (Analytic Apps) |
|--|-------------------|---------------------------|
| **Donde** | Dentro de Data Actions | En Analytic Applications / Stories |
| **Para que** | Calcular y transformar datos del modelo | Controlar UI, widgets, eventos, versiones |
| **Acceso a datos** | Directo (RESULTLOOKUP, DATA) | Indirecto (APIs de widgets y DataSource) |
| **Sintaxis** | Propia de SAP (tipo SQL/formulas) | JavaScript/TypeScript estandar |
| **Ejecucion** | Backend (sobre el modelo) | Frontend (navegador del usuario) |
| **Complejidad** | Logica de datos pesada | Orquestacion, UX, automatizacion |

Son complementarios: el JavaScript puede **disparar** Data Actions y pasarles parametros,
pero la logica pesada de transformacion de datos va en Advanced Formulas.
Ver la seccion de [Advanced Formulas](../01-CONCEPTOS-BASE.md) para la referencia completa del lenguaje de Data Actions.

## Donde se escribe el JavaScript

1. Seleccionas un widget en el canvas (boton, tabla, dropdown, etc.)
2. Panel lateral → pestana **Scripting** o icono `</>`
3. Eliges el evento (onClick, onSelect, onResultChanged, etc.)
4. Escribes el script en el editor inline

Tambien puedes crear **ScriptObjects** para funciones reutilizables entre widgets.

## Eventos disponibles por widget

| Widget | Eventos principales |
|--------|-------------------|
| **Button** | `onClick` |
| **Dropdown** | `onSelect` |
| **Table** | `onSelect`, `onResultChanged` |
| **Chart** | `onSelect`, `onResultChanged` |
| **Input Field** | `onChange`, `onEnter` |
| **Radio Button** | `onSelect` |
| **Checkbox** | `onClick` |
| **Application** | `onInitialization`, `onTimer` |
| **Popup** | `onOpen`, `onClose` |

## Acceso a datos

```javascript
// Obtener el DataSource de un widget
var ds = Table_1.getDataSource();

// Aplicar filtro
ds.setDimensionFilter("Account", ["Revenue", "Cost"]);

// Quitar filtro
ds.removeDimensionFilter("Account");

// Refrescar datos
ds.refreshData();

// Obtener datos de una celda (requiere selection object)
var value = ds.getData({"Account": "Revenue", "Date": "202601"});
```

## Mensajes y navegacion

```javascript
// Mensaje al usuario
Application.showMessage(ApplicationMessageType.Success, "Operacion completada");
Application.showMessage(ApplicationMessageType.Warning, "Revisa los datos");
Application.showMessage(ApplicationMessageType.Error, "Fallo al ejecutar");

// Indicador de carga
Application.showBusyIndicator();
// ... operacion larga ...
Application.hideBusyIndicator();

// Navegar a otra pagina
Application.openPage("Page_2");
```

## Ejecutar Data Actions desde JavaScript

```javascript
// Ejecutar Data Action sin parametros
DataAction_Forecast.execute();

// Ejecutar con parametros (se mapean a los %PARAM% del Advanced Formula)
DataAction_Forecast.execute({
    "VERSION": "Plan",
    "TARGET_YEAR": "2026"
});
```

## Visibilidad de widgets

```javascript
// Mostrar/ocultar
Panel_Detail.setVisible(true);
Panel_Detail.setVisible(false);

// Toggle
Panel_Detail.setVisible(!Panel_Detail.isVisible());
```

## Variables compartidas (ScriptObject)

Para compartir logica entre widgets, creas un **ScriptObject** (objeto global):

```javascript
// En ScriptObject_Utils, defines funciones:
function getSelectedVersion() {
    return Dropdown_Version.getSelectedKey();
}

// Desde cualquier widget lo llamas:
var version = ScriptObject_Utils.getSelectedVersion();
```

## Diferencias con JavaScript normal

- **No hay DOM**: no existe `document`, `window`, `querySelector`
- **No hay fetch/HTTP**: no puedes llamar APIs externas directamente
- **No hay async/await**: ejecucion sincrona
- **No hay npm/imports**: solo la API de SAC disponible
- **Tipado**: el editor ofrece autocompletado basado en el tipo de widget
- **Scope**: cada evento tiene su propio scope; variables compartidas van en ScriptObject

## Analytic Application vs Optimized Story

SAC tiene dos contextos donde se puede usar JavaScript. **No son equivalentes:**

| | Analytic Application | Optimized Story |
|--|---------------------|----------------|
| **Scripting completo** | Si (todas las APIs) | Subconjunto limitado |
| **Planning API** | Si (versiones, master data, locking) | Limitado |
| **ScriptObjects** | Si | Si |
| **Custom widgets** | Si | No |
| **Performance** | Buena | Mejor (optimizada para consumo) |
| **Caso de uso** | Apps de planning interactivas | Dashboards de consumo |

> **Importante:** Toda esta guia asume que trabajas en **Analytic Applications**,
> que es donde tienes acceso completo a la API de scripting.

## Limitaciones importantes

- **Timeout de ejecucion:** Los scripts tienen un limite de ~30 segundos. Bucles sobre muchos miembros o multiples llamadas API pueden exceder este limite.
- **Sin try/catch:** SAC no soporta bloques try/catch en todas las versiones. Si un error ocurre, el script se detiene.
- **Sin acceso a red:** No puedes hacer llamadas HTTP, fetch, ni acceder a APIs externas.
- **Ejecucion sincrona:** Todo se ejecuta de forma secuencial, sin async/await.

## Permisos necesarios

No todas las operaciones estan disponibles para todos los usuarios:

| Operacion | Permiso requerido |
|-----------|------------------|
| Leer datos (filtros, getResultSet) | Viewer |
| Ejecutar Data Actions | Content Creator / Planner |
| Gestionar versiones (copy, publish, delete) | Planning permissions |
| Master Data CRUD (createMembers, etc.) | Planning Admin / Content Creator |
| Data Locking (setState) | Planning Admin |

## Depuracion

SAC ofrece varias herramientas de depuracion:

### 1. Debug Mode con DevTools del navegador

Anade `&debug=true` al final de la URL de la aplicacion para activar el modo debug:

```
https://tenant.sapanalytics.cloud/story/STORY_ID&debug=true
```

Con debug mode activo:
1. Abre DevTools (`F12` o `Ctrl+Shift+J`)
2. En la pestana **Sources**, busca `sandbox.worker.main` (o `Ctrl+P` y busca por nombre del script)
3. Haz clic en el numero de linea para colocar un **breakpoint**
4. Usa Step Over / Step Into / Step Out para recorrer el codigo
5. Conditional breakpoints: clic derecho en la linea → "Add conditional breakpoint"

```javascript
// Insertar en cualquier script para pausar la ejecucion en debug mode
debugger;
```

### 2. Console.log para inspeccion

```javascript
// Solo funciona en debug mode con DevTools abierto
// NO aparece al usuario final
console.log("Variable:", myVar);

// Inspeccionar datos complejos
var resultSet = Table_1.getDataSource().getResultSet();
console.log("ResultSet:", JSON.stringify(resultSet));
```

### 3. Application.showMessage como assert manual

```javascript
// Patron de depuracion visible al usuario
var ds = Table_1.getDataSource();
var members = ds.getMembers("Region");
Application.showMessage(ApplicationMessageType.Info, "Miembros encontrados: " + members.length.toString());
```

### 4. Panel de logs del editor

El editor de scripts muestra errores de compilacion y runtime directamente. Util para errores de sintaxis.

### 5. Errores comunes en runtime

| Error | Causa probable |
|-------|---------------|
| `Cannot read property 'x' of undefined` | Widget o variable no existe |
| `Method not found` | Metodo no disponible para ese tipo de widget |
| Script termina sin mensaje | Timeout (~30 segundos) o error silencioso |
| `Type mismatch` | Asignacion entre tipos incompatibles (SAC es estrictamente tipado) |
