# SAC Temas Avanzados de Scripting

## 1. Clases de utilidad

SAC incluye clases de utilidad para operaciones comunes que no requieren widgets:

### ArrayUtils

```javascript
// Buscar en arrays
var index = ArrayUtils.findIndex(myArray, function(item) {
    return item.id === "target";
});
```

### StringUtils

```javascript
// Operaciones con strings
// Utiles para formatear valores antes de mostrarlos
```

### ConvertUtils

```javascript
// Conversion entre tipos
// SAC es estrictamente tipado; no permite type casting directo
```

### NumberFormat y DateFormat

```javascript
// Formatear numeros y fechas para visualizacion
// Respetan la configuracion regional del tenant
```

> **Nota:** La documentacion completa de estas clases esta en la
> [API Reference oficial](https://help.sap.com/doc/958d4c11261f42e992e8d01a4c0dde25/release/en-US/index.html).
> Esta seccion cubre los patrones mas comunes.

---

## 2. Export programatico

Desde scripts puedes exportar contenido a PDF, Excel, CSV y PowerPoint:

```javascript
// Exportar a PDF
var exportPdf = Application.getExportPdf();
exportPdf.export();

// Exportar a Excel
var exportExcel = Application.getExportExcel();
exportExcel.export();

// Exportar a CSV
var exportCsv = Application.getExportCsv();
exportCsv.export();

// Exportar a PowerPoint
var exportPptx = Application.getExportPptx();
exportPptx.export();
```

Util para automatizar la generacion de informes desde botones en la aplicacion.

---

## 3. Composites (componentes reutilizables)

Los Composites son el mecanismo oficial de reutilizacion **entre stories**:

- Encapsulan widgets + scripting + layout en un componente modular
- Exponen interfaces: **properties**, **functions** y **events**
- Se actualizan centralmente; las stories consumidoras reciben la version mas reciente al ejecutarse
- Funcion "Where Used" permite identificar que stories usan un Composite antes de modificarlo

### Cuando usar Composites vs ScriptObjects

| Necesidad | Solucion |
|-----------|----------|
| Reutilizar logica dentro de UNA aplicacion | ScriptObject |
| Reutilizar widgets + logica entre MULTIPLES stories | Composite |
| Compartir un panel de filtros estandar entre dashboards | Composite |
| Funcion helper para formatear datos | ScriptObject |

### Crear un Composite

1. File → New → Composite
2. Disena el layout con widgets
3. Define la **interfaz publica**:
   - Properties: valores que la story consumidora puede configurar
   - Functions: metodos que la story consumidora puede llamar
   - Events: notificaciones que el Composite emite hacia la story

### Usar un Composite desde una story

```javascript
// Llamar una funcion publica del Composite
Composite_Filters.applyFilters("Region", "EMEA");

// Leer una propiedad publica del Composite
var selectedValue = Composite_Filters.getProperty("selectedRegion");

// Escuchar un evento del Composite (configurado en el Designer)
// En el evento onCustomEvent del Composite:
var detail = Composite_Filters.getEventDetail();
```

> **Novedad 2025:** Los Script Objects dentro de un Composite pueden exponerse
> automaticamente como funciones de interfaz publica.

---

## 4. Navegacion entre stories con parametros

### Crear URL con parametros

```javascript
// Crear URL que abre otra story con parametros predefinidos
var params = [
    UrlParameter.create("p_year", "2025"),
    UrlParameter.create("p_region", "EU")
];
var url = Application.createStoryUrl("STORY_ID", params);
Application.openUrl(url, true); // true = nueva ventana
```

### Recibir parametros en la story destino

Los parametros se mapean a Script Variables en la story destino.
La URL resultante tiene formato:

```
https://tenant.sapanalytics.cloud/story/STORY_ID;p_year=2025;p_region=EU
```

---

## 5. Seguridad

### Data Access Control (DAC)

- Se activa por dimension en cada modelo
- Los scripts **respetan el DAC automaticamente**: `getMembers()` solo devuelve miembros autorizados
- La seguridad real esta en el backend (DAC), no en ocultar widgets

### Validar URLs dinamicas

```javascript
// PELIGRO: abrir URL construida con input del usuario sin validar
Application.openUrl(InputField_Url.getValue(), true); // NO hacer esto

// SEGURO: validar dominio antes de abrir
var url = InputField_Url.getValue();
if (url.indexOf("https://mycompany.sap.com") === 0) {
    Application.openUrl(url, true);
} else {
    Application.showMessage(ApplicationMessageType.Error, "URL no permitida");
}
```

### Patron role-based UI

```javascript
// Ocultar widgets segun informacion del usuario
// IMPORTANTE: esto es solo UX, no seguridad real
// La seguridad real debe estar en el DAC del modelo
Panel_Admin.setVisible(false); // ocultar por defecto
```

---

## 6. Custom Widgets

Los Custom Widgets permiten integrar librerias JavaScript externas (D3.js, Chart.js, etc.)
dentro de SAC como Web Components:

### Estructura de un Custom Widget

```
my-widget/
  widget.json      # Manifiesto con id, version, propiedades, eventos
  main.js          # Web Component (extends HTMLElement)
  styling.css      # Estilos opcionales
  icon.png         # Icono 48x48
  /libs/           # Librerias externas
```

### Ciclo de vida

| Metodo | Cuando se ejecuta |
|--------|-------------------|
| `onCustomWidgetInit()` | Inicializacion |
| `onCustomWidgetBeforeUpdate()` | Antes de cambios de propiedades/datos |
| `onCustomWidgetAfterUpdate()` | Despues de actualizar |
| `onCustomWidgetResize()` | Cambio de tamano del contenedor |
| `onCustomWidgetDestroy()` | Eliminacion del widget |

### Acceso a datos desde Custom Widget

```javascript
onCustomWidgetAfterUpdate(changedProperties) {
    var dataBinding = this.dataBinding.getDataBinding();
    var dimensions = dataBinding.dimensions;
    var measures = dataBinding.measures;
    var data = dataBinding.data;
    // Renderizar con D3, Chart.js, etc.
}
```

> **Restriccion:** No se puede usar builder panel Y data bindings simultaneamente.

> **Repositorio oficial de ejemplos:** https://github.com/SAP-samples/analytics-cloud-custom-widget

---

## 7. Nuevas APIs (2025)

| API | Descripcion | Version minima |
|-----|-------------|---------------|
| Chart Variance | Control programatico de varianzas en graficos | 2025.21 |
| Time Series Forecast | Ejecutar forecasting desde script | 2025.21 |
| Comments | Leer/escribir comentarios en widgets y celdas | 2025.21 |
| Widget API | Embeber charts de SAC en apps externas | 2025.08 |
| Performance Recommendations | Tab por widget con sugerencias automaticas | 2025.17 |
| AI Formula Assistant | Generacion de formulas via prompt (requiere AI units) | 2025.17 |

---

## Resumen de la Application API completa

| Metodo | Que hace |
|--------|----------|
| `Application.showMessage(type, text)` | Muestra mensaje al usuario |
| `Application.showBusyIndicator()` | Muestra indicador de carga |
| `Application.hideBusyIndicator()` | Oculta indicador de carga |
| `Application.openPage(pageId)` | Navega a otra pagina (Stories) |
| `Application.openUrl(url, newWindow?)` | Abre URL externa |
| `Application.createStoryUrl(storyId, params[])` | Crea URL con parametros |
| `Application.getExportPdf()` | Obtiene objeto de exportacion PDF |
| `Application.getExportExcel()` | Obtiene objeto de exportacion Excel |
| `Application.getExportCsv()` | Obtiene objeto de exportacion CSV |
| `Application.getExportPptx()` | Obtiene objeto de exportacion PowerPoint |
