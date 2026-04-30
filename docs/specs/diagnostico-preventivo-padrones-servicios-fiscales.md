# Diagnostico preventivo de padrones y servicios fiscales

## Contexto

Producto afectado: Producto Localizaciones Argentina, con impacto sobre Liquidacion de Impuestos.

Periodo analizado: 30/01/2026 al 30/04/2026.

Criterio de seleccion: tickets de soporte asociados a Localizaciones / Loc Argentina y Localizaciones / Loc Argentina / Liquidacion de Impuestos.

## Problema actual y evidencia

Los tickets por ARBA, ARCA, AFIP, padrones y servicios fiscales son recurrentes. En el periodo analizado se detectaron:

- 86 tickets relacionados con ARBA o A-122R.
- 52 tickets relacionados con ARCA o AFIP.
- 66 tickets etiquetados como ARBA A-122R.
- 35 tickets con SLA fallido dentro del universo total del producto Argentina.
- 45 tickets con mas de 72 horas de cierre dentro del universo total del producto Argentina.

Ejemplos representativos:

- "NO LEVANTA LAS PERCEEPCIONES ARBA"
- "ARBA 122R - Conexion webservices"
- "18 alertas webservice arba"
- "Importacion Mis Comprobantes ARCA (USD)"
- "[18] Boton Verificar clave ARBA"

El patron indica que muchos tickets nacen cuando el usuario descubre el problema durante la operacion, sin una validacion preventiva clara ni un diagnostico funcional que indique si el origen esta en credenciales, disponibilidad externa, padrones o configuracion.

## Cambio propuesto

Incorporar una experiencia funcional de diagnostico preventivo fiscal para que el usuario pueda validar, antes de operar, si las credenciales, servicios fiscales, padrones y conexiones estan en condiciones.

La experiencia debe permitir:

- Ver el estado de los servicios fiscales relevantes para la operatoria argentina.
- Identificar si una credencial esta ausente, vencida, invalida o pendiente de configuracion.
- Conocer la fecha y resultado de la ultima verificacion exitosa.
- Diferenciar problemas propios de la configuracion del cliente de indisponibilidad temporal de organismos externos.
- Recibir una accion sugerida clara antes de abrir un ticket.
- Acceder a un resumen comprensible para compartir con soporte cuando el problema requiera intervencion.

El diagnostico debe estar redactado en terminos funcionales, evitando mensajes tecnicos crudos como errores internos, trazas o fallas de script.

## Impacto esperado

Reducir la recurrencia de tickets por conexion, credenciales, padrones desactualizados y dudas operativas vinculadas a ARBA, ARCA y AFIP.

Tambien deberia disminuir el tiempo de resolucion de los casos que sigan llegando a soporte, porque el usuario y el equipo de soporte partirian de un diagnostico previo y de una causa probable.

## Riesgo o tradeoff

Puede haber falsos positivos por indisponibilidad temporal de organismos fiscales. El mensaje debe distinguir entre:

- problema de configuracion del cliente;
- indisponibilidad externa;
- falta de informacion suficiente para diagnosticar;
- estado correcto sin accion requerida.

Tambien existe el riesgo de sobrecargar al usuario con controles. Por eso el resumen inicial debe ser simple y permitir ampliar detalle solo cuando haya inconsistencias.

## Criterio de validacion

Durante los 60 dias posteriores a la salida:

- Reducir al menos 30% los tickets nuevos relacionados con ARBA, A-122R, ARCA o AFIP frente al promedio mensual de febrero-abril 2026.
- En los tickets remanentes, soporte debe poder identificar en el primer contacto el estado de credenciales, conexion y ultima consulta.
- Los mensajes visibles para el usuario deben indicar causa probable y proxima accion sin requerir interpretacion tecnica.
