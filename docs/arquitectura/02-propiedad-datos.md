# Propiedad de datos del sistema RTV

**Estado:** propuesta para revisión del equipo.

**Base de trabajo:** [Catálogo de dominio y servicios](01-dominio-servicios.md) y [referencia del PFC RTV](../../PFC-RTV-referencia-completa-para-GLMR.md), secciones 5, 9 a 11, 17 y 19.

Cada servicio es la **fuente oficial** de los datos que crea y modifica. Tiene una base privada y credenciales propias; ningún otro servicio ni el Gateway consulta o escribe en ella directamente. Las referencias externas se guardan como identificadores, no como claves foráneas entre bases. En E1, las dos bases de la demostración podían residir en una misma instalación PostgreSQL: eso no les quita independencia lógica, pero sí supone una dependencia común de infraestructura.

## Bases privadas y reglas locales

| Servicio y base | Datos de los que es fuente oficial | Reglas de integridad locales |
|---|---|---|
| **`MS-01` Turnos y Pagos** — `rtv_turnos_pagos` (E1) | Turnos, habilitación, estado o confirmación del pago, autorizaciones y claves de idempotencia de autorización. En el diseño ampliado: tarifa operativa y valor aplicado al trámite. | El pago y la autorización se vinculan al turno dentro de la misma base. Una nueva autorización exige turno existente, habilitado y con pago confirmado. En E1 se permite una autorización por turno; una clave ya asociada a otro turno no se reutiliza. Emitir una autorización no modifica el pago. |
| **`MS-02` Inspecciones** — `rtv_inspecciones` (E1) | Inspección, identificador y estado; registro de inicio y claves de idempotencia propias. En el diseño ampliado: catálogo de líneas y asignación de línea a la revisión. | Solo se registra un nuevo inicio tras recibir una autorización válida para el mismo turno. En E1, turno, autorización y clave son únicos entre las inspecciones registradas. La asignación de línea requiere categoría válida; el servicio no la recalcula. |
| **`MS-03` Identidad y Acceso** — `rtv_identidad_acceso` (propuesta) | Cuentas del personal y su estado, credenciales protegidas, sesiones, roles, permisos, asignaciones y registros de auditoría de sus propias operaciones. | Una cuenta inactiva no puede iniciar sesión. Una asignación solo usa roles y cuentas existentes; si algún dato es inválido, no se guarda ningún cambio. Los permisos modificados se aplican desde la siguiente solicitud protegida. Los registros de asignación de roles no se actualizan ni se eliminan desde la aplicación. |
| **`MS-04` Ficha y Clasificación Vehicular** — `rtv_ficha_vehicular` (propuesta) | Fichas e historial por vehículo; placa y período; tipo, uso, ejes y peso; catálogos de clasificación; categoría y fecha de clasificación; indicación de impronta y registros de auditoría de la ficha. | No duplicar una placa normalizada dentro del mismo período. Validar tipo y uso contra catálogo, ejes contra el tipo y peso positivo en kilogramos. Si el peso está fuera del rango esperado para el tipo, no guardarlo sin confirmación. Recalcular la categoría cuando cambien los datos de entrada y no conservar como vigente una categoría que ya no corresponda. Auditar el alta de placa y los cambios de uso. |

## Información que cruza los límites

| Consumidor | Fuente oficial | Información necesaria y uso |
|---|---|---|
| Inspecciones | Turnos y Pagos | Envía el identificador del turno y recibe la autorización correspondiente. Conserva las referencias al turno y a la autorización; no necesita consultar ni replicar el estado de pago para decidir el inicio. |
| Turnos y Pagos | Ficha y Clasificación Vehicular | Consulta la categoría válida para aplicar la tarifa. Puede conservar la categoría y la fecha o referencia de clasificación usadas al calcular el importe, como dato histórico del trámite. |
| Inspecciones | Ficha y Clasificación Vehicular | Consulta la categoría válida para asignar una línea. Conserva una referencia a la versión de ficha usada en la revisión para futuras comparaciones; también puede conservar la categoría empleada como dato histórico. |
| Ficha y Clasificación Vehicular | Inspecciones | Identifica la última revisión pertinente y la ficha o versión utilizada en ella. Compara el uso de esa versión con el actual para detectar si corresponde impronta. La inspección acredita cuál fue la revisión; Ficha Vehicular es dueña del valor de uso. |
| Servicios que protegen operaciones | Identidad y Acceso | Usan la identidad del actor y los permisos vigentes. Un dato de sesión o una copia desactualizada no debe permitir operar con permisos ya revocados. |
| Identidad y Acceso, para la consulta de auditoría | Servicios que registran las acciones | Obtiene los registros de los servicios de origen y aplica los filtros de consulta. Ficha Vehicular conserva los registros de alta de placa y cambios de uso; Identidad conserva los cambios de roles. |

Las copias permitidas son **referencias o instantáneas históricas**, como la categoría con la que se calculó una tarifa. No reemplazan la categoría vigente para una decisión nueva ni otorgan al consumidor el derecho de modificar el dato de origen. No se propone replicar credenciales, permisos como fuente oficial, ni estados de pago. El equipo aún debe acordar cómo identificar la versión de ficha usada en la última revisión y cómo consultar esos datos al asignar una línea.

## Auditoría sin un servicio adicional

1. El servicio que ejecuta el cambio guarda el registro de auditoría en su propia base. Para las operaciones incluidas, se propone guardar el registro y el cambio de negocio en la misma transacción local.
2. Cada entrada incluye identificador del registro, actor, fecha y hora, acción, identificador del objeto afectado y resultado. Para cambios de uso, incluye también el valor anterior y el nuevo. Se excluyen contraseñas y tokens de sesión completos.
3. Identidad y Acceso verifica el permiso de auditoría y ofrece filtros por usuario, fecha y acción. Su consulta no modifica registros de origen ni presenta como completo un resultado al que le falte alguna fuente.

## Límites del diseño

Las reglas de integridad descritas se aplican donde vive el dato. En el flujo de E1, puede quedar una autorización registrada en Turnos y Pagos sin un inicio confirmado en Inspecciones; no se presupone una transacción única entre bases. La recuperación, los contratos de consulta y el manejo de dependencias no disponibles se acuerdan con Kevin y Eduardo. Las reglas de vigencia, cancelación y reinspección no se deducen de las restricciones del prototipo.
