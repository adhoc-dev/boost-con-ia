# Spec tecnica - Monitor de facturacion automatica de suscripciones (MVP)

Fecha: 2026-05-07

Origen funcional: Spec 1 de `docs/specs/ventas-suscripciones-mejoras-asistencia.md`.

## Decision de alcance

Se toma como desarrollo mas facil el monitor de facturacion automatica, porque permite entregar valor temprano sin tocar logica de precio, promociones ni recalculos masivos.

Objetivo MVP:

- Registrar cada corrida del cron de facturacion de suscripciones.
- Clasificar resultados (ok, warning, fail, interrupted).
- Exponer un tablero funcional para soporte y operaciones.
- Disparar aviso interno cuando existan pendientes o errores.

Fuera de alcance MVP:

- Reintentos inteligentes automativos por causa.
- Auto-remediacion de errores externos.
- Cambios en reglas de pricing o wizard de actualizacion masiva.

## Modulos y componentes

Modulo objetivo sugerido: extension sobre `sale_subscription` (en repo de addons custom del equipo).

Componentes tecnicos:

- Modelo cabecera de corrida.
- Modelo detalle por suscripcion procesada/omitida/fallida.
- Wrapper del proceso de cron de facturacion.
- Reglas de clasificacion de omisiones y errores.
- Vistas (list/form/search) para monitoreo.
- Mensaje interno de resumen post-corrida.

## Modelo de datos

### 1) Cabecera de corrida

Entidad propuesta: `subscription.billing.run`

Campos minimos:

- `name` (secuencia legible)
- `started_at`, `finished_at`
- `state`: `running`, `success`, `warning`, `failed`, `interrupted`
- `expected_count`
- `processed_count`
- `omitted_count`
- `invoice_created_count`
- `invoice_draft_count`
- `invoice_pending_validation_count`
- `error_count`
- `omission_counters_json` (resumen por causa)
- `next_action` (texto corto)
- `trigger_type`: `cron` o `manual`
- `trigger_user_id`

Indices sugeridos:

- `(state, started_at)`
- `(started_at)`

### 2) Detalle por suscripcion

Entidad propuesta: `subscription.billing.run.line`

Campos minimos:

- `run_id` (m2o a cabecera)
- `subscription_id`
- `result`: `processed`, `omitted`, `failed`
- `omission_cause`: enum funcional
- `error_type`: enum tecnico acotado
- `invoice_id` (si aplica)
- `message_short` (resumen legible)

Indices sugeridos:

- `(run_id, result)`
- `(run_id, omission_cause)`
- `(subscription_id, run_id)`

### Catalogo de causas funcionales

Valores iniciales para `omission_cause`:

- `missing_configuration`
- `non_billable_state`
- `existing_invoice`
- `fiscal_validation_pending`
- `technical_error`
- `transient_flag`
- `business_rule`

## Flujo tecnico de ejecucion

### Paso A - Inicio de corrida

1. Antes de iniciar el cron, crear cabecera en `running` con `started_at` y `expected_count`.
2. Guardar referencia de corrida en contexto de proceso.

### Paso B - Procesamiento

1. Por cada suscripcion candidata, ejecutar el flujo actual de facturacion.
2. Registrar una linea de resultado (`processed`/`omitted`/`failed`).
3. Si falla una suscripcion, continuar con la siguiente (batch tolerant) y sumar error.

### Paso C - Cierre de corrida

1. Consolidar contadores.
2. Determinar estado final:
   - `success`: sin errores ni omisiones relevantes.
   - `warning`: con omisiones o pendientes manuales.
   - `failed`: error global de corrida o porcentaje de fallas sobre umbral.
   - `interrupted`: cancelacion o corte de proceso.
3. Guardar `finished_at` y `next_action`.

### Paso D - Aviso interno

Publicar mensaje interno solo si estado en `warning`/`failed`/`interrupted`.

Contenido minimo:

- id de corrida
- estado
- totales
- link a vista de corrida

Sin exponer listado de clientes/facturas en el cuerpo del mensaje.

## Vistas y UX

### Lista de corridas

Columnas:

- fecha inicio/fin
- estado
- esperadas/procesadas/omitidas/errores
- facturas creadas/draft/pendientes validacion

Filtros:

- por estado
- por rango de fecha
- por omisiones > 0
- por errores > 0

### Form de corrida

Secciones:

- resumen ejecutivo
- contadores por causa
- smart button a lineas omitidas/fallidas
- trazabilidad (quien disparo y tipo)

### Lineas de corrida

Filtros rapidos:

- `result`
- `omission_cause`
- `subscription_id`

## Seguridad

Permisos sugeridos:

- Grupo soporte funcional: lectura completa de corridas y lineas.
- Grupo operaciones/facturacion: lectura + accion de reintento manual (si existe en fase 2).
- Usuarios generales: sin acceso por defecto.

Regla de privacidad:

- Mensajes internos con resumen agregado, sin detalle sensible.

## Observabilidad y operacion

- Retencion inicial de corridas: 180 dias (parametrizable).
- Limite de lineas por corrida configurable para evitar crecimiento sin control.
- Log tecnico en servidor se mantiene, pero la investigacion funcional debe resolverse desde Odoo.

## Plan de implementacion

Fase 1 (MVP):

1. Modelos + seguridad + secuencias.
2. Hook del cron con apertura/cierre de corrida.
3. Registro basico de lineas por resultado.
4. Vistas list/form/search.
5. Aviso interno con resumen.

Fase 2 (mejora):

1. Umbrales por severidad configurables.
2. Reintento asistido por causa transitoria.
3. Dashboard con metricas historicas.

## Criterios de aceptacion tecnicos

- Cada corrida del cron genera exactamente una cabecera de monitor.
- El estado final nunca queda en `running` cuando el proceso termina.
- Las suscripciones omitidas/fallidas quedan con causa registrada.
- Se puede filtrar en UI por estado y por causa.
- Si hay omisiones o errores, se genera un mensaje interno resumido.
- No se rompe el comportamiento existente de facturacion cuando el monitor esta activo.

## Pruebas minimas

Casos de prueba:

1. Corrida completamente exitosa.
2. Corrida con omisiones por configuracion.
3. Corrida con error tecnico recuperable por suscripcion.
4. Corrida interrumpida.
5. Corrida con facturas pendientes de validacion externa.

Validaciones:

- integridad de contadores
- transicion correcta de estados
- permisos de acceso a vistas
- contenido de mensaje interno sin datos sensibles

## Riesgos tecnicos y mitigaciones

- Riesgo: alto volumen de lineas por corrida.
  Mitigacion: indices + retencion + limites configurables.

- Riesgo: falsa clasificacion de causa.
  Mitigacion: catalogo acotado inicial y ajuste iterativo con soporte.

- Riesgo: ruido de notificaciones.
  Mitigacion: publicar solo en estados no exitosos y con umbrales.
