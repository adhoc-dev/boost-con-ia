# Asistente de control para retenciones, percepciones y TXT

## Contexto

Producto afectado: Liquidacion de Impuestos y Producto Localizaciones Argentina.

Periodo analizado: 30/01/2026 al 30/04/2026.

Criterio de seleccion: tickets de soporte asociados a Localizaciones / Loc Argentina y Localizaciones / Loc Argentina / Liquidacion de Impuestos.

## Problema actual y evidencia

Retenciones, percepciones y archivos de presentacion concentran una parte significativa de la recurrencia del producto. En el periodo analizado se detectaron:

- 160 tickets que mencionan retenciones o percepciones.
- 83 de esos tickets en Liquidacion de Impuestos.
- 77 de esos tickets en Localizaciones Argentina.
- Mas de 213 horas imputadas en tickets vinculados a retenciones o percepciones.
- 59 tickets que mencionan TXT o archivo.

Ejemplos representativos:

- "POS - Percepciones automaticas y retenciones sufridas"
- "RETENCIONES SANTA FE SEGUN PADRON"
- "Percepciones IIBB Ventas"
- "retenciones en pagos"
- "TXT percepciones y RetencionesIIBB"
- "Error en txt ARCIBA"
- "SIRE exportacion TXT No funciona"
- "PROBLEMA CON ARCHIVO TXT POR PERCEPCIONES ARBA"

El patron indica que los usuarios necesitan entender por que una retencion o percepcion fue aplicada o no aplicada, que datos fiscales faltan y si el archivo generado esta listo para presentarse.

## Cambio propuesto

Crear un flujo guiado de control antes de liquidar, revisar o exportar informacion de retenciones, percepciones y archivos TXT.

El flujo debe permitir:

- Validar datos obligatorios antes de generar una liquidacion o archivo.
- Explicar por que se aplico o no se aplico una retencion o percepcion.
- Mostrar comprobantes incluidos, excluidos y observados.
- Identificar diferencias esperadas versus diferencias que requieren correccion.
- Alertar sobre configuraciones fiscales faltantes o inconsistentes.
- Validar el archivo antes de descargarlo o presentarlo.
- Ofrecer un resumen funcional de inconsistencias para soporte cuando el caso no pueda resolverse desde el flujo.

La prioridad es prevenir tickets antes de que el usuario llegue a una presentacion fallida, un calculo dudoso o una diferencia dificil de auditar.

## Impacto esperado

Reducir tickets por calculo de retenciones y percepciones, archivos TXT invalidos, dudas sobre padrones y consultas de auditoria operativa.

Tambien deberia reducir tickets largos, ya que varios casos representativos consumieron muchas horas de soporte y resolucion.

## Riesgo o tradeoff

Agregar controles puede hacer mas largo el flujo para usuarios expertos. Para mitigarlo:

- La validacion inicial debe ser rapida.
- El detalle debe mostrarse solo cuando haya inconsistencias o cuando el usuario quiera auditar.
- El flujo no debe bloquear tareas por advertencias menores si la operacion sigue siendo funcionalmente valida.

## Criterio de validacion

Durante los 60 dias posteriores a la salida:

- Reducir al menos 25% los tickets nuevos relacionados con retenciones, percepciones, TXT o archivos frente al promedio mensual de febrero-abril 2026.
- Reducir la cantidad de tickets de cierre mayor a 72 horas asociados a estos temas.
- En tickets remanentes, soporte debe poder identificar rapidamente comprobantes incluidos, excluidos, causa funcional de la diferencia y validacion previa del archivo.
