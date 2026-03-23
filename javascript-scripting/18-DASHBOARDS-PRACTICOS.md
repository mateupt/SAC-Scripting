# Dashboards Practicos en SAC

> Ejemplos completos de dashboards interactivos en Analytic Applications.
> Cada ejemplo incluye la estructura de widgets, el script y el resultado esperado.

---

## 1. Dashboard Ejecutivo con KPIs y Drill-Down

### Estructura de widgets

```
Canvas_Main
├── Panel_Header
│   ├── Text_Title          ("Dashboard Financiero Q1 2026")
│   ├── Dropdown_Year
│   └── Dropdown_Region
├── Panel_KPIs
│   ├── Text_KPI_Revenue    (KPI card)
│   ├── Text_KPI_Margin     (KPI card)
│   ├── Text_KPI_Headcount  (KPI card)
│   └── Text_KPI_Forecast   (KPI card)
├── Panel_Charts
│   ├── Chart_Revenue_Trend  (line chart)
│   └── Chart_Region_Split   (donut chart)
└── Panel_Detail
    └── Table_Detail          (tabla detalle, oculta por defecto)
```

### Script: onInitialization de la Application

```javascript
// Inicializar filtros
var currentYear = DateUtils.now().getFullYear().toString();
Dropdown_Year.setSelectedKey(currentYear);

// Poblar dropdown de regiones
var ds = Chart_Revenue_Trend.getDataSource();
var regions = ds.getMembers("Region");
var items = [];
items.push({"key": "ALL", "text": "Todas las regiones"});
for (var i = 0; i < regions.length; i++) {
    items.push({
        "key": regions[i].id,
        "text": regions[i].description
    });
}
Dropdown_Region.setItems(items);
Dropdown_Region.setSelectedKey("ALL");

// Ocultar panel detalle al inicio
Panel_Detail.setVisible(false);

// Cargar KPIs
updateKPIs();
```

### Script: funcion updateKPIs (ScriptObject)

```javascript
// ScriptObject → funcion updateKPIs()
var ds = Table_Detail.getDataSource();

// Revenue
var rev = ds.getData({"Account": "Revenue", "Version": "Actual"});
if (rev !== undefined) {
    Text_KPI_Revenue.setText(rev.formattedValue);
    var revPlan = ds.getData({"Account": "Revenue", "Version": "Plan"});
    if (revPlan !== undefined) {
        var actual = ConvertUtils.stringToNumber(rev.rawValue);
        var plan = ConvertUtils.stringToNumber(revPlan.rawValue);
        var pct = ((actual - plan) / plan * 100).toFixed(1);
        if (actual >= plan) {
            Text_KPI_Revenue.setCssClass("kpi-green");
        } else {
            Text_KPI_Revenue.setCssClass("kpi-red");
        }
    }
}

// Margin
var margin = ds.getData({"Account": "Gross_Margin_Pct", "Version": "Actual"});
if (margin !== undefined) {
    Text_KPI_Margin.setText(margin.formattedValue);
}

// Headcount
var hc = ds.getData({"Account": "Headcount", "Version": "Actual"});
if (hc !== undefined) {
    Text_KPI_Headcount.setText(hc.formattedValue);
}

// Forecast Accuracy
var fcActual = ds.getData({"Account": "Revenue", "Version": "Actual"});
var fcForecast = ds.getData({"Account": "Revenue", "Version": "Forecast"});
if (fcActual !== undefined && fcForecast !== undefined) {
    var a = ConvertUtils.stringToNumber(fcActual.rawValue);
    var f = ConvertUtils.stringToNumber(fcForecast.rawValue);
    var accuracy = (100 - Math.abs((a - f) / f * 100)).toFixed(1);
    Text_KPI_Forecast.setText(accuracy + "%");
}
```

### Script: Dropdown_Region → onSelect

```javascript
var selected = Dropdown_Region.getSelectedKey();
var ds = Chart_Revenue_Trend.getDataSource();

if (selected === "ALL") {
    ds.removeFilters("Region");
} else {
    ds.setFilter("Region", selected);
}

// Sincronizar todos los widgets con el mismo DataSource
Table_Detail.getDataSource().setFilter("Region",
    selected === "ALL" ? undefined : selected);
Chart_Region_Split.getDataSource().setFilter("Region",
    selected === "ALL" ? undefined : selected);

// Actualizar KPIs
ScriptObjects.updateKPIs();
```

### Script: Chart_Revenue_Trend → onSelect (drill-down a tabla)

```javascript
var sel = Chart_Revenue_Trend.getSelections();
if (sel.length > 0) {
    var month = sel[0]["Date"];
    if (month !== undefined) {
        Table_Detail.getDataSource().setFilter("Date", month);
        Panel_Detail.setVisible(true);
    }
}
```

---

## 2. Dashboard Comparativo Plan vs Actual

### Estructura

```
Canvas_PlanVsActual
├── Panel_Filters
│   ├── Dropdown_Version1   (default: "Actual")
│   ├── Dropdown_Version2   (default: "Plan")
│   └── Button_Compare      ("Comparar")
├── Panel_Variance
│   ├── Table_Variance       (tabla con columnas calculadas)
│   └── Chart_Waterfall      (waterfall de varianzas)
└── Panel_TopBottom
    ├── Chart_Top5            (bar chart top 5 desviaciones positivas)
    └── Chart_Bottom5         (bar chart top 5 desviaciones negativas)
```

### Script: Button_Compare → onClick

```javascript
Application.showBusyIndicator("Calculando varianzas...");

var v1 = Dropdown_Version1.getSelectedKey();
var v2 = Dropdown_Version2.getSelectedKey();

if (v1 === v2) {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Selecciona dos versiones diferentes"
    );
    Application.hideBusyIndicator();
    return;
}

// Configurar tabla de varianzas
var dsTable = Table_Variance.getDataSource();

// Mostrar ambas versiones como columnas
dsTable.removeFilters("Version");
dsTable.setFilter("Version", [v1, v2]);

// Configurar waterfall
var dsWaterfall = Chart_Waterfall.getDataSource();
dsWaterfall.removeFilters("Version");
dsWaterfall.setFilter("Version", [v1, v2]);

// Top 5 / Bottom 5 - usar ranking
Chart_Top5.getDataSource().setFilter("Version", v1);
Chart_Top5.rankBy("Revenue", SortOrder.Descending, 5);

Chart_Bottom5.getDataSource().setFilter("Version", v1);
Chart_Bottom5.rankBy("Revenue", SortOrder.Ascending, 5);

Application.hideBusyIndicator();
```

### Script: Table_Variance → onResultChanged (colorear varianzas)

```javascript
var rs = Table_Variance.getDataSource().getResultSet();

for (var i = 0; i < rs.length; i++) {
    var row = rs[i];
    var actual = row["Revenue_Actual"];
    var plan = row["Revenue_Plan"];

    if (actual !== undefined && plan !== undefined) {
        var a = ConvertUtils.stringToNumber(actual.rawValue);
        var p = ConvertUtils.stringToNumber(plan.rawValue);
        var variance = a - p;

        // Aplicar formato condicional via CSS
        if (variance >= 0) {
            // Verde - por encima del plan
        } else {
            // Rojo - por debajo del plan
        }
    }
}
```

---

## 3. Dashboard de Planning Interactivo

### Estructura

```
Canvas_Planning
├── Panel_Header
│   ├── Dropdown_Version     (versiones privadas del usuario)
│   ├── Button_Save          ("Guardar")
│   ├── Button_Publish       ("Publicar")
│   └── Text_Status          (estado: "Borrador" / "Publicado")
├── Table_Planning           (tabla editable con Data Entry)
├── Panel_Actions
│   ├── Button_TopDown       ("Distribuir Top-Down")
│   ├── Button_Copy          ("Copiar de Actual")
│   └── Button_Revert        ("Deshacer cambios")
└── Chart_Preview            (preview del plan vs actual)
```

### Script: onInitialization - poblar versiones privadas

```javascript
var ds = Table_Planning.getDataSource();
var versions = ds.getMembers("Version");
var items = [];

for (var i = 0; i < versions.length; i++) {
    var v = versions[i];
    // Solo mostrar versiones privadas (no publicadas)
    if (v.properties !== undefined && v.properties["PRIVATE"] === true) {
        items.push({"key": v.id, "text": v.description});
    }
}

Dropdown_Version.setItems(items);

if (items.length > 0) {
    Dropdown_Version.setSelectedKey(items[0].key);
    ds.setFilter("Version", items[0].key);
    Text_Status.setText("Borrador");
    Text_Status.setCssClass("status-draft");
}
```

### Script: Button_Save → onClick

```javascript
Application.showBusyIndicator("Guardando...");

try {
    var ds = Table_Planning.getDataSource();
    ds.submitData();
    Application.showMessage(
        ApplicationMessageType.Success,
        "Datos guardados correctamente"
    );
} catch (e) {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Error al guardar: " + e.message
    );
}

Application.hideBusyIndicator();
```

### Script: Button_Publish → onClick

```javascript
var version = Dropdown_Version.getSelectedKey();

Application.showMessage(
    ApplicationMessageType.Confirm,
    "Publicar la version '" + version + "'? Esta accion no se puede deshacer.",
    "Confirmar publicacion"
);
```

### Script: Application → onMessage (capturar respuesta del confirm)

```javascript
// Solo actuar si es la confirmacion de publicacion
if (message === "Confirm") {
    Application.showBusyIndicator("Publicando version...");

    var ds = Table_Planning.getDataSource();
    var version = Dropdown_Version.getSelectedKey();

    ds.publishVersion(version);

    Text_Status.setText("Publicado");
    Text_Status.setCssClass("status-published");

    // Deshabilitar edicion
    Table_Planning.setEnabled(false);
    Button_Save.setEnabled(false);
    Button_TopDown.setEnabled(false);

    Application.hideBusyIndicator();
}
```

### Script: Button_Copy → onClick (copiar Actual a Plan)

```javascript
Application.showBusyIndicator("Copiando datos de Actual...");

var ds = Table_Planning.getDataSource();
var targetVersion = Dropdown_Version.getSelectedKey();

// Ejecutar Data Action para copiar
var params = {
    "SOURCE_VERSION": "Actual",
    "TARGET_VERSION": targetVersion
};
ds.runDataAction("DA_CopyVersion", params);

// Refrescar vista
ds.refreshData();
Chart_Preview.getDataSource().refreshData();

Application.hideBusyIndicator();
Application.showMessage(
    ApplicationMessageType.Success,
    "Datos copiados desde Actual a " + targetVersion
);
```

### Script: Button_TopDown → onClick (distribuir total por ratio)

```javascript
Application.showBusyIndicator("Distribuyendo top-down...");

var ds = Table_Planning.getDataSource();
var version = Dropdown_Version.getSelectedKey();

var params = {
    "VERSION": version,
    "METHOD": "PROPORTIONAL"  // Proporcional al Actual
};
ds.runDataAction("DA_TopDownDistribution", params);

ds.refreshData();
Chart_Preview.getDataSource().refreshData();

Application.hideBusyIndicator();
```

---

## 4. Dashboard Multi-Pagina con Navegacion

### Estructura (3 paginas)

```
Page_Overview    ← Pagina resumen con KPIs
Page_Detail      ← Detalle por dimension
Page_Settings    ← Configuracion de filtros y preferencias
```

### Script: Navegacion entre paginas

```javascript
// Button_GoDetail → onClick
NavigationUtils.navigateToPage("Page_Detail");

// Button_GoOverview → onClick
NavigationUtils.navigateToPage("Page_Overview");

// Button_GoSettings → onClick
NavigationUtils.navigateToPage("Page_Settings");
```

### Script: Pasar contexto entre paginas via Script Variables

```javascript
// En Page_Overview, al hacer click en un KPI:
// Button_KPI_Revenue → onClick
ScriptVariables.setString("SelectedAccount", "Revenue");
ScriptVariables.setString("SelectedYear", Dropdown_Year.getSelectedKey());
NavigationUtils.navigateToPage("Page_Detail");

// En Page_Detail → onInitialization:
var account = ScriptVariables.getString("SelectedAccount");
var year = ScriptVariables.getString("SelectedYear");

if (account !== "" && year !== "") {
    var ds = Table_Detail.getDataSource();
    ds.setFilter("Account", account);
    ds.setFilter("Date", year + "01", year + "12");  // Rango anual
    Text_Title.setText("Detalle: " + account + " - " + year);
}
```

---

## 5. Dashboard con Refresh Automatico (Timer)

### Script: Auto-refresh cada 5 minutos

```javascript
// onInitialization
Timer_Refresh.start(300000);  // 300,000 ms = 5 minutos

// Timer_Refresh → onTimeout
Application.showBusyIndicator("Actualizando datos...");

// Refrescar todos los DataSources
Chart_Revenue.getDataSource().refreshData();
Chart_Pipeline.getDataSource().refreshData();
Table_Alerts.getDataSource().refreshData();

// Actualizar timestamp
var now = DateUtils.now();
var timestamp = now.getHours().toString().padStart(2, "0") + ":" +
                now.getMinutes().toString().padStart(2, "0");
Text_LastUpdate.setText("Ultima actualizacion: " + timestamp);

Application.hideBusyIndicator();

// Reiniciar timer
Timer_Refresh.start(300000);
```

---

## 6. Dashboard con Export y Compartir

### Script: Exportar tabla a CSV

```javascript
// Button_Export → onClick
var ds = Table_Report.getDataSource();
var rs = ds.getResultSet();

if (rs.length === 0) {
    Application.showMessage(
        ApplicationMessageType.Warning,
        "No hay datos para exportar"
    );
    return;
}

// Usar Export API
Table_Report.exportToCSV("Reporte_" + DateUtils.now().toISOString().slice(0, 10));
```

### Script: Generar enlace para compartir con filtros

```javascript
// Button_Share → onClick
var region = Dropdown_Region.getSelectedKey();
var year = Dropdown_Year.getSelectedKey();

// Construir URL con parametros
var baseUrl = Application.getUrl();
var params = "?Region=" + region + "&Year=" + year;
var shareUrl = baseUrl + params;

// Copiar al clipboard via popup
InputField_ShareLink.setValue(shareUrl);
Popup_Share.open();
```

---

## Patrones comunes en dashboards

### CSS Classes reutilizables

En SAC, las CSS classes se definen en Application → Styling y se aplican con `setCssClass()`:

| Clase | Uso tipico |
|-------|-----------|
| `kpi-green` | KPI por encima del objetivo (color verde) |
| `kpi-red` | KPI por debajo del objetivo (color rojo) |
| `kpi-yellow` | KPI en rango de alerta (color amarillo) |
| `status-draft` | Version borrador |
| `status-published` | Version publicada |
| `highlight-row` | Fila destacada en tabla |

### Patron: Sincronizar filtros entre widgets

```javascript
// Funcion reutilizable en ScriptObject
// syncFilter(dimension, value)
function syncFilter(dim, val) {
    var widgets = [Chart_Revenue, Chart_Margin, Table_Detail, Chart_Donut];

    for (var i = 0; i < widgets.length; i++) {
        var ds = widgets[i].getDataSource();
        if (val === "ALL" || val === undefined) {
            ds.removeFilters(dim);
        } else {
            ds.setFilter(dim, val);
        }
    }
}
```

### Patron: Loading state global

```javascript
// ScriptObject → startLoading()
function startLoading(message) {
    Application.showBusyIndicator(message || "Cargando...");
    Button_Export.setEnabled(false);
    Button_Refresh.setEnabled(false);
}

// ScriptObject → stopLoading()
function stopLoading() {
    Application.hideBusyIndicator();
    Button_Export.setEnabled(true);
    Button_Refresh.setEnabled(true);
}
```
