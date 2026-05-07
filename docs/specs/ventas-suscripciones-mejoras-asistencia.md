# Specs funcionales - Ventas / Suscripciones

Fecha: 2026-05-07

Fuente del analisis: tickets de `helpdesk.ticket` consultados via Tuqui MCP en modo lectura.

Periodo analizado: 2025-11-07 a 2026-05-07, hora Argentina.

Producto: Ventas / Suscripciones.

Equipos incluidos:

- Equipo de Asistencia Funcional.
- Bugfix.
- Asistencia de actualizacion funcional.

Equipos excluidos:

- Pregunta Comercial.
- Pregunta Administrativa.

Universo filtrado: 155 tickets.

Campos y fuentes revisadas: titulo, descripcion del ticket, descripcion, solucion, solucion vinculada y mensajeria/chatter.

## Contexto del analisis

El analisis de contenido muestra que la recurrencia principal no esta concentrada en dudas comerciales, sino en problemas operativos y funcionales dentro de Odoo relacionados con facturacion automatica, crons, filtros de suscripciones a facturar, actualizacion masiva de precios recurrentes y diferencias entre precio esperado y precio facturado.

Patrones detectados en contenido, descripciones, soluciones y mensajes:

- Facturacion: 90 tickets.
- Error/traceback: 54 tickets.
- Precio/descuento/lista: 41 tickets.
- Automatico: 37 tickets.
- Cron: 21 tickets.
- Migracion/upgrade: 11 tickets.
- Filtro / "a facturar": 10 tickets.
- Script de listas de precios y precios recurrentes: 11 tickets.

## Spec 1 - Monitor de facturacion automatica y recuperacion de cron

### Problema / dolor

Los usuarios y el equipo de soporte no tienen dentro de Odoo una senal clara cuando el cron de facturacion de suscripciones falla, se detiene, omite registros o deja facturas pendientes de validar. El diagnostico termina dependiendo de logs externos, videos, revision tecnica manual o mensajes dispersos en el chatter.

Casos revisados muestran situaciones como:

- Cron que se desactiva o queda interrumpido.
- Suscripciones pendientes de facturar sin causa visible.
- Facturas generadas pero no validadas por errores externos.
- Facturas que se crean o envian automaticamente sin trazabilidad funcional suficiente.
- Necesidad de avisar a soporte o a un canal interno cuando quedan pendientes luego de una corrida.

Cantidad de tickets relacionados:

- Cron: 21 tickets.
- Automatico: 37 tickets.
- Facturacion: 90 tickets.

### Propuesta funcional

Desarrollar dentro de Odoo un monitor de facturacion automatica de suscripciones.

El monitor debe registrar cada corrida del proceso automatico de facturacion y dejar evidencia funcional consultable por usuarios internos con permisos. El objetivo es que soporte pueda responder desde Odoo que paso en una corrida, que suscripciones fueron procesadas, cuales quedaron omitidas y por que.

El monitor debe incluir, como minimo:

- Fecha y hora de inicio de la corrida.
- Fecha y hora de finalizacion.
- Estado final: exitosa, exitosa con advertencias, fallida, interrumpida.
- Cantidad de suscripciones esperadas.
- Cantidad de suscripciones procesadas.
- Cantidad de suscripciones omitidas.
- Cantidad de facturas creadas.
- Cantidad de facturas en borrador.
- Cantidad de facturas pendientes de validacion o autorizacion.
- Cantidad de errores.
- Clasificacion funcional de errores u omisiones.
- Proximo intento o accion esperada, cuando aplique.

Cuando la corrida termine con pendientes, errores o facturas que requieren intervencion, Odoo debe publicar un aviso interno con resumen y acceso al detalle. El aviso no debe exponer listados masivos ni datos sensibles; debe mostrar totales y permitir ingresar al detalle segun permisos.

### Criterios de aceptacion

- Al finalizar cada corrida de facturacion automatica, Odoo crea un registro de monitor con estado, fechas, cantidades y resultado.
- Desde Odoo se puede acceder a una vista/listado de corridas de facturacion automatica de suscripciones.
- El usuario puede abrir una corrida y ver las suscripciones procesadas, omitidas o fallidas segun permisos.
- Cada suscripcion omitida queda asociada a una causa funcional, por ejemplo: configuracion incompleta, estado no facturable, factura previa existente, validacion fiscal pendiente, error tecnico, flag transitorio o condicion de negocio.
- Si una factura se crea pero no se valida por una validacion externa, la factura queda identificada como pendiente de validacion manual con motivo visible.
- Si la corrida termina con error, pendientes u omisiones relevantes, Odoo publica un mensaje interno con resumen y link al registro de monitor.
- El resumen publicado no expone datos sensibles ni listados completos de clientes o facturas.
- El monitor permite filtrar corridas por estado, fecha, cantidad de errores y cantidad de omitidas.
- El monitor permite filtrar suscripciones omitidas por causa.
- Si una corrida queda interrumpida, Odoo debe dejarla en estado interrumpida o fallida, no como exitosa.
- La funcionalidad debe permitir distinguir un problema de configuracion de un incidente que requiere Bugfix.

### Riesgo

Puede generar ruido operativo si cada advertencia produce mensajes internos. Se recomienda definir umbrales, permisos y severidades para evitar saturar canales o usuarios. Tambien existe riesgo de exponer informacion sensible si el detalle no respeta permisos.

## Spec 3 - Wizard de actualizacion masiva de precios recurrentes

### Problema / dolor

La actualizacion de precios recurrentes y listas de precios genera tickets por dispersion de precios, diferencias entre lo esperado y lo facturado, dudas sobre procesos en background, parches puntuales por cliente y falta de visibilidad del avance.

Casos revisados muestran situaciones como:

- Actualizaciones masivas que requieren scripts o parches.
- Usuarios que no saben si el proceso corrio o quedo pendiente.
- Diferencias entre precio de orden de venta, suscripcion y factura.
- Promociones o descuentos que alteran el resultado esperado.
- Necesidad de volver a actualizar precios por dispersion o resultados inconsistentes.

Cantidad de tickets relacionados:

- Precio/descuento/lista: 41 tickets.
- Script de listas de precios y precios recurrentes: 11 tickets.
- Facturacion: 90 tickets, como impacto posterior cuando el precio recurrente termina facturandose.

### Propuesta funcional

Desarrollar dentro de Odoo un wizard de actualizacion masiva de precios recurrentes de suscripciones.

El wizard debe permitir simular, revisar, confirmar y ejecutar cambios masivos de precios recurrentes con trazabilidad funcional. Para lotes grandes, la ejecucion debe correr en background con estado visible para el usuario.

El proceso debe tener cuatro etapas funcionales:

1. Seleccion del universo de suscripciones afectadas.
2. Simulacion de cambios antes de aplicar.
3. Confirmacion y ejecucion del cambio.
4. Resumen final con resultados, omisiones, errores y advertencias.

La simulacion debe mostrar el precio actual, el precio esperado, la variacion y las reglas que afectan el resultado, especialmente listas de precios, descuentos o promociones vigentes.

### Criterios de aceptacion

- El usuario puede abrir un wizard desde Odoo para actualizar precios recurrentes de suscripciones.
- El wizard permite seleccionar el universo de suscripciones a actualizar usando criterios funcionales.
- Antes de aplicar cambios, Odoo muestra una simulacion con cantidad de suscripciones afectadas.
- La simulacion muestra precio actual, precio nuevo esperado y variacion.
- La simulacion advierte si existen promociones, descuentos, listas de precios o condiciones activas que pueden modificar el resultado.
- El usuario debe confirmar explicitamente antes de aplicar cambios.
- Para lotes grandes, la actualizacion se ejecuta en background y muestra estado, progreso y fecha/hora de inicio.
- El usuario puede consultar el avance del job desde Odoo.
- Al finalizar, Odoo muestra cantidad de suscripciones actualizadas, omitidas y fallidas.
- Cada omision o falla debe tener una causa funcional visible.
- Si la variacion detectada queda fuera de una tolerancia definida, Odoo alerta antes de permitir facturar esas suscripciones.
- El resultado final queda trazado con usuario ejecutor, fecha, criterio usado y resumen de cambios.
- Soporte puede descargar o consultar un resumen acotado para diagnostico sin revisar logs externos.
- La funcionalidad no debe modificar facturas ya emitidas.
- Si existe una necesidad de rollback, solo debe estar disponible para cambios no facturados o bajo una accion controlada con permisos especificos.

### Riesgo

El principal riesgo es generar inconsistencias si se permite revertir o modificar precios que ya impactaron en facturas emitidas. Tambien puede haber resultados confusos si las reglas de listas de precios, descuentos o promociones legacy no estan completamente modeladas en la simulacion.

