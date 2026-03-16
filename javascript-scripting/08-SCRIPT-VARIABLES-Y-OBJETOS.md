# SAC Script Variables, ScriptObjects y Buenas Practicas

## 1. Script Variables

Las Script Variables son variables globales que persisten durante toda la sesion
de la aplicacion. Se crean en el Designer (panel Outline → Script Variables).

### Tipos soportados

| Tipo | Ejemplo | Uso |
|------|---------|-----|
| `string` | "EMEA" | Textos, IDs, nombres |
| `number` | 42, 3.14 | Contadores, ratios |
| `boolean` | true/false | Flags, estados |

### Configuracion en el Designer

- **Name**: nombre tecnico (ej: `ScriptVar_Region`)
- **Type**: string, number, boolean
- **Default Value**: valor inicial
- **Set As Array**: permite guardar multiples valores

### Uso en scripts

```javascript
// Leer valor
var region = ScriptVar_Region.getValue();

// Escribir valor
ScriptVar_Region.setValue("APAC");

// Con arrays (si "Set As Array" esta activado)
ScriptVar_Regions.setValue(["EMEA", "APAC", "AMER"]);
var regions = ScriptVar_Regions.getValue();
```

### Pasar variables por URL

Las Script Variables se pueden inicializar desde la URL con el prefijo `;p_`:

```
https://tenant.sapanalytics.cloud/story/ID;p_ScriptVar_Region=EMEA;p_ScriptVar_Year=2026
```

Util para:
- Links entre aplicaciones con contexto
- Bookmarks que preservan estado
- Integraciones con otras herramientas SAP

---

## 2. ScriptObjects (funciones reutilizables)

Un ScriptObject es un contenedor de funciones que se pueden llamar desde cualquier
widget de la aplicacion. Evita duplicar codigo.

### Crear un ScriptObject

1. Panel Outline → "+" junto a "Script Objects"
2. Nombre: ej. `Utils`, `FilterHelper`, `PlanningOps`
3. Dentro, crear funciones

### Definir funciones

```javascript
// En ScriptObject "Utils"

// Funcion sin parametros
function refreshAll() {
    Table_1.getDataSource().refreshData();
    Chart_1.getDataSource().refreshData();
    KPI_Total.getDataSource().refreshData();
}

// Funcion con parametros
function applyFilter(dimensionName, value) {
    Table_1.getDataSource().setDimensionFilter(dimensionName, value);
    Chart_1.getDataSource().setDimensionFilter(dimensionName, value);
}

// Funcion que devuelve valor
function getSelectedVersion() {
    return Dropdown_Version.getSelectedKey();
}

// Funcion con logica compleja
function linkedFilter(sourceDimension, sourceValue) {
    var ds_table = Table_1.getDataSource();
    var ds_chart1 = Chart_Revenue.getDataSource();
    var ds_chart2 = Chart_Cost.getDataSource();

    ds_table.setDimensionFilter(sourceDimension, sourceValue);
    ds_chart1.setDimensionFilter(sourceDimension, sourceValue);
    ds_chart2.setDimensionFilter(sourceDimension, sourceValue);
}
```

### Llamar desde widgets

```javascript
// Desde onClick de cualquier boton
Utils.refreshAll();

// Desde onSelect de un dropdown
Utils.applyFilter("Region", Dropdown_Region.getSelectedKey());

// Usar valor devuelto
var version = Utils.getSelectedVersion();
```

### Acceso a widgets desde ScriptObject

Dentro de un ScriptObject tienes acceso a todos los widgets, variables y popups
de la aplicacion. Puedes referenciarlos directamente por nombre.

---

## 3. Buenas practicas

### Rendimiento

```javascript
// MAL: getDataSource() en cada linea
Table_1.getDataSource().setDimensionFilter("A", "1");
Table_1.getDataSource().setDimensionFilter("B", "2");
Table_1.getDataSource().setDimensionFilter("C", "3");

// BIEN: guardar en variable
var ds = Table_1.getDataSource();
ds.setDimensionFilter("A", "1");
ds.setDimensionFilter("B", "2");
ds.setDimensionFilter("C", "3");
```

```javascript
// MEJOR: pausar refresco durante multiples cambios
var ds = Table_1.getDataSource();
ds.setRefreshPaused(PauseMode.On);
ds.setDimensionFilter("A", "1");
ds.setDimensionFilter("B", "2");
ds.setDimensionFilter("C", "3");
ds.setRefreshPaused(PauseMode.Off);
```

### Validacion de inputs

```javascript
// Siempre validar antes de operar
var name = InputField_Name.getValue();

if (name === "" || name === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "El campo nombre es obligatorio");
    return;  // sale del script
}
```

### Busy Indicator para operaciones largas

```javascript
// Siempre mostrar/ocultar en pares
Application.showBusyIndicator();

// ... operacion ...

Application.hideBusyIndicator();  // NO olvidar esto
```

### Verificar Planning habilitado

```javascript
// Antes de cualquier operacion de planning
if (!Table_1.getPlanning().isEnabled()) {
    Application.showMessage(ApplicationMessageType.Error, "Planning no esta habilitado");
    return;
}
```

### Verificar DataLocking habilitado

```javascript
var locking = Table_1.getPlanning().getDataLocking();
if (locking === undefined) {
    Application.showMessage(ApplicationMessageType.Warning, "Data Locking no configurado");
    return;
}
```

### Nombrado consistente

| Tipo | Convencion | Ejemplo |
|------|-----------|---------|
| Tabla | `Table_NombreDescriptivo` | `Table_PnL`, `Table_Headcount` |
| Chart | `Chart_NombreDescriptivo` | `Chart_Revenue`, `Chart_Trend` |
| Dropdown | `Dropdown_Dimension` | `Dropdown_Region`, `Dropdown_Year` |
| Boton | `Button_Accion` | `Button_Calculate`, `Button_Lock` |
| Popup | `Popup_Proposito` | `Popup_NewVersion`, `Popup_Confirm` |
| ScriptObject | `Utils_Area` | `Utils_Filter`, `Utils_Planning` |
| ScriptVariable | `ScriptVar_Nombre` | `ScriptVar_Region`, `ScriptVar_IsAdmin` |
| Input | `InputField_Campo` | `InputField_VersionName` |
| Panel | `Panel_Seccion` | `Panel_Main`, `Panel_Detail` |

### Estructura recomendada de ScriptObjects

Para aplicaciones grandes, separar logica en varios ScriptObjects:

| ScriptObject | Responsabilidad |
|-------------|----------------|
| `Utils_Filter` | Funciones de filtrado y linked analysis |
| `Utils_Planning` | Operaciones de planning (versiones, Data Actions) |
| `Utils_MasterData` | CRUD de miembros de dimensiones |
| `Utils_Navigation` | Navegacion, paneles, popups |
| `Utils_Bookmark` | Gestion de bookmarks |
| `Utils_Validation` | Validaciones de inputs |
