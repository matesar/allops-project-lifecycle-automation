# ALLOPS – CapEx Project Provisioning (Power Apps + Power Automate)

La solución combina **Power Automate**, **Power Apps**, **SharePoint** y **Power BI** para:
- Crear la estructura documental base del proyecto.
- Centralizar los metadatos del proyecto en una lista maestra.
- Proveer al Project Manager (PM) enlaces clave (documentos, Gantt, cost tracking, dashboard, carpeta compartida).
- Facilitar la comunicación con un canal de Microsoft Teams.
- Ofrecer una interfaz amigable para la creación y consulta de proyectos.

Desarrollado para empresa multinacional para el uso en la región LATAM.

> **Nota:** En este repositorio se usa el nombre **ALLOPS** para referirse al sitio corporativo de SharePoint donde se centralizan los proyectos CapEx. El flujo puede adaptarse a cualquier entorno similar cambiando las listas y rutas de SharePoint.

---

## Objetivo

Reducir el trabajo manual al crear un nuevo proyecto CapEx:

- Estandarizar la estructura de carpetas, archivos iniciales y listas asociadas.
- Centralizar la información del proyecto en una lista maestra de SharePoint.
- Proveer al usuario final todos los enlaces clave en un solo correo automático.
- Facilitar el seguimiento de costos y la planificación y permitir visualización rápida de avance (Gantt / MS Project / dashboard).
- Centralizar la comunicación del proyecto mediante un canal dedicado de Teams.

---

## Componentes de la solución

### 1. Flujos de Power Automate (`flows/`)

Automatizan el alta y la preparación del proyecto:

- **Parent – `CPXinitializationARParent-*.json`**  
  - Se dispara cuando se crea un nuevo ítem en la lista de proyectos.  
  - Valida datos básicos, asigna un **ID único** al proyecto y corre el flujo CHILD.

- **Child – `CPXProvisioningChild-*.json`**  
  - Crea la estructura documental del proyecto en SharePoint (carpetas y archivos desde plantillas).  
  - Actualiza la lista maestra de proyectos con enlaces generados.  
  - Crea (opcionalmente) un canal de Teams y mensaje de bienvenida.  
  - Envía un correo automático al responsable del proyecto con todos los enlaces clave.

Tecnologías utilizadas en los flujos:

- **Power Automate (cloud)**
- **SharePoint Online**
- **Outlook 365** (envío de correos)
- **REST API de SharePoint** (`CreateCopyJobs`) para copiar plantillas de carpetas/archivos.
- **REST API de Microsoft Graph** para crear canal y publicaciones de bienvenida.

Más detalle técnico en [`flows_info`](flows_info).

---

### 2. Aplicación Power Apps (`powerapps/`)

Frontend para PMs y equipo de ingeniería:

- Pantallas orientadas a:
  - **Alta de proyectos** con pocos campos obligatorios (vista simplificada).
  - **Consulta de proyectos** y navegación a enlaces clave.
  - Visualización embebida de **dashboards de Power BI** filtrados por ID de proyecto.
  - Accesos a Gantt, listas de tareas, cost tracking y otros recursos.

Archivos:
- `CapexProjectInitApp.msapp` – export de la app de Power Apps.  
- `powerapps/app_info.md` – detalle funcional y guía de importación (pendiente de completar).

---

### 3. SharePoint Online

La solución asume:

- **Lista maestra de proyectos** (por ejemplo, `AR Projects Master list`):
  - Contiene metadatos del proyecto (país, compañía, ID interno, nombre, complejidad, PM, etc.).
  - Almacena enlaces a:
    - Carpeta documental principal
    - Gantt / lista de tareas
    - Archivo de cost tracking
    - Dashboard de Power BI
    - Carpeta compartida / “carpeta azul”
- **Bibliotecas y plantillas**:
  - Ubicación de las plantillas de carpetas y archivos que el flujo copia al crear el proyecto.

---

### 4. Power BI

- Dashboards que muestran:
  - Avance de costos (committed / spent).
  - Estado general del CapEx.
  - Información consolidada por proyecto.

- Algunos mosaicos (tiles) son **embebidos en Power Apps** y filtrados por ID de proyecto (CPX ID) mediante parámetros en el URL del tile.

---

## Arquitectura general

El flujo de alto nivel es:

1. **Creación de un nuevo proyecto**
   - El PM completa un formulario (lista de SharePoint o app de Power Apps).
   - Se crea un ítem con los datos mínimos del proyecto.

2. **Ejecución del flujo PARENT (Power Automate)**
   - Asigna un ID único al ítem.
   - Llama al flujo CHILD, pasando los datos del proyecto.

3. **Provisionamiento (flujo CHILD)**
   - Crea estructura documental desde plantillas.
   - Genera listas/gantt/cost tracking según corresponda.
   - Actualiza la lista maestra con todos los enlaces.

4. **Comunicación al PM**
   - Envío de correo automático con:
     - Resumen del proyecto.
     - Enlace principal al proyecto en ALLOPS.
     - Enlaces a instructivos y dashboards.

5. **Front-end de consulta**
   - El PM y el equipo usan la app de Power Apps para:
     - Ver detalles del proyecto.
     - Acceder a todos los recursos asociados.
     - Consultar dashboards filtrados por proyecto.

Ver diagrama y detalles técnicos en [`docs/arquitectura.md`](docs/arquitectura.md).

---

## Estructura del repositorio

```text
.
├── README.md                      # Visión general de TODA la solución
├── flows/
│   ├── <CPXProvisioningChild-292B8380-6DC1-F011-BBD2-6045BD9F321D>.json       # Export del flujo de Power Automate CHILD
│   └── <CPXinitializationARParent-CFFF159C-D4CA-F011-8544-7CED8D592B6B>.json  # Export del flujo de Power Automate PARENT
│   └── flows_info                                                             # Detalle de los flujos
├── powerapps/
│   ├── CapexProjectInitApp.msapp          # Export de la app
│   └── app_info.md                        # Detalle de la app
└── assets/
    └── screenshots
    └── docs                         
        └── arquitectura.md                 # Documentación técnica de la arquitectura del flujo
