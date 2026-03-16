# SAC Popups, Dialogos y Navegacion

## 1. Popups y Dialogos

### Crear un Popup (en el Designer)

1. Panel Outline → icono "+" junto a "Popups"
2. Se crea un popup donde puedes arrastrar widgets dentro (inputs, dropdowns, tablas)
3. Toggle "Enable header & footer" para convertirlo en dialogo con botones OK/Cancel

### Abrir y cerrar un Popup desde script

```javascript
// Abrir popup
Popup_Config.open();

// Cerrar popup (normalmente desde dentro del propio popup)
Popup_Config.close();
```

### Eventos del Popup

| Evento | Cuando se dispara |
|--------|------------------|
| `onOpen` | Al abrirse el popup |
| `onClose` | Al cerrarse |
| `onOK` | Al pulsar boton OK (si es dialogo) |
| `onCancel` | Al pulsar Cancel (si es dialogo) |

### Ejemplo: Popup para crear version

**Boton principal (onClick):**
```javascript
Popup_NewVersion.open();
```

**Popup → evento onOK:**
```javascript
Application.showBusyIndicator();

var newName = InputField_VersionName.getValue();

if (newName === "") {
    Application.showMessage(ApplicationMessageType.Error, "Introduce un nombre");
    Application.hideBusyIndicator();
} else {
    var actual = Table_1.getPlanning().getPublicVersion("Actual");
    actual.copy(newName, PlanningCopyOption.NoData);

    var privateCopy = Table_1.getPlanning().getPrivateVersion(newName);
    privateCopy.publishAs(newName, PlanningCategory.Planning);

    Table_1.getDataSource().refreshData();
    Application.hideBusyIndicator();
    Application.showMessage(ApplicationMessageType.Success, "Version '" + newName + "' creada");

    InputField_VersionName.setValue("");
    Popup_NewVersion.close();
}
```

**Popup → evento onCancel:**
```javascript
InputField_VersionName.setValue("");
Popup_NewVersion.close();
```

---

## 2. Mensajes al usuario

```javascript
// Exito (verde)
Application.showMessage(ApplicationMessageType.Success, "Operacion completada");

// Informacion (azul)
Application.showMessage(ApplicationMessageType.Info, "No hay cambios pendientes");

// Warning (amarillo)
Application.showMessage(ApplicationMessageType.Warning, "Revisa los datos antes de publicar");

// Error (rojo)
Application.showMessage(ApplicationMessageType.Error, "Fallo al ejecutar la Data Action");
```

---

## 3. Indicador de carga (Busy Indicator)

Para operaciones que tardan (Data Actions, copias de version, etc.):

```javascript
Application.showBusyIndicator();

// ... operacion larga ...

Application.hideBusyIndicator();
```

**Importante:** Siempre asegurate de llamar `hideBusyIndicator()` incluso si hay error,
o la aplicacion quedara bloqueada.

---

## 4. Navegacion entre paginas

**Nota:** Las paginas existen en **Stories**. Las **Analytic Applications** no tienen paginas,
usan paneles y visibilidad para simular navegacion.

### En Stories con scripting

```javascript
// Ir a otra pagina
Application.openPage("Page_Detail");

// Volver a pagina anterior
Application.openPage("Page_Main");
```

### En Analytic Applications (simular con paneles)

```javascript
// Ocultar panel actual, mostrar el destino
Panel_Main.setVisible(false);
Panel_Detail.setVisible(true);
```

### Navegacion a otra aplicacion

```javascript
// Abrir otra aplicacion/story en nueva ventana
Application.openUrl("https://tenant.sapanalytics.cloud/story/STORY_ID");
```

---

## 5. Bookmarks (guardar estado)

Los bookmarks guardan el estado completo del dashboard: filtros, selecciones,
visibilidad de widgets, valores de script variables, jerarquias, ordenacion.

### Prerequisito

Crear un **Bookmark Set** en el Designer (panel Outline → Bookmark Sets).

### API de Bookmarks

```javascript
// Obtener el bookmark set
var bookmarkSet = Application.getBookmarkSet("MyBookmarkSet");

// Guardar bookmark nuevo
bookmarkSet.saveBookmark("Mi Vista Q1", {
    // properties opcionales
});

// Listar bookmarks guardados
var bookmarks = bookmarkSet.getBookmarks();

// Abrir un bookmark
bookmarkSet.openBookmark(bookmarkId);

// Actualizar bookmark existente
bookmarkSet.updateBookmark(bookmarkId);

// Eliminar bookmark
bookmarkSet.deleteBookmark(bookmarkId);

// Compartir bookmark
bookmarkSet.shareBookmark(bookmarkId);
```

### Ejemplo: Panel de bookmarks con botones

```javascript
// Boton "Guardar vista"
var name = InputField_BookmarkName.getValue();
bookmarkSet.saveBookmark(name);
Application.showMessage(ApplicationMessageType.Success, "Vista '" + name + "' guardada");

// Boton "Cargar vista" (desde dropdown con bookmarks)
var selectedBookmark = Dropdown_Bookmarks.getSelectedKey();
bookmarkSet.openBookmark(selectedBookmark);
```

### Script Variables y Bookmarks

Los bookmarks guardan automaticamente el valor de las Script Variables.
Puedes pasar variables por URL:

```
https://tenant.sapanalytics.cloud/story/STORY_ID;p_ScriptVariable_1=value1;p_ScriptVariable_2=value2
```

---

## 6. Timer (ejecucion periodica)

El evento `onTimer` de la Application permite ejecutar logica periodicamente:

```javascript
// En Application → onTimer
// Se ejecuta cada X segundos (configurado en el Designer)
Table_1.getDataSource().refreshData();
```

Util para dashboards que necesitan refrescar datos automaticamente.

---

## Resumen

| Accion | Metodo |
|--------|--------|
| Abrir popup | `Popup_Name.open()` |
| Cerrar popup | `Popup_Name.close()` |
| Mensaje exito | `Application.showMessage(ApplicationMessageType.Success, text)` |
| Mensaje error | `Application.showMessage(ApplicationMessageType.Error, text)` |
| Carga on | `Application.showBusyIndicator()` |
| Carga off | `Application.hideBusyIndicator()` |
| Ir a pagina | `Application.openPage("Page_Name")` |
| Abrir URL | `Application.openUrl(url)` |
| Toggle panel | `Panel.setVisible(bool)` |
| Guardar bookmark | `bookmarkSet.saveBookmark(name)` |
| Abrir bookmark | `bookmarkSet.openBookmark(id)` |
