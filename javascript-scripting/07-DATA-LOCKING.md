# SAC Data Locking via Scripting

## Concepto

Data Locking permite **proteger areas de datos** para que no se modifiquen durante
el proceso de planning. Los usuarios solo pueden editar celdas que esten "abiertas".

Es fundamental para workflows de planning donde distintos equipos trabajan en
distintas regiones/periodos y necesitas controlar quien puede editar que.

---

## Estados de bloqueo

| Estado | Significado |
|--------|------------|
| **Open** | Desbloqueado. Cualquier usuario con permisos puede editar |
| **Restricted** | Solo el propietario del lock puede editar |
| **Locked** | Nadie puede editar (ni el propietario) |

---

## Configuracion previa (en el modelo)

1. En el modelo de Planning → Settings → **Data Locking**
2. Activar Data Locking
3. Definir **Driving Dimensions**: las dimensiones que determinan la granularidad del bloqueo
   (ej: Version + Region + Date → puedes bloquear "Plan / EMEA / Q1" independientemente)

---

## Acceder al objeto DataLocking

```javascript
var locking = Table_1.getPlanning().getDataLocking();
```

Si el modelo no tiene data locking habilitado, devuelve `undefined`:

```javascript
var locking = Table_1.getPlanning().getDataLocking();
if (locking === undefined) {
    Application.showMessage(ApplicationMessageType.Warning, "Data Locking no esta habilitado");
}
```

---

## Metodos principales

### getState - Consultar estado de bloqueo

```javascript
// Consultar el estado de una combinacion de dimensiones
var state = locking.getState({
    "Version": "Plan",
    "Region": "EMEA",
    "Date": "2026Q1"
});

// state puede ser: "OPEN", "RESTRICTED", "LOCKED"
if (state === "LOCKED") {
    Application.showMessage(ApplicationMessageType.Warning, "Esta area esta bloqueada");
}
```

### setState - Cambiar estado de bloqueo

```javascript
// Bloquear una combinacion
locking.setState({
    "Version": "Plan",
    "Region": "EMEA",
    "Date": "2026Q1"
}, "LOCKED");

// Desbloquear
locking.setState({
    "Version": "Plan",
    "Region": "EMEA",
    "Date": "2026Q1"
}, "OPEN");

// Restringir (solo el propietario puede editar)
locking.setState({
    "Version": "Plan",
    "Region": "EMEA",
    "Date": "2026Q1"
}, "RESTRICTED");
```

---

## Ejemplos practicos

### Boton "Bloquear Q1 para EMEA"

```javascript
// onClick
var locking = Table_1.getPlanning().getDataLocking();

if (locking !== undefined) {
    locking.setState({
        "Version": "Plan",
        "Region": "EMEA",
        "Date": "2026Q1"
    }, "LOCKED");

    Application.showMessage(ApplicationMessageType.Success, "Q1 EMEA bloqueado");
} else {
    Application.showMessage(ApplicationMessageType.Error, "Data Locking no habilitado");
}
```

### Boton "Bloquear todas las regiones de un trimestre"

```javascript
// onClick
var locking = Table_1.getPlanning().getDataLocking();
var regions = ["EMEA", "APAC", "AMER", "LATAM"];
var quarter = Dropdown_Quarter.getSelectedKey();

Application.showBusyIndicator();

for (var i = 0; i < regions.length; i++) {
    locking.setState({
        "Version": "Plan",
        "Region": regions[i],
        "Date": quarter
    }, "LOCKED");
}

Application.hideBusyIndicator();
Application.showMessage(ApplicationMessageType.Success,
    "Todas las regiones bloqueadas para " + quarter);
```

### Verificar antes de ejecutar Data Action

```javascript
// onClick del boton "Ejecutar Forecast"
var locking = Table_1.getPlanning().getDataLocking();
var targetRegion = Dropdown_Region.getSelectedKey();

var state = locking.getState({
    "Version": "Forecast",
    "Region": targetRegion
});

if (state === "LOCKED") {
    Application.showMessage(ApplicationMessageType.Error,
        "La region " + targetRegion + " esta bloqueada. No se puede ejecutar el forecast.");
} else {
    DataAction_Forecast.execute({
        "REGION": targetRegion
    });
    Table_1.getDataSource().refreshData();
    Application.showMessage(ApplicationMessageType.Success, "Forecast ejecutado");
}
```

### Bloquear multiples versiones

```javascript
// Bloquear una region en todas las versiones publicas
var locking = Table_1.getPlanning().getDataLocking();
var versions = Table_1.getPlanning().getPublicVersions();

for (var i = 0; i < versions.length; i++) {
    locking.setState({
        "Version": versions[i].getId(),
        "Region": "EMEA",
        "Date": "2026Q4"
    }, "LOCKED");
}
```

---

## Resumen

| Metodo | Que hace |
|--------|----------|
| `getDataLocking()` | Obtiene el objeto DataLocking (undefined si no habilitado) |
| `getState(selection)` | Consulta estado: "OPEN", "RESTRICTED", "LOCKED" |
| `setState(selection, state)` | Cambia estado de bloqueo |
