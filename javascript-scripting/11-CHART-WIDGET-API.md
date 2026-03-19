# Chart Widget API

## Metodos disponibles en el widget Chart

Referencia confirmada en la API Reference Guide (version 2025.14+):

| Metodo | Descripcion |
|--------|-------------|
| `getDataSource()` | Devuelve el DataSource vinculado al chart |
| `getSelections()` | Devuelve las selecciones del usuario en el chart |
| `getDimensions()` | Devuelve las dimensiones actuales del chart |
| `getMeasures(feed)` | Devuelve las medidas en un feed especifico |
| `getMembers(dimension)` | Devuelve los miembros visibles de una dimension |
| `getNumberFormat()` | Devuelve el objeto de formato numerico |
| `getEffectiveAxisScale()` | Devuelve la escala actual del eje |
| `getForecast()` | Devuelve la configuracion de forecast |
| `getSmartGrouping()` | Devuelve la configuracion de smart grouping |
| `getDataChangeInsights()` | Devuelve insights sobre cambios en los datos |
| `isEnabled()` | Devuelve si el chart esta habilitado |
| `isVisible()` | Devuelve si el chart es visible |
| `setEnabled(boolean)` | Habilita/deshabilita el chart |
| `setVisible(boolean)` | Muestra/oculta el chart |
| `addDimension(dimension, feed)` | Anade una dimension a un feed |
| `removeDimension(dimension, feed)` | Quita una dimension de un feed |
| `addMeasure(measure, feed)` | Anade una medida a un feed |
| `removeMeasure(measure, feed)` | Quita una medida de un feed |
| `addMember(dimension, member)` | Anade un miembro visible |
| `removeMember(dimension, member)` | Quita un miembro visible |
| `setAxisScale(config)` | Configura la escala del eje |
| `setBreakGroupingEnabled(boolean)` | Activa/desactiva break grouping |
| `setContextMenuVisible(boolean)` | Muestra/oculta menu contextual |
| `setQuickActionsVisibility(boolean)` | Muestra/oculta acciones rapidas |
| `sortByMember(dimension, sortOrder)` | Ordena por miembros |
| `sortByValue(measure, sortOrder)` | Ordena por valores |
| `rankBy(measure, sortOrder, count)` | Aplica ranking |
| `removeSorting()` | Quita ordenacion |
| `removeRanking()` | Quita ranking |
| `openInNewStory()` | Abre en nuevo story |

### Eventos

| Evento | Cuando se dispara |
|--------|------------------|
| `onSelect` | El usuario hace click en un punto de datos |
| `onResultChanged` | Los datos del chart cambian |

## Feed Types (constantes)

Los feeds determinan donde se posiciona una dimension o medida en el chart:

| Feed | Uso |
|------|-----|
| `Feed.CategoryAxis` | Eje de categorias (X en bar chart) |
| `Feed.CategoryAxis2` | Segundo eje de categorias |
| `Feed.ValueAxis` | Eje de valores (Y en bar chart) |
| `Feed.ValueAxis2` | Segundo eje de valores |
| `Feed.Color` | Dimension de color/leyenda |
| `Feed.Size` | Dimension de tamano (bubble chart) |

## getSelections() - Capturar selecciones

```javascript
// En el evento onSelect de Chart_1
var selections = Chart_1.getSelections();

if (selections.length > 0) {
    // Cada seleccion contiene pares dimension:miembro
    var first = selections[0];
    var region = first["Region"];

    Application.showMessage(
        ApplicationMessageType.Info,
        "Region seleccionada: " + region
    );
}
```

### Patron: Click en chart filtra tabla

```javascript
// En onSelect de Chart_Overview
var sel = Chart_Overview.getSelections();

if (sel.length > 0) {
    var selectedProduct = sel[0]["Product"];
    var dsTable = Table_Detail.getDataSource();
    dsTable.setDimensionFilter("Product", selectedProduct);
} else {
    // Click fuera de un punto de datos = deseleccion
    var dsTable = Table_Detail.getDataSource();
    dsTable.removeDimensionFilter("Product");
}
```

## Cambio dinamico de medidas

Este es uno de los patrones mas utiles: permitir al usuario elegir que medidas ver.

### Patron: Dropdown cambia medidas del chart

```javascript
// En onSelect de Dropdown_Measure
var selectedMeasure = Dropdown_Measure.getSelectedKey();

// Quitar todas las medidas actuales
var currentMeasures = Chart_1.getMeasures(Feed.ValueAxis);
for (var i = 0; i < currentMeasures.length; i++) {
    Chart_1.removeMeasure(currentMeasures[i], Feed.ValueAxis);
}

// Anadir la medida seleccionada
Chart_1.addMeasure(selectedMeasure, Feed.ValueAxis);
```

### Patron: CheckboxGroup para multiples medidas

```javascript
// En onSelect de CheckboxGroup_Measures
var selectedKeys = CheckboxGroup_Measures.getSelectedKeys();

// Quitar medidas actuales del ValueAxis
var current = Chart_1.getMeasures(Feed.ValueAxis);
for (var i = 0; i < current.length; i++) {
    Chart_1.removeMeasure(current[i], Feed.ValueAxis);
}

// Anadir medidas seleccionadas
for (var j = 0; j < selectedKeys.length; j++) {
    Chart_1.addMeasure(selectedKeys[j], Feed.ValueAxis);
}
```

## Cambio dinamico de dimensiones

```javascript
// En onSelect de Dropdown_Dimension
var selectedDim = Dropdown_Dimension.getSelectedKey();

// Quitar dimensiones actuales del CategoryAxis
var currentDims = Chart_1.getDimensions();
for (var i = 0; i < currentDims.length; i++) {
    Chart_1.removeDimension(currentDims[i], Feed.CategoryAxis);
}

// Anadir la nueva dimension
Chart_1.addDimension(selectedDim, Feed.CategoryAxis);
```

## Sobre setChartType()

> **Importante:** No existe un metodo `setChartType()` en la API de Analytics Designer.
> El tipo de chart (bar, line, pie, etc.) se configura en **design time** y no se puede
> cambiar programaticamente en runtime.

### Workaround: Multiples charts con toggle de visibilidad

```javascript
// Patron para simular cambio de tipo de chart
// Creas multiples charts con el mismo DataSource pero distinto tipo

// En onSelect de Dropdown_ChartType
var chartType = Dropdown_ChartType.getSelectedKey();

// Ocultar todos
Chart_Bar.setVisible(false);
Chart_Line.setVisible(false);
Chart_Pie.setVisible(false);

// Mostrar el seleccionado
if (chartType === "BAR") {
    Chart_Bar.setVisible(true);
} else if (chartType === "LINE") {
    Chart_Line.setVisible(true);
} else if (chartType === "PIE") {
    Chart_Pie.setVisible(true);
}
```

> Este workaround es la practica estandar en la comunidad SAC. Requiere mantener
> multiples widgets sincronizados con los mismos filtros.

## Escala de ejes

```javascript
// Configurar escala fija del eje
Chart_1.setAxisScale({
    minValue: 0,
    maxValue: 1000000
});

// Obtener la escala efectiva actual
var scale = Chart_1.getEffectiveAxisScale();
```

## Ordenacion y ranking en charts

```javascript
// Ordenar por valor descendente
Chart_1.sortByValue("Revenue", SortOrder.Descending);

// Top 10 por Revenue
Chart_1.rankBy("Revenue", SortOrder.Descending, 10);

// Quitar ordenacion
Chart_1.removeSorting();
Chart_1.removeRanking();
```

## Manipulacion de leyenda y data labels

> **Limitacion:** No existen metodos de scripting para manipular la leyenda,
> data labels, o formato de etiquetas de datos. Estas configuraciones son
> exclusivas del **design time** (panel de propiedades del chart).
>
> Lo que si puedes hacer:
> - Cambiar que medidas/dimensiones aparecen (que afecta la leyenda indirectamente)
> - Usar `setCssClass()` para aplicar estilos CSS globales al chart
> - Cambiar el formato numerico con `getNumberFormat()`

## Patron completo: Dashboard interactivo

```javascript
// En onInitialization de Application
// Configurar chart inicial con las medidas por defecto
Chart_Main.addMeasure("Revenue", Feed.ValueAxis);
Chart_Main.addMeasure("Cost", Feed.ValueAxis);
Chart_Main.addDimension("Region", Feed.CategoryAxis);

// En onSelect de Chart_Main
var sel = Chart_Main.getSelections();
if (sel.length > 0) {
    var region = sel[0]["Region"];

    // Actualizar label
    Text_Selected.setText("Detalle de: " + region);

    // Filtrar tabla de detalle
    Table_Detail.getDataSource().setDimensionFilter("Region", region);

    // Filtrar segundo chart
    Chart_Trend.getDataSource().setDimensionFilter("Region", region);
}
```

## Pitfalls comunes

1. **removeMeasure/removeDimension falla silenciosamente:** Si intentas quitar una medida que no esta en el feed especificado, no da error pero no hace nada. Verifica con `getMeasures(feed)` antes.

2. **Chart se queda vacio:** Si quitas todas las medidas y/o dimensiones sin anadir otras, el chart se queda en blanco. Siempre anade antes de quitar, o usa una secuencia quitar-todo + anadir-nuevo.

3. **Feed incorrecto:** Usar `Feed.ValueAxis` cuando deberia ser `Feed.ValueAxis2` (o viceversa) no da error pero los datos aparecen en el eje equivocado.

4. **getSelections() en onResultChanged:** No confundir eventos. `getSelections()` tiene sentido en `onSelect`, no en `onResultChanged`.

5. **No existe setChartType:** El autocompletado del editor puede sugerir metodos que no existen en tu version. Siempre verifica contra la API Reference de tu version.
