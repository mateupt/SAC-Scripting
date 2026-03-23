# Table Widget API

## Metodos disponibles en el widget Table

El widget Table es uno de los mas ricos en API dentro de SAC Analytics Designer.
Estos son los metodos confirmados en la API Reference Guide (version 2025.14+):

| Metodo | Descripcion |
|--------|-------------|
| `getDataSource()` | Devuelve el DataSource vinculado a la tabla |
| `getSelections()` | Devuelve un array de objetos Selection con las celdas seleccionadas |
| `getPlanning()` | Devuelve el objeto Planning de la tabla (undefined si el modelo no soporta planning) |
| `getRowCount()` | Devuelve el numero de filas visibles |
| `getDimensionsOnRows()` | Devuelve las dimensiones en filas |
| `getDimensionsOnColumns()` | Devuelve las dimensiones en columnas |
| `getNumberFormat()` | Devuelve el objeto TableNumberFormat |
| `isEnabled()` | Devuelve si la tabla esta habilitada |
| `isVisible()` | Devuelve si la tabla es visible |
| `isCompactDisplayEnabled()` | Devuelve si el modo compacto esta activo |
| `isZeroSuppressionEnabled()` | Devuelve si la supresion de ceros esta activa |
| `setEnabled(boolean)` | Habilita/deshabilita la tabla |
| `setVisible(boolean)` | Muestra/oculta la tabla |
| `setCompactDisplayEnabled(boolean)` | Activa/desactiva modo compacto |
| `setZeroSuppressionEnabled(boolean)` | Activa/desactiva supresion de ceros |
| `setModel(modelId)` | Cambia el modelo vinculado |
| `setContextMenuVisible(boolean)` | Muestra/oculta el menu contextual |
| `setQuickActionsVisibility(boolean)` | Muestra/oculta acciones rapidas |
| `setBreakGroupingEnabled(boolean)` | Activa/desactiva el break grouping |
| `setActiveDimensionProperties(...)` | Configura las propiedades activas de una dimension |
| `sortByMember(dimension, sortOrder)` | Ordena por miembros de una dimension |
| `sortByValue(measure, sortOrder)` | Ordena por valores de una medida |
| `rankBy(measure, sortOrder, count)` | Aplica ranking por medida |
| `removeSorting()` | Quita cualquier ordenacion aplicada |
| `removeRanking()` | Quita el ranking aplicado |
| `removeDimension(dimension)` | Quita una dimension de la tabla |
| `openNavigationPanel()` | Abre el panel de navegacion |
| `openInNewStory()` | Abre los datos en un nuevo story |
| `openSelectModelDialog()` | Abre el dialogo de seleccion de modelo |

### Eventos

| Evento | Cuando se dispara |
|--------|------------------|
| `onSelect` | El usuario selecciona celdas en la tabla |
| `onResultChanged` | Los datos de la tabla cambian (filtro, refresh, etc.) |

## getSelections() - Leer selecciones del usuario

`getSelections()` devuelve un array de objetos `Selection`. Cada objeto es un diccionario con pares dimension-miembro que identifican la celda seleccionada.

```javascript
// En el evento onSelect de Table_1
var selections = Table_1.getSelections();

if (selections.length > 0) {
    // Cada selection es un objeto {dimension: miembro, ...}
    var firstSelection = selections[0];

    // Acceder a una dimension especifica
    var region = firstSelection["Region"];
    var account = firstSelection["Account"];

    Application.showMessage(
        ApplicationMessageType.Info,
        "Seleccionado: " + region + " - " + account
    );
}
```

### Patron: Filtrar otro widget basado en seleccion de tabla

```javascript
// En onSelect de Table_Resumen
var sel = Table_Resumen.getSelections();

if (sel.length > 0) {
    var selectedRegion = sel[0]["Region"];

    // Aplicar filtro al DataSource de otra tabla
    var dsDetalle = Table_Detalle.getDataSource();
    dsDetalle.setDimensionFilter("Region", selectedRegion);
}
```

### Patron: Seleccion multiple

```javascript
// En onSelect de Table_1
var selections = Table_1.getSelections();
var regionList = [];

for (var i = 0; i < selections.length; i++) {
    var region = selections[i]["Region"];
    if (regionList.indexOf(region) === -1) {
        regionList.push(region);
    }
}

// Aplicar filtro con multiples valores
if (regionList.length > 0) {
    var dsChart = Chart_1.getDataSource();
    dsChart.setDimensionFilter("Region", regionList);
}
```

## Ordenacion programatica

### sortByMember

Ordena los miembros de una dimension alfabeticamente.

```javascript
// Ordenar Region ascendente
Table_1.sortByMember("Region", SortOrder.Ascending);

// Ordenar descendente
Table_1.sortByMember("Region", SortOrder.Descending);
```

### sortByValue

Ordena por los valores de una medida.

```javascript
// Ordenar por Revenue descendente
Table_1.sortByValue("Revenue", SortOrder.Descending);

// Ordenar por Amount ascendente
Table_1.sortByValue("Amount", SortOrder.Ascending);
```

### rankBy

Muestra solo los N primeros/ultimos.

```javascript
// Top 10 por Revenue
Table_1.rankBy("Revenue", SortOrder.Descending, 10);

// Bottom 5 por Cost
Table_1.rankBy("Cost", SortOrder.Ascending, 5);

// Quitar ranking
Table_1.removeRanking();
```

### Quitar ordenacion

```javascript
Table_1.removeSorting();
```

> **Pitfall:** `sortByMember` y `sortByValue` fueron introducidos progresivamente.
> En versiones antiguas de Analytics Designer podian no hacer nada sin error.
> Si no funcionan, verifica la version de tu tenant. Estos metodos estan disponibles
> de forma completa a partir de versiones 2024+.

## Manipulacion de dimensiones

```javascript
// Quitar una dimension de la tabla
Table_1.removeDimension("Product");

// Nota: No existe addDimension() en Table de forma directa.
// Para anadir dimensiones, normalmente se usa el Builder Panel
// o se reconfigura el modelo.
```

## Supresion de ceros

```javascript
// Activar supresion de ceros (oculta filas con todos los valores a 0)
Table_1.setZeroSuppressionEnabled(true);

// Verificar estado
var suppressed = Table_1.isZeroSuppressionEnabled();
```

## Formato numerico

```javascript
// Obtener el objeto de formato numerico
var numFormat = Table_1.getNumberFormat();

// Cambiar decimales
numFormat.setDecimalPlaces(2);
```

## Comentarios en tabla

La Table API incluye metodos para gestionar comentarios en celdas:

```javascript
// Anadir un comentario a una seleccion
var sel = Table_1.getSelections();
if (sel.length > 0) {
    Table_1.addComment(sel[0], "Revisar este valor con Finance");
}

// Obtener comentario de una celda
var comment = Table_1.getComment(sel[0]);

// Eliminar comentario
Table_1.removeComment(sel[0]);

// Eliminar todos los comentarios
Table_1.removeAllComments();
```

## Conditional Formatting via Script

> **Importante:** SAC no expone un metodo tipo `applyConditionalFormat()` en la API de scripting.
> El Conditional Formatting se configura en **design time** desde el Builder Panel de la tabla,
> no programaticamente.

Lo que SI puedes hacer por script es combinar datos con logica visual:

### Patron: Formato condicional simulado con CSS Classes

```javascript
// En onResultChanged de Table_1
// Usar setCssClass en la tabla completa (no por celda)
var ds = Table_1.getDataSource();
var revenue = ds.getData({"Account": "Revenue"});

if (revenue !== undefined) {
    var value = ConvertUtils.stringToNumber(revenue.rawValue);
    if (value < 0) {
        Table_1.setCssClass("table-negative");
    } else {
        Table_1.setCssClass("table-positive");
    }
}
```

> **Limitacion:** `setCssClass()` se aplica a nivel de widget completo, no por celda individual.
> Para formato condicional por celda, usa la configuracion de design time.

## setColumnWidth y freezeColumns

> **Estado actual:** No existen metodos `setColumnWidth()` ni `freezeColumns()` en la
> API de scripting de Analytics Designer. Estas funcionalidades existen **solo en la UI
> de design time** (panel de propiedades de la tabla) o a traves del menu contextual
> en runtime.
>
> Si necesitas columnas congeladas o anchos especificos, configuralos en design time.
> No es posible cambiarlos dinamicamente por script.

## Data Entry con Table Planning API

Para escribir datos de vuelta al modelo, ver la guia [Data Entry y Planning](./12-DATA-ENTRY-PLANNING.md).

## Pitfalls comunes

1. **getSelections() devuelve array vacio:** Ocurre si el usuario hace click fuera de una celda de datos, o si llamas `getSelections()` fuera del evento `onSelect`.

2. **sortByMember/sortByValue no hace nada:** Verificar la version del tenant. En versiones pre-2024, estos metodos podian estar disponibles en el autocompletado pero no tener efecto.

3. **getPlanning() devuelve undefined:** El modelo vinculado a la tabla no tiene planning habilitado, o la tabla esta en modo solo lectura.

4. **Diferencia entre Selection y Filter:** `getSelections()` devuelve lo que el usuario ha clickado visualmente. No es lo mismo que un filtro del DataSource. Para obtener los filtros actuales, usa `getDataSource().getDimensionFilters()`.

5. **onResultChanged se dispara multiples veces:** Al aplicar varios filtros seguidos, `onResultChanged` se dispara por cada cambio. Usa logica de control para evitar cascadas.
