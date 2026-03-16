# SAC DataSource API - Acceso y Manipulacion de Datos

## Obtener el DataSource

Cada widget con datos (Table, Chart, etc.) tiene un DataSource asociado:

```javascript
var ds = Table_1.getDataSource();
```

**Buena practica:** guardar en variable si lo vas a usar varias veces (mejor rendimiento):

```javascript
// MAL: llamar getDataSource() en cada iteracion
for (var i = 0; i < items.length; i++) {
    Table_1.getDataSource().setDimensionFilter("Dim", items[i]);
}

// BIEN: guardar en variable
var ds = Table_1.getDataSource();
for (var i = 0; i < items.length; i++) {
    ds.setDimensionFilter("Dim", items[i]);
}
```

---

## 1. Filtros de dimensiones

### setDimensionFilter - Aplicar filtro

```javascript
// Filtrar por un miembro
ds.setDimensionFilter("Account", "Revenue");

// Filtrar por varios miembros (array)
ds.setDimensionFilter("Account", ["Revenue", "Cost", "Profit"]);

// Filtrar por rango de tiempo (TimeRange)
ds.setDimensionFilter("Date", {
    start: "202601",
    end: "202612"
});
```

### removeDimensionFilter - Quitar filtro

```javascript
// Quitar filtro de una dimension
ds.removeDimensionFilter("Account");

// Quitar filtros de varias dimensiones
ds.removeDimensionFilter("Account");
ds.removeDimensionFilter("Region");
ds.removeDimensionFilter("Date");
```

### getDimensionFilters - Consultar filtros activos

```javascript
var filters = ds.getDimensionFilters("Account");
```

### copyDimensionFilterFrom - Copiar filtros entre DataSources

```javascript
// Copiar todos los filtros de otro datasource
ds.copyDimensionFilterFrom(Chart_1.getDataSource());

// Copiar solo el filtro de una dimension
ds.copyDimensionFilterFrom(Chart_1.getDataSource(), "Region");
```

---

## 2. Leer datos

### getResultSet - Obtener filas completas

Devuelve un array de objetos, cada uno con todas las dimensiones de la fila:

```javascript
var resultSet = ds.getResultSet();

// Cada fila tiene las dimensiones como propiedades
for (var i = 0; i < resultSet.length; i++) {
    var row = resultSet[i];
    var accountId = row["Account"].id;
    var accountDesc = row["Account"].description;
    var region = row["Region"].id;
}
```

### getData - Obtener valor de una celda

```javascript
// Valor por seleccion de propiedades
var value = ds.getData("Account", "Revenue");
```

### getDataSelections - Obtener selecciones

```javascript
// Sin parametros: todas las selecciones
var selections = ds.getDataSelections();

// Con filtro, offset y limite
var selections = ds.getDataSelections("Account", 0, 10);
```

### isResultEmpty - Verificar si hay datos

```javascript
if (ds.isResultEmpty()) {
    Application.showMessage(ApplicationMessageType.Warning, "No hay datos para mostrar");
}
```

---

## 3. Dimensiones y miembros

### getDimensions - Listar dimensiones

```javascript
var dimensions = ds.getDimensions();
// Devuelve array con todas las dimensiones (filas, columnas, filtros)
```

### getDimensionProperties - Propiedades de una dimension

```javascript
// Solo disponible en widgets de tabla
var properties = ds.getDimensionProperties("Product");
```

### getMembers - Obtener miembros de una dimension

```javascript
// Todos los miembros
var members = ds.getMembers("Region");

for (var i = 0; i < members.length; i++) {
    var id = members[i].id;
    var desc = members[i].description;
}
```

### getMember - Obtener un miembro especifico

```javascript
// Jerarquia plana
var member = ds.getMember("Entity", "E1000");
// Devuelve: {id: "E1000", description: "InsightCubes UAE", dimensionId: "Entity", ...}

// Jerarquia activa (notacion de nodo)
var member = ds.getMember("Entity", "[ENTITY].[parentId].&[E1000]");
```

### getResultMember - Miembro desde resultado

```javascript
// Obtener miembro a partir de una seleccion en el resultado
var member = ds.getResultMember("Region", selection);
```

### setMemberDisplayMode / getMemberDisplayMode

```javascript
// Mostrar solo ID, solo descripcion, o ambos
ds.setMemberDisplayMode("Account", MemberDisplayMode.Key);
ds.setMemberDisplayMode("Account", MemberDisplayMode.Text);
ds.setMemberDisplayMode("Account", MemberDisplayMode.KeyAndText);

var mode = ds.getMemberDisplayMode("Account");
```

---

## 4. Jerarquias

```javascript
// Listar jerarquias disponibles para una dimension
var hierarchies = ds.getHierarchies("Region");

// Obtener jerarquia activa
var activeHierarchy = ds.getHierarchy("Region");

// Cambiar jerarquia activa
ds.setHierarchy("Region", "Geographic");

// Nivel de jerarquia
var level = ds.getHierarchyLevel("Region");
ds.setHierarchyLevel("Region", 2);

// Expandir/colapsar nodo
ds.expandNode("Region", selection);
ds.collapseNode("Region", selection);
```

---

## 5. Variables de modelo

```javascript
// Listar todas las variables
var vars = ds.getVariables();

// Obtener valores de una variable
var values = ds.getVariableValues("VAR_YEAR");

// Establecer valor de una variable (string simple)
ds.setVariableValue("VAR_YEAR", "2026");

// Establecer valor con objeto
ds.setVariableValue("VAR_REGION", {value: "EMEA"});

// Excluir un valor
ds.setVariableValue("VAR_REGION", {exclude: true, value: "APAC"});

// Quitar valor de variable
ds.removeVariableValue("VAR_YEAR");

// Copiar variable de otro datasource
ds.copyVariableValueFrom(Chart_1.getDataSource(), "VAR_YEAR");

// Abrir dialogo de prompt de variables
ds.openPromptDialog();
```

---

## 6. Control de refresco

```javascript
// Refrescar datos
ds.refreshData();

// Pausar refresco (util cuando haces muchos cambios de filtro)
ds.setRefreshPaused(PauseMode.On);

// ... aplicar muchos filtros sin que se refresque cada vez ...
ds.setDimensionFilter("Account", "Revenue");
ds.setDimensionFilter("Region", "EMEA");
ds.setDimensionFilter("Date", "202601");

// Reactivar refresco
ds.setRefreshPaused(PauseMode.Off);

// Modo auto: solo refresca widgets activos/visibles
ds.setRefreshPaused(PauseMode.Auto);

// Comprobar si esta pausado
var isPaused = ds.getRefreshPaused();
```

---

## 7. Medidas

```javascript
// Obtener todas las medidas del datasource
var measures = ds.getMeasures();
```

---

## Resumen de metodos (36 en total)

| Categoria | Metodo | Que hace |
|-----------|--------|----------|
| **Filtros** | `setDimensionFilter(dim, value)` | Aplica filtro |
| | `removeDimensionFilter(dim)` | Quita filtro |
| | `getDimensionFilters(dim)` | Consulta filtros activos |
| | `copyDimensionFilterFrom(ds, dim?)` | Copia filtros de otro DS |
| **Datos** | `getResultSet()` | Todas las filas con dimensiones |
| | `getData(selection)` | Valor de celda |
| | `getDataSelections(sel?, offset?, limit?)` | Selecciones de datos |
| | `isResultEmpty()` | Verifica si hay datos |
| | `getDataExplorer()` | Explorador de datos |
| | `getInfo()` | Info del datasource |
| **Dimensiones** | `getDimensions()` | Lista dimensiones |
| | `getDimensionProperties(dim)` | Propiedades de dimension |
| | `getMembers(dim, options?)` | Miembros de dimension |
| | `getMember(dim, id, hier?)` | Un miembro especifico |
| | `getResultMember(dim, selection)` | Miembro de resultado |
| | `getMeasures()` | Lista medidas |
| | `setMemberDisplayMode(dim, mode)` | Modo visualizacion |
| | `getMemberDisplayMode(dim)` | Consulta modo |
| **Jerarquias** | `getHierarchies(dim)` | Lista jerarquias |
| | `getHierarchy(dim)` | Jerarquia activa |
| | `setHierarchy(dim, name)` | Cambia jerarquia |
| | `getHierarchyLevel(dim)` | Nivel actual |
| | `setHierarchyLevel(dim, level?)` | Cambia nivel |
| | `expandNode(dim, selection)` | Expande nodo |
| | `collapseNode(dim, selection)` | Colapsa nodo |
| **Variables** | `getVariables()` | Lista variables |
| | `getVariableValues(name)` | Valores de variable |
| | `setVariableValue(name, value)` | Establece valor |
| | `removeVariableValue(name)` | Quita valor |
| | `copyVariableValueFrom(ds, name?)` | Copia variable |
| | `openPromptDialog()` | Dialogo de prompt |
| **Refresco** | `refreshData()` | Refresca datos |
| | `setRefreshPaused(mode)` | Pausa/reanuda refresco |
| | `getRefreshPaused()` | Estado de pausa |
| **Planning** | `getPlanning()` | Objeto Planning (ver 02-VERSION-MANAGEMENT) |
