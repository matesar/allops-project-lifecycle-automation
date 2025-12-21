# Arquitectura – Provisionamiento de proyectos CapEx (ALLOPS)

Este documento describe la arquitectura técnica de la solución de alta/provisionamiento de proyectos CapEx en el entorno **ALLOPS**, construida sobre **Power Automate**, **Power Apps**, **SharePoint Online**, **Power BI**, **Outlook 365** y **Microsoft Teams**.

---

## 1. Alcance y visión general

Cuando un usuario crea un nuevo proyecto CapEx mediante un formulario de SharePoint o la app de Power Apps, el sistema:

1. Registra la solicitud en una lista de SharePoint.
2. Asigna un identificador único al proyecto.
3. Crea la estructura documental estándar a partir de plantillas.
4. Actualiza una **lista maestra de proyectos** con metadatos y enlaces clave.
5. (Opcional) crea un canal de Teams para el proyecto.
6. Envía un correo automático al responsable con todos los enlaces relevantes.
7. Expone la información del proyecto y dashboards de Power BI a través de una app de Power Apps como front end.

El proceso se divide en dos flujos de Power Automate y una app canvas de Power Apps:

- **Flujo PARENT**: `CPXinitializationARParent-…`
- **Flujo CHILD**: `CPXProvisioningChild-…`
- **App de Power Apps**: `CapexProjectInitApp.msapp`

---

## 2. Componentes principales

### 2.1 SharePoint Online

1. **Sitio ALLOPS**  
   Sitio de SharePoint/Teams donde se centralizan los proyectos CapEx.

2. **Lista de proyectos / solicitudes de proyecto** (alta inicial)  
   - Origen del disparador del flujo PARENT.  
   - Columnas típicas:
     - `CompanyISOCod` (país/compañía)
     - `ProjectName` (nombre del proyecto)
     - `InternalID` (ID interno CPX o similar)
     - `Requester` (solicitante / PM)
     - `Complexity` (Baja / Media / Alta)
     - `FiscalYear`
     - Otros campos específicos según la organización.

   > En algunas implementaciones esta lista ya es la **lista maestra de proyectos**; en otras existen una lista de “solicitudes” y una lista “maestra” separada.

3. **Lista maestra de proyectos**  
   Ejemplo de nombre: `Projects Master list`  
   Almacena la información consolidada del proyecto una vez provisionado:
   - ID único de proyecto (ALLOPS Project ID / CPX ID).
   - Metadatos (país, planta, responsable, FY, monto estimado, etc.).
   - Enlaces a:
     - Carpeta raíz del proyecto.
     - Archivo de cost tracking.
     - Gantt / lista de tareas / MS Project.
     - Dashboard de Power BI.
     - Carpeta compartida / “carpeta azul”.
     - Canal de Teams.

4. **Biblioteca de plantillas**  
   Ejemplo de ruta:  
   `/Shared Documents/Templates/ALLOPS/CPX/Issued/Latest/`  

   Contiene la estructura “modelo” de carpetas y archivos:
   - Subcarpetas por área (EHSQ, Ingeniería, Finanzas, etc.).
   - Planilla de cost tracking.
   - Plantilla de MS Project.
   - Instructivos (PDF/Word).

   Esta biblioteca es la **fuente de origen** para la API `CreateCopyJobs`.

---

### 2.2 Power Automate (cloud flows)

1. **Flujo PARENT**  
   Archivo: `CPXinitializationARParent-….json`  

   **Rol:**
   - Recibir la solicitud de proyecto desde SharePoint.
   - Validar datos mínimos.
   - Calcular/generar el ID único del proyecto.
   - Preparar un objeto de configuración con los parámetros necesarios.
   - Invocar al flujo CHILD, pasando los datos del proyecto.

2. **Flujo CHILD**  
   Archivo: `CPXProvisioningChild-….json`  

   **Rol:**
   - Crear la estructura documental del proyecto usando `CreateCopyJobs`.
   - Actualizar la lista maestra de proyectos con enlaces generados y metadatos.
   - Crear un canal de Teams y mensaje de bienvenida (si está habilitado).
   - Enviar el correo automático al responsable del proyecto.
   - Registrar errores y devolver estado.

3. **Conectores principales**
   - `SharePoint` (acciones estándar + llamadas HTTP a `_api`).
   - `Office 365 Outlook` (envío de correos).
   - `HTTP` (Microsoft Graph para Teams).
   - Acciones estándar de Power Automate para scopes, variables, condiciones, bucles, etc.

---

### 2.3 App canvas de Power Apps

**Archivo:** `powerapps/CapexProjectInitApp.msapp`  

Aplicación canvas utilizada por PMs y equipo de ingeniería como front end principal.

**Orígenes de datos:**

- **Lista maestra de proyectos** de SharePoint (origen principal).
- Opcionalmente:
  - Listas de tareas / Gantt por proyecto.
  - Listas adicionales (tablas de referencia, complejidad, plantas, etc.).

**Conceptos clave:**

- Barra de **navegación horizontal** (galería o botones) que actúa como pestañas para cambiar de pantalla:
  - *Nuevo proyecto* (campos mínimos obligatorios).
  - *Detalles del proyecto* (campos extendidos).
  - *Dashboards* (Power BI embebido).
  - *Tareas / Gantt* (lista de tareas de SharePoint).
  - *Recursos y enlaces* (accesos directos a documentos, carpetas, etc.).

- La app se conecta a la misma lista de SharePoint que usan los flujos, por lo que no hay duplicación de datos: un proyecto creado en la app es visible para los flujos y viceversa.

Más detalles en la sección **6. App de Power Apps – pantallas y comportamiento**.

---

### 2.4 Microsoft Teams / Graph

- **Equipo ALLOPS** (Team donde se crean los canales de proyecto).  

El flujo CHILD utiliza **Microsoft Graph** para:

- Crear un **canal por proyecto**:
  - `POST /teams/{team-id}/channels`
  - Nombre del canal basado en el ID de proyecto y el nombre del proyecto.

- Publicar un **mensaje de bienvenida**:
  - `POST /teams/{team-id}/channels/{channel-id}/messages`
  - El mensaje incluye descripción y enlaces principales (SharePoint, cost tracking, dashboard, etc.).

---

### 2.5 Outlook 365

Usado por el flujo CHILD para enviar un correo al PM:

- Saludo personalizado (primer nombre).
- Resumen del proyecto.
- Enlaces a:
  - Sitio/área del proyecto en SharePoint.
  - Carpeta raíz / carpeta compartida.
  - Cost tracking.
  - Gantt / MS Project o lista de tareas.
  - Dashboard.
  - Canal de Teams.
- Enlaces a instructivos (RPA, sincronización, etc.) según corresponda.

---

### 2.6 Power BI

- Dashboards que muestran:
  - Evolución de costos (committed / spent).
  - Estado general del CapEx.
  - Información consolidada por proyecto.

- Algunos **mosaicos (tiles)** se incrustan en la app de Power Apps y se filtran por ID de proyecto (CPX ID) usando parámetros en la URL (`&filter=...`).

---

## 3. Proceso end-to-end (vista general)

1. **Creación de un nuevo proyecto**
   - El PM completa un formulario:
     - directamente sobre la lista de SharePoint, **o**
     - desde la app de Power Apps (pantalla simplificada de “Nuevo proyecto”).
   - Se crea un ítem con los campos mínimos obligatorios.

2. **Ejecución del flujo PARENT**
   - Se dispara con el nuevo ítem de SharePoint.
   - Valida los datos clave.
   - Genera un **ID único de proyecto**.
   - Actualiza el ítem original con ese ID (trazabilidad).
   - Llama al flujo CHILD con todos los parámetros necesarios.

3. **Provisionamiento (flujo CHILD)**
   - Crea la estructura documental desde plantillas mediante `CreateCopyJobs`.
   - Crea o vincula listas de tareas/Gantt y archivos de cost tracking.
   - Actualiza la lista maestra con enlaces y metadatos generados.
   - Crea el canal de Teams y el mensaje de bienvenida (si está habilitado).

4. **Notificación**
   - Envía un correo automático al PM con:
     - Resumen del proyecto.
     - Enlace principal en ALLOPS.
     - Enlaces a instructivos y dashboards.

5. **Uso del front end (Power Apps)**
   - El PM y el equipo utilizan la app para:
     - Ver detalles del proyecto.
     - Abrir todos los recursos asociados desde un único lugar.
     - Visualizar dashboards de Power BI ya filtrados por proyecto.
     - Navegar a listas de tareas / Gantt y archivos de cost tracking.

---

## 4. Flujo PARENT – detalle de funcionamiento

### 4.1 Disparador

- **Trigger:** `When an item is created` (SharePoint).
- Origen: lista de solicitudes / proyectos.
- Los campos del formulario se obtienen desde `triggerBody()`.

### 4.2 Validación y asignación de ID

Pasos típicos:

1. **Lectura de campos clave**
   - País / compañía.
   - Nombre del proyecto.
   - Solicitante (correo).
   - ID interno (si viene informado).
   - Año fiscal, planta, etc.

2. **Cálculo de ID único**
   - Consultar la lista maestra de proyectos.
   - Contar ítems existentes o leer la última secuencia.
   - Generar un ID secuencial (ej. `AR-2025-0012` o `2025ARIT083`).
   - Implementar lógica de reintento (`Do until` / condiciones) para evitar colisiones.

3. **Actualización del ítem de origen**
   - Escribir el ID generado en el ítem original de la lista.

### 4.3 Llamada al flujo CHILD

- Acción: **Run a child flow**.
- Parámetros típicos:
  - ID único del proyecto.
  - Campos de la solicitud (país, planta, nombre, responsable, FY, etc.).
  - Rutas de origen/destino para plantillas.
  - Correos del PM y otros interesados.
  - Flags para “crear canal de Teams”, “enviar correo”, etc. (si se usan).

---

## 5. Flujo CHILD – detalle de funcionamiento

### 5.1 Parámetros de entrada

El flujo CHILD recibe, como mínimo:

- `ProjectId` (ID único generado por el PARENT).
- `ProjectName`.
- `CountryCompany`.
- `RequesterEmail` / `PmEmail`.
- `FiscalYear`.
- Rutas de SharePoint:
  - Biblioteca de plantillas.
  - Biblioteca / carpeta destino del proyecto.
- URLs de instructivos (si se usan en el correo).
- Flags opcionales: crear canal de Teams, enviar correo, etc.

### 5.2 Creación de estructura documental (`CreateCopyJobs`)

1. **Construcción del payload JSON**
   - Cuerpo de la llamada a `CreateCopyJobs` con:
     - `exportObjectUris`: rutas de carpetas/archivos de origen (plantillas).
     - `destinationUri`: carpeta destino específica del proyecto.

2. **Llamada HTTP**
   - Método: `POST`
   - URI: `/_api/site/CreateCopyJobs?copy=true`
   - Cabeceras:
     - `Accept: application/json;odata=nometadata`
     - `Content-Type: application/json;odata=nometadata`

3. **Resultado**
   - Se obtiene la URL de la carpeta raíz del proyecto y subcarpetas relevantes para usar en pasos siguientes y guardarlas en la lista maestra.

### 5.3 Canal de Teams y mensaje de bienvenida

1. **Creación del canal**
   - Llamada HTTP a Microsoft Graph:
     - `POST /teams/{team-id}/channels`
   - El nombre del canal suele combinar el ID del proyecto y el nombre del proyecto.

2. **Mensaje de bienvenida**
   - `POST /teams/{team-id}/channels/{channel-id}/messages`
   - Contenido:
     - Breve descripción del proyecto.
     - Enlaces a carpeta raíz, cost tracking, dashboard, etc.

### 5.4 Actualización de la lista maestra

- Se actualiza o crea el ítem correspondiente en la **lista maestra de proyectos** con:
  - Metadatos provenientes de la solicitud.
  - URL de la carpeta raíz.
  - URL de la lista/Gantt de tareas.
  - URL de cost tracking.
  - URL del dashboard.
  - URL del canal de Teams.

### 5.5 Envío de correo automático

- Acción: `Send an email (V2)` (Outlook).
- Asunto: dinámico, usualmente incluyendo campos clave (ej. país, planta, nombre del proyecto).
- Cuerpo HTML:
  - Saludo con el primer nombre del PM.
  - Resumen del proyecto (ID, nombre, planta, FY).
  - Lista de enlaces (SharePoint, cost tracking, Gantt, dashboard, Teams).
  - Enlaces a instructivos.

---

## 6. App de Power Apps – pantallas y comportamiento

### 6.1 Conexiones de datos

- Origen principal: **lista maestra de proyectos** de SharePoint.
- Orígenes opcionales:
  - Listas de tareas / Gantt por proyecto.
  - Listas de referencia (plantas, niveles de complejidad, etc.).

### 6.2 Modelo de navegación

- Una **barra de navegación horizontal** (galería o conjunto de botones) en la parte superior actúa como control de pestañas:
  - Cada botón actualiza una variable (por ejemplo, `varCurrentTab`) y ejecuta `Navigate()` hacia la pantalla correspondiente.
  - Ejemplos de pestañas:
    - `Nuevo proyecto`
    - `Detalles del proyecto`
    - `Dashboards`
    - `Tareas / Gantt`
    - `Recursos`

### 6.3 Pantallas principales

1. **Pantalla “Nuevo proyecto” (formulario simplificado)**
   - Muestra solo un **conjunto reducido de campos obligatorios**, por ejemplo:
     - Nombre del proyecto
     - País / compañía
     - Año fiscal
     - Complejidad
     - PM / solicitante
   - Usa un control de formulario de SharePoint en modo **New**, vinculado a la lista maestra.
   - En `OnSuccess` se puede:
     - Navegar a la pantalla de **Detalles** con el registro recién creado seleccionado.

2. **Pantalla “Detalles del proyecto” (formulario extendido)**
   - Formulario completo de SharePoint en modo **Edit** con campos adicionales:
     - Presupuesto, alcance, planta, categoría, etc.
     - Columnas de URL para cost tracking, Gantt, dashboards, “carpeta azul”, etc.
   - Para evitar mostrar valores “dummy” (p.ej. `.` usado en columnas con formato JSON de hipervínculo), la app:
     - Oculta esas columnas en el formulario, y
     - Usa **iconos o imágenes** con `OnSelect` que llaman a `Launch(URLCampo)`.

3. **Pantalla “Dashboards” (Power BI embebido)**
   - Inserta un mosaico de Power BI usando:
     - El control de **Power BI tile**, o
     - Un control de **HTML**/imagen con la URL de embed.
   - La URL incluye el parámetro `filter=` que utiliza el ID CPX del proyecto seleccionado (por ejemplo, variable `varCPXId`) para mostrar métricas solo de ese proyecto.
   - El proyecto seleccionado suele provenir de:
     - Una galería vinculada a la lista maestra, **o**
     - El ítem actual del formulario.

4. **Pantalla “Tareas / Gantt”**
   - Conectada a una lista de tareas de SharePoint (o varias) donde se almacena el Gantt del proyecto.
   - Puede mostrar:
     - Una galería o tabla de tareas.
     - Enlaces para abrir la vista Gantt nativa de SharePoint o el archivo de MS Project.

5. **Pantalla “Recursos / enlaces”**
   - Centro de accesos con iconos/imágenes clicables a:
     - Carpeta raíz del proyecto.
     - Carpeta compartida / “carpeta azul”.
     - Archivo de cost tracking.
     - Instructivos y plantillas del proyecto.
   - Los enlaces se obtienen de las columnas de la lista maestra.

### 6.4 Reutilización y extensión

- Al personalizar formularios de otras listas, se pueden **copiar pantallas** entre apps y volver a vincularlas a nuevos orígenes de datos manteniendo layout y navegación.
- El patrón de navegación por pestañas y el enfoque “ocultar campo dummy, mostrar icono con Launch()” se puede reutilizar en distintas pantallas.

---

## 7. Modelo de datos (resumen)

### 7.1 Formulario / lista maestra de proyectos (campos mínimos recomendados)

- `CapExName` (nombre del proyecto)
- `CPX ID#` (clave principal / ID único de proyecto)
- `Country` / `CompanyISOCod`
- `Plant`
- `Requester` / `PmEmail`
- `FiscalYear`
- `Complexity`
- `CreatedBy`
- Campos de estado (por ejemplo: `Status`, `ProvisioningStatus`)

### 7.2 Campos de enlaces / recursos

Columnas de tipo URL típicas:

- `ProjectFolderUrl`
- `CostTrackingUrl`
- `GanttUrl` o `TasksListUrl`
- `DashboardUrl`
- `TeamsChannelUrl`
- `SharedFolderUrl` (carpeta azul)

Estos campos:

- Son **completados por el flujo CHILD** durante el provisionamiento.
- Son **consumidos por la app de Power Apps** para abrir recursos y filtrar dashboards.

---

## 8. Manejo de errores

- Cada bloque principal de los flujos está envuelto en un **Scope**:
  - `Scope – Create structure`
  - `Scope – Update master list`
  - `Scope – Create Teams channel`
  - `Scope – Send email`

- En caso de error:
  - Se actualiza una variable (por ejemplo `varError`) con el nombre del bloque que falló.
  - El flujo puede:
    - Registrar el error en una lista de logs de SharePoint.
    - Actualizar el ítem de la lista maestra con un estado de error.
    - Enviar un correo de alerta al equipo de soporte.

La app de Power Apps puede opcionalmente leer campos de estado/error desde la lista maestra y mostrarlos al usuario.

---

## 9. Seguridad y permisos

Las cuentas de conexión usadas por los flujos deben tener:

- Permisos de **Edición** sobre:
  - Lista maestra de proyectos.
  - Biblioteca de plantillas.
  - Biblioteca destino de proyectos.
- Permisos para:
  - Enviar correos (Outlook).
  - Crear canales y publicar mensajes en el equipo ALLOPS (Graph / Teams).

Recomendaciones:

- Usar una **service account** dedicada para automatizaciones en lugar de una cuenta personal.
- Restringir quién puede crear nuevos proyectos en la app y/o lista mediante permisos de SharePoint y configuración de compartición en Power Apps.

---

## 10. Parametrización y adaptación

Para adaptar la solución a otro país/sitio:

- Ajustar rutas de SharePoint:
  - URL del sitio.
  - Biblioteca de plantillas.
  - Biblioteca destino de proyectos.
- Actualizar nombres de listas:
  - Requests / lista maestra.
- Revisar y actualizar URLs de instructivos.
- Adaptar el formato del ID único (prefijos por país, año fiscal, etc.).
- En Power Apps:
  - Apuntar los orígenes de datos a las nuevas listas.
  - Ajustar etiquetas de pestañas y campos visibles según requisitos locales.
- En Power BI:
  - Asegurarse de que los filtros usen los nombres correctos de tablas/columnas para el CPX ID y claves de proyecto.

---

