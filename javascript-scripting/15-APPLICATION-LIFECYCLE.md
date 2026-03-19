# Application Lifecycle Events

## Eventos del ciclo de vida

### Application-level events

| Evento | Cuando se dispara |
|--------|------------------|
| `onInitialization` | Una sola vez al cargar la aplicacion |
| `onTimer` | Periodicamente segun el intervalo configurado con `setTimer()` |

### Page-level events (Optimized Stories)

| Evento | Cuando se dispara |
|--------|------------------|
| `onInitialization` | Una sola vez al cargar la pagina por primera vez |
| `onActive` | Cada vez que la pagina se activa (incluye la primera vez, despues de onInitialization) |

> **Diferencia clave:** En Analytic Applications clasicas, `onInitialization` esta en el
> objeto Application. En Optimized Stories / Stories 2.0, `onInitialization` y `onActive`
> estan en el objeto `ApplicationPage`.

## onInitialization - Buenas practicas

### Regla de oro: Configura en design time, no en script

El error mas comun es usar `onInitialization` para configurar estados que podrian
configurarse en design time:

```javascript
// MAL: Esto causa un refresh extra innecesario
// En onInitialization
Table_1.getDataSource().setDimensionFilter("Year", "2026");
Chart_1.getDataSource().setDimensionFilter("Year", "2026");

// BIEN: Configura el filtro "Year = 2026" en design time
// desde el panel de propiedades del DataSource
// onInitialization queda vacio o con logica que SI necesita ser dinamica
```

### Cuando SI usar onInitialization

1. **Configuracion que depende del usuario actual:**

```javascript
// En onInitialization
var userInfo = Application.getUserInfo();
Text_Welcome.setText("Bienvenido, " + userInfo.displayName);

// Filtrar por la region del usuario (almacenada en un mapeo)
var userRegion = ScriptObject_UserMapping.getRegionForUser(userInfo.id);
if (userRegion !== "") {
    Table_1.getDataSource().setDimensionFilter("Region", userRegion);
}
```

2. **Configuracion que depende de URL parameters:**

```javascript
// En onInitialization
// Si la app se abrio con parametros de URL
if (ScriptVariable_Region !== "") {
    Table_1.getDataSource().setDimensionFilter("Region", ScriptVariable_Region);
    Dropdown_Region.setSelectedKey(ScriptVariable_Region);
}
```

3. **Poblar dropdowns con datos dinamicos:**

```javascript
// En onInitialization
var ds = Table_1.getDataSource();
var years = ds.getMembers("Year");

Dropdown_Year.removeAllItems();
for (var i = 0; i < years.length; i++) {
    Dropdown_Year.addItem(years[i].id, years[i].description);
}

// Seleccionar el anio actual por defecto
Dropdown_Year.setSelectedKey("2026");
```

## Pause Refresh API - Optimizar onInitialization

Si necesitas hacer multiples cambios en onInitialization, usa Pause Refresh para evitar
que cada cambio dispare un roundtrip al backend:

```javascript
// En onInitialization
// Pausar refreshes de todos los widgets
var ds1 = Table_1.getDataSource();
var ds2 = Chart_1.getDataSource();

ds1.setRefreshPaused(true);
ds2.setRefreshPaused(true);

// Hacer multiples cambios sin roundtrips intermedios
ds1.setDimensionFilter("Year", "2026");
ds1.setDimensionFilter("Version", "Plan");
ds2.setDimensionFilter("Year", "2026");
ds2.setDimensionFilter("Version", "Plan");

// Reanudar refreshes - ahora se hace UN solo roundtrip
ds1.setRefreshPaused(false);
ds2.setRefreshPaused(false);
```

> **Documentado por SAP:** Esta es la practica recomendada oficialmente para optimizar
> el rendimiento de aplicaciones que necesitan configurar multiples filtros en la inicializacion.

### Pausar widgets invisibles

```javascript
// En onInitialization
// Si tienes tabs con widgets que no se ven inicialmente,
// pausa su refresh hasta que sean visibles

var dsTab2 = Table_Tab2.getDataSource();
dsTab2.setRefreshPaused(true);
// Table_Tab2 no carga datos hasta que se haga setRefreshPaused(false)

// En el evento del tab switch
dsTab2.setRefreshPaused(false);
// Ahora si carga datos
```

## Timer API - Patrones avanzados

### Configurar y usar el Timer

```javascript
// Iniciar timer (intervalo en milisegundos)
Application.setTimer(5000); // Cada 5 segundos

// Detener timer
Application.clearTimer();
```

### onTimer event

El evento `onTimer` se ejecuta cada vez que el intervalo configurado se cumple.

```javascript
// En onTimer de Application
// Refrescar datos automaticamente
Table_Live.getDataSource().refreshData();
```

### Patron: Auto-refresh con control de usuario

```javascript
// En onInitialization
ScriptVariable_AutoRefresh = false;

// En onClick de Toggle_AutoRefresh
ScriptVariable_AutoRefresh = !ScriptVariable_AutoRefresh;
if (ScriptVariable_AutoRefresh) {
    Application.setTimer(30000); // Refresh cada 30 segundos
    Toggle_AutoRefresh.setText("Auto-refresh: ON");
} else {
    Application.clearTimer();
    Toggle_AutoRefresh.setText("Auto-refresh: OFF");
}

// En onTimer
if (ScriptVariable_AutoRefresh) {
    Table_Live.getDataSource().refreshData();
    Chart_Live.getDataSource().refreshData();
    Text_LastUpdate.setText("Ultima actualizacion: " + ConvertUtils.dateToString(new Date()));
}
```

### Patron: Countdown / temporizador visual

```javascript
// En onInitialization
ScriptVariable_SecondsLeft = 60;
Application.setTimer(1000); // Cada segundo

// En onTimer
ScriptVariable_SecondsLeft = ScriptVariable_SecondsLeft - 1;
Text_Countdown.setText("Tiempo restante: " + ScriptVariable_SecondsLeft.toString() + "s");

if (ScriptVariable_SecondsLeft <= 0) {
    Application.clearTimer();
    // Ejecutar accion al llegar a 0
    Application.showMessage(ApplicationMessageType.Warning, "Tiempo agotado");
    DataAction_AutoSave.execute();
}
```

### Patron: Monitorear ejecucion en background

```javascript
// En onClick de Button_RunHeavyProcess
var bgResult = DataAction_Heavy.executeInBackground();
ScriptVariable_ExecutionId = bgResult.executionId;
ScriptVariable_PollCount = 0;
Application.setTimer(3000); // Verificar cada 3s
Application.showBusyIndicator("Procesando...");

// En onTimer
if (ScriptVariable_ExecutionId !== "") {
    ScriptVariable_PollCount = ScriptVariable_PollCount + 1;

    var progress = DataAction_Heavy.getExecutionProgress();

    if (progress.status === "COMPLETED") {
        Application.clearTimer();
        Application.hideBusyIndicator();
        ScriptVariable_ExecutionId = "";
        Application.showMessage(ApplicationMessageType.Success, "Proceso completado");
        Table_Result.getDataSource().refreshData();
    } else if (progress.status === "FAILED") {
        Application.clearTimer();
        Application.hideBusyIndicator();
        ScriptVariable_ExecutionId = "";
        Application.showMessage(ApplicationMessageType.Error, "Proceso fallido");
    } else if (ScriptVariable_PollCount > 60) {
        // Timeout: 60 * 3s = 3 minutos
        Application.clearTimer();
        Application.hideBusyIndicator();
        ScriptVariable_ExecutionId = "";
        Application.showMessage(ApplicationMessageType.Warning, "Timeout esperando resultado");
    }
}
```

## Gestion de estado de la aplicacion

SAC no tiene un "state manager" como React o similar. El estado se gestiona con **Script Variables**.

### Patron: Estado centralizado en ScriptObject

```javascript
// ScriptObject_AppState

// Variables de estado (declaradas como Script Variables en el app)
// ScriptVariable_CurrentPage: string
// ScriptVariable_SelectedRegion: string
// ScriptVariable_SelectedYear: string
// ScriptVariable_IsEditMode: boolean
// ScriptVariable_IsDirty: boolean

function initState() {
    ScriptVariable_CurrentPage = "Overview";
    ScriptVariable_SelectedRegion = "";
    ScriptVariable_SelectedYear = "2026";
    ScriptVariable_IsEditMode = false;
    ScriptVariable_IsDirty = false;
}

function setRegion(region) {
    ScriptVariable_SelectedRegion = region;
    // Aplicar a todos los widgets relevantes
    Table_Main.getDataSource().setDimensionFilter("Region", region);
    Chart_Main.getDataSource().setDimensionFilter("Region", region);
}

function setYear(year) {
    ScriptVariable_SelectedYear = year;
    Table_Main.getDataSource().setDimensionFilter("Year", year);
    Chart_Main.getDataSource().setDimensionFilter("Year", year);
}

function setDirty(dirty) {
    ScriptVariable_IsDirty = dirty;
    Button_Save.setEnabled(dirty);
    Button_Discard.setEnabled(dirty);
}
```

```javascript
// En onInitialization
ScriptObject_AppState.initState();

// En onSelect de Dropdown_Region
ScriptObject_AppState.setRegion(Dropdown_Region.getSelectedKey());
```

### Paso de estado entre paginas (Optimized Stories)

En Stories 2.0, usa Script Variables compartidas para pasar contexto entre paginas:

```javascript
// En Page_1 - onSelect de Table_Summary
var sel = Table_Summary.getSelections();
if (sel.length > 0) {
    // Guardar contexto en Script Variable (compartida entre paginas)
    ScriptVariable_SelectedRegion = sel[0]["Region"];
    // Navegar a pagina de detalle
    Application.openPage("Page_Detail");
}

// En Page_Detail - onActive
if (ScriptVariable_SelectedRegion !== "") {
    Table_Detail.getDataSource().setDimensionFilter("Region", ScriptVariable_SelectedRegion);
    Text_Title.setText("Detalle de: " + ScriptVariable_SelectedRegion);
}
```

> **Documentado por SAP:** Para cross-page interaction, SAP recomienda pasar contexto
> via Script Variables compartidas, no referenciando widgets de otras paginas directamente.

## Gestion de sesion

> **Limitacion:** SAC no expone APIs de gestion de sesion en el scripting.
> No puedes detectar timeout de sesion, refrescar la sesion, ni controlar la
> autenticacion desde scripts.
>
> Lo que SI puedes hacer:
> - Detectar al usuario actual con `Application.getUserInfo()`
> - Detectar roles con `Application.getRolesInfo()`
> - Detectar equipos con `Application.getTeamsInfo()`

```javascript
// En onInitialization
var user = Application.getUserInfo();
var roles = Application.getRolesInfo();

console.log("Usuario: " + user.id + " (" + user.displayName + ")");
console.log("Roles: " + roles.length.toString());

for (var i = 0; i < roles.length; i++) {
    console.log("  Rol: " + roles[i]);
}
```

## Pitfalls comunes

1. **onInitialization se ejecuta despues del render inicial:** Los widgets ya se han renderizado con su configuracion de design time. Cualquier cambio en onInitialization causa un segundo render (roundtrip adicional).

2. **setTimer sin clearTimer:** Si olvidas `clearTimer()`, el timer sigue ejecutandose indefinidamente. Esto puede causar problemas de rendimiento si el onTimer hace refreshData.

3. **onActive vs onInitialization:** En Stories 2.0, `onActive` se ejecuta CADA VEZ que navegas a la pagina. `onInitialization` solo la primera vez. Si pones logica de inicializacion en `onActive`, se ejecutara repetidamente.

4. **Widgets invisibles en onInitialization:** Si un widget esta en un container invisible y lo referencias en onInitialization, puede funcionar pero SAP recomienda que sea hijo directo del Canvas para evitar problemas.

5. **Orden de ejecucion no garantizado:** Si multiples widgets tienen `onResultChanged`, el orden de ejecucion entre ellos no esta garantizado. No asumas que uno se ejecuta antes que otro.
