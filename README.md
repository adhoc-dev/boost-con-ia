# boost-con-ia

Repositorio para una capacitación de Boost con IA orientada a trabajo asistido con agentes sobre datos de Odoo usando Tuqui MCP.

## Objetivo

El ejercicio consiste en analizar información de Odoo de Adhoc para detectar oportunidades de mejora en los productos del equipo y convertir ese análisis en propuestas accionables.

## Herramientas esperadas

- Codex
- Tuqui MCP
- Skills de apoyo cuando apliquen, por ejemplo para commits

## Configurar Tuqui MCP en Codex

Desde la terminal, ejecutar:

```bash
codex mcp add tuqui --url https://tuqui.ai/mcp/adhoc
```

Luego volver a entrar a Codex y correr `/mcp`. Deberían aparecer las tools correspondientes (`tuqui_context`, `odoo_schema_discover`, `odoo_fields_get`, `odoo_read_group`, `odoo_search_read`, entre otras).

## Enunciado

- identificar los productos `adhoc.product` del equipo;
- analizar los tickets de los últimos 3 meses;
- redactar un informe breve con hallazgos;
- definir 2 especificaciones de mejora para reducir tickets;
- abrir uno o más PRs implementando las specs.

### Entregables esperados

- 1 informe breve con evidencia y hallazgos
- 2 specs priorizadas para bajar recurrencia de tickets
- uno o más PRs implementando las specs
