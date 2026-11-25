# Dominio y microservicios del sistema RTV

**Estado:** propuesta para revisión del equipo.

**Base de trabajo:** [Referencia del PFC RTV](../../PFC-RTV-referencia-completa-para-GLMR.md), secciones 2 a 10, 14 y 17. Allí se enlazan los requisitos, diagramas y evidencias de E1.

## Alcance del diseño

El Centro de Revisión Técnica Vehicular del Municipio de Valencia necesita coordinar el trabajo de atención, recaudación e inspección. En el proceso documentado, recepción gestiona el turno, recaudación confirma el pago y el personal técnico ejecuta la revisión. La regla crítica es que **no se inicia una inspección sin autorización para un turno existente, habilitado y con pago confirmado**.

El prototipo de E1 demuestra la autorización y el registro del inicio entre dos servicios. El taller conserva ese flujo y diseña, además, el acceso del personal, la auditoría y la ficha vehicular. La gestión completa de turnos, el tarifario y la asignación de línea se describen donde afectan al diseño, sin presentarlos como funcionalidades verificadas en el prototipo. La presencia de otras funciones en el código de origen no las incorpora al alcance: las integraciones con ANT o SRI, las reinspecciones y la revocación de autorizaciones quedan pendientes de acuerdo.

| Actor | Participación relevante |
|---|---|
| Administrador autorizado | Asigna roles y, si tiene permiso de auditoría, consulta las acciones registradas. |
| Personal de recepción | Gestiona el turno y coordina la asignación de línea. |
| Personal de recaudación | Registra la confirmación del pago del trámite. |
| Inspector técnico | Gestiona los datos de la ficha e inicia la inspección cuando se cumplen las condiciones. |
| Propietario y conductor | Participan en el trámite; la documentación revisada no les asigna cuentas en este diseño. |

## Alternativas de descomposición

| Alternativa | Límites | Ventajas | Costos y límites |
|---|---|---|---|
| **A. Cuatro servicios — recomendada** | Turnos y Pagos; Inspecciones; Identidad y Acceso; Ficha y Clasificación Vehicular. Los registros de auditoría se conservan donde se realiza cada acción y se consultan desde Identidad y Acceso. | Conserva el flujo crítico de E1. Permite guardar el cambio de negocio y su registro de auditoría en la misma transacción local. No añade otro despliegue para auditoría. | La consulta unificada debe combinar filtros y resultados de distintas fuentes e informar si alguna no puede responder. |
| **B. Cinco servicios — alternativa** | Los cuatro anteriores más un servicio independiente de Auditoría, con base propia para la consulta centralizada. | Separa las consultas de auditoría de la operación diaria y facilita filtrar y ordenar en un único almacén. | Añade un despliegue y exige asegurar la entrega de registros, manejar duplicados y definir cuánto retraso se acepta antes de que una acción sea visible. |

Se propone **A** porque ya cubre la necesidad de auditar cambios de roles y de fichas sin añadir otro servicio. Turnos, pagos y autorización se mantienen juntos para que la elegibilidad se resuelva sobre datos propios. La inspección se mantiene separada porque su estado técnico no es el estado administrativo del turno. Ficha y clasificación permanecen juntas: los cambios de tipo, ejes y peso afectan directamente el cálculo de la categoría. Separarlos añadiría coordinación para una regla que puede mantenerse en una sola base.

## Catálogo y componentes internos

Los identificadores son propuestos para que el equipo utilice los mismos en el catálogo, los contratos y los diagramas. Los componentes son lógicos; no representan clases ni endpoints definitivos.

| ID y servicio | Responsabilidad y operaciones principales | Componentes propuestos y dependencias |
|---|---|---|
| **`MS-01` Turnos y Pagos** | Gestionar el estado administrativo del trámite: registrar y consultar turnos, registrar confirmaciones de pago y autorizar el inicio para turnos elegibles. Autorizar no equivale a cobrar. | Entrada de operaciones; gestión de turnos y pagos; validador de elegibilidad y autorización; repositorio privado. Para aplicar la tarifa cuando corresponda, necesita la categoría oficial emitida por Ficha Vehicular. |
| **`MS-02` Inspecciones** | Controlar el inicio y estado de la inspección. Solicitar la autorización, registrar el inicio y consultar la inspección. En la evolución del diseño, administrar líneas y su asignación técnica. | Entrada de operaciones; control de inicio e idempotencia; integración con Turnos y Pagos; componente de asignación de línea que consume la categoría de Ficha Vehicular; repositorio privado. |
| **`MS-03` Identidad y Acceso** | Autenticar al personal, gestionar sesiones y asignaciones de roles y resolver permisos vigentes. Ofrecer la consulta unificada de auditoría administrativa, sin apropiarse de los registros que generan otros servicios. | Autenticación y sesiones; gestión de roles y permisos; registro local de cambios de acceso; consulta de auditoría con filtros; repositorio privado. Para la consulta unificada, obtiene los registros de los servicios que originaron las acciones. |
| **`MS-04` Ficha y Clasificación Vehicular** | Registrar y consultar la ficha del vehículo; validar placa, tipo, uso, ejes y peso; detectar cambios de uso y calcular la categoría. | Entrada de operaciones; gestión de ficha e historial; validación de catálogos; cálculo de categoría; registro local de auditoría; repositorio privado. Para comparar con la última revisión, usa la referencia a la ficha utilizada en la revisión anterior, no simplemente la última edición. |

El Gateway es un punto de entrada, no un servicio de negocio ni dueño de estos datos. La verificación de permisos no se limita al menú ni al Gateway: cada servicio protege su operación y aplica sus reglas de negocio.

## Matriz de trazabilidad funcional

Se emplean nombres descriptivos para identificar los diez requisitos de la referencia del PFC. Los colaboradores aportan datos o aplican controles, sin asumir la responsabilidad principal.

| Requisito | Responsable principal | Colaboradores | Criterio de cobertura en el diseño |
|---|---|---|---|
| Autenticar al personal | Identidad y Acceso | — | Iniciar sesión solo con cuenta activa y credenciales válidas; no crear sesión con credenciales inválidas o incompletas. |
| Asignar roles a una cuenta | Identidad y Acceso | — | Solo un administrador autorizado puede asignar roles existentes a una cuenta registrada. Un cambio inválido no modifica asignaciones; un cambio confirmado se aplica desde la siguiente solicitud protegida. |
| Restringir operaciones por permisos | Identidad y Acceso | Turnos y Pagos, Inspecciones y Ficha Vehicular | Identidad determina los permisos vigentes; el servicio que recibe la operación aplica la restricción incluso si la llamada no proviene del menú. |
| Consultar el registro de acciones | Identidad y Acceso | Ficha Vehicular, para las acciones de ficha | El administrador con permiso consulta por usuario, fecha o acción. El registro incluye actor, fecha y hora, acción, objeto y resultado; no se edita ni se elimina desde la aplicación y no contiene contraseñas ni tokens completos. |
| Registrar la placa en la ficha | Ficha y Clasificación Vehicular | Identidad y Acceso | Exigir placa válida, asociarla a la ficha y evitar duplicados por placa en el mismo período. Registrar el alta para auditoría. |
| Registrar el tipo de vehículo | Ficha y Clasificación Vehicular | — | Usar un tipo obligatorio del catálogo; si cambia, recalcular la categoría antes de la asignación de línea. |
| Registrar el uso del vehículo | Ficha y Clasificación Vehicular | Inspecciones, para identificar la revisión anterior; Identidad y Acceso, para la consulta de auditoría | Exigir un uso del catálogo, indicar que requiere impronta si difiere del de la revisión anterior y auditar el cambio con el valor anterior y el nuevo. |
| Registrar el número de ejes | Ficha y Clasificación Vehicular | — | Validar que sea entero, esté dentro del rango acordado y sea compatible con el tipo; recalcular la categoría al modificarlo. |
| Registrar el peso | Ficha y Clasificación Vehicular | — | Exigir un valor positivo en kilogramos; pedir confirmación si está fuera del rango esperado para el tipo y recalcular la categoría al modificarlo. |
| Clasificar automáticamente el vehículo | Ficha y Clasificación Vehicular | Turnos y Pagos, para la tarifa; Inspecciones, para la línea | Aplicar la norma NTE INEN 2656 a tipo, uso, ejes y peso; registrar categoría y fecha. Indicar los datos faltantes o la inconsistencia y no asignar línea sin una categoría válida. |

### Flujo crítico documentado en E1

Turnos y Pagos es responsable de comprobar elegibilidad y emitir la autorización. Inspecciones es responsable de confirmar el inicio después de guardar su propio registro. La justificación de ambos servicios se apoya en el objetivo del PFC, el contrato de autorización y las evidencias de E1 (secciones 3, 10 y 14 de la referencia); estas operaciones no se presentan como otro requisito funcional numerado.

## Restricciones no funcionales que orientan el diseño

| Restricción | Consecuencia para el diseño |
|---|---|
| Rendimiento del inicio | El objetivo de E1 es un p95 local de hasta 100 ms con 1, 5 y 10 usuarios simultáneos. No se atribuye esa medición al nuevo recorrido con Gateway. |
| Fallos o demora al autorizar | En E1, Inspecciones espera hasta 3 segundos por la dependencia. Si no puede confirmar una autorización válida, no registra un nuevo inicio. |
| Integridad de turno y pago | Un turno inexistente, inhabilitado o con pago no confirmado no genera una autorización válida ni una inspección. |
| Reintentos y concurrencia | Repetir la misma operación debe recuperar la autorización ya emitida; solicitudes simultáneas no deben duplicar la inspección. |
| Persistencia tras reinicios | Cada servicio conserva los registros confirmados con sus identificadores y estados en su base privada. La evidencia de E1 corresponde a los dos servicios del prototipo. |
| Acceso entre servicios | La autorización de inicio rechaza solicitudes sin una credencial de servicio válida sin cambiar el estado de negocio. Esto es distinto del inicio de sesión del personal. |

## Acuerdos pendientes

- Validar con el equipo la responsabilidad de Identidad y Acceso sobre la consulta de auditoría, los identificadores de servicio y la asignación de línea en Inspecciones.
- Confirmar con el responsable del dominio el formato de placa, la definición del período, los catálogos, los rangos por tipo y las reglas normativas de clasificación. No trasladar automáticamente las reglas del código de origen al diseño.
- Entregar los límites y [la propiedad de datos](02-propiedad-datos.md) a Kevin, Carlos y Eduardo para acordar contratos, diagramas y comportamiento ante fallos.
