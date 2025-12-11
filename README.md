# ALLOPS – Flujo de alta de proyectos CapEx (Power Automate)

Automatización del alta de proyectos de inversión (CapEx) en ALLOPS utilizando Power Automate, SharePoint Online y Microsoft 365.  
El flujo crea la estructura documental base del proyecto, genera enlaces útiles (Gantt, MS Project, cost tracking, dashboard, carpeta compartida, etc.) y envía un correo automático de bienvenida al responsable del proyecto.

---

## Objetivo

Reducir el trabajo manual al crear un nuevo proyecto CapEx:

- Estandarizar la estructura de carpetas y archivos iniciales.
- Centralizar la información del proyecto en una lista maestra de SharePoint.
- Proveer al usuario final todos los enlaces clave en un solo correo automático.
- Facilitar el seguimiento de costos y la planificación (Gantt / MS Project / dashboard).
- Proveer un medio de comunicación (canal de MS Teams) para todos los miembros del proyecto y el equipo de ingeniería

---

## Tecnologías utilizadas

- **Power Automate (cloud)**
- **SharePoint Online**
- **Microsoft Forms / Power Apps** (origen de los datos del proyecto)
- **Outlook 365** (envío de correos)
- **REST API de SharePoint** (`CreateCopyJobs`) para copiar plantillas de carpetas/archivos.

---

## Arquitectura general

Flujo de alto nivel:

1. **Disparador**  
   - Envío de formulario (Forms) o envío de datos desde una Power App.
   - Se reciben campos como: país / compañía, ID interno del proyecto, nombre del proyecto, nombre del solicitante, complejidad, etc.

2. **Inicialización de variables de configuración**  
   - Objeto `cfg` con enlaces base del proyecto (por ejemplo `varLink` a la ficha del proyecto en ALLOPS).
   - Objeto `cfg2` con enlaces derivados:
     - `ganttlink` – Gantt básico en SharePoint.
     - `mpplink` – Archivo de MS Project precargado según complejidad.
     - `ctlink` – Archivo de cost tracking.
     - `dblink` – Tablero (dashboard) de seguimiento.
     - `shrd_fold_link` – Carpeta compartida con externos.
     - `bluefld_link` – “Carpeta azul” con estructura estándar del proyecto.

3. **Creación de estructura documental**  
   - Acción HTTP → `/_api/site/CreateCopyJobs?copy=true`  
   - Copia de carpetas/archivos desde una ruta de **templates** a una nueva ruta específica del proyecto.

4. **Actualización / creación de item en lista maestra**  
   - Lista SharePoint (por ejemplo: `PE Projects Master list`).
   - Se almacenan metadatos del proyecto y los principales enlaces generados.

5. **Envío de correo automático al responsable**  
   - Asunto dinámico:  
     `@{triggerBody()?['text_3']}_@{triggerBody()?['text_1']}_@{triggerBody()?['text_4']}`
   - Cuerpo HTML con:
     - Saludo personalizado usando el **primer nombre** del solicitante.
     - Enlace principal al proyecto en ALLOPS.
     - Enlaces a instructivos (RPA, sincronización con MS Project, actualización de cost tracking, etc.).
     - Enlaces a Gantt, MS Project, cost tracking, dashboard, carpeta compartida y carpeta azul.

6. **Registro / manejo básico de errores**  
   - Acciones de registro (log) en caso de fallo de la API de copia o de actualización en la lista.

Para más detalle, ver [`docs/arquitectura.md`](docs/arquitectura.md).

---

## Estructura del repositorio

```text
.
├── README.md                 # Descripción general del proyecto
├── flows/
│   └── <CPXProvisioningChild-292B8380-6DC1-F011-BBD2-6045BD9F321D>.json   # Export del flujo de Power Automate CHILD
│   └── <CPXinitializationARParent-CFFF159C-D4CA-F011-8544-7CED8D592B6B>.json  # Export del flujo de Power Automate PARENT
└── docs/
    └── arquitectura.md       # Documentación técnica de la arquitectura del flujo
