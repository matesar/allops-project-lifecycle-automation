# Arquitectura del flujo de alta de proyectos CapEx (ALLOPS)

Este documento describe la arquitectura técnica de los flujos de alta de proyectos CapEx en el entorno **ALLOPS**, basado en Power Automate, SharePoint Online, Outlook 365 y Microsoft Teams.

---

## 1. Alcance y visión general

Cuando un usuario carga un nuevo proyecto CapEx mediante un formulario en SharePoint, el sistema:

1. Registra la solicitud en una lista de SharePoint.
2. Asigna un identificador único al proyecto.
3. Crea la estructura documental estándar a partir de plantillas.
4. Actualiza una **lista maestra de proyectos** con los metadatos y enlaces clave.
5. Crea un canal en Teams para el proyecto.
6. Envía un correo automático al responsable con todos los enlaces relevantes.

El proceso está dividido en dos flujos de Power Automate:

- **PARENT**: `CPXinitializationARParent-…`
- **CHILD**: `CPXProvisioningChild-…`

---

## 2. Componentes principales

### 2.1 SharePoint Online

1. **Sitio ALLOPS**  
   Sitio de SharePoint/Teams donde se centralizan los proyectos CapEx.

2. **Formulario de proyecto** (alta inicial)  
     
   - Origen del disparador del flujo PARENT.  
   - Columnas típicas:
     - `CompanyISOCod` (País)
     - `ProjectName` (Nombre del proyecto)
     - `InternalID` (ID interno CPX o equivalente)
     - `Requester` (Persona que solicita / PM)
     - `Complexity` (Baja / Media / Alta)
     - `FiscalYear`
     - Otros campos específicos según la organización.

3. **Lista maestra de proyectos**  
   Ejemplo de nombre: `Projects Master list`  
   - Guarda la información consolidada del proyecto una vez generado:
     - ID único del proyecto (ALLOPS Project ID).
     - Metadatos (país, planta, responsable, FY, monto estimado…).
     - Enlace a carpeta raíz del proyecto.
     - Enlace a cost tracking.
     - Enlace a Gantt / MS Project.
     - Enlace a dashboard.
     - Enlace a canal de Teams.

4. **Biblioteca de documentos de plantillas**  
   Ejemplo de ruta:  
   `/Shared Documents/Templates/ALLOPS/CPX/Issued/Latest/`  
   - Contiene la estructura “modelo” de carpetas y archivos:
     - Subcarpetas por área (EHSQ, Ingeniería, Finanzas, etc.).
     - Planilla de cost tracking.
     - Plantilla de MS Project.
     - Instructivos (PDF/Word).
   - Es la fuente de origen para la API `CreateCopyJobs`.

---

### 2.2 Power Automate (cloud flows)

1. **Flujo PARENT**  
   Archivo: `CPXinitializationARParent-….json`  
   Rol:
   - Recibir la solicitud desde SharePoint.
   - Validar datos mínimos.
   - Calcular/generar el ID único del proyecto.
   - Preparar el objeto de configuración.
   - Invocar al flujo CHILD pasando los parámetros necesarios.

2. **Flujo CHILD**  
   Archivo: `CPXProvisioningChild-….json`  
   Rol:
   - Crear la estructura documental mediante `CreateCopyJobs`.
   - Crear el canal de Teams y mensaje de bienvenida.
   - Enviar el correo automático al responsable.
   - Registrar errores y devolver estado.

3. **Conectores principales**
   - `SharePoint` (acciones estándar + HTTP con `_api`).
   - `Office 365 Outlook` (envío de correo).
   - `HTTP` (Microsoft Graph para Teams).
   - Conectores internos de Power Automate para scopes, variables, condiciones, etc.

---

### 2.3 Microsoft Teams / Graph

- **Team ALLOPS** (equipo donde se crean los canales de proyecto).  
- El flujo CHILD utiliza **Microsoft Graph** para:
  - Crear un **canal compartido** por proyecto.
  - Publicar un **mensaje de bienvenida** con los enlaces principales (SharePoint, cost tracking, dashboard, etc.).

---

### 2.4 Outlook 365

- Envío de correo al responsable (PM) con:
  - Saludo personalizado (primer nombre).
  - Resumen del proyecto.
  - Enlaces a:
    - Sitio del proyecto en SharePoint.
    - Carpeta raíz / carpeta compartida.
    - Cost tracking.
    - Gantt / MS Project.
    - Dashboard.
    - Canal de Teams.
  - Enlaces a instructivos (RPA, sincronización, etc.).

---

## 3. Flujo PARENT – Detalle de funcionamiento

### 3.1 Disparador

- **Trigger**: `When an item is created` (SharePoint).
- Origen: lista de solicitudes de proyecto.
- Los campos del formulario se leen desde `triggerBody()`.

### 3.2 Validación y asignación de ID

Pasos típicos:

1. **Leer campos clave**
   - País / Compañía.
   - Nombre del proyecto.
   - Solicitante (correo).
   - ID interno (si ya viene definido).
   - Año fiscal, planta, etc.

2. **Calcular ID único del proyecto**
   - Consultar la lista maestra de proyectos.
   - Contar los elementos existentes.
   - Generar un ID secuencial (ej.: `AR-2025-0012`).
   - Implementar lógica de reintento (Do until / condición) si existe riesgo de colisión.

3. **Actualizar ítem de la solicitud**
   - Guardar el ID único generado en la lista de solicitudes (para trazabilidad).


### 3.3 Llamada al flujo CHILD

- Acción: **Run a child flow**.
- Parámetros típicos que se envían:
  - ID único del proyecto.
  - Campos de la solicitud (país, planta, nombre, responsable, FY, etc.).
  - Rutas de origen/destino para plantillas.
  - Correos del responsable y otros interesados.

---

## 4. Flujo CHILD – Detalle de funcionamiento

### 4.1 Parámetros de entrada

El flujo CHILD espera como entrada, al menos:

- `ProjectId` (ID único generado por el PARENT).
- `ProjectName`.
- `CountryCompany`.
- `RequesterEmail` / `PmEmail`.
- `FiscalYear`.
- Rutas de SharePoint:
  - Biblioteca de plantillas.
  - Biblioteca destino para el proyecto.
- URLs de instructivos (si se usan en el correo).

### 4.2 Creación de estructura documental (CreateCopyJobs)

1. **Construcción de payload JSON**
   - Se arma un cuerpo para `CreateCopyJobs` con:
     - `exportObjectUris`: rutas de carpetas/archivos de origen (plantillas).
     - `destinationUri`: carpeta destino específica del proyecto.
2. **Llamada HTTP**
   - Método: `POST`
   - URI: `/_api/site/CreateCopyJobs?copy=true`
   - Cabeceras:
     - `Accept: application/json;odata=nometadata`
     - `Content-Type: application/json;odata=nometadata`
3. **Resultado**
   - Obtención de la URL de la carpeta raíz del proyecto y/o subcarpetas clave para usar en pasos siguientes.

### 4.3 Creación de canal de Teams y mensaje de bienvenida

1. **Creación de canal**
   - Uso de HTTP con Microsoft Graph:
     - `POST /teams/{team-id}/channels`
   - Nombre del canal basado en ID + nombre del proyecto.
2. **Mensaje de bienvenida**
   - `POST /teams/{team-id}/channels/{channel-id}/messages`
   - Contenido:
     - Breve descripción del proyecto.
     - Enlaces a carpeta raíz, cost tracking, dashboard, etc.

### 4.5 Envío de correo automático

- Acción: `Send an email (V2)` de Outlook.
- Asunto dinámico (ejemplo):  
  `@{triggerBody()?['text_3']}_@{triggerBody()?['text_1']}_@{triggerBody()?['text_4']}`
- Cuerpo HTML:
  - Saludo con primer nombre del PM.
  - Resumen del proyecto (ID, nombre, planta, año).
  - Lista de enlaces (SharePoint, cost tracking, Gantt, dashboard, Teams).
  - Enlaces a instructivos.

---

## 5. Modelo de datos (resumen)

### 5.1 Formulario de proyecto

Campos mínimos recomendados:

- `CapExName`
- `País`
- `FiscalYear`

### 5.2 Lista maestra de proyectos (`Projects Master list`)

- `CapExName` (Nombre del proyecto).
- `CPX ID#` (clave principal).
- `País`
- `Planta`
- `Creado por`
- `PmEmail`
- `FiscalYear`
- `ProjectFolderUrl`
- `CostTrackingUrl`
- `DashboardUrl`
- `TeamsChannelUrl`
- `Status`

---

## 6. Manejo de errores

- Cada bloque principal está envuelto en un **Scope**:
  - `Scope – Create structure`
  - `Scope – Update master list`
  - `Scope – Create Teams channel`
  - `Scope – Send email`
- En caso de error:
  - Se actualiza `varError` con el nombre del bloque donde falló.
  - Se puede:
    - Registrar el error en una lista de logs.
    - Actualizar el ítem de la lista maestra con el estado de error.
    - Enviar un correo de alerta al equipo de soporte.

---

## 7. Seguridad y permisos

- La cuenta (o cuentas de conexión) usadas por los flujos debe tener:
  - Permisos de **Editar** en:
    - Lista maestra de proyectos.
    - Biblioteca de plantillas.
    - Biblioteca destino de proyectos.
  - Permisos para:
    - Enviar correo (Outlook).
    - Crear canales y enviar mensajes en el Team ALLOPS (Graph / Teams).
- Se recomienda usar una **service account** dedicada a automatizaciones, en lugar de una cuenta personal.

---

## 8. Parametrización y adaptación

- Para adaptar el flujo a otro país/sitio:
  - Ajustar rutas de SharePoint (sitio, biblioteca de plantillas, biblioteca destino).
  - Actualizar nombres de listas (Requests / Master list).
  - Revisar URLs de instructivos.
  - Ajustar formato del ID único (prefijos por país, año fiscal, etc.).
  

---

