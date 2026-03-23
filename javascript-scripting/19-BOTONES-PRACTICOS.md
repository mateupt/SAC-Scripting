# Botones - Ejemplos Practicos

> Patrones completos de botones en Analytic Applications.
> Cada ejemplo incluye el evento, el script y cuando usarlo.

---

## Referencia rapida del widget Button

| Propiedad/Metodo | Descripcion |
|------------------|-------------|
| `onClick` | Evento principal: se dispara al hacer click |
| `setText(string)` | Cambia el texto del boton |
| `getText()` | Devuelve el texto actual |
| `setEnabled(boolean)` | Habilita/deshabilita (gris si false) |
| `isEnabled()` | Devuelve si esta habilitado |
| `setVisible(boolean)` | Muestra/oculta |
| `isVisible()` | Devuelve si esta visible |
| `setCssClass(string)` | Aplica clase CSS (estilo visual) |
| `getCssClass()` | Devuelve la clase CSS actual |
| `setIcon(string)` | Aplica un icono SAP (sap-icon://nombre) |
| `setBadgeValue(string)` | Muestra un badge numerico en el boton |
| `setTooltip(string)` | Texto al pasar el raton |

---

## 1. Boton Guardar con validacion

**Caso:** El usuario edita datos en una tabla de planning. Al pulsar Guardar, se valida que no haya celdas vacias ni valores negativos.

```javascript
// Button_Save → onClick
var ds = Table_Planning.getDataSource();
var rs = ds.getResultSet();
var errors = [];

// Validar cada fila
for (var i = 0; i < rs.length; i++) {
    var row = rs[i];
    var revenue = row["Revenue"];

    if (revenue === undefined || revenue.rawValue === "") {
        errors.push("Fila " + (i + 1) + ": Revenue vacio");
    } else {
        var val = ConvertUtils.stringToNumber(revenue.rawValue);
        if (val < 0) {
            errors.push("Fila " + (i + 1) + ": Revenue negativo (" + revenue.formattedValue + ")");
        }
    }
}

if (errors.length > 0) {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Errores encontrados:\n" + errors.join("\n")
    );
    return;
}

// Si pasa validacion, guardar
Application.showBusyIndicator("Guardando...");
ds.submitData();
Application.hideBusyIndicator();
Application.showMessage(
    ApplicationMessageType.Success,
    "Datos guardados correctamente (" + rs.length + " filas)"
);
```

---

## 2. Boton Toggle (activar/desactivar vista)

**Caso:** Alternar entre vista de tabla y vista de grafico.

```javascript
// Button_Toggle → onClick
if (Table_Data.isVisible()) {
    // Cambiar a vista grafico
    Table_Data.setVisible(false);
    Chart_Data.setVisible(true);
    Button_Toggle.setText("Ver Tabla");
    Button_Toggle.setIcon("sap-icon://table-view");
} else {
    // Cambiar a vista tabla
    Table_Data.setVisible(true);
    Chart_Data.setVisible(false);
    Button_Toggle.setText("Ver Grafico");
    Button_Toggle.setIcon("sap-icon://bar-chart");
}
```

---

## 3. Boton con confirmacion (eliminar version)

**Caso:** Eliminar una version privada. Requiere doble confirmacion para evitar errores.

```javascript
// Button_Delete → onClick
var version = Dropdown_Version.getSelectedKey();
var versionName = Dropdown_Version.getSelectedText();

if (version === "" || version === "Actual") {
    Application.showMessage(
        ApplicationMessageType.Error,
        "No puedes eliminar la version Actual"
    );
    return;
}

// Guardar contexto para el callback
ScriptVariables.setString("PendingDeleteVersion", version);

Application.showMessage(
    ApplicationMessageType.Confirm,
    "Eliminar la version '" + versionName + "'?\nEsta accion no se puede deshacer."
);
```

```javascript
// Application → onMessage
var pendingVersion = ScriptVariables.getString("PendingDeleteVersion");

if (pendingVersion !== "" && message === "Confirm") {
    Application.showBusyIndicator("Eliminando version...");

    var ds = Table_Planning.getDataSource();
    ds.deleteVersion(pendingVersion);

    // Limpiar estado
    ScriptVariables.setString("PendingDeleteVersion", "");

    // Refrescar dropdown de versiones
    ScriptObjects.refreshVersionDropdown();

    Application.hideBusyIndicator();
    Application.showMessage(
        ApplicationMessageType.Success,
        "Version eliminada"
    );
}
```

---

## 4. Boton de filtro rapido (chips)

**Caso:** Fila de botones que actuan como filtros rapidos (como chips/tags).

```javascript
// Button_FilterQ1 → onClick
applyQuarterFilter("Q1", ["202601", "202602", "202603"]);

// Button_FilterQ2 → onClick
applyQuarterFilter("Q2", ["202604", "202605", "202606"]);

// Button_FilterQ3 → onClick
applyQuarterFilter("Q3", ["202607", "202608", "202609"]);

// Button_FilterQ4 → onClick
applyQuarterFilter("Q4", ["202610", "202611", "202612"]);

// Button_FilterAll → onClick
applyQuarterFilter("ALL", []);
```

```javascript
// ScriptObject → applyQuarterFilter(quarter, months)
function applyQuarterFilter(quarter, months) {
    var buttons = {
        "Q1": Button_FilterQ1,
        "Q2": Button_FilterQ2,
        "Q3": Button_FilterQ3,
        "Q4": Button_FilterQ4,
        "ALL": Button_FilterAll
    };

    // Resaltar boton activo
    for (var key in buttons) {
        if (key === quarter) {
            buttons[key].setCssClass("chip-active");
        } else {
            buttons[key].setCssClass("chip-inactive");
        }
    }

    // Aplicar filtro
    var ds = Chart_Revenue.getDataSource();
    if (quarter === "ALL") {
        ds.removeFilters("Date");
    } else {
        ds.setFilter("Date", months);
    }

    // Sincronizar tabla
    var dsTable = Table_Detail.getDataSource();
    if (quarter === "ALL") {
        dsTable.removeFilters("Date");
    } else {
        dsTable.setFilter("Date", months);
    }
}
```

---

## 5. Boton Exportar a PDF / Excel

**Caso:** Exportar el contenido visible del dashboard.

```javascript
// Button_ExportPDF → onClick
Application.showBusyIndicator("Generando PDF...");

// Exportar pagina actual como PDF
Application.export(ExportType.PDF, {
    "fileName": "Dashboard_" + Dropdown_Year.getSelectedKey(),
    "paperSize": "A4",
    "orientation": "Landscape",
    "includeFilters": true
});

Application.hideBusyIndicator();
```

```javascript
// Button_ExportExcel → onClick
var ds = Table_Report.getDataSource();
var rs = ds.getResultSet();

if (rs.length === 0) {
    Application.showMessage(
        ApplicationMessageType.Warning,
        "No hay datos para exportar"
    );
    return;
}

Table_Report.exportToCSV(
    "Reporte_" + Dropdown_Region.getSelectedKey() + "_" +
    Dropdown_Year.getSelectedKey()
);
```

---

## 6. Boton Ejecutar Data Action con parametros

**Caso:** Boton que lanza un calculo complejo en el backend (Data Action).

```javascript
// Button_RunForecast → onClick
var year = Dropdown_Year.getSelectedKey();
var region = Dropdown_Region.getSelectedKey();
var method = Dropdown_Method.getSelectedKey();  // "LINEAR", "CAGR", "WEIGHTED"

if (year === "" || region === "" || method === "") {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Selecciona año, region y metodo antes de ejecutar"
    );
    return;
}

// Deshabilitar boton mientras se ejecuta
Button_RunForecast.setEnabled(false);
Button_RunForecast.setText("Ejecutando...");
Application.showBusyIndicator("Ejecutando forecast " + method + "...");

var ds = Table_Planning.getDataSource();
var params = {
    "YEAR": year,
    "REGION": region,
    "METHOD": method,
    "VERSION_TARGET": "Forecast"
};

try {
    ds.runDataAction("DA_Forecast_Calculation", params);
    ds.refreshData();
    Chart_Preview.getDataSource().refreshData();

    Application.showMessage(
        ApplicationMessageType.Success,
        "Forecast generado para " + region + " - " + year
    );
} catch (e) {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Error en forecast: " + e.message
    );
}

Button_RunForecast.setEnabled(true);
Button_RunForecast.setText("Ejecutar Forecast");
Application.hideBusyIndicator();
```

---

## 7. Boton con estado de progreso (multi-step)

**Caso:** Proceso de cierre mensual con varios pasos secuenciales.

```javascript
// Button_MonthlyClose → onClick
var steps = [
    {"action": "DA_ValidateData", "label": "Validando datos...", "params": {}},
    {"action": "DA_CalculateAllocations", "label": "Calculando allocations...", "params": {}},
    {"action": "DA_CurrencyConversion", "label": "Conversion de moneda...", "params": {}},
    {"action": "DA_CloseMonth", "label": "Cerrando mes...", "params": {}}
];

var ds = Table_Planning.getDataSource();
var month = Dropdown_Month.getSelectedKey();
var totalSteps = steps.length;

Button_MonthlyClose.setEnabled(false);

for (var i = 0; i < totalSteps; i++) {
    var step = steps[i];
    var progress = "Paso " + (i + 1) + "/" + totalSteps + ": " + step.label;
    Application.showBusyIndicator(progress);
    Button_MonthlyClose.setText(progress);

    step.params["MONTH"] = month;

    try {
        ds.runDataAction(step.action, step.params);
    } catch (e) {
        Application.hideBusyIndicator();
        Button_MonthlyClose.setEnabled(true);
        Button_MonthlyClose.setText("Cierre Mensual");
        Application.showMessage(
            ApplicationMessageType.Error,
            "Error en paso " + (i + 1) + ": " + e.message
        );
        return;
    }
}

ds.refreshData();
Application.hideBusyIndicator();
Button_MonthlyClose.setEnabled(true);
Button_MonthlyClose.setText("Cierre Mensual");
Application.showMessage(
    ApplicationMessageType.Success,
    "Cierre completado para " + month
);
```

---

## 8. Boton Undo/Redo (deshacer cambios de planning)

```javascript
// Button_Undo → onClick
var ds = Table_Planning.getDataSource();
ds.undoDataInput();
Application.showMessage(ApplicationMessageType.Info, "Ultimo cambio deshecho");

// Button_Redo → onClick
var ds = Table_Planning.getDataSource();
ds.redoDataInput();
```

---

## 9. Boton de navegacion con contexto

**Caso:** Boton en una tabla que abre el detalle de la fila seleccionada en otra pagina.

```javascript
// Button_ViewDetail → onClick (dentro de una celda custom o al lado de la tabla)
var sel = Table_Summary.getSelections();

if (sel.length === 0) {
    Application.showMessage(
        ApplicationMessageType.Warning,
        "Selecciona una fila primero"
    );
    return;
}

// Capturar contexto de la seleccion
var selectedRow = sel[0];
var costCenter = selectedRow["CostCenter"];
var account = selectedRow["Account"];

// Pasar a pagina de detalle
ScriptVariables.setString("DrillCostCenter", costCenter);
ScriptVariables.setString("DrillAccount", account);
NavigationUtils.navigateToPage("Page_Detail");
```

---

## 10. Boton Refresh inteligente (solo si hay cambios)

```javascript
// Button_Refresh → onClick
Application.showBusyIndicator("Comprobando cambios...");

var ds = Chart_Revenue.getDataSource();
var oldTotal = ScriptVariables.getString("LastTotal");
ds.refreshData();

var newData = ds.getData({"Account": "Revenue"});
var newTotal = newData !== undefined ? newData.rawValue : "0";

if (newTotal !== oldTotal) {
    // Hay cambios: refrescar todo
    Table_Detail.getDataSource().refreshData();
    Chart_Margin.getDataSource().refreshData();
    ScriptObjects.updateKPIs();
    ScriptVariables.setString("LastTotal", newTotal);

    var now = DateUtils.now();
    Text_LastUpdate.setText(
        "Actualizado: " + now.getHours().toString().padStart(2, "0") + ":" +
        now.getMinutes().toString().padStart(2, "0")
    );

    Application.showMessage(ApplicationMessageType.Info, "Datos actualizados");
} else {
    Application.showMessage(ApplicationMessageType.Info, "Sin cambios");
}

Application.hideBusyIndicator();
```

---

## 11. Boton con badge (notificaciones pendientes)

```javascript
// En onInitialization o en un Timer
var ds = Table_Alerts.getDataSource();
var rs = ds.getResultSet();
var pendingCount = 0;

for (var i = 0; i < rs.length; i++) {
    if (rs[i]["Status"] !== undefined && rs[i]["Status"].rawValue === "PENDING") {
        pendingCount++;
    }
}

if (pendingCount > 0) {
    Button_Alerts.setBadgeValue(pendingCount.toString());
    Button_Alerts.setCssClass("btn-alert");
} else {
    Button_Alerts.setBadgeValue("");
    Button_Alerts.setCssClass("btn-normal");
}
```

---

## 12. Grupo de botones de accion (toolbar)

**Caso:** Toolbar con multiples acciones de planning.

```javascript
// ScriptObject → setupToolbar()
// Llamar desde onInitialization
function setupToolbar() {
    var ds = Table_Planning.getDataSource();
    var version = Dropdown_Version.getSelectedKey();

    // Verificar permisos de la version
    var members = ds.getMembers("Version");
    var isEditable = false;

    for (var i = 0; i < members.length; i++) {
        if (members[i].id === version) {
            isEditable = members[i].properties !== undefined &&
                         members[i].properties["IS_WRITABLE"] === true;
            break;
        }
    }

    // Habilitar/deshabilitar botones segun permisos
    Button_Save.setEnabled(isEditable);
    Button_Publish.setEnabled(isEditable);
    Button_TopDown.setEnabled(isEditable);
    Button_Revert.setEnabled(isEditable);
    Button_Delete.setEnabled(isEditable && version !== "Actual" && version !== "Plan");

    // Actualizar tooltips
    if (!isEditable) {
        Button_Save.setTooltip("Version de solo lectura");
        Button_Publish.setTooltip("Version de solo lectura");
    } else {
        Button_Save.setTooltip("Guardar cambios pendientes");
        Button_Publish.setTooltip("Publicar version para todos los usuarios");
    }
}
```

---

## Resumen de iconos SAP mas usados en botones

| Icono | Codigo | Uso tipico |
|-------|--------|-----------|
| Guardar | `sap-icon://save` | Guardar datos |
| Editar | `sap-icon://edit` | Modo edicion |
| Eliminar | `sap-icon://delete` | Borrar registros |
| Refrescar | `sap-icon://refresh` | Actualizar datos |
| Exportar | `sap-icon://download` | Exportar/descargar |
| Filtrar | `sap-icon://filter` | Abrir filtros |
| Grafico | `sap-icon://bar-chart` | Vista grafico |
| Tabla | `sap-icon://table-view` | Vista tabla |
| Configurar | `sap-icon://action-settings` | Configuracion |
| Navegar | `sap-icon://navigation-right-arrow` | Ir a detalle |
| Atras | `sap-icon://navigation-left-arrow` | Volver |
| Copiar | `sap-icon://copy` | Duplicar/copiar |
| Alertas | `sap-icon://bell` | Notificaciones |
| PDF | `sap-icon://pdf-attachment` | Exportar PDF |
| Excel | `sap-icon://excel-attachment` | Exportar Excel |
