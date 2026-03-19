# Data Actions Avanzado

## La API de DataAction

En Analytics Designer, un Data Action se anade al canvas como un objeto no visual.
La API expone los siguientes metodos (confirmados en API Reference 2025.14+):

| Metodo | Descripcion |
|--------|-------------|
| `execute()` | Ejecuta el Data Action de forma sincrona. Devuelve `DataActionExecutionResponse` |
| `executeInBackground()` | Ejecuta en background. Devuelve `DataActionBackgroundExecutionResponse` |
| `setParameterValue(name, value)` | Establece el valor de un parametro antes de ejecutar |
| `getParameterValue(name)` | Devuelve el valor actual de un parametro |
| `getExecutionProgress()` | Devuelve el progreso de una ejecucion en background |

### Tipos de respuesta

| Tipo | Propiedades |
|------|------------|
| `DataActionExecutionResponse` | `status` (string) |
| `DataActionBackgroundExecutionResponse` | `executionId`, `status` |

## Ejecucion basica

```javascript
// Ejecucion simple sin parametros
var result = DataAction_Calculate.execute();

if (result.status === "OK") {
    Application.showMessage(ApplicationMessageType.Success, "Calculo completado");
} else {
    Application.showMessage(ApplicationMessageType.Error, "Error: " + result.status);
}
```

## Parametros de Data Actions

Los parametros definidos en el Data Action se mapean a los `%PARAMETER_NAME%` usados
en las Advanced Formulas dentro del Data Action.

### setParameterValue - Antes de ejecutar

```javascript
// Establecer parametros
DataAction_CopyVersion.setParameterValue("SOURCE_VERSION", "Actual");
DataAction_CopyVersion.setParameterValue("TARGET_VERSION", "Plan");
DataAction_CopyVersion.setParameterValue("YEAR", "2026");

// Ejecutar con los parametros establecidos
var result = DataAction_CopyVersion.execute();
```

### Parametros desde Dropdowns

```javascript
// En onClick de Button_Execute
var sourceVersion = Dropdown_Source.getSelectedKey();
var targetVersion = Dropdown_Target.getSelectedKey();
var selectedYear = Dropdown_Year.getSelectedKey();

DataAction_CopyVersion.setParameterValue("SOURCE_VERSION", sourceVersion);
DataAction_CopyVersion.setParameterValue("TARGET_VERSION", targetVersion);
DataAction_CopyVersion.setParameterValue("YEAR", selectedYear);

Application.showBusyIndicator("Ejecutando copia de version...");
var result = DataAction_CopyVersion.execute();
Application.hideBusyIndicator();

if (result.status === "OK") {
    Application.showMessage(ApplicationMessageType.Success, "Copia completada");
    // Refrescar tabla para ver datos nuevos
    Table_Plan.getDataSource().refreshData();
}
```

### Parametros con jerarquias (Date dimension)

Cuando un parametro es de tipo fecha y la dimension tiene jerarquia, debes especificar
la jerarquia en el valor:

```javascript
// Formato: [Dimension].[Jerarquia].&[Miembro]
DataAction_Forecast.setParameterValue("PERIOD", "[Date].[YQM].&[202601]");

// Para dimensiones sin jerarquia, basta con el ID del miembro
DataAction_Forecast.setParameterValue("REGION", "EMEA");
```

> **Pitfall critico:** La dimension Date en SAC tiene jerarquia por defecto (Year > Quarter > Month).
> Si pasas solo `"202601"` sin la jerarquia, el parametro puede no resolverse correctamente.
> Incluir siempre el formato `[Dimension].[Jerarquia].&[Miembro]` para fechas.

### getParameterValue - Verificar valores

```javascript
// Verificar que los parametros estan correctos antes de ejecutar
var source = DataAction_Copy.getParameterValue("SOURCE_VERSION");
var target = DataAction_Copy.getParameterValue("TARGET_VERSION");

console.log("Source: " + source + ", Target: " + target);

if (source === target) {
    Application.showMessage(
        ApplicationMessageType.Warning,
        "Version origen y destino no pueden ser la misma"
    );
    return;
}

DataAction_Copy.execute();
```

## Ejecucion en background

Para Data Actions de larga duracion, puedes ejecutar en background para no bloquear la UI:

```javascript
// Ejecutar en background
var bgResult = DataAction_HeavyCalc.executeInBackground();
var executionId = bgResult.executionId;

Application.showMessage(
    ApplicationMessageType.Info,
    "Data Action iniciada en background. ID: " + executionId
);
```

### Monitorear progreso con Timer

```javascript
// En onInitialization o en el boton que dispara la ejecucion
ScriptVariable_ExecutionId = "";

// En onClick de Button_Execute
var bgResult = DataAction_Heavy.executeInBackground();
ScriptVariable_ExecutionId = bgResult.executionId;
Application.setTimer(3000); // Verificar cada 3 segundos

// En onTimer de Application
if (ScriptVariable_ExecutionId !== "") {
    var progress = DataAction_Heavy.getExecutionProgress();

    if (progress.status === "COMPLETED") {
        Application.clearTimer();
        ScriptVariable_ExecutionId = "";
        Application.showMessage(ApplicationMessageType.Success, "Data Action completada");
        Table_Plan.getDataSource().refreshData();
    } else if (progress.status === "FAILED") {
        Application.clearTimer();
        ScriptVariable_ExecutionId = "";
        Application.showMessage(ApplicationMessageType.Error, "Data Action fallo");
    }
    // Si status es "RUNNING", el timer se vuelve a disparar
}
```

> **Conocimiento comunitario:** Los valores exactos de `status` en las respuestas
> no estan completamente documentados por SAP. "OK" es el valor estandar para ejecucion
> exitosa sincrona. Para background, los valores comunes son "COMPLETED", "RUNNING", "FAILED",
> pero verifica en tu version especifica.

## Encadenar multiples Data Actions

### Secuencial simple

```javascript
// Ejecutar tres Data Actions en secuencia
Application.showBusyIndicator("Paso 1: Validando datos...");
var r1 = DataAction_Validate.execute();

if (r1.status !== "OK") {
    Application.hideBusyIndicator();
    Application.showMessage(ApplicationMessageType.Error, "Validacion fallo");
    return;
}

Application.showBusyIndicator("Paso 2: Calculando forecast...");
var r2 = DataAction_Forecast.execute();

if (r2.status !== "OK") {
    Application.hideBusyIndicator();
    Application.showMessage(ApplicationMessageType.Error, "Forecast fallo");
    return;
}

Application.showBusyIndicator("Paso 3: Publicando...");
var r3 = DataAction_Publish.execute();
Application.hideBusyIndicator();

if (r3.status === "OK") {
    Application.showMessage(ApplicationMessageType.Success, "Pipeline completado");
    Table_Plan.getDataSource().refreshData();
} else {
    Application.showMessage(ApplicationMessageType.Error, "Publicacion fallo");
}
```

### Secuencial con parametros compartidos

```javascript
// Mismos parametros para multiples Data Actions
var year = Dropdown_Year.getSelectedKey();
var version = Dropdown_Version.getSelectedKey();

// Configurar todos los Data Actions
var actions = [DataAction_Step1, DataAction_Step2, DataAction_Step3];
var stepNames = ["Limpieza", "Calculo", "Agregacion"];

for (var i = 0; i < actions.length; i++) {
    actions[i].setParameterValue("YEAR", year);
    actions[i].setParameterValue("VERSION", version);
}

// Ejecutar en secuencia
var allOk = true;
for (var j = 0; j < actions.length; j++) {
    Application.showBusyIndicator("Paso " + (j + 1).toString() + ": " + stepNames[j]);
    var result = actions[j].execute();

    if (result.status !== "OK") {
        Application.hideBusyIndicator();
        Application.showMessage(
            ApplicationMessageType.Error,
            "Error en paso " + (j + 1).toString() + ": " + stepNames[j]
        );
        allOk = false;
        break;
    }
}

if (allOk) {
    Application.hideBusyIndicator();
    Application.showMessage(ApplicationMessageType.Success, "Todos los pasos completados");
}
```

## Pasar contexto de filtro a Data Actions

### Desde filtros de tabla

```javascript
// Obtener el filtro actual de la tabla y pasarlo como parametro
var ds = Table_Plan.getDataSource();

// Obtener el filtro de Region
var regionFilter = ds.getDimensionFilters("Region");
if (regionFilter.length > 0) {
    DataAction_Calculate.setParameterValue("REGION", regionFilter[0]);
}

DataAction_Calculate.execute();
```

### Desde seleccion de usuario

```javascript
// Usar la seleccion de tabla como contexto del Data Action
var sel = Table_Plan.getSelections();
if (sel.length > 0) {
    var region = sel[0]["Region"];
    var period = sel[0]["Date"];

    DataAction_DrillDown.setParameterValue("REGION", region);
    DataAction_DrillDown.setParameterValue("PERIOD", period);
    DataAction_DrillDown.execute();
}
```

## Disparar Data Actions desde eventos

### Desde onSelect de Chart

```javascript
// En onSelect de Chart_Overview
var sel = Chart_Overview.getSelections();
if (sel.length > 0) {
    var product = sel[0]["Product"];
    DataAction_ProductDetail.setParameterValue("PRODUCT", product);
    DataAction_ProductDetail.execute();
    Table_Detail.getDataSource().refreshData();
}
```

### Desde onChange de Input Field

```javascript
// En onChange de InputField_Threshold
var threshold = InputField_Threshold.getValue();
DataAction_ApplyThreshold.setParameterValue("THRESHOLD", threshold);
// No ejecutar inmediatamente - esperar a que el usuario confirme
```

### Desde onResultChanged (auto-ejecutar)

```javascript
// En onResultChanged de Table_Plan
// CUIDADO: esto se ejecuta cada vez que los datos cambian
// Usar un flag para evitar bucles infinitos

if (ScriptVariable_AutoCalcEnabled && !ScriptVariable_IsCalculating) {
    ScriptVariable_IsCalculating = true;
    DataAction_AutoCalc.execute();
    ScriptVariable_IsCalculating = false;
}
```

> **Pitfall:** Ejecutar Data Actions en `onResultChanged` puede causar bucles infinitos
> si el Data Action modifica los mismos datos que disparan el evento. Siempre usa un
> flag de control.

## Popup de confirmacion antes de ejecutar

```javascript
// Patron completo con popup de confirmacion
// En onClick de Button_Execute
Popup_Confirm.open();

// En el boton "Confirmar" dentro del popup
Popup_Confirm.close();
Application.showBusyIndicator("Ejecutando...");

DataAction_Critical.setParameterValue("VERSION", Dropdown_Version.getSelectedKey());
var result = DataAction_Critical.execute();

Application.hideBusyIndicator();

if (result.status === "OK") {
    Application.showMessage(ApplicationMessageType.Success, "Completado");
    Table_Plan.getDataSource().refreshData();
} else {
    Application.showMessage(ApplicationMessageType.Error, "Error en la ejecucion");
}
```

## Pitfalls comunes

1. **setParameterValue despues de execute:** Los parametros deben establecerse ANTES de llamar a `execute()`. Si los estableces despues, solo aplican a la siguiente ejecucion.

2. **Parametro no definido en el Data Action:** Si llamas `setParameterValue()` con un nombre de parametro que no existe en el Data Action, no da error pero el valor se ignora.

3. **execute() es sincrono y bloqueante:** `execute()` bloquea la UI hasta que termina. Para Data Actions largos, usa `executeInBackground()`.

4. **Refresh de datos post-ejecucion:** `execute()` no refresca automaticamente los widgets. Debes llamar `getDataSource().refreshData()` explicitamente en las tablas/charts afectados.

5. **Timeout en Data Actions largas:** Si el Data Action tarda mas de ~5 minutos, puede fallar por timeout del servidor. Usa `executeInBackground()` para procesos largos.

6. **Formato de parametros con jerarquia:** No olvides el formato `[Dim].[Hierarchy].&[Member]` para dimensiones con jerarquia activa, especialmente Date.
