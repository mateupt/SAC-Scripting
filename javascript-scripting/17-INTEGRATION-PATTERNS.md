# Integration Patterns

## Navegacion entre aplicaciones

### NavigationUtils - API de navegacion

SAC proporciona `NavigationUtils` para navegacion entre aplicaciones, stories y URLs externas.

| Metodo | Descripcion |
|--------|-------------|
| `NavigationUtils.openApplication(appId, params[])` | Abre una Analytic Application |
| `NavigationUtils.openStory(storyId, pageId, params[], newTab)` | Abre un Story |
| `NavigationUtils.openDataAnalyzer(config)` | Abre el Data Analyzer |

### Application.openURL

| Metodo | Descripcion |
|--------|-------------|
| `Application.openURL(url)` | Abre una URL externa en una nueva pestana |

## Pasar parametros entre aplicaciones

### UrlParameter.create

Para pasar valores entre aplicaciones, se usa `UrlParameter.create()`:

```javascript
// Sintaxis
UrlParameter.create("p_NombreVariable", "valor")
```

> **Convencion obligatoria:** El nombre del parametro DEBE tener el prefijo `p_`.
> Si tu Script Variable se llama `SelectedRegion`, el parametro es `p_SelectedRegion`.

### Ejemplo: Navegar a otra app con parametros

```javascript
// En onClick de Button_OpenDetail
var region = Dropdown_Region.getSelectedKey();
var year = Dropdown_Year.getSelectedKey();

NavigationUtils.openApplication(
    "APP_ID_DETAIL_123",
    [
        UrlParameter.create("p_Region", region),
        UrlParameter.create("p_Year", year)
    ]
);
```

### Recibir parametros en la app destino

En la app destino:

1. Crea Script Variables con los mismos nombres (sin prefijo `p_`)
2. Marca la opcion **"Expose variable via URL parameter"** en cada Script Variable
3. Los valores se asignan automaticamente al abrir la app

```javascript
// En onInitialization de la app destino
// ScriptVariable_Region ya tiene el valor pasado por URL
// ScriptVariable_Year ya tiene el valor pasado por URL

if (ScriptVariable_Region !== "") {
    Table_Detail.getDataSource().setDimensionFilter("Region", ScriptVariable_Region);
    Text_Title.setText("Detalle de " + ScriptVariable_Region + " - " + ScriptVariable_Year);
}
```

> **Importante:** La Script Variable debe existir en la app destino y debe estar
> marcada como "Expose variable via URL parameter". Si no esta marcada, el valor
> del URL se ignora.

## Navegar a Stories

```javascript
// Abrir un story en la misma pestana
NavigationUtils.openStory(
    "STORY_ID_456",       // Story ID
    "Page_Overview",       // Page ID (opcional, "" para pagina por defecto)
    [                      // URL parameters
        UrlParameter.create("p_Year", "2026")
    ],
    false                  // false = misma pestana, true = nueva pestana
);
```

### Abrir en nueva pestana

```javascript
// Abrir en nueva pestana
NavigationUtils.openStory(
    "STORY_ID_456",
    "",
    [],
    true  // Nueva pestana
);
```

## Abrir Data Analyzer

```javascript
// Abrir Data Analyzer con filtros predefinidos
NavigationUtils.openDataAnalyzer({
    modelId: "MODEL_ID_789",
    filters: [
        {dimensionId: "Region", memberId: "EMEA"},
        {dimensionId: "Year", memberId: "2026"}
    ]
});
```

## Application.openURL - URLs externas

```javascript
// Abrir URL externa
Application.openURL("https://example.com/report");

// Abrir URL con parametros dinamicos
var orderId = InputField_OrderId.getValue();
Application.openURL("https://erp.company.com/order/" + orderId);
```

> **Limitacion:** `openURL()` siempre abre en una nueva pestana del navegador.
> No puedes embeber contenido externo dentro de la aplicacion SAC
> (excepto con custom widgets que usen iframes).

## URL API para embedding

### Embeber SAC en aplicaciones externas

SAC permite embeber stories y analytic applications en iframes:

```
https://<SAC_HOST>/sap/fpa/ui/tenants/<TENANT>/app.html#;mode=present;view_id=story;storyId=<STORY_ID>&url_api=true
```

Parametros de la URL API:

| Parametro | Descripcion |
|-----------|-------------|
| `mode=present` | Modo presentacion (sin editor) |
| `mode=embed` | Modo embebido (sin barra de navegacion) |
| `url_api=true` | Activa la URL API |
| `view_id=story` | Para stories |
| `view_id=appBuilding` | Para analytic applications |
| `f01Dim=<dimension>&f01Val=<value>` | Filtro por URL |
| `v01Par=<variable>&v01Val=<value>` | Variable por URL |
| `p_<variable>=<value>` | Script Variable por URL |

### Ejemplo de URL con filtros

```
https://mycompany.sapanalytics.cloud/sap/fpa/ui/tenants/myco/app.html
#;mode=embed;view_id=story;storyId=ABC123
&url_api=true
&f01Dim=Region&f01Val=EMEA
&f01Op=in
&f02Dim=Year&f02Val=2026
```

### Requisitos de configuracion

Para que el embedding funcione:

1. **Administracion > App Integration:** Habilitar "Allow embedding in iframes"
2. Configurar los dominios permitidos en la whitelist de CORS
3. El usuario que accede debe tener una sesion activa en SAC

> **Documentado por SAP:** La URL API esta documentada oficialmente.
> Hay diferencias entre la URL API para Stories y para Analytic Applications.

## Patron: Landing page con navegacion

```javascript
// Aplicacion que sirve como hub de navegacion
// En onInitialization
var roles = Application.getRolesInfo();

// Mostrar/ocultar tiles segun permisos
var isPlanner = false;
var isAdmin = false;

for (var i = 0; i < roles.length; i++) {
    if (roles[i] === "BI_User_Planner") { isPlanner = true; }
    if (roles[i] === "Planning_Admin") { isAdmin = true; isPlanner = true; }
}

Panel_PlanningTile.setVisible(isPlanner);
Panel_AdminTile.setVisible(isAdmin);

// En onClick de Button_OpenRevenue
NavigationUtils.openApplication(
    "APP_REVENUE_PLAN",
    [UrlParameter.create("p_Year", "2026")]
);

// En onClick de Button_OpenCost
NavigationUtils.openApplication(
    "APP_COST_PLAN",
    [UrlParameter.create("p_Year", "2026")]
);

// En onClick de Button_OpenReport
NavigationUtils.openStory(
    "STORY_EXEC_REPORT",
    "",
    [UrlParameter.create("p_Year", "2026")],
    true // nueva pestana
);

// En onClick de Button_OpenSAP
Application.openURL("https://erp.company.com/fiori");
```

## Integracion con SAP BW y BPC

### Modelos BPC en SAC

SAC puede conectarse a modelos BPC (Business Planning and Consolidation). La integracion
se configura a nivel de conexion/modelo, no de scripting. Desde scripting:

```javascript
// Ejecutar Planning Sequence de BPC
PlanningSequence_1.getBpcPlanningSequenceDataSource().setVariableValue("CATEGORY", "PLAN");
PlanningSequence_1.execute();
```

### Conexiones live a BW

Para modelos con conexion live a BW/4HANA o BW:

- Los datos no se copian a SAC; las queries se ejecutan en BW
- Las variables BW se exponen como variables SAC
- Puedes establecer valores de variables desde script:

```javascript
// Establecer variable BW desde script
var ds = Table_BW.getDataSource();
ds.setVariableValue("0CALMONTH", "202601");
ds.setVariableValue("0COMP_CODE", "1000");
```

> **Limitaciones con conexiones live:**
> - No hay Planning API (planning es solo para modelos SAC nativos o BPC embedded)
> - getResultSet puede tener limitaciones de volumen
> - Las variables deben estar marcadas como "Ready for Input" en BW

### Patron: Seleccionar en SAC, abrir detalle en BW

```javascript
// En onSelect de Table_Summary
var sel = Table_Summary.getSelections();
if (sel.length > 0) {
    var costCenter = sel[0]["CostCenter"];
    var period = sel[0]["CalMonth"];

    // Abrir transaccion BW/SAP GUI via URL
    // (requiere Web GUI o Fiori Launchpad configurado)
    var url = "https://fiori.company.com/sap/bc/ui5_ui5/ui2/ushell/shells/abap/FioriLaunchpad.html"
        + "#CostCenter-analyze?CostCenter=" + costCenter
        + "&Period=" + period;

    Application.openURL(url);
}
```

## Patron: Contexto compartido entre apps SAC

```javascript
// App origen: guardar contexto y abrir app destino
// En onClick de Button_DrillDown
var sel = Table_Main.getSelections();
if (sel.length === 0) {
    Application.showMessage(ApplicationMessageType.Warning, "Selecciona una fila");
    return;
}

var region = sel[0]["Region"];
var year = sel[0]["Year"];
var version = Dropdown_Version.getSelectedKey();

NavigationUtils.openApplication(
    "APP_DRILL_DOWN",
    [
        UrlParameter.create("p_Region", region),
        UrlParameter.create("p_Year", year),
        UrlParameter.create("p_Version", version),
        UrlParameter.create("p_Source", "APP_MAIN")  // Para saber de donde viene
    ]
);
```

```javascript
// App destino: recibir contexto
// En onInitialization
// Las Script Variables p_Region, p_Year, p_Version, p_Source
// ya tienen valores asignados automaticamente

if (ScriptVariable_Region !== "") {
    var ds = Table_Detail.getDataSource();
    ds.setDimensionFilter("Region", ScriptVariable_Region);
    ds.setDimensionFilter("Year", ScriptVariable_Year);
    ds.setDimensionFilter("Version", ScriptVariable_Version);

    Text_Breadcrumb.setText("Desde: " + ScriptVariable_Source + " > " + ScriptVariable_Region);
}

// Boton para volver
// En onClick de Button_Back
NavigationUtils.openApplication(ScriptVariable_Source, []);
```

## Pitfalls comunes

1. **Prefijo p_ obligatorio:** Olvidar el prefijo `p_` en `UrlParameter.create()` hace que el parametro no se mapee a la Script Variable destino.

2. **Script Variable no expuesta:** Si la Script Variable destino no tiene marcada "Expose variable via URL parameter", el valor se pierde silenciosamente.

3. **IDs de aplicacion/story:** Los IDs son strings como `"t.1.ABC123DEF456"`. Un caracter incorrecto y la navegacion falla. Copia el ID desde la URL de la aplicacion.

4. **openURL y popups bloqueados:** Los navegadores pueden bloquear `openURL()` si no se ejecuta como resultado directo de una accion del usuario (click). No llames openURL desde onInitialization o onTimer.

5. **Encoding de parametros URL:** Valores con caracteres especiales (espacios, &, =) pueden romper la URL. SAC codifica automaticamente en la mayoria de casos, pero verifica con valores complejos.

6. **Live connections y variables:** No todas las variables BW se exponen en SAC. Solo las marcadas como "Ready for Input" son accesibles via `setVariableValue()`.

7. **CORS para embedding:** Si embedes SAC en un iframe externo y no has configurado CORS, el iframe mostrara una pagina en blanco sin error visible.
