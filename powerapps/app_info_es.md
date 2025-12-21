# CapEx Project Init App (Power Apps)

Aplicación de lienzo (Canvas app) utilizada como front-end para la creación y consulta de proyectos CapEx en **ALLOPS**.  
Se apoya en las mismas listas de SharePoint usadas por los flujos de Power Automate y expone enlaces, dashboards y documentos de forma amigable para el usuario.

---

## 1. Propósito

- Guiar a los Project Managers en la **creación** de un nuevo proyecto CapEx con solo unos pocos campos obligatorios.
- Evitar abrumar al usuario dividiendo la información en **pestañas/secciones**.
- Centralizar el acceso a:
  - Metadatos del proyecto (quién / dónde / cuándo / estado).
  - Documentos del proyecto (carpeta raíz, “carpeta azul”, cost tracking, etc.).
  - Gantt / listas de tareas.
  - Dashboards de Power BI filtrados por **CPX ID**.
  - Canal de Teams y otros recursos de colaboración.

La app es opcional: los flujos pueden ejecutarse usando un formulario estándar de SharePoint, pero la app ofrece una mejor experiencia de usuario y un punto único de trabajo.

---

## 2. Resumen de arquitectura

- **Tipo**: Canvas app (diseño tipo tablet).
- **Archivo**: `CapexProjectInitApp.msapp`
- **Origen de datos principal**: lista maestra de proyectos de SharePoint  
  (por ejemplo `AR Projects Master list`, la misma lista usada por los flujos).
- **Otros recursos**:
  - Bibliotecas de documentos de SharePoint donde viven las carpetas y plantillas de proyecto.
  - Dashboards/tiles de Power BI embebidos vía URL.
  - Enlaces (URLs) almacenados en columnas de la lista y abiertos con `Launch()`.

Flujo de alto nivel desde la perspectiva de la app:

1. El usuario abre la app y:
   - Crea un nuevo proyecto, o
   - Selecciona un proyecto existente.
2. La app escribe/lee en la **Projects Master List**.
3. Los flujos de Power Automate aprovisionan el proyecto (carpetas, listas, Teams, etc.).
4. La app muestra los enlaces actualizados, Gantt/tareas y dashboards para ese proyecto.

---

## 3. Pantallas y navegación

La app se organiza en pantallas/secciones, navegadas mediante una **barra de pestañas horizontal** (galería horizontal):

- **New / Edit Project**
  - Formulario de SharePoint conectado a la Projects Master List.
  - La primera pestaña solo muestra un **conjunto mínimo de campos obligatorios** (por ejemplo):
    - Nombre del proyecto
    - Área organizacional
    - Año fiscal
    - Planta
  - El resto de los campos se distribuyen en otras secciones (detalles, financieros, etc.).
  - La visibilidad de las DataCards se controla para que la vista de “New” se mantenga simple.

- **Project Overview**
  - Vista de solo lectura de los metadatos clave del proyecto seleccionado.
  - Usa el mismo ítem de la lista que el formulario (no se duplica el origen de datos).

- **Links & Documents**
  - Iconos / botones que llaman a `Launch()` con URLs almacenadas en la lista maestra:
    - Carpeta raíz del proyecto
    - Carpeta compartida / “carpeta azul”
    - Archivo de cost tracking
    - Lista de tareas / Gantt
    - Canal de Teams
  - Algunas columnas de SharePoint pueden contener valores *dummy* (por ejemplo `"."`) para permitir formato JSON en las vistas;  
    en la app, el usuario interactúa con **iconos/botones personalizados**, no con ese texto dummy.

- **Analytics (Power BI)**
  - Embebe un **tile de Power BI** que muestra KPIs del proyecto (committed/spent, estado, etc.).
  - El tile se filtra por el **CPX ID** del proyecto actual usando parámetros en la URL.

> Los nombres exactos de pantallas/pestañas pueden ajustarse a tu entorno; el `.msapp` funciona como implementación de referencia.

---

## 4. Orígenes de datos

Conexiones principales esperadas:

- **SharePoint**
  - Sitio: ALLOPS (o equivalente en tu tenant).
  - Lista: lista maestra de proyectos (por ejemplo `AR Projects Master list`).
  - La app asume que esta lista expone:
    - Metadatos básicos (nombre, país, planta, FY, owner, etc.).
    - Columnas de tipo URL para:
      - `ProjectFolderUrl`
      - `CostTrackingUrl`
      - `DashboardUrl`
      - `TeamsChannelUrl`
      - `TaskListUrl` / `GanttUrl`
      - `BlueFolderUrl` (si se utiliza)

- **Power BI**
  - Uno o más dashboards ya publicados en el servicio de Power BI.
  - Al menos un **tile** con un campo que pueda filtrarse por CPX ID (por ejemplo `CPX ID#`).

Si los nombres de columnas son distintos, las fórmulas dentro de la app deben ajustarse en consecuencia.

---

## 5. Integración con Power BI y filtrado

La app embebe un **tile de dashboard de Power BI** usando una URL del siguiente tipo:

```text
https://app.powerbi.com/embed
  ?dashboardId=<DashboardId>
  &tileId=<TileId>
  &filter=<TableName>/<ColumnName> eq '<CPX_ID>'
