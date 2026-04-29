# AGENTS.md

Guía operativa para Codex en este repo. Configuración del MCP y del entorno: ver `README.md`.

## Contexto

Capacitación de Boost con IA. El flujo simula la colaboración entre PO y devs apoyada por agentes:

1. **PO**: usa Tuqui MCP para analizar tickets de los últimos 3 meses sobre los productos `adhoc.product` del equipo y redacta specs **funcionales** (sin referencias a código) que apunten a reducir la recurrencia de tickets.
2. **PO**: sube esas specs a una rama y las entrega a los devs.
3. **Devs**: se pasan a la rama, abren el entorno de desarrollo y, con la spec funcional + contexto del código, redactan una spec **técnica**.
4. **Devs**: implementan la spec técnica y abren el PR.

No asumir nombres técnicos de modelos o campos: validarlos con Tuqui antes de consultar. No escribir en Odoo.

## Skills del repo

Las skills viven en `.agents/skills/`. Usarlas en lugar de inventar formatos.

- `git-branch`: nombres de rama estilo ADHOC (`<version>-<h|t>-<slug>-<nickname>`). Usar al crear ramas para specs o PRs.
- `git-commit`: subjects de commit estilo ADHOC/Odoo (`[ADD|IMP|REF|I18N|FIX|DEL] module_name: summary`). Usar al proponer o redactar commits.
- `skill-creator`: para crear o evaluar nuevas skills si surge la necesidad.

## Modo PO (análisis + specs funcionales)

### Forma de trabajo con Tuqui MCP

1. Empezar siempre con `tuqui_context`.
2. Descubrir modelos relevantes con `odoo_schema_discover` (keywords: `product`, `adhoc.product`, `ticket`, `helpdesk`, `support`, `issue`).
3. Validar campos con `odoo_fields_get` o `odoo_fields_batch` antes de armar dominios.
4. Usar `odoo_read_group` para agregados, tendencias y top-N.
5. Usar `odoo_search_read` sólo para muestras o detalle puntual, con `limit` y lista explícita de campos.
6. Si `has_more` es `true`, paginar con `next_offset`.
7. No inventar datos: si algo no aparece en Odoo, decirlo explícitamente.

### Alcance del análisis

Como mínimo cruzar:

- productos o módulos del equipo;
- tickets de soporte/helpdesk de los últimos 3 meses;
- volumen por producto;
- temas o categorías repetidas;
- clientes más afectados;
- tickets reabiertos, bloqueados o de resolución lenta (si hay datos).

### Informe

Incluir período analizado, criterios de selección de productos y tickets, top problemas con evidencia numérica, oportunidades detectadas y las specs finales priorizadas.

### Specs funcionales (entregable PO)

2 specs, sin referencias a código. Cada una con:

- problema actual y evidencia (volumen, ejemplos);
- módulo o producto afectado;
- cambio propuesto en términos de producto/proceso;
- impacto esperado en recurrencia de tickets;
- riesgo o tradeoff;
- criterio de validación.

Preferir mejoras que **prevengan** tickets sobre arreglos reactivos.

### Entrega al equipo de devs

1. Crear rama con la skill `git-branch`.
2. Commitear las specs con la skill `git-commit`.
3. Pushear la rama y dejarla lista para los devs.

## Modo Dev (spec técnica + implementación)

1. Checkout de la rama del PO con las specs funcionales.
2. Levantar el entorno de desarrollo del módulo afectado.
3. Leer la spec funcional y explorar el código para mapear puntos de cambio.
4. Redactar una spec **técnica** por cada spec funcional: archivos/modelos a tocar, cambios concretos, migraciones o datos, tests, riesgos.
5. Implementar.
6. Para nombres de rama y commits, usar las skills `git-branch` y `git-commit`.
7. Abrir uno o más PRs implementando las specs.

## Restricciones

- No escribir en Odoo desde Tuqui MCP.
- No asumir acceso fuera de Tuqui MCP.
- Mantener consultas acotadas (filtros estrechos, `limit`, campos explícitos).
- No exponer credenciales ni copiar configuraciones sensibles al repo.
