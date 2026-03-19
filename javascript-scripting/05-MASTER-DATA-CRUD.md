# SAC Master Data - Crear, Leer, Actualizar y Eliminar Miembros

## Concepto

Desde JavaScript en Analytic Applications puedes gestionar los **miembros de dimensiones genericas**
del modelo de planning: crear nuevos, actualizar propiedades y eliminar.

**Requisito:** Debes anadir el Planning Model como objeto en el Designer
(panel Outline → Planning Models Objects).

**Limitacion:** Solo funciona con **dimensiones genericas**. NO soporta
dimensiones de tipo Date, Version, Account ni Organization.

---

## Objeto PlanningModel

Una vez anadido el modelo al designer, accedes con su nombre tecnico:

```javascript
// PlanningModel_1 es el nombre asignado en el Outline
PlanningModel_1.createMembers(...);
PlanningModel_1.updateMembers(...);
PlanningModel_1.deleteMembers(...);
PlanningModel_1.getMember(...);
PlanningModel_1.getMembers(...);
```

---

## 1. Leer miembros (getMember / getMembers)

```javascript
// Obtener un miembro especifico
var member = PlanningModel_1.getMember("CostCenter", "CC_1000");

// Obtener todos los miembros de una dimension
var members = PlanningModel_1.getMembers("CostCenter");

for (var i = 0; i < members.length; i++) {
    // Nota: console.log() solo funciona para depuracion en el editor, no es API oficial
    Application.showMessage(ApplicationMessageType.Info, members[i].id + " - " + members[i].description);
}
```

---

## 2. Crear miembros (createMembers)

Crea uno o varios miembros nuevos. El ID no debe existir previamente.

```javascript
// Crear un solo miembro
PlanningModel_1.createMembers("CostCenter", [{
    "id": "CC_2000",
    "description": "Marketing EMEA",
    "properties": {
        "Region": "EMEA",
        "Department": "Marketing"
    }
}]);

// Crear varios miembros de golpe
PlanningModel_1.createMembers("Product", [
    {
        "id": "PROD_100",
        "description": "Widget Alpha",
        "properties": {
            "ProductGroup": "Widgets",
            "Status": "Active"
        }
    },
    {
        "id": "PROD_101",
        "description": "Widget Beta",
        "properties": {
            "ProductGroup": "Widgets",
            "Status": "Active"
        }
    }
]);
```

---

## 3. Actualizar miembros (updateMembers)

Actualiza propiedades de miembros existentes. El ID debe existir.

```javascript
// Actualizar la descripcion y una propiedad
PlanningModel_1.updateMembers("CostCenter", [{
    "id": "CC_2000",
    "description": "Marketing EMEA - Updated",
    "properties": {
        "Status": "Inactive"
    }
}]);

// Actualizar varios
PlanningModel_1.updateMembers("Product", [
    {"id": "PROD_100", "properties": {"Status": "Discontinued"}},
    {"id": "PROD_101", "properties": {"Status": "Discontinued"}}
]);
```

---

## 4. Eliminar miembros (deleteMembers)

Elimina miembros existentes. Los datos asociados tambien se eliminan.

```javascript
// Eliminar un miembro
PlanningModel_1.deleteMembers("CostCenter", ["CC_2000"]);

// Eliminar varios
PlanningModel_1.deleteMembers("Product", ["PROD_100", "PROD_101"]);
```

---

## 5. Ejemplo completo: Formulario CRUD con botones

### Crear: boton + input fields

```javascript
// onClick del boton "Crear"
var newId = InputField_Id.getValue();
var newDesc = InputField_Description.getValue();
var newGroup = Dropdown_Group.getSelectedKey();

if (newId === "" || newDesc === "") {
    Application.showMessage(ApplicationMessageType.Error, "ID y Descripcion son obligatorios");
} else {
    PlanningModel_1.createMembers("Product", [{
        "id": newId,
        "description": newDesc,
        "properties": {
            "ProductGroup": newGroup
        }
    }]);

    // Publicar cambios en master data
    Table_1.getPlanning().getPublicVersion("Actual").publish();
    Table_1.getDataSource().refreshData();

    Application.showMessage(ApplicationMessageType.Success, "Miembro '" + newId + "' creado");

    // Limpiar campos
    InputField_Id.setValue("");
    InputField_Description.setValue("");
}
```

### Actualizar: boton + input fields

```javascript
// onClick del boton "Actualizar"
var targetId = InputField_TargetId.getValue();
var newDesc = InputField_NewDescription.getValue();

if (targetId === "" || targetId === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "Introduce el ID del miembro a actualizar");
    return;
}

if (newDesc === "" || newDesc === undefined) {
    Application.showMessage(ApplicationMessageType.Error, "Introduce la nueva descripcion");
    return;
}

PlanningModel_1.updateMembers("Product", [{
    "id": targetId,
    "description": newDesc
}]);

Table_1.getPlanning().getPublicVersion("Actual").publish();
Table_1.getDataSource().refreshData();
Application.showMessage(ApplicationMessageType.Success, "Miembro '" + targetId + "' actualizado");
```

### Eliminar: boton con confirmacion

```javascript
// onClick del boton "Eliminar"
var targetId = InputField_DeleteId.getValue();

if (targetId === "") {
    Application.showMessage(ApplicationMessageType.Error, "Introduce el ID a eliminar");
} else {
    PlanningModel_1.deleteMembers("Product", [targetId]);

    Table_1.getPlanning().getPublicVersion("Actual").publish();
    Table_1.getDataSource().refreshData();
    Application.showMessage(ApplicationMessageType.Success, "Miembro '" + targetId + "' eliminado");
}
```

---

## 6. Publicar cambios de master data

Despues de crear/actualizar/eliminar miembros, los cambios estan en estado "draft".
Para que sean visibles para otros usuarios, hay que **publicar**:

```javascript
// Publicar la version donde se hicieron los cambios
Table_1.getPlanning().getPublicVersion("Actual").publish();
```

Sin este paso, los cambios de master data solo son visibles para ti.

---

## Resumen de metodos

| Metodo | Parametros | Que hace |
|--------|-----------|----------|
| `getMember(dim, id)` | dimension, memberId | Lee un miembro |
| `getMembers(dim)` | dimension | Lee todos los miembros |
| `createMembers(dim, members[])` | dimension, array de objetos | Crea miembros nuevos |
| `updateMembers(dim, members[])` | dimension, array de objetos | Actualiza propiedades |
| `deleteMembers(dim, ids[])` | dimension, array de IDs | Elimina miembros |
