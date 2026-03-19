# Manipulacion Condicional de Widgets

## Metodos comunes a todos los widgets

La mayoria de widgets en SAC exponen estos metodos base:

| Metodo | Descripcion |
|--------|-------------|
| `setVisible(boolean)` | Muestra u oculta el widget |
| `isVisible()` | Devuelve si el widget es visible |
| `setEnabled(boolean)` | Habilita o deshabilita el widget (inputs, botones) |
| `isEnabled()` | Devuelve si el widget esta habilitado |
| `setCssClass(className)` | Aplica una clase CSS al widget |
| `getCssClass()` | Devuelve la clase CSS actual |

> **Nota:** `setEnabled()` esta disponible en widgets interactivos (Button, Dropdown,
> InputField, Table, Chart, etc.). Widgets estaticos como Text o Image no tienen `setEnabled()`.

## Show/Hide basado en datos

### Patron: Ocultar panel si no hay datos

```javascript
// En onResultChanged de Table_Detail
var ds = Table_Detail.getDataSource();
var rs = ds.getResultSet();

if (rs.length === 0) {
    Panel_Detail.setVisible(false);
    Text_NoData.setVisible(true);
    Text_NoData.setText("No hay datos para los filtros seleccionados");
} else {
    Panel_Detail.setVisible(true);
    Text_NoData.setVisible(false);
}
```

### Patron: Mostrar alerta si valor critico

```javascript
// En onResultChanged de Table_KPI
var ds = Table_KPI.getDataSource();
var revenue = ds.getData({"Account": "Revenue", "Version": "Actual"});

if (revenue !== undefined) {
    var value = ConvertUtils.stringToNumber(revenue.rawValue);
    if (value < 0) {
        Panel_Alert.setVisible(true);
        Text_Alert.setText("ALERTA: Revenue negativo (" + revenue.formattedValue + ")");
    } else {
        Panel_Alert.setVisible(false);
    }
}
```

### Patron: Toggle de secciones con botones

```javascript
// En onClick de Button_ToggleChart
Chart_Analysis.setVisible(!Chart_Analysis.isVisible());

if (Chart_Analysis.isVisible()) {
    Button_ToggleChart.setText("Ocultar Grafico");
} else {
    Button_ToggleChart.setText("Mostrar Grafico");
}
```

## Habilitar/Deshabilitar basado en rol de usuario

```javascript
// En onInitialization
var roles = Application.getRolesInfo();

// Verificar si el usuario tiene rol de planificador
var isPlanner = false;
for (var i = 0; i < roles.length; i++) {
    if (roles[i] === "BI_User_Planner" || roles[i] === "Planning_Professional") {
        isPlanner = true;
        break;
    }
}

// Habilitar/deshabilitar widgets de planning
Button_SavePlan.setEnabled(isPlanner);
Button_PublishVersion.setEnabled(isPlanner);
Button_RunDataAction.setEnabled(isPlanner);
Panel_PlanningTools.setVisible(isPlanner);

if (!isPlanner) {
    Text_RoleInfo.setText("Modo lectura. Contacta al administrador para permisos de planning.");
    Text_RoleInfo.setVisible(true);
}
```

### Patron: Distintos niveles de acceso

```javascript
// En onInitialization
var roles = Application.getRolesInfo();

var accessLevel = "VIEWER"; // Por defecto

for (var i = 0; i < roles.length; i++) {
    var role = roles[i];
    if (role === "Planning_Admin") {
        accessLevel = "ADMIN";
        break;
    } else if (role === "BI_User_Planner") {
        accessLevel = "PLANNER";
        // No break, por si tiene ADMIN tambien
    }
}

// Aplicar permisos segun nivel
if (accessLevel === "VIEWER") {
    // Solo lectura
    Button_Edit.setVisible(false);
    Button_Publish.setVisible(false);
    Button_Delete.setVisible(false);
    Panel_DataEntry.setVisible(false);
} else if (accessLevel === "PLANNER") {
    // Puede editar pero no publicar/eliminar
    Button_Edit.setVisible(true);
    Button_Publish.setVisible(false);
    Button_Delete.setVisible(false);
    Panel_DataEntry.setVisible(true);
} else if (accessLevel === "ADMIN") {
    // Acceso completo
    Button_Edit.setVisible(true);
    Button_Publish.setVisible(true);
    Button_Delete.setVisible(true);
    Panel_DataEntry.setVisible(true);
}
```

## Habilitar/Deshabilitar basado en estado de datos

### Patron: Deshabilitar Save si no hay cambios

```javascript
// En onInitialization
ScriptVariable_IsDirty = false;
Button_Save.setEnabled(false);

// En cualquier evento que modifique datos
// (InputField onChange, setUserInput, etc.)
ScriptVariable_IsDirty = true;
Button_Save.setEnabled(true);
Button_Discard.setEnabled(true);

// En onClick de Button_Save
var planning = Table_Plan.getPlanning();
planning.submitData();
ScriptVariable_IsDirty = false;
Button_Save.setEnabled(false);
Button_Discard.setEnabled(false);
```

### Patron: Deshabilitar acciones durante ejecucion

```javascript
// En onClick de Button_Execute
// Deshabilitar controles durante la ejecucion
Button_Execute.setEnabled(false);
Dropdown_Version.setEnabled(false);
Dropdown_Year.setEnabled(false);

Application.showBusyIndicator("Ejecutando...");

var result = DataAction_Calc.execute();

Application.hideBusyIndicator();

// Rehabilitar controles
Button_Execute.setEnabled(true);
Dropdown_Version.setEnabled(true);
Dropdown_Year.setEnabled(true);

if (result.status === "OK") {
    Application.showMessage(ApplicationMessageType.Success, "Completado");
}
```

## Styling programatico con CSS Classes

### Definir clases CSS

En el editor CSS de la aplicacion (Styling > CSS), defines las clases:

```css
/* En el editor CSS de la aplicacion */
.highlight-positive {
    background-color: #e8f5e9;
    border-left: 4px solid #4caf50;
}

.highlight-negative {
    background-color: #ffebee;
    border-left: 4px solid #f44336;
}

.highlight-warning {
    background-color: #fff3e0;
    border-left: 4px solid #ff9800;
}

.dimmed {
    opacity: 0.5;
}

.hidden-border {
    border: none;
}
```

### Aplicar clases CSS por script

```javascript
// Aplicar clase CSS a un widget
Panel_Status.setCssClass("highlight-positive");

// Cambiar clase basado en condicion
var ds = Table_KPI.getDataSource();
var margin = ds.getData({"Account": "Margin"});

if (margin !== undefined) {
    var marginValue = ConvertUtils.stringToNumber(margin.rawValue);
    if (marginValue >= 0.1) {
        Panel_MarginStatus.setCssClass("highlight-positive");
        Text_MarginStatus.setText("Margen saludable");
    } else if (marginValue >= 0) {
        Panel_MarginStatus.setCssClass("highlight-warning");
        Text_MarginStatus.setText("Margen bajo");
    } else {
        Panel_MarginStatus.setCssClass("highlight-negative");
        Text_MarginStatus.setText("Margen negativo");
    }
}
```

### Quitar clase CSS

```javascript
// Para quitar una clase, establece una cadena vacia
Panel_Status.setCssClass("");
```

> **Limitacion:** `setCssClass()` aplica la clase al widget **completo**.
> No puedes aplicar estilos a celdas individuales de una tabla o a puntos
> individuales de un chart. Para formato condicional a nivel de celda,
> usa la funcionalidad de Conditional Formatting del design time.

## Patron: Tabs manuales con botones

```javascript
// Simular tabs usando botones y paneles
// En onInitialization
Panel_Tab1.setVisible(true);
Panel_Tab2.setVisible(false);
Panel_Tab3.setVisible(false);
Button_Tab1.setCssClass("tab-active");

// En onClick de Button_Tab1
Panel_Tab1.setVisible(true);
Panel_Tab2.setVisible(false);
Panel_Tab3.setVisible(false);
Button_Tab1.setCssClass("tab-active");
Button_Tab2.setCssClass("");
Button_Tab3.setCssClass("");

// En onClick de Button_Tab2
Panel_Tab1.setVisible(false);
Panel_Tab2.setVisible(true);
Panel_Tab3.setVisible(false);
Button_Tab1.setCssClass("");
Button_Tab2.setCssClass("tab-active");
Button_Tab3.setCssClass("");

// Optimizacion: pausar refresh de tabs no visibles
Table_Tab2.getDataSource().setRefreshPaused(true);

// Al activar Tab2, reanudar refresh
Table_Tab2.getDataSource().setRefreshPaused(false);
```

## Patron: Formulario dinamico

```javascript
// En onSelect de Dropdown_InputType
var inputType = Dropdown_InputType.getSelectedKey();

// Mostrar/ocultar campos segun el tipo seleccionado
InputField_Amount.setVisible(inputType === "MANUAL");
Dropdown_Factor.setVisible(inputType === "FACTOR");
Panel_Upload.setVisible(inputType === "UPLOAD");

// Habilitar/deshabilitar boton segun si los campos estan completos
if (inputType === "MANUAL") {
    Button_Apply.setEnabled(InputField_Amount.getValue() !== "");
} else if (inputType === "FACTOR") {
    Button_Apply.setEnabled(Dropdown_Factor.getSelectedKey() !== "");
} else {
    Button_Apply.setEnabled(true);
}
```

## Patron completo: Panel de administracion

```javascript
// En onInitialization
var userInfo = Application.getUserInfo();
var roles = Application.getRolesInfo();

// Determinar permisos
var canEdit = false;
var canAdmin = false;

for (var i = 0; i < roles.length; i++) {
    if (roles[i] === "BI_User_Planner") { canEdit = true; }
    if (roles[i] === "Planning_Admin") { canAdmin = true; canEdit = true; }
}

// Configurar UI segun permisos
Panel_AdminTools.setVisible(canAdmin);
Panel_EditTools.setVisible(canEdit);
Button_MasterData.setEnabled(canAdmin);
Button_DataLocking.setEnabled(canAdmin);

// Configurar saludo
Text_Header.setText("Dashboard de Planning - " + userInfo.displayName);

// Configurar estado visual
if (canAdmin) {
    Text_RoleBadge.setText("Administrador");
    Text_RoleBadge.setCssClass("badge-admin");
} else if (canEdit) {
    Text_RoleBadge.setText("Planificador");
    Text_RoleBadge.setCssClass("badge-planner");
} else {
    Text_RoleBadge.setText("Lectura");
    Text_RoleBadge.setCssClass("badge-viewer");
}
```

## Pitfalls comunes

1. **setVisible en widgets dentro de containers:** Si un container (Panel) es invisible, hacer `setVisible(true)` en un widget hijo no lo muestra. Primero haz visible el container padre.

2. **setEnabled no cambia la apariencia visual:** `setEnabled(false)` deshabilita la interaccion pero visualmente el cambio puede ser sutil. Combina con `setCssClass("dimmed")` para hacer mas evidente el estado.

3. **Performance con muchos setVisible:** Llamar `setVisible()` en muchos widgets en un bucle puede causar multiples re-renders. Si necesitas cambiar muchos widgets a la vez, agrupa la logica.

4. **setCssClass sobrescribe, no anade:** `setCssClass("class-a")` seguido de `setCssClass("class-b")` deja SOLO `class-b`. No se acumulan. Para aplicar multiples clases, usa `setCssClass("class-a class-b")`.

5. **getRolesInfo devuelve strings:** Los nombres de roles son strings. Un typo en la comparacion (`"BI_user_Planner"` vs `"BI_User_Planner"`) hace que la verificacion falle silenciosamente. Usa console.log para verificar los nombres exactos.
