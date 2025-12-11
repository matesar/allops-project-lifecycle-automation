# ALLOPS – Flujo de alta de proyectos CapEx (Power Automate)

Automatización del alta de proyectos de inversión (CapEx) en ALLOPS utilizando Power Automate, SharePoint Online y Microsoft 365.  
El flujo crea la estructura documental base del proyecto, genera enlaces útiles (Gantt, MS Project, cost tracking, dashboard, carpeta compartida, etc.) y envía un correo automático de bienvenida al responsable del proyecto.

Desarrollado para empresa multinacional para el uso en la región LATAM.

> **Nota:** En este repositorio se usa el nombre **ALLOPS** para referirse al sitio corporativo de SharePoint donde se centralizan los proyectos CapEx. El flujo puede adaptarse a cualquier entorno similar cambiando las listas y rutas de SharePoint.

---

## Objetivo

Reducir el trabajo manual al crear un nuevo proyecto CapEx:

- Estandarizar la estructura de carpetas y archivos iniciales.
- Centralizar la información del proyecto en una lista maestra de SharePoint.
- Proveer al usuario final todos los enlaces clave en un solo correo automático.
- Facilitar el seguimiento de costos y la planificación (Gantt / MS Project / dashboard).
- Proveer un medio de comunicación (canal de MS Teams) para el equipo del proyecto y el área de ingeniería.

---

## Tecnologías utilizadas

- **Power Automate (cloud)**
- **SharePoint Online**
- **Outlook 365** (envío de correos)
- **REST API de SharePoint** (`CreateCopyJobs`) para copiar plantillas de carpetas/archivos.
- **REST API de Microsoft Graph** para crear canal y publicaciones de bienvenida.
  
---

## Arquitectura general

Flujo de alto nivel:

1. **Disparador**  
- Trigger: **When an item is created** en una lista de SharePoint (formulario simple de alta de proyecto).
- El formulario lo completa el **Project Manager** o responsable del proyecto.
- Campos a completar:
  - Nombre del proyecto
  - Area organizacional
  - Año fiscal
  - Planta
- El flujo **PARENT** asigna un ID único al registro de cada país (contando ítems existentes y sumando 1, con lógica de reintento para evitar colisiones).

2. **Inicialización de variables de configuración**  
- Se inicializa un objeto de configuración `cfg` con:
  - Rutas base de SharePoint donde se crearán las carpetas del proyecto.
  - Enlace a la ficha del proyecto en el sitio ALLOPS.
  - URLs de plantillas (Gantt, MS Project, cost tracking, etc.).
- Se inicializa `varError` para registrar en qué parte del flujo se produce un fallo (cada bloque principal está dentro de un **Scope** para facilitar el manejo de errores).

3. **Creación de estructura documental**  
- Acción HTTP a SharePoint:
  `POST /_api/site/CreateCopyJobs?copy=true`
- A partir de una carpeta de **templates** (estructura “modelo”), se copia:
  - La carpeta raíz del proyecto.
  - Subcarpetas por área (por ej. EHSQ, Ingeniería, Finanzas, etc.).
  - Archivos base: planilla de cost tracking, plantilla de MS Project, instructivos, etc.
- La estructura se crea en una ruta específica del proyecto, por ejemplo:  
  `/Shared Documents/ALLOPS/CPX/<Año>/<ID_Proyecto>_<Nombre>/…`

4. **Actualización / creación de item en lista maestra**  
- Se actualiza o crea un registro en una **lista maestra de proyectos** (ej.: `AR Projects Master list`).
- Se guardan:
  - Metadatos clave del proyecto (ID, país, planta, responsable, año fiscal, monto estimado, estado inicial, etc.).
  - Enlaces generados: carpeta raíz del proyecto, cost tracking, Gantt, dashboard, canal de Teams, etc.
- Esta lista funciona como **fuente única de verdad** para consultar el estado y enlaces de cualquier proyecto CapEx de la región LATAM de la empresa.
   
5. **Creación de canal en Teams y correo automático al responsable**

- Con Microsoft Graph se crea un **canal de Teams** para el proyecto (en un Team específico de ALLOPS) y se publica un mensaje de bienvenida con los enlaces principales.
- Se envía un correo al responsable del proyecto:
  - Asunto dinámico (ejemplo):  
    `@{triggerBody()?['text_3']}_@{triggerBody()?['text_1']}_@{triggerBody()?['text_4']}`  
    > Equivale a combinar nombre del proyecto, país/compañía y un ID interno.
  - Cuerpo HTML con:
    - Saludo personalizado usando el **primer nombre** del PM.
    - Enlace principal al proyecto en SharePoint.
    - Enlaces a instructivos (RPA, sincronización MS Project, actualización del cost tracking, etc.).
    - Enlaces directos a:
      - Gantt / MS Project
      - Planilla de cost tracking
      - Dashboard de seguimiento
      - Carpeta compartida del proyecto
      - Carpeta “azul” (documentación accesible para áreas no técnicas, si aplica)
        
6. **Registro y manejo básico de errores**

- Cada bloque crítico (copias de SharePoint, actualización de listas, creación de canal, envío de mail) se ejecuta dentro de un **Scope**.
- Si alguna acción falla:
  - Se actualiza `varError` indicando en qué etapa se produjo el error.
  - Se registran los detalles (por ejemplo en una lista de logs o en el propio ítem del proyecto).
  - Opcionalmente, se puede enviar un correo de alerta al equipo de Ingeniería / IT.

Para más detalle, ver [`docs/arquitectura.md`](docs/arquitectura.md).

---

## Estructura del repositorio

```text
.
├── README.md                 # Descripción general del proyecto
├── flows/
│   └── <CPXProvisioningChild-292B8380-6DC1-F011-BBD2-6045BD9F321D>.json   # Export del flujo de Power Automate CHILD
│   └── <CPXinitializationARParent-CFFF159C-D4CA-F011-8544-7CED8D592B6B>.json  # Export del flujo de Power Automate PARENT
├── assets/
│   └── screenshots
└── docs/
    └── arquitectura.md       # Documentación técnica de la arquitectura del flujo
