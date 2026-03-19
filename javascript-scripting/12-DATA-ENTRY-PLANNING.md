# Data Entry y Planning Input

## La Planning API de Table

Cuando una tabla esta vinculada a un modelo con planning habilitado, puedes acceder a la Planning API:

```javascript
var planning = Table_Plan.getPlanning();
// Si el modelo no soporta planning, devuelve undefined
```

## Metodos del objeto Planning

El objeto devuelto por `getPlanning()` expone los siguientes metodos (confirmados en API Reference 2025.14+):

| Metodo | Descripcion |
|--------|-------------|
| `setUserInput(selection, value)` | Escribe un valor en una celda especifica |
| `submitData()` | Envia todos los cambios pendientes al backend |
| `isDataEntryEnabled()` | Devuelve si la entrada de datos esta habilitada |
| `isDataLockingEnabled()` | Devuelve si el data locking esta activo |
| `getPrivateVersion()` | Devuelve la version privada actual |
| `getPrivateVersions()` | Devuelve todas las versiones privadas |
| `getPublicVersion(versionName)` | Devuelve una version publica por nombre |

## setUserInput - Escribir datos en celdas

`setUserInput(selection, value)` es el metodo principal para escribir datos de vuelta al modelo.

### Parametros

- **selection**: Un objeto `Selection` obtenido de `getSelections()`. Debe ser una celda individual de la tabla.
- **value**: Un string con el valor numerico a escribir. Formateado segun las configuraciones regionales del usuario.

### Retorno

Devuelve `true` si la operacion fue exitosa, `false` si fallo.

### Ejemplo basico

```javascript
// En un boton "Aplicar valor"
var selections = Table_Plan.getSelections();
var planning = Table_Plan.getPlanning();

if (selections.length > 0 && planning !== undefined) {
    var targetCell = selections[0];
    var newValue = InputField_Value.getValue();

    var success = planning.setUserInput(targetCell, newValue);

    if (success) {
        // Enviar al backend
        planning.submitData();
        Application.showMessage(
            ApplicationMessageType.Success,
            "Valor actualizado correctamente"
        );
    } else {
        Application.showMessage(
            ApplicationMessageType.Error,
            "No se pudo actualizar el valor"
        );
    }
}
```

## submitData - Enviar cambios al backend

`submitData()` persiste todos los cambios hechos con `setUserInput()`.

### Reglas importantes

1. **Llamar una sola vez** despues de todos los `setUserInput()`. No llamar submitData despues de cada celda individual.
2. Devuelve `true` si la operacion fue exitosa.
3. Automaticamente refresca la tabla con los datos actualizados del backend.

```javascript
// Patron correcto: multiples setUserInput + un submitData
var planning = Table_Plan.getPlanning();
var selections = Table_Plan.getSelections();

for (var i = 0; i < selections.length; i++) {
    planning.setUserInput(selections[i], "100");
}

// Un solo submitData para todo
var submitted = planning.submitData();
if (submitted) {
    Application.showMessage(ApplicationMessageType.Success, "Datos enviados");
}
```

## Patron: Entrada de datos desde Input Fields

```javascript
// Layout: InputField_Revenue, InputField_Cost, Button_Save
// La tabla tiene una seleccion previa del usuario

// En onClick de Button_Save
var planning = Table_Plan.getPlanning();
if (planning === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "Planning no disponible");
    return;
}

var sel = Table_Plan.getSelections();
if (sel.length === 0) {
    Application.showMessage(ApplicationMessageType.Warning, "Selecciona una fila primero");
    return;
}

var baseSelection = sel[0];

// Crear selecciones especificas para cada cuenta
// Nota: la seleccion debe incluir todas las dimensiones necesarias
var revSelection = {};
var costSelection = {};

// Copiar todas las dimensiones de la seleccion base
for (var key in baseSelection) {
    revSelection[key] = baseSelection[key];
    costSelection[key] = baseSelection[key];
}

// Sobreescribir la dimension Account
revSelection["Account"] = "Revenue";
costSelection["Account"] = "Cost";

var revValue = InputField_Revenue.getValue();
var costValue = InputField_Cost.getValue();

// Validar antes de escribir
if (revValue === "" || costValue === "") {
    Application.showMessage(ApplicationMessageType.Warning, "Completa todos los campos");
    return;
}

planning.setUserInput(revSelection, revValue);
planning.setUserInput(costSelection, costValue);
planning.submitData();
```

## Valor con factor multiplicativo

Si el valor pasado a `setUserInput()` empieza con `*`, se aplica como factor al valor actual de la celda:

```javascript
// Incrementar el valor actual en un 10%
planning.setUserInput(selection, "*1.1");

// Reducir a la mitad
planning.setUserInput(selection, "*0.5");
```

> **Documentado en SAP:** El valor string con prefijo `*` se interpreta como multiplicador.

## Validacion de datos antes de entrada

```javascript
// Patron de validacion completo
function validateAndSave() {
    var planning = Table_Plan.getPlanning();
    if (planning === undefined) {
        Application.showMessage(ApplicationMessageType.Error, "Modelo no soporta planning");
        return;
    }

    if (!planning.isDataEntryEnabled()) {
        Application.showMessage(ApplicationMessageType.Error, "La entrada de datos esta deshabilitada");
        return;
    }

    var sel = Table_Plan.getSelections();
    if (sel.length === 0) {
        Application.showMessage(ApplicationMessageType.Warning, "Selecciona una celda");
        return;
    }

    var value = InputField_Value.getValue();

    // Validar que sea numerico
    var numValue = ConvertUtils.stringToNumber(value);
    if (numValue === undefined) {
        Application.showMessage(ApplicationMessageType.Error, "Introduce un valor numerico valido");
        return;
    }

    // Validar rango
    if (numValue < 0 || numValue > 999999999) {
        Application.showMessage(ApplicationMessageType.Error, "Valor fuera de rango permitido");
        return;
    }

    // Escribir
    var success = planning.setUserInput(sel[0], value);
    if (success) {
        planning.submitData();
        Application.showMessage(ApplicationMessageType.Success, "Guardado correctamente");
    } else {
        Application.showMessage(ApplicationMessageType.Error, "Error al escribir el valor");
    }
}
```

## Planning Sequences (BPC)

Si tu modelo esta basado en BPC, puedes ejecutar Planning Sequences desde script:

```javascript
// PlanningSequence_1 es un objeto BPC Planning Sequence Trigger anadido al canvas
// Configurar variables de la Planning Sequence
PlanningSequence_1.getBpcPlanningSequenceDataSource().setVariableValue("CATEGORY", "PLAN");
PlanningSequence_1.getBpcPlanningSequenceDataSource().setVariableValue("YEAR", "2026");

// Ejecutar
PlanningSequence_1.execute();
```

> **Solo BPC:** Las Planning Sequences son especificas de modelos BPC.
> Para modelos SAC nativos, se usan Data Actions (ver guia 14-DATA-ACTIONS-AVANZADO.md).

## Versiones y flujo de planning completo

```javascript
// Flujo tipico de planning:
// 1. Entrar en modo edicion
var planning = Table_Plan.getPlanning();
var privateVersion = planning.getPublicVersion("Plan");

// 2. El usuario hace cambios via UI o script
planning.setUserInput(selection, "50000");
planning.submitData();

// 3. Publicar cuando este listo (ver guia 02-VERSION-MANAGEMENT.md)
// privateVersion.publish();
```

## Undo/Redo

> **Limitacion:** No existe una API de undo/redo en el scripting de SAC.
> Si el usuario necesita deshacer cambios:
> - **Antes de submitData:** Los cambios no se han enviado, se pueden rehacer con nuevos setUserInput.
> - **Despues de submitData:** La unica forma es revertir la version privada (ver Version Management).
>
> Para simular undo, puedes guardar los valores originales en Script Variables antes de modificarlos:

```javascript
// Guardar valor original antes de cambiar
var ds = Table_Plan.getDataSource();
var originalData = ds.getData({"Account": "Revenue", "Region": "EMEA", "Date": "202601"});
ScriptVariable_OriginalValue = originalData.rawValue;

// ... usuario modifica ...

// Para "deshacer":
planning.setUserInput(selection, ScriptVariable_OriginalValue);
planning.submitData();
```

## Pitfalls comunes

1. **setUserInput en celda null (#):** En modelos BPC, no puedes usar setUserInput en celdas que muestran `#`. Solo funciona en celdas con valores existentes o con dash (`-`).

2. **Formato del valor:** El string debe respetar el formato regional del usuario. Si el usuario tiene configuracion europea, el separador decimal es coma: `"1.000,50"`. Esto puede causar errores silenciosos.

3. **Scalar maximo 7 caracteres:** Segun la documentacion SAP, el valor escalar puede tener un maximo de 7 caracteres.

4. **submitData sin setUserInput:** Llamar submitData sin haber hecho ningun setUserInput devuelve true pero no hace nada.

5. **Seleccion de celda agregada:** Si seleccionas una celda que es un total/subtotal, `setUserInput` distribuye el valor proporcionalmente entre las celdas hoja (leaf members). Esto es por diseno, no un bug.

6. **Planning no disponible en Optimized Stories:** La Planning API completa (`getPlanning()`, `setUserInput()`) es exclusiva de **Analytic Applications**. En Optimized Stories, la entrada de datos es via UI directa, no via script.

7. **isDataEntryEnabled() es false:** Verifica que:
   - El modelo tiene planning habilitado
   - El usuario tiene permisos de planner
   - La version privada esta creada
   - No hay data locking activo en las celdas objetivo
