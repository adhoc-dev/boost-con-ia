# AGENTS.md

## Contexto

Este repositorio es para una capacitación de Boost con IA.

El objetivo es que los agentes trabajen usando el MCP de Tuqui para consultar datos de la base de Odoo de Adhoc y producir análisis útiles, concretos y accionables para el equipo.

No asumir que los nombres técnicos de todos los modelos o campos son obvios. Validarlos con las herramientas de Tuqui antes de consultar.

## Objetivo del ejercicio

Tomar como foco los productos `adhoc.product` del equipo en el que trabaja la persona usuaria, analizar los tickets de los últimos 3 meses y producir:

1. Un informe breve con hallazgos.
2. Tres especificaciones de mejora para nuestros módulos, orientadas a reducir recurrencia de tickets y hacer el mantenimiento más sustentable.
3. Un commit por cada especificación.

Si no hay implementación real para commitear en este repo, al menos generar un mensaje de commit propuesto por cada spec usando la skill de commits.

## Forma de trabajo esperada

1. Empezar siempre cargando contexto con `tuqui_context`.
2. Descubrir los modelos relevantes antes de consultar datos.
3. Validar campos con `odoo_fields_get` o `odoo_fields_batch` antes de armar dominios.
4. Usar `odoo_read_group` para agregados, tendencias y top-N.
5. Usar `odoo_search_read` sólo para muestras concretas o detalle puntual.
6. Limitar siempre los resultados y mantener las consultas acotadas.
7. No inventar datos. Si algo no aparece en Odoo, decirlo explícitamente.

## Alcance sugerido del análisis

Buscar y relacionar, como mínimo:

- Productos o módulos del equipo.
- Tickets de soporte o helpdesk de los últimos 3 meses.
- Volumen por producto.
- Temas o categorías repetidas.
- Clientes más afectados.
- Tickets reabiertos, bloqueados o de resolución lenta, si esos datos existen.

Si `adhoc.product` no fuera el nombre técnico exacto del modelo, usar `odoo_schema_discover` para encontrar el modelo correcto.

## Flujo recomendado con Tuqui MCP

1. Ejecutar `tuqui_context`.
2. Usar `odoo_schema_discover` con keywords como `product`, `adhoc.product`, `ticket`, `helpdesk`, `support` e `issue`.
3. Validar los campos de fecha, producto, estado, equipo, categoría y cliente con `odoo_fields_get`.
4. Armar dominios sólo después de confirmar los nombres reales de campos y valores de selección.
5. Obtener agregados de los últimos 3 meses con `odoo_read_group`.
6. Leer una muestra chica de tickets representativos con `odoo_search_read`.
7. Redactar el informe con evidencia concreta.
8. Proponer 3 specs priorizadas por impacto y factibilidad.
9. Generar 1 commit por spec, o al menos 1 mensaje de commit por spec si no hay implementación.
10. Para los commits o mensajes de commit, usar la skill `$git-commit-message` en lugar de definir reglas de formato en este archivo.

## Criterios para las 3 specs

Cada spec debe:

- atacar una causa repetida de tickets;
- reducir soporte manual, ambigüedad o errores operativos;
- indicar módulo afectado;
- indicar problema actual;
- indicar cambio propuesto;
- indicar impacto esperado;
- indicar riesgo o tradeoff;
- indicar cómo validar la mejora.

Preferir mejoras de producto y proceso que prevengan tickets, no sólo arreglos reactivos.

## Formato esperado del informe

Incluir:

- período analizado;
- criterios usados para identificar productos y tickets;
- top problemas detectados;
- evidencia resumida con números;
- oportunidades de mejora;
- 3 specs finales priorizadas.

## Restricciones

- No escribir en Odoo.
- No asumir acceso fuera de Tuqui MCP.
- No usar consultas excesivamente amplias.
- No exponer credenciales ni copiar configuraciones sensibles al repo.

## Cómo agregar Tuqui MCP a Codex

Si Tuqui MCP no está configurado en Codex, preferir este comando:

```bash
codex mcp add tuqui --url https://tuqui.ai/mcp/adhoc
```

Alternativamente, agregar una entrada en `~/.codex/config.toml`:

```toml
[mcp_servers.tuqui]
url = "https://tuqui.ai/mcp/adhoc"
```

Pasos:

1. Ejecutar `codex mcp add tuqui --url https://tuqui.ai/mcp/adhoc`.
2. Si se usa configuración manual, abrir `~/.codex/config.toml` y agregar el bloque de `mcp_servers.tuqui`.
3. Reiniciar Codex.
4. Verificar que las herramientas de Tuqui estén disponibles, por ejemplo intentando usar `tuqui_context`.

Si ya existe una sección `mcp_servers.tuqui`, no duplicarla: revisar o actualizar su `url`.
