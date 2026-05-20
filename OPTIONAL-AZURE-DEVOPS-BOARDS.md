# Lección Opcional: Integración con Azure DevOps Boards

> Esta lección es complementaria al flujo principal del taller. Agrega la capacidad de leer historias de usuario desde Azure DevOps Boards y sincronizar las tareas generadas por Spec Kit de vuelta al board. Requiere acceso a una organización de Azure DevOps con permisos de Contributor o superior.

## Contexto y justificación

En muchos equipos enterprise (gobierno, banca, telco), la planificación vive en Azure DevOps Boards mientras el código vive en GitHub. El flujo estándar de Spec Kit asume que los requerimientos se escriben desde cero en `spec.md`, pero en la práctica esos requerimientos ya existen como User Stories o PBIs en un board.

Esta integración permite dos cosas:

1. **Leer** historias de usuario existentes en ADO Boards y usarlas como input para `/speckit.specify`
2. **Escribir** las tareas generadas por `/speckit.tasks` de vuelta como work items en ADO

Existen dos mecanismos para lograr esto. Ambos son opcionales e independientes entre sí.

---

## Opción A: MCP Server Oficial de Azure DevOps (Recomendada)

Esta es la ruta más robusta. Usa el MCP server oficial de Microsoft (`microsoft/azure-devops-mcp`) que se integra directamente con GitHub Copilot en modo Agente dentro de VS Code.

### Por qué esta opción y no otra

El MCP server oficial es mantenido por Microsoft, soporta lectura y escritura bidireccional de work items, usa autenticación OAuth nativa via Azure CLI (sin PAT tokens), y funciona en cualquier plataforma donde corra VS Code. Las alternativas comunitarias que existen a la fecha solo sincronizan en una dirección (Spec Kit hacia ADO) y tienen actividad de desarrollo mínima.

### Prerrequisitos adicionales

Además de los prerrequisitos del taller principal:

1. **Node.js 20+**: El MCP server corre sobre Node.
   Validar: `node --version`

2. **Azure CLI**: Necesaria para autenticación OAuth.
   - Windows: https://aka.ms/installazurecliwindows
   - macOS: `brew install azure-cli`
   - Linux: https://docs.microsoft.com/cli/azure/install-azure-cli

   Validar: `az --version`

3. **Autenticación activa**:
   ```bash
   az login
   ```

4. **Acceso a Azure DevOps**: Permisos Contributor en el proyecto destino.

### Configuración del MCP Server

Crea (o agrega al existente) el archivo `.vscode/mcp.json` en la raíz del proyecto:

```json
{
  "inputs": [
    {
      "id": "ado_org",
      "type": "promptString",
      "description": "Nombre de la organización de Azure DevOps (ej. 'contoso')"
    }
  ],
  "servers": {
    "ado": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@azure-devops/mcp",
        "${input:ado_org}",
        "-d", "core", "work-items"
      ]
    }
  }
}
```

Una copia de este archivo está incluida en este repositorio en `.vscode/mcp.json`.

El flag `-d` controla qué dominios de herramientas se cargan. Para este taller solo necesitamos `core` (proyectos, equipos) y `work-items` (lectura y escritura de PBIs, User Stories, Tasks). Azure DevOps expone muchas herramientas (repos, pipelines, wikis, builds, tests) y cargar todas satura al modelo con opciones irrelevantes. Si en un escenario futuro necesitas más dominios, agrégalos al array.

### Activar el MCP Server en VS Code

1. Abre VS Code en el proyecto del taller.
2. Abre el panel de GitHub Copilot Chat.
3. Cambia a **Agent Mode** (el ícono de agente en la parte superior del chat).
4. En el menú de herramientas (Tools), activa **ado** (Azure DevOps MCP).
5. La primera vez te pedirá el nombre de la organización de ADO.

### Uso: Leer historias de usuario de ADO Boards

Una vez activo el MCP server, puedes pedirle a Copilot en Agent Mode que lea las historias antes de especificar:

```
@workspace Lee las User Stories asignadas al Sprint actual en el proyecto "BankingAPI" 
de Azure DevOps y genera un resumen estructurado que pueda servir como input 
para /speckit.specify
```

Copilot usará las herramientas del MCP server para consultar los work items y te devolverá el contenido. Puedes entonces usar ese resumen como base del prompt de `/speckit.specify` para que la especificación esté alineada con lo que ya existe en el board.

### Uso: Crear work items en ADO desde tasks.md

Después de generar el backlog con `/speckit.tasks`, puedes pedirle a Copilot que cree los work items:

```
@workspace Lee el archivo specs/001-banking-api/tasks.md y crea un Task en Azure DevOps 
por cada tarea listada, vinculándolos como hijos del PBI "Consulta de Saldo" (ID 12345) 
en el proyecto "BankingAPI".
```

Este flujo es manual e intencional. No hay auto-sync. El asistente del taller decide qué tareas subir y a qué PBI vincularlas.

---

## Opción B: Extensión Comunitaria spec-kit-azure-devops

Esta extensión agrega comandos `/speckit.adosync` directamente al flujo de Spec Kit. Es más sencilla de usar pero tiene limitaciones que conviene tener claras.

### Limitaciones

* Solo sincroniza en una dirección: de Spec Kit hacia ADO. No puede leer historias de ADO para traerlas como contexto.
* Está escrita en PowerShell/Shell, lo cual puede generar problemas en macOS/Linux dependiendo de la configuración del entorno.
* Es un proyecto comunitario con actividad mínima (2 commits, 1 contribuidor, sin issues ni discusión activa).
* No ha sido auditada ni respaldada por el equipo de Spec Kit. La documentación oficial lo dice explícitamente: las extensiones comunitarias son responsabilidad de sus autores.

### Instalación

```bash
# Opción 1: Desde el catálogo comunitario
export SPECKIT_CATALOG_URL="https://raw.githubusercontent.com/github/spec-kit/main/extensions/catalog.community.json"
specify extension add azure-devops

# Opción 2: Directamente por URL
specify extension add azure-devops --from https://github.com/pragya247/spec-kit-azure-devops/archive/refs/tags/v1.0.0.zip
```

Validar instalación:

```bash
specify extension list
```

### Configuración

La primera vez que ejecutes el comando, te pedirá interactivamente:

1. **Organization**: Nombre de tu organización de ADO (ej. "contoso" de `https://dev.azure.com/contoso`)
2. **Project**: Nombre del proyecto en ADO
3. **Area Path**: Ruta del área de trabajo (ej. "BankingAPI\Backend")

La configuración se guarda en `~/.speckit/ado-config.json`.

### Uso

Después de `/speckit.tasks`, ejecuta en el chat de Copilot:

```
/speckit.adosync
```

Te mostrará las historias/tareas encontradas y te dejará elegir cuáles sincronizar.

---

## Cuál usar y cuándo

Si el objetivo es demostrar la lectura de historias desde ADO Boards hacia Spec Kit, la única opción funcional es la Opción A (MCP Server). La extensión comunitaria no tiene esa capacidad.

Si el objetivo es solo empujar tareas generadas hacia ADO, ambas opciones funcionan, pero la Opción A es más confiable y no depende de plataforma.

Para el contexto de este taller de 3 horas, la recomendación es hacer una demo rápida de la Opción A al final del laboratorio (después de la Parte 6 de implementación), mostrando cómo Copilot en Agent Mode puede leer work items y crear nuevos desde el contexto del proyecto.

---

## Referencia: Alternativas en desarrollo

Existe una propuesta activa en la comunidad de Spec Kit (Discussion #1235) para un comando `/speckit.ado` nativo que haría sync bidireccional con reconciliación, estimaciones de tiempo, y validación de jerarquía. A la fecha es un experimento individual y no una extensión publicada, pero merece seguimiento si la integración con ADO es prioritaria para tu organización.

---

## Tiempo estimado adicional

| Actividad | Tiempo |
| --- | --- |
| Setup del MCP Server (config + auth) | 5-10 min |
| Demo: Leer historias de ADO | 5 min |
| Demo: Crear work items desde tasks.md | 5 min |

Total: 15-20 minutos. Si el taller lo permite, puede reemplazar parte del tiempo de "Demo + Cierre" de la Parte 7.
