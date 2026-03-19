# SAC Version Management via Scripting

## Conceptos clave

En SAC Planning, las **versiones** son copias del modelo de datos donde los usuarios
pueden trabajar sin afectar a los demas. Hay dos tipos:

| Tipo | Visibilidad | Uso tipico |
|------|-------------|-----------|
| **Publica** | Todos los usuarios | Actual, Plan, Forecast (versiones "oficiales") |
| **Privada** | Solo el creador | Borradores, simulaciones, backups temporales |

**Flujo habitual para crear una version nueva:**
1. Copiar una version existente → se crea como **privada**
2. Modificar datos en la copia privada
3. Publicar (`publish` o `publishAs`) → se convierte en **publica**

## Acceder al objeto Planning

Todo empieza obteniendo el objeto Planning de una tabla habilitada para planning:

```javascript
var planning = Table_1.getPlanning();
```

**Verificar que planning esta habilitado antes de operar:**

```javascript
if (Table_1.getPlanning().isEnabled()) {
    // ejecutar logica de versiones
}
```

---

## 1. Obtener versiones

```javascript
// Una version publica especifica
var actual = planning.getPublicVersion("Actual");

// Una version privada especifica
var myDraft = planning.getPrivateVersion("myDraft");

// Todas las publicas
var allPublic = planning.getPublicVersions();

// Todas las privadas
var allPrivate = planning.getPrivateVersions();

// Metadatos de una version
var id = actual.getId();
var displayId = actual.getDisplayId();

// Propietario de version privada
var owner = myDraft.getOwnerId();
```

---

## 2. Copiar versiones (copy)

La copia siempre crea una **version privada**. Es el unico mecanismo para crear versiones nuevas.

### Firma

```javascript
copy(
    newVersionName: string,
    planningCopyOption: PlanningCopyOption,
    versionCategory?: PlanningCategory,
    planningAreaFilters?: PlanningAreaFilters
): boolean
```

### PlanningCopyOption (que datos copiar)

| Valor | Que copia |
|-------|-----------|
| `PlanningCopyOption.AllData` | Todos los datos del modelo |
| `PlanningCopyOption.NoData` | Solo estructura (version vacia) |
| `PlanningCopyOption.PlanningArea` | Solo datos del planning area definido en el modelo |
| `PlanningCopyOption.VisibleData` | Solo lo visible en la tabla actual |
| `PlanningCopyOption.CustomizedPlanningArea` | Planning area con filtros personalizados |

### PlanningCategory (tipo de version)

| Valor | Tipo |
|-------|------|
| `PlanningCategory.Planning` | Planificacion |
| `PlanningCategory.Forecast` | Forecast |
| `PlanningCategory.Budget` | Presupuesto |

### Ejemplos basicos

```javascript
// Copiar Actual completo a version privada "Backup_Q1"
var actual = Table_1.getPlanning().getPublicVersion("Actual");
actual.copy("Backup_Q1", PlanningCopyOption.AllData);

// Copiar sin datos (version vacia para rellenar despues)
actual.copy("Plan_2027", PlanningCopyOption.NoData);

// Copiar solo planning area, categorizando como Forecast
actual.copy("Forecast_2026", PlanningCopyOption.PlanningArea, PlanningCategory.Forecast);

// Copiar con nombre dinamico (timestamp para backups)
actual.copy("Backup_" + Date.now().toString(), PlanningCopyOption.AllData);

// Copiar desde version privada
var myDraft = Table_1.getPlanning().getPrivateVersion("myDraft");
myDraft.copy("myDraft_v2", PlanningCopyOption.AllData);
```

### Copia con filtros personalizados (CustomizedPlanningArea)

Permite copiar solo un subconjunto de datos filtrado por dimensiones:

```javascript
// Obtener la configuracion de planning area
var areaFilters = Table_1.getPlanning().getPlanningAreaInfo();

// Quitar el filtro de fecha (copiar todas las fechas)
areaFilters.removeFilter("Date");

// Filtrar por CostCenter: solo OPERATIONS dentro de jerarquia H1
areaFilters.changeFilter("CostCenter", {
    "hierarchy": "H1",
    "members": ["[CostCenter].[H1].&[OPERATIONS]"]
});

// Ejecutar la copia con los filtros
var filters = areaFilters.getFilters();
Table_1.getPlanning().getPublicVersion("Actual").copy(
    "Actual_Ops_Only",
    PlanningCopyOption.CustomizedPlanningArea,
    undefined,      // sin cambiar categoria
    filters
);
```

---

## 3. Publicar versiones (publish)

### Publicar version publica (guardar cambios editados)

```javascript
var forecast = Table_1.getPlanning().getPublicVersion("Forecast");
forecast.publish();
```

### Publicar version privada (merge al modelo publico)

```javascript
var myDraft = Table_1.getPlanning().getPrivateVersion("myDraft");
myDraft.publish();
```

### Publicar solo datos modificados a una version destino

```javascript
// Segundo parametro true = solo delta (datos cambiados)
var myDraft = Table_1.getPlanning().getPrivateVersion("myDraft");
var forecast = Table_1.getPlanning().getPublicVersion("Forecast");
myDraft.publish(forecast, true);
```

### Crear version publica nueva desde privada (publishAs)

```javascript
// publishAs convierte una version privada en una publica nueva
var temp = Table_1.getPlanning().getPrivateVersion("TempDraft");
temp.publishAs("Budget_2026", PlanningCategory.Budget);
```

### Gestion de conflictos al publicar

Cuando otros usuarios han modificado la misma version:

```javascript
// Version publica: tres opciones
forecast.publish(PublicPublishConflict.ShowWarning);            // dialogo al usuario
forecast.publish(PublicPublishConflict.PublishWithoutWarning);   // sobreescribir
forecast.publish(PublicPublishConflict.RevertWithoutWarning);    // revertir sin dialogo

// Version privada: dos opciones
myDraft.publish(PrivatePublishConflict.ShowWarning);
myDraft.publish(PrivatePublishConflict.PublishWithoutWarning);
```

| Enum | Valores |
|------|---------|
| `PublicPublishConflict` | `ShowWarning`, `PublishWithoutWarning`, `RevertWithoutWarning` |
| `PrivatePublishConflict` | `ShowWarning`, `PublishWithoutWarning` |

---

## 4. Revertir cambios (revert)

Deshace todos los cambios no publicados de una version publica:

```javascript
Table_1.getPlanning().getPublicVersion("Forecast").revert();
```

---

## 5. Eliminar versiones (deleteVersion)

```javascript
// Eliminar version privada
Table_1.getPlanning().getPrivateVersion("OldDraft").deleteVersion();

// Eliminar version publica (cuidado: irreversible)
Table_1.getPlanning().getPublicVersion("Test_Version").deleteVersion();
```

---

## 6. Verificar estado (isDirty)

Comprueba si una version tiene cambios pendientes de publicar:

```javascript
var forecast = Table_1.getPlanning().getPublicVersion("Forecast");

if (forecast.isDirty()) {
    // hay cambios sin publicar
    forecast.publish();
} else {
    Application.showMessage(ApplicationMessageType.Info, "No hay cambios pendientes");
}
```

---

## 7. Modo edicion (startEditMode)

Inicia el modo edicion en areas de planificacion especificas de una version publica:

```javascript
Table_1.getPlanning().getPublicVersion("Forecast").startEditMode();
```

---

## 8. Guardar datos editados (submitData)

Diferente de publish: `submitData` guarda los cambios del usuario en la version privada
o en el buffer de edicion, sin publicar al modelo publico.

```javascript
var success = Table_1.getPlanning().submitData();
if (success) {
    Application.showMessage(ApplicationMessageType.Success, "Datos guardados");
}
```

---

## Ejemplos completos

### Boton "Crear Version Plan"

```javascript
// onClick
var newName = InputField_VersionName.getValue();

// Validar input
if (newName === "" || newName === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "Introduce un nombre para la version");
    return;
}

// Verificar planning habilitado
if (!Table_1.getPlanning().isEnabled()) {
    Application.showMessage(ApplicationMessageType.Error, "Planning no esta habilitado");
    return;
}

Application.showBusyIndicator();

var actual = Table_1.getPlanning().getPublicVersion("Actual");

// 1. Copiar sin datos (copy devuelve boolean)
var success = actual.copy(newName, PlanningCopyOption.NoData);

if (success) {
    // 2. Publicar como version publica tipo Planning
    var privateCopy = Table_1.getPlanning().getPrivateVersion(newName);
    privateCopy.publishAs(newName, PlanningCategory.Planning);

    // 3. Refrescar y notificar
    Table_1.getDataSource().refreshData();
    Application.showMessage(ApplicationMessageType.Success, "Version '" + newName + "' creada");
} else {
    Application.showMessage(ApplicationMessageType.Error, "Error al copiar la version");
}

Application.hideBusyIndicator();
```

### Boton "Backup antes de Data Action"

```javascript
// onClick
Application.showBusyIndicator();

// 1. Crear backup con timestamp
var backupName = "Backup_" + Date.now().toString();
var success = Table_1.getPlanning().getPublicVersion("Forecast").copy(
    backupName,
    PlanningCopyOption.AllData
);

if (!success) {
    Application.hideBusyIndicator();
    Application.showMessage(ApplicationMessageType.Error, "Error al crear backup. Data Action cancelada.");
    return;
}

// 2. Ejecutar Data Action (solo si el backup fue exitoso)
DataAction_Calculo.execute({
    "VERSION": "Forecast",
    "YEAR": Dropdown_Year.getSelectedKey()
});

// 3. Refrescar
Table_1.getDataSource().refreshData();
Application.hideBusyIndicator();
Application.showMessage(ApplicationMessageType.Success,
    "Backup '" + backupName + "' creado y Data Action ejecutada");
```

### Boton "Publicar con confirmacion"

```javascript
// onClick
var forecast = Table_1.getPlanning().getPublicVersion("Forecast");

if (forecast.isDirty()) {
    forecast.publish(PublicPublishConflict.ShowWarning);
    Table_1.getDataSource().refreshData();
    Application.showMessage(ApplicationMessageType.Success, "Forecast publicado");
} else {
    Application.showMessage(ApplicationMessageType.Info, "No hay cambios que publicar");
}
```

### Boton "Limpiar versiones privadas antiguas"

```javascript
// onClick
var privates = Table_1.getPlanning().getPrivateVersions();
var count = 0;

for (var i = 0; i < privates.length; i++) {
    var v = privates[i];
    // Eliminar todas las que empiecen por "Backup_"
    if (v.getDisplayId().indexOf("Backup_") === 0) {
        v.deleteVersion();
        count = count + 1;
    }
}

Application.showMessage(ApplicationMessageType.Success,
    count.toString() + " backups eliminados");
```

---

## Resumen de metodos

| Metodo | Sobre | Que hace |
|--------|-------|----------|
| `getPlanning()` | Table/Widget | Obtiene objeto Planning |
| `isEnabled()` | Planning | Verifica si planning esta activo |
| `getPublicVersion(id)` | Planning | Obtiene version publica |
| `getPrivateVersion(id)` | Planning | Obtiene version privada |
| `getPublicVersions()` | Planning | Lista todas las publicas |
| `getPrivateVersions()` | Planning | Lista todas las privadas |
| `copy(name, option, cat?, filters?)` | Version | Crea copia privada |
| `publish()` | Version | Publica cambios |
| `publish(target, deltaOnly)` | PrivateVersion | Publica a destino especifico |
| `publishAs(name, category)` | PrivateVersion | Crea nueva version publica |
| `revert()` | PublicVersion | Revierte cambios no publicados |
| `deleteVersion()` | Version | Elimina la version |
| `isDirty()` | Version | Comprueba cambios pendientes |
| `startEditMode()` | PublicVersion | Inicia modo edicion |
| `submitData()` | Planning | Guarda datos sin publicar |
| `getId()` | Version | ID interno |
| `getDisplayId()` | Version | ID visible |
| `getOwnerId()` | PrivateVersion | ID del creador |
