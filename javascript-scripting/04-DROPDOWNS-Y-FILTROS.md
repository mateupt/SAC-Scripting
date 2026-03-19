# SAC Dropdowns, Filtros y Cascading Filters

## 1. Dropdown basico - Poblar y filtrar

### Poblar dropdown al iniciar la aplicacion

En el evento `onInitialization` del Canvas/Application:

**Opcion A: con getResultSet() (todas las dimensiones)**

```javascript
// Limpiar primero para evitar duplicados si onInitialization se ejecuta mas de una vez
Dropdown_Product.removeAllItems();

var resultSet = Table_1.getDataSource().getResultSet();

for (var i = 0; i < resultSet.length; i++) {
    Dropdown_Product.addItem(resultSet[i]["Product"].id, resultSet[i]["Product"].description);
}
```

**Opcion B: con getMembers() (una dimension)**

```javascript
// Limpiar primero para evitar duplicados
Dropdown_Product.removeAllItems();

var members = Table_1.getDataSource().getMembers("Product");

for (var i = 0; i < members.length; i++) {
    Dropdown_Product.addItem(members[i].id, members[i].description);
}
```

**Diferencia clave:**
- `getResultSet()` devuelve filas con TODAS las dimensiones → accedes con `row["DimName"].id`
- `getMembers("Dim")` devuelve solo miembros de UNA dimension → accedes con `member.id`

### Filtrar al seleccionar

En el evento `onSelect` del Dropdown:

```javascript
var selected = Dropdown_Product.getSelectedKey();
Table_1.getDataSource().setDimensionFilter("Product", selected);
```

---

## 2. Metodos del widget Dropdown

| Metodo | Que hace |
|--------|----------|
| `addItem(key, text)` | Anade un item |
| `removeAllItems()` | Limpia todos los items |
| `getSelectedKey()` | Devuelve el key seleccionado |
| `getSelectedText()` | Devuelve el texto seleccionado |
| `setSelectedKey(key)` | Selecciona un item por key |
| `setVisible(bool)` | Muestra/oculta |
| `setEnabled(bool)` | Habilita/deshabilita |

---

## 3. Cascading Filters (filtros en cascada)

Patron clasico: al seleccionar en un dropdown "padre", se actualizan los items
del dropdown "hijo" para mostrar solo los valores validos.

Ejemplo: Publisher → Game Name (al elegir Publisher, solo aparecen sus juegos)

### onInitialization (poblar todos los dropdowns)

```javascript
var resultSet = Table_1.getDataSource().getResultSet();
var yearMembers = Table_1.getDataSource().getMembers("Year");

// Poblar dropdowns de dimensiones del result set
for (var i = 0; i < resultSet.length; i++) {
    Dropdown_Name.addItem(resultSet[i]["Name"].id, resultSet[i]["Name"].description);
    Dropdown_Publisher.addItem(resultSet[i]["Publisher"].id, resultSet[i]["Publisher"].description);
}

// Poblar dropdown de Year con getMembers
for (var z = 0; z < yearMembers.length; z++) {
    Dropdown_Year.addItem(yearMembers[z].id, yearMembers[z].description);
}
```

### onSelect del Dropdown padre (Publisher)

```javascript
// 1. Quitar filtros para obtener datos completos
Table_1.getDataSource().removeDimensionFilter("Name");
Table_1.getDataSource().removeDimensionFilter("Publisher");

// 2. Obtener result set sin filtros
var resultSet = Table_1.getDataSource().getResultSet();
var selectedPublisher = Dropdown_Publisher.getSelectedKey();

// 3. Limpiar dropdown hijo
Dropdown_Name.removeAllItems();

// 4. Repoblar con solo los items que pertenecen al publisher seleccionado
for (var i = 0; i < resultSet.length; i++) {
    if (resultSet[i]["Publisher"].id === selectedPublisher) {
        Dropdown_Name.addItem(resultSet[i]["Name"].id, resultSet[i]["Name"].description);
    }
}

// 5. Aplicar filtro a la tabla
Table_1.getDataSource().setDimensionFilter("Publisher", selectedPublisher);
```

### onSelect del Dropdown hijo (Name)

```javascript
var selectedName = Dropdown_Name.getSelectedKey();
Table_1.getDataSource().setDimensionFilter("Name", selectedName);
```

### Patron resumido del cascading

```
1. Quitar filtros de las dimensiones afectadas
2. Leer datos sin filtrar (getResultSet)
3. Limpiar dropdown hijo (removeAllItems)
4. Loop: si el item coincide con la seleccion padre → addItem al hijo
5. Aplicar filtro del padre al datasource
```

---

## 4. Filtro con multiples widgets (Linked Analysis)

Cuando un chart o tabla filtra a otros widgets al hacer clic:

### En el evento onSelect del Chart origen

```javascript
// Obtener la seleccion del usuario en el chart
var selections = Chart_Region.getSelections();

if (selections.length > 0) {
    var selectedRegion = selections[0]["Region"].id;

    // Aplicar como filtro a otros widgets
    Table_Detail.getDataSource().setDimensionFilter("Region", selectedRegion);
    Chart_Revenue.getDataSource().setDimensionFilter("Region", selectedRegion);
    Chart_Cost.getDataSource().setDimensionFilter("Region", selectedRegion);
}
```

### Patron reutilizable con ScriptObject

Crear un ScriptObject `Utils` con funcion `linkedFilter`:

```javascript
// En ScriptObject "Utils", funcion "linkedFilter"
// Parametros: sourceDimension (string), sourceValue (string)
function linkedFilter(sourceDimension, sourceValue) {
    // Lista de widgets a filtrar
    Table_Detail.getDataSource().setDimensionFilter(sourceDimension, sourceValue);
    Chart_Revenue.getDataSource().setDimensionFilter(sourceDimension, sourceValue);
    Chart_Cost.getDataSource().setDimensionFilter(sourceDimension, sourceValue);
    KPI_Total.getDataSource().setDimensionFilter(sourceDimension, sourceValue);
}
```

Llamar desde cualquier widget:

```javascript
// En onSelect del Chart
var selections = Chart_Region.getSelections();
if (selections.length > 0) {
    Utils.linkedFilter("Region", selections[0]["Region"].id);
}
```

---

## 5. Boton para limpiar todos los filtros

```javascript
// onClick del boton "Limpiar filtros"
var ds = Table_1.getDataSource();

ds.removeDimensionFilter("Account");
ds.removeDimensionFilter("Region");
ds.removeDimensionFilter("Product");
ds.removeDimensionFilter("Date");

// Resetear dropdowns a "Sin seleccion"
Dropdown_Account.setSelectedKey("");
Dropdown_Region.setSelectedKey("");
Dropdown_Product.setSelectedKey("");

Application.showMessage(ApplicationMessageType.Info, "Filtros eliminados");
```

---

## 6. Optimizacion: pausar refresco durante multiples filtros

Cuando aplicas muchos filtros seguidos, la tabla se refresca en cada uno.
Pausar el refresco evita renders intermedios:

```javascript
var ds = Table_1.getDataSource();

// Pausar
ds.setRefreshPaused(PauseMode.On);

// Aplicar todos los filtros sin render intermedio
ds.setDimensionFilter("Account", Dropdown_Account.getSelectedKey());
ds.setDimensionFilter("Region", Dropdown_Region.getSelectedKey());
ds.setDimensionFilter("Product", Dropdown_Product.getSelectedKey());
ds.setDimensionFilter("Date", Dropdown_Date.getSelectedKey());

// Reactivar (renderiza una sola vez con todos los filtros)
ds.setRefreshPaused(PauseMode.Off);
```

**Tip adicional:** En el Builder, configura los widgets como "Refresh active Widgets Only"
para que solo se refresquen los widgets visibles.
