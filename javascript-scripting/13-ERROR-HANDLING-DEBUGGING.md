# Error Handling y Debugging

## Soporte de try/catch en SAC

> **Estado actual (2025+):** SAC soporta bloques `try/catch` en versiones recientes
> del Analytics Designer. Sin embargo, el soporte no ha existido siempre y en versiones
> antiguas los scripts crasheaban silenciosamente sin posibilidad de capturar errores.
>
> La guia introductoria de este repositorio menciona "sin try/catch" como limitacion.
> Esto era cierto en versiones iniciales. A partir de ~2023-2024, SAP anadio soporte
> para try/catch, pero con limitaciones.

### Uso basico de try/catch

```javascript
try {
    var ds = Table_1.getDataSource();
    var data = ds.getData({"Account": "Revenue", "Region": "EMEA"});
    Application.showMessage(ApplicationMessageType.Info, "Valor: " + data.rawValue);
} catch (e) {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Error al obtener datos: " + e.message
    );
}
```

### Limitaciones del try/catch en SAC

- No todas las excepciones son capturables. Errores de infraestructura (timeout, conexion perdida) no se capturan.
- Los errores de compilacion (sintaxis) no se capturan en runtime.
- Algunos metodos de la API fallan silenciosamente devolviendo `undefined` o `false` en lugar de lanzar excepciones.
- La propiedad `e.message` puede contener mensajes genericos poco utiles.

### Patron defensivo recomendado

```javascript
// En vez de depender solo de try/catch, valida antes de operar
var ds = Table_1.getDataSource();
if (ds === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "DataSource no disponible");
    return;
}

var planning = Table_1.getPlanning();
if (planning === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "Planning no disponible");
    return;
}

var selections = Table_1.getSelections();
if (selections.length === 0) {
    Application.showMessage(ApplicationMessageType.Warning, "No hay seleccion");
    return;
}

// Solo ahora operar con seguridad
var success = planning.setUserInput(selections[0], "100");
```

## console.log() en SAC

`console.log()` **funciona** en SAC y es la herramienta principal de depuracion junto con `Application.showMessage()`.

### Como ver la salida de console.log

1. Ejecuta la aplicacion en modo View o Preview
2. Abre las Developer Tools del navegador: `F12` o `Ctrl+Shift+J`
3. Ve a la pestana **Console**
4. Los mensajes de `console.log()` aparecen ahi

```javascript
// Depuracion con console.log
var ds = Table_1.getDataSource();
var members = ds.getMembers("Region");
console.log("Numero de miembros Region: " + members.length.toString());

for (var i = 0; i < members.length; i++) {
    console.log("Miembro " + i.toString() + ": " + members[i].id + " - " + members[i].description);
}
```

> **Importante:** `console.log()` no es API oficial de SAC. Es una funcionalidad del
> navegador que SAC no bloquea. No aparece en la documentacion oficial de SAP, pero
> funciona y la comunidad lo usa extensivamente para debugging.

### console.log vs Application.showMessage

| | `console.log()` | `Application.showMessage()` |
|--|-----------------|---------------------------|
| **Visible al usuario final** | No (solo Developer Tools) | Si (toast en pantalla) |
| **Volumen de datos** | Ilimitado | Un mensaje a la vez |
| **Tipos de datos** | Cualquiera | Solo strings |
| **Documentado por SAP** | No | Si |
| **Uso recomendado** | Desarrollo/debugging | Feedback al usuario + debugging basico |

## Debug Mode via URL

Puedes activar el modo debug anadiendo `&debug=true` a la URL de la aplicacion:

```
https://<HOST>/sap/fpa/ui/app.html?tenant=<TENANT>#;mode=present;view_id=appBuilding;appId=<APP_ID>&debug=true
```

### Que habilita el debug mode

1. **Comentarios en codigo generado:** Los comentarios de tus scripts se preservan en el JavaScript transformado, facilitando encontrar tu codigo en DevTools.
2. **Breakpoints funcionales:** Puedes poner breakpoints en tus scripts desde el editor.
3. **Mejor trazabilidad:** Los errores en consola incluyen mas contexto.

## Breakpoints en el editor de SAC

### Breakpoints nativos del editor

1. Abre el script en el editor de SAC (design time)
2. Haz click en el numero de linea a la izquierda
3. Aparece un marcador azul indicando el breakpoint
4. Ejecuta la aplicacion: el script se pausa en ese punto

### Sentencia debugger

Puedes insertar `debugger;` directamente en tu codigo:

```javascript
// El script se pausara aqui si DevTools esta abierto
var ds = Table_1.getDataSource();
debugger;
var data = ds.getData({"Account": "Revenue"});
// Desde DevTools puedes inspeccionar 'ds' y 'data'
```

> **Requisito:** Las Developer Tools del navegador deben estar abiertas ANTES de que
> se ejecute la linea `debugger;`. Si no estan abiertas, la sentencia se ignora.

## Encontrar tus scripts en Developer Tools

1. Abre DevTools (`F12`)
2. Ve a la pestana **Sources**
3. En el arbol de archivos, busca la entrada que empieza con `sandbox.worker.main`
4. Dentro, busca **AnalyticApplication** > nombre de tu aplicacion
5. Ahi encontraras todos los scripts que se han ejecutado

## Errores comunes de runtime

### 1. Cannot read property 'x' of undefined

```javascript
// ERROR: si getDataSource() devuelve undefined
var value = Table_1.getDataSource().getData({"Account": "Revenue"});

// SOLUCION: validar cada paso
var ds = Table_1.getDataSource();
if (ds !== undefined) {
    var data = ds.getData({"Account": "Revenue"});
    if (data !== undefined) {
        // usar data.rawValue
    }
}
```

### 2. Method not found

Ocurre cuando llamas un metodo que no existe en el tipo de widget. Ejemplo: llamar `getPlanning()` en un Chart.

```javascript
// Verificar disponibilidad del metodo
// No hay un typeof check fiable, pero puedes verificar el tipo de widget
// La mejor practica es conocer que metodos tiene cada widget
```

### 3. Script termina sin mensaje (timeout)

Los scripts tienen un limite de ejecucion de aproximadamente 30 segundos. Si un bucle procesa muchos elementos o hace muchas llamadas API, puede exceder este limite.

```javascript
// PROBLEMA: bucle sobre miles de miembros
var members = ds.getMembers("CostCenter");
for (var i = 0; i < members.length; i++) {
    // Si CostCenter tiene 5000 miembros, esto puede exceder timeout
    ds.setDimensionFilter("CostCenter", members[i].id);
    // ... procesamiento ...
}

// SOLUCION: limitar el procesamiento o usar Data Actions para logica masiva
```

### 4. Error silencioso en setDimensionFilter

```javascript
// Si el miembro no existe en la dimension, no da error pero el filtro no se aplica
ds.setDimensionFilter("Region", "REGION_QUE_NO_EXISTE");
// No hay error, pero la tabla puede quedar vacia
```

### 5. getResultSet devuelve datos inesperados

```javascript
// getResultSet devuelve los datos tal como estan filtrados en la tabla
// Si hay filtros activos, solo devuelve los datos visibles
var rs = ds.getResultSet();
// rs puede estar vacio si los filtros excluyen todo
```

## Patron: Wrapper de error handling

```javascript
// En un ScriptObject (ScriptObject_ErrorHandler)
function safeExecute(description) {
    // Registrar en consola para debugging
    console.log("[INFO] Ejecutando: " + description);
}

function logError(operation, errorMsg) {
    console.log("[ERROR] " + operation + ": " + errorMsg);
    Application.showMessage(
        ApplicationMessageType.Error,
        "Error en " + operation + ": " + errorMsg
    );
}

function logWarning(operation, msg) {
    console.log("[WARN] " + operation + ": " + msg);
    Application.showMessage(
        ApplicationMessageType.Warning,
        msg
    );
}
```

```javascript
// Uso desde cualquier evento
ScriptObject_ErrorHandler.safeExecute("Aplicar filtro Region");

var ds = Table_1.getDataSource();
if (ds === undefined) {
    ScriptObject_ErrorHandler.logError("Filtro Region", "DataSource no disponible");
    return;
}

var sel = Dropdown_Region.getSelectedKey();
if (sel === "" || sel === undefined) {
    ScriptObject_ErrorHandler.logWarning("Filtro Region", "No hay region seleccionada");
    return;
}

ds.setDimensionFilter("Region", sel);
```

## Depuracion de Data Actions

Los errores de Data Actions son especialmente dificiles de depurar porque se ejecutan en el backend:

```javascript
// Ejecutar Data Action y verificar resultado
var result = DataAction_Calculate.execute();

// El resultado tiene una propiedad status
if (result.status === "OK") {
    Application.showMessage(ApplicationMessageType.Success, "Data Action completada");
} else {
    Application.showMessage(
        ApplicationMessageType.Error,
        "Data Action fallo con status: " + result.status
    );
}
```

> **Nota:** El objeto `DataActionExecutionResponse` devuelto por `execute()` contiene
> el campo `status`. Los valores posibles no estan completamente documentados, pero
> incluyen "OK" para ejecucion exitosa.

## Estrategia de debugging recomendada

1. **Primero:** Usa `Application.showMessage()` para verificar valores intermedios rapidamente
2. **Segundo:** Anade `console.log()` para datos volumosos o inspecciones detalladas
3. **Tercero:** Activa debug mode (`&debug=true`) y usa breakpoints para stepping
4. **Cuarto:** Usa `debugger;` en puntos especificos donde necesites inspeccionar variables
5. **Siempre:** Valida `undefined` antes de acceder a propiedades o metodos de objetos

## Pitfalls de debugging

1. **console.log en produccion:** No olvides quitar o comentar los `console.log()` antes de publicar. No causan errores pero ensucian la consola.

2. **Breakpoints no funcionan:** Si los breakpoints no se activan, verifica que `debug=true` esta en la URL y que DevTools esta abierto.

3. **showMessage se acumula:** Si pones showMessage dentro de un bucle, los mensajes se acumulan y solo se ve el ultimo. Usa console.log para bucles.

4. **Error oculto por try/catch:** Un catch vacio oculta errores. Siempre loggea algo en el catch.

5. **Scripts en widgets invisibles:** Si un widget es invisible y su script se ejecuta, puede fallar silenciosamente si depende de widgets que aun no se han renderizado.
