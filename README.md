# boost-con-ia

Repositorio para una capacitación de Boost con IA orientada a trabajo asistido con agentes sobre datos de Odoo usando Tuqui MCP.

## Objetivo

El ejercicio consiste en analizar información de Odoo de Adhoc para detectar oportunidades de mejora en los productos del equipo y convertir ese análisis en propuestas accionables.

El foco sugerido es:

- identificar los productos `adhoc.product` del equipo;
- analizar los tickets de los últimos 3 meses;
- redactar un informe breve con hallazgos;
- definir 3 especificaciones de mejora para reducir tickets;
- generar 1 commit o mensaje de commit por cada spec.

## Herramientas esperadas

- Codex
- Tuqui MCP
- Skills de apoyo cuando apliquen, por ejemplo para commits

## Configurar Tuqui MCP en Codex

Si Tuqui MCP no está configurado localmente, agregar esto en `~/.codex/config.toml`:

```toml
[mcp_servers.tuqui]
url = "https://tuqui.ai/mcp/adhoc"
```

Luego reiniciar Codex y verificar que estén disponibles herramientas como `tuqui_context`, `odoo_schema_discover`, `odoo_fields_get`, `odoo_read_group` y `odoo_search_read`.

## Estructura del repo

- [AGENTS.md](/home/joa/enterprise/boost-con-ia/AGENTS.md): contexto operativo del ejercicio y lineamientos para agentes
- [`.agents/`](/home/joa/enterprise/boost-con-ia/.agents): skills de proyecto disponibles en el repo

## Entregables esperados

- 1 informe breve con evidencia y hallazgos
- 3 specs priorizadas para bajar recurrencia de tickets
- 3 commits o mensajes de commit asociados a esas specs

## Notas

- No asumir nombres técnicos de modelos o campos sin validarlos en Tuqui.
- Preferir agregados con `odoo_read_group` y usar `odoo_search_read` sólo para detalle puntual.
- No escribir en Odoo desde este ejercicio.
