# Analisis de tickets: Saas, Nube y Portal Adhoc

Periodo analizado: 30/01/2026 00:00 ART al 30/04/2026 12:46 ART.

Producto considerado: "Saas, Nube y Portal Adhoc", dentro de "Su relacion con Adhoc", con equipo tecnico DevOps.

## Criterios de seleccion

- Tickets creados dentro del periodo analizado.
- Tickets asociados al producto "Saas, Nube y Portal Adhoc".
- Se usaron agregados para volumen, equipos, etapas, prioridades, SLA, clientes y temas recurrentes.
- Para temas recurrentes se usaron titulos y etiquetas. Los grupos pueden superponerse porque un mismo ticket puede hablar, por ejemplo, de acceso y base de datos.

## Evidencia cuantitativa

Total: 140 tickets.

Volumen mensual:

- Enero 2026 parcial: 2 tickets.
- Febrero 2026: 46 tickets.
- Marzo 2026: 54 tickets.
- Abril 2026 hasta el corte: 38 tickets.

Estado actual:

- Resuelto: 122.
- Esperando respuesta: 7.
- Cancelada: 6.
- Ongoing: 2.
- En tarea: 2.
- Esperando otra accion: 1.
- Abiertos no cerrados: 10.

Equipo receptor:

- Equipo de Asistencia Funcional: 82 tickets.
- Pregunta Comercial: 35 tickets.
- Bugfix: 18 tickets.
- Pregunta Administrativa: 5 tickets.

Prioridad:

- Alta: 79.
- Media: 50.
- Urgente: 9.
- Baja: 2.

SLA y bloqueo:

- SLA fallido: 18 tickets.
- SLA alcanzado tarde: 16 tickets.
- Bloqueados: 0.
- Rotting: 0.
- No se encontro un dato validado de tickets reabiertos; no se infiere reapertura.

Horas imputadas:

- Total del periodo: 38,48 h.
- Asistencia Funcional: 29,85 h.
- Bugfix: 5,87 h.
- Pregunta Comercial: 2,77 h.
- Pregunta Administrativa: 0 h.

Clientes con mas recurrencia:

- Ediciones Logos SA: 6 tickets.
- oh! Gift Card US LLC: 5 tickets.
- Greenlite SRL: 4 tickets.
- Yoo SRL: 4 tickets.
- Industrias Polimera SA: 4 tickets.
- Bankers SA: 4 tickets.
- Regex LLC, Provar y Reparaciones Android SRL: 3 tickets cada uno.

## Temas repetidos

Los principales grupos detectados por titulo/etiqueta fueron:

- Gestion de bases, backups, bases train/test, restauraciones, duplicaciones o base inactiva: 50 tickets. De estos, 37 entraron por Asistencia Funcional, 6 por Comercial, 5 por Bugfix y 2 por Administrativa.
- Acceso, portal, usuarios, permisos, login, documentacion o cursos: 34 tickets. De estos, 28 entraron por Asistencia Funcional, 4 por Comercial y 2 por Bugfix.
- Suscripcion, licencias, renovacion, facturacion, cotizacion, horas o baja: 25 tickets. De estos, 18 entraron por Comercial, 4 por Administrativa y 3 por Bugfix.
- Casos tecnicos etiquetados como Producto/Sistemas/traceback/error log: 22 tickets.
- Dominio, web, SSL o redireccionamiento: 9 tickets.
- Lentitud, caida, no funciona o servicio no disponible: 8 tickets.
- Seguridad, advisory, CVE o contrasena vulnerada: 6 tickets.

Ejemplos representativos:

- Bases/backups/train: "No me deja crear bases de datos", "Dar de alta BD train", "Backup completo con filestore", "Reestablecer base old", "Base de train borrada - recuperar", "Extensión de base de entrenamiento".
- Acceso/portal/usuarios: "ERROR AL INGRESAR AL PORTAL ADHOC", "Error al acceder al portal Adhoc", "Acceso a portal con link equivocado", "Usuarios con acceso a carga de tickets", "Permisos para usuarios para entrar a Documentación".
- Suscripcion/licencias/facturacion: "Renovar licencias anuales", "Suscripción de Odoo vence en 1 dia", "mensaje suscripción vencida", "Facturacion Adhoc - error empresa pdf", "Comprar horas de consultoria dedicada".

## Oportunidades

1. Reducir tickets de operacion repetitiva de bases con autoservicio guiado y estado visible.
2. Reducir tickets de acceso al portal con un centro de permisos, diagnostico y recuperacion accionable.
3. Separar mejor comunicaciones comerciales/administrativas de alertas tecnicas para evitar tickets que llegan al equipo funcional por dudas prevenibles.
4. Mejorar mensajes preventivos de vencimiento, inactividad, train/test y links de acceso para que el cliente sepa que puede hacer sin abrir ticket.

## Spec funcional 1: Autoservicio guiado para bases, backups y ambientes train/test

Problema actual y evidencia:

El mayor grupo de tickets esta relacionado con bases, backups, ambientes train/test, restauraciones, duplicaciones, base inactiva o vencimientos: 50 de 140 tickets del periodo. Hay ejemplos repetidos como creacion o extension de bases train, recuperacion de bases borradas, backups completos, restauracion de bases old y dudas sobre base inactiva o vencida.

Modulo o producto afectado:

Saas, Nube y Portal Adhoc.

Cambio propuesto:

Incorporar en el portal del cliente una experiencia de autoservicio para bases y backups que muestre, para cada ambiente del cliente:

- Estado visible de cada base: activa, inactiva, por vencer, vencida, train/test, old o pendiente de accion.
- Fecha de expiracion o desactivacion cuando aplique.
- Acciones disponibles segun el estado: solicitar backup, extender train/test, pedir restauracion, solicitar nueva base, consultar base old o pedir revision de inactividad.
- Mensajes claros antes de abrir ticket: que se puede resolver automaticamente, que requiere aprobacion, tiempos esperados y quien debe aprobar.
- Confirmaciones y seguimiento visible para solicitudes ya iniciadas, para evitar tickets duplicados.
- Alertas preventivas antes de vencimientos o borrados, con accion directa desde el portal.

Impacto esperado en recurrencia:

Reducir entre 30% y 40% los tickets del grupo bases/backups/train. Si se toma la base de 50 tickets, el objetivo inicial es evitar al menos 15 tickets trimestrales, especialmente los de extension, creacion, backup y consultas por estado.

Riesgo o tradeoff:

Exponer acciones de infraestructura al cliente exige controles de autorizacion, mensajes muy claros y limites operativos para no habilitar cambios destructivos o ambiguos. Algunas solicitudes van a seguir requiriendo intervencion humana, pero deberian entrar ya clasificadas y con datos completos.

Criterio de validacion:

- En los 90 dias posteriores al lanzamiento, los tickets por palabras clave base, train, backup, restauracion, duplicacion, old, vencida o inactiva bajan al menos 30% contra los 50 tickets del periodo base.
- Al menos 70% de las solicitudes de backup, extension o creacion de train/test se inician desde el portal y no por ticket manual.
- Las solicitudes creadas desde el portal llegan con estado, ambiente, motivo y aprobador identificados.
- No aumenta la cantidad de tickets urgentes asociados a acciones incorrectas sobre bases.

## Spec funcional 2: Centro de acceso, permisos y salud del portal

Problema actual y evidencia:

El segundo grupo mas recurrente agrupa acceso, portal, usuarios, permisos, login, documentacion y cursos: 34 de 140 tickets. Hay ejemplos como errores al ingresar al portal, links equivocados, usuarios sin acceso, permisos para cargar tickets, acceso a documentacion, usuario adicional y problemas de perfil.

Modulo o producto afectado:

Saas, Nube y Portal Adhoc.

Cambio propuesto:

Crear en el portal un centro de acceso y permisos orientado al cliente, con estas capacidades funcionales:

- Ver usuarios/contactos habilitados para cargar tickets, ver documentacion y acceder a recursos contratados.
- Solicitar alta, baja o cambio de permisos de contactos desde un flujo guiado, con aprobacion del responsable definido por el cliente.
- Recuperar acceso con diagnostico visible: usuario no encontrado, invitacion vencida, correo no validado, link incorrecto, permiso faltante o cuenta bloqueada.
- Mostrar links oficiales de portal, documentacion y capacitaciones desde un unico lugar, evitando URLs viejas o mal copiadas.
- Mostrar mensajes de error accionables: que paso, que puede hacer el cliente y cuando corresponde abrir ticket.
- Notificar preventivamente al responsable del cliente cuando haya contactos sin correo validado o permisos inconsistentes.

Impacto esperado en recurrencia:

Reducir entre 30% y 35% los tickets de acceso/portal/usuarios. Sobre 34 tickets del periodo, el objetivo inicial es evitar al menos 10 tickets trimestrales y bajar el volumen de urgentes por imposibilidad de ingreso.

Riesgo o tradeoff:

El principal riesgo es de seguridad y gobierno de accesos: un autoservicio demasiado amplio puede habilitar permisos indebidos. El flujo debe priorizar aprobaciones claras, trazabilidad y mensajes que no expongan informacion sensible.

Criterio de validacion:

- En los 90 dias posteriores al lanzamiento, los tickets por palabras clave portal, acceso, usuario, permiso, login, cuenta, documentacion o curso bajan al menos 30% contra los 34 tickets del periodo base.
- Al menos 70% de altas/cambios de contactos para ticketera o documentacion se gestionan desde el centro de acceso.
- Los tickets que sigan entrando por acceso llegan con diagnostico generado por el portal y no como "no puedo ingresar" sin contexto.
- No aumenta la cantidad de incidentes de permisos asignados incorrectamente.
