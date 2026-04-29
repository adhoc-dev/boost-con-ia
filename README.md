# Boost con IA

Repositorio para una capacitación de Boost con IA orientada a trabajo asistido con agentes sobre datos de Odoo usando Tuqui MCP.

## Objetivo

El ejercicio consiste en analizar información de Odoo de Adhoc para detectar oportunidades de mejora en los productos del equipo y convertir ese análisis en propuestas accionables.

## Herramientas esperadas

- Codex
- Tuqui MCP
- Skills de apoyo cuando apliquen, por ejemplo para commits

## Clonar el repo

- **POs** (via HTTPS):

  ```bash
  git clone https://github.com/adhoc-dev/boost-con-ia.git
  ```

- **Devs** (via SSH):

  ```bash
  git clone git@github.com:adhoc-dev/boost-con-ia.git
  ```

Luego entrar al directorio y abrir Codex desde ahí:

```bash
cd boost-con-ia
codex
```

## Codex

Si todavía no tenés Codex instalado:

```bash
npm install -g @openai/codex
```

El login debe hacerse con la cuenta de ChatGPT compartida por el equipo (`team-*@adhoc.inc`), no con cuentas personales.

### Configurar Tuqui MCP

El registro del MCP se hace **por fuera de Codex**, desde la terminal:

```bash
codex mcp add tuqui --url https://tuqui.ai/mcp/adhoc
```

Una vez registrado, volver a entrar a Codex con `codex` y correr `/mcp`. Deberían aparecer las tools correspondientes (`tuqui_context`, `odoo_schema_discover`, `odoo_fields_get`, `odoo_read_group`, `odoo_search_read`, entre otras).

## Enunciado

El flujo simula la colaboración entre PO y devs apoyada por agentes:

1. **PO**: analiza los tickets de los últimos 3 meses sobre los productos `adhoc.product` del equipo vía Tuqui MCP, y redacta specs **funcionales** (sin referencias a código) que apunten a reducir la recurrencia de tickets.
2. **PO**: sube esas specs a una rama y las entrega a los devs (usando Codex + skills).
3. **Devs**: se pasan a la rama y, con la spec funcional, abren el entorno de desarrollo.
4. **Devs**: con la spec funcional y el contexto del código, generan una spec **técnica** de implementación.
5. **Devs**: implementan la spec técnica y abren el PR correspondiente.

### Entregables esperados

- 1 informe breve con evidencia y hallazgos del análisis de tickets
- 2 specs funcionales priorizadas para bajar recurrencia de tickets
- 2 specs técnicas derivadas de las funcionales
- uno o más PRs implementando las specs
