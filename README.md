# Reto Banorte — Agente financiero que genera su propia UI

App bancaria donde el usuario le pide al agente una pantalla (una meta de ahorro,
un análisis de gastos, un resumen de ingresos) y el agente la **genera como UI
estructurada** —nunca texto libre— usando un catálogo de componentes. El usuario
puede organizarla y exportarla como *widget* a su Inicio.

## Requisitos

- Node.js 20+
- Una API key de Anthropic

## Inicialización

```bash
npm install
```

Crea un archivo `.env.local` a partir de `.env.local.example` y coloca tu key:

```
ANTHROPIC_API_KEY=sk-ant-...
```

Opcional: `MCP_SERVER_URL` (por defecto `http://localhost:3000/api/mcp`).

## Correr el proyecto

```bash
npm run dev
```

Abre http://localhost:3000. El dashboard de Inicio carga datos reales de las
tools del MCP y la pestaña **Agente financiero** genera pantallas.

> El agente llama a las tools del MCP por HTTP contra `/api/mcp`, así que el
> servidor de `npm run dev` debe estar corriendo para que funcione.

### Otros scripts

```bash
npm run chat        # CLI para probar el agente (requiere npm run dev en otra terminal)
npm run typecheck   # tsc --noEmit
npm run lint        # eslint
npm run build       # build de producción
```

El CLI acepta un usuario: `npx tsx scripts/chat-cli.mts u_carlos` (default `u_ana`).

## Arquitectura

```
Usuario (UI Next.js)
   │  prompt / interacción con un componente
   ▼
/api/chat  ──►  Agente (orchestrator.ts)
                   │  loop de tool-use (Anthropic)
                   ├──►  MCP client  ──HTTP──►  /api/mcp (6 tools)
                   │                                  │
                   │                             datos + cálculos puros
                   ▼
             pantalla (AgentScreen: bloques)
                   │
                   ▼
             A2UI (surface)  ──►  Catálogo de componentes  ──►  Widget en Inicio
```

### Usuario

`src/app` es la app bancaria. Tiene dos pestañas (`app-shell.tsx`):

- **Inicio** (`dashboard.tsx`): tarjeta, saldo, acciones rápidas y resumen, más
  los widgets que el usuario exportó. Los widgets se guardan en `localStorage`
  (`src/widgets/store.ts`), se pueden arrastrar/redimensionar y los que dependen
  de datos se refrescan solos cada 60 s reenviando su historial al agente.
- **Agente financiero** (`agent-screen.tsx`): el usuario escribe un prompt, ve la
  pantalla generada, puede pedir "más chico/grande", organizar los bloques
  (arrastrar/redimensionar) y **exportarla como widget**.

El usuario activo es un fixture (`u_ana` en `src/data/fixtures/usuarios.ts`); el
`usuarioId` viaja en cada request para que los datos y la meta de ahorro sean
suyos.

### Agente

`src/agent/orchestrator.ts` es un loop de *tool-use* con Anthropic:

- El modelo **no puede responder con texto libre**: cada turno debe terminar
  llamando la tool `emit_ui`.
- La salida de `emit_ui` se valida contra el schema Zod (`AgentScreenSchema`)
  antes de confiar en ella; si no cumple, se devuelve el error al modelo para
  que lo corrija.
- En cada iteración el modelo puede llamar las tools del MCP para obtener datos
  reales (ingresos, gastos, simulación, meta) y luego emite la pantalla.
- La pantalla resultante (`title`, `reply`, `scale`, `exportable`, `refreshable`
  y `blocks`) se devuelve al frontend vía `/api/chat`.

`src/agent/mcp-client.ts` es el cliente MCP real: habla HTTP contra `/api/mcp`
en vez de importar las funciones del MCP directamente.

### MCP

`src/mcp/` define las herramientas financieras y se expone como **servidor MCP
real** en `src/app/api/mcp/route.ts` con `mcp-handler`. Las 6 tools
(`src/mcp/tools/index.ts`):

| Tool | Qué hace |
| --- | --- |
| `get_income_summary` | Total y promedio mensual de ingresos |
| `get_expense_breakdown` | Desglose de gastos por categoría |
| `get_transactions` | Transacciones crudas en un rango |
| `simulate_savings_goal` | Simula una meta sin persistir |
| `create_savings_goal` | Crea/actualiza la meta (acción persistente) |
| `get_savings_goal` | Consulta la meta activa |

Los **cálculos financieros son puros** (`src/mcp/finance.ts`): el LLM nunca
inventa números, solo los consume. Los contratos de entrada/salida están en
`src/schemas/mcp-tools.ts`. La meta de ahorro se persiste en
`src/data/db.local.json` (`src/data/store.ts`).

### A2UI

`src/a2ui/` implementa un subconjunto del protocolo **A2UI v0.9.1**
(Agent-to-User Interface), que modela la UI como una lista plana de componentes
referenciados por `id` más un data model (`src/schemas/a2ui.ts`).

- `surface.ts`: convierte una pantalla (bloques con `x/y/w/h`) en una *surface*
  con un componente `root` de tipo `Screen` y cada bloque como hijo con su
  `layout`. También serializa la surface a mensajes A2UI (`createSurface`,
  `updateComponents`, `updateDataModel`).
- `catalog.ts`: catálogo propio que mapea cada tipo del catálogo de Banorte a su
  nombre A2UI (`kpi_card` ↔ `KpiCard`), resuelve valores dinámicos (`{ path }`)
  contra el data model y valida props al reconstruir componentes.
- `render.tsx`: `A2uiSurfaceView` reconstruye el árbol desde `root` y lo pinta en
  una grilla de 8 columnas respetando el `layout` de cada componente.

### Componentes

`src/schemas/ui-catalog.ts` es el **contrato** del catálogo: el agente solo puede
usar estos tipos con estas props (nunca HTML/JSX arbitrario).

- **Diálogos** (slot-filling): `number_input_dialog`, `choice_dialog`,
  `confirmation_dialog`.
- **Display**: `kpi_card`, `progress_tracker`, `scenario_comparison`,
  `breakdown_chart`, `trend_chart`, `summary_table`, `slider`, `timeline`,
  `alert_card`, `cta_button`.
- **Fallback** (la demo nunca se rompe): `text_block`, `list_block`.

`src/ui-catalog/registry.tsx` es el dispatcher: valida el JSON crudo contra Zod
y renderiza el componente correspondiente desde `src/ui-catalog/` (carpetas
`dialogs/`, `display/`, `fallback/`). Si el tipo es desconocido o no valida, cae
a `UnknownComponentFallback`. Para agregar un componente nuevo: primero el
schema en `schemas/ui-catalog.ts`, luego el componente y su caso en el registry.
