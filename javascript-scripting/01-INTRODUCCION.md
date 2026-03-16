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

// Obtener datos de una celda (requiere tabla o chart)
var value = ds.getData("Account", "Revenue");
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
