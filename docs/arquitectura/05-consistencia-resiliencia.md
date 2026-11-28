# Consistencia y resiliencia del sistema RTV

**Responsable:** REINOSO VELEZ EDUARDO DAVID  
**Actividad:** GA-TL-02  
**Estado:** propuesta para revisión del equipo. Incluye los escenarios de fallo, el análisis de consistencia entre servicios y su relación con la matriz de comunicaciones (`04`) y los diagramas C4.

**Base de trabajo:** [01-dominio-servicios.md](01-dominio-servicios.md), [02-propiedad-datos.md](02-propiedad-datos.md), [03-api-gateway.md](03-api-gateway.md), [04-comunicaciones-contratos.md](04-comunicaciones-contratos.md), los diagramas C4 de [docs/diagramas](../diagramas/), las secciones 6, 10, 11 y 12 de [PFC-RTV-referencia-completa-para-GLMR.md](../../PFC-RTV-referencia-completa-para-GLMR.md) y el [contrato REST v0.2 de E1](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1/blob/f675258e151c9458494707b4ae239afb9131636a/docs/contratos/contrato-inspecciones-turnos-pagos.md).

## 1. Alcance y respaldo de cada escenario

Los servicios MS-01 a MS-04 son la descomposición de `01`, confirmada por el equipo para el taller. De todo el diseño, solo la llamada de Inspecciones (MS-02) a Turnos y Pagos (MS-01) está implementada y probada en E1, con REST síncrono. La columna **Respaldo** de cada escenario indica cuál de estos casos aplica:

- **E1 probado:** el comportamiento tiene evidencia registrada en E1 (RNF-02, RNF-03, RNF-04 y RNF-05).
- **Contrato E1:** el comportamiento está definido en el contrato v0.2. La referencia no detalla una prueba específica de ese caso.
- **Diseño:** análisis del diseño del taller. No hay medición ni prueba.

Este documento no promete tiempos de disponibilidad ni porcentajes de éxito. La única medición disponible es el p95 local de E1 sin Gateway (RNF-01).

## 2. Escenarios de fallo

La idempotencia es la propiedad de repetir una operación con la misma clave sin duplicar su resultado. En este sistema la clave viaja en la cabecera `Idempotency-Key`.

| ID | Qué falla | Impacto | Qué sigue funcionando | Detección y estrategia | Respaldo |
|---|---|---|---|---|---|
| FR-01 | MS-01 detenido o inalcanzable cuando MS-02 pide la autorización | No se registra un nuevo inicio de inspección. MS-02 responde `503 TURNOS_NO_DISPONIBLE`. Si la respuesta de MS-01 llega con formato inesperado, responde `502 RESPUESTA_TURNOS_INVALIDA`. | Recuperar desde la base de MS-02 un inicio ya guardado con la misma clave. | MS-02 detecta el error de conexión al llamar a MS-01. No inicia sin una autorización válida. El cliente repite la operación con la misma clave. | E1 probado (RNF-02) |
| FR-02 | MS-01 tarda más de 3 s | MS-02 responde `504 TURNOS_TIEMPO_AGOTADO`. MS-01 puede seguir procesando y guardar la autorización después de que MS-02 dejó de esperar. | Igual que FR-01. | Vence el tiempo de espera `TURNOS_TIMEOUT` (3 s inicial). MS-02 conserva la clave. El reintento con la misma clave y turno recupera la autorización guardada. | E1 probado (RNF-02) |
| FR-03 | Se pierde la respuesta de MS-01 después de guardar la autorización | Queda una autorización guardada en MS-01 sin inicio confirmado en MS-02. | Los registros ya confirmados de ambos servicios. | El cliente no recibe respuesta. Repite con la misma clave y turno. MS-01 devuelve la autorización original con `200` y MS-02 completa el inicio. | E1 probado (RNF-02 y RNF-04) |
| FR-04 | MS-02 no puede guardar el inicio después de recibir la autorización | Autorización guardada sin inspección registrada. MS-02 responde `503 PERSISTENCIA_NO_DISPONIBLE`. | MS-01 y sus autorizaciones ya guardadas. | El reintento con la clave original recupera la misma autorización y completa el registro local. | Contrato E1 |
| FR-05 | La base de MS-01 no está disponible | MS-01 no puede emitir autorizaciones. Responde `503 PERSISTENCIA_NO_DISPONIBLE` y MS-02 lo trata como FR-01. | MS-02 para inicios ya guardados. | MS-01 detecta el fallo de acceso a su base y no crea ninguna autorización. | Contrato E1 |
| FR-06 | Cae la instalación de PostgreSQL que aloja varias bases | Si MS-01 y MS-02 comparten instalación, como en la demostración de E1, pierden la persistencia a la vez. Se pierde el aislamiento de fallos entre sus bases. | Los procesos de los servicios siguen vivos, pero no pueden escribir ni leer datos. | Los servicios responden `503 PERSISTENCIA_NO_DISPONIBLE`. Al volver la instalación, los registros confirmados se conservan con sus identificadores y estados (RNF-05, probado con reinicio de servicios y PostgreSQL). La decisión de usar instancias separadas está pendiente (sección 5). | E1 probado (recuperación); Diseño (aislamiento) |
| FR-07 | Cae el Gateway | Los clientes web y móvil no pueden acceder a ninguna ruta externa. | Los servicios siguen activos. La llamada interna de MS-02 a MS-01 no pasa por el Gateway, así que lo que estaba en curso entre ellos continúa. | Detección por una comprobación de estado (health check) del Gateway y por la falta de solicitudes nuevas en los registros de los servicios. Propuesta: que el Gateway no guarde estado de negocio, para poder desplegar más de una instancia. No se afirma disponibilidad. | Diseño |
| FR-08 | El Gateway espera menos que la suma de las esperas internas | El cliente recibe `504` aunque MS-02 habría devuelto el error previsto del contrato o habría completado el inicio. | El inicio puede quedar completado en MS-02 sin que el cliente lo sepa. | La espera del Gateway para COM-01 debe superar la suma de tres tiempos: la verificación de permisos previa de MS-02 contra MS-03 (COM-05), los 3 s de COM-02 y el procesamiento local, más un margen. Según `04`, los 3 s de COM-02 ya incluyen las verificaciones que MS-01 haga durante esa llamada, así que no se suman otra vez. El cliente reintenta con la misma `Idempotency-Key`. El criterio está en la sección 5. El valor numérico no se fija sin medir el recorrido. | Diseño |
| FR-09 | Un cliente supera la cuota del Gateway | El Gateway responde `429 Too Many Requests` con `Retry-After`. | Los servicios quedan protegidos del exceso de tráfico. | Si la cuota es demasiado baja, también rechaza usuarios legítimos en horas de más uso. Los valores se fijan con pruebas de carga, según `03-api-gateway.md`. | Diseño |
| FR-10 | Cae MS-03 (Identidad y Acceso) | Según `03` y `04` (COM-05), cada servicio consulta a MS-03 en cada solicitud protegida. Sin MS-03 no hay inicio de sesión (RF-01) ni cambio de roles (RF-02), y además toda operación protegida responde `503` sin ejecutarse, incluido el inicio de inspección por COM-01. | Las lecturas y escrituras que no exigen verificar permisos, si las hubiera, y los registros ya confirmados. `04` no define operaciones protegidas sin verificación. | `04` descarta aceptar la operación con una copia de permisos. Con esa decisión se cumple RF-02, criterio 4 (el cambio rige desde la siguiente solicitud), a cambio de que MS-03 sea una dependencia crítica de toda la ruta. El tiempo de espera de COM-05 no está definido en `04` y requiere pruebas. | Diseño |
| FR-11 | Cae MS-04 (Ficha y Clasificación Vehicular) | No se registra ni consulta la ficha (RF-05 a RF-10). MS-01 no puede calcular una tarifa nueva y MS-02 no puede asignar línea, porque necesitan la categoría vigente. | La autorización y el inicio de inspección. Según `01`, MS-01 resuelve la elegibilidad con datos propios y no consulta a MS-04. | Los consumidores pueden conservar la categoría como dato histórico, pero según `02` no sirve para una decisión nueva. Las operaciones que dependen de la categoría responden con error y se reintentan. | Diseño |
| FR-12 | Una fuente no responde durante la consulta de auditoría | La consulta unificada de MS-03 no puede combinar los registros de MS-04. El resultado no se debe presentar como completo y debe indicar qué fuente falló (RF-04). | Los registros de auditoría no se pierden: cada servicio los guarda en su propia base, en la misma transacción local que el cambio de negocio. | MS-03 detecta la fuente caída y devuelve el resultado parcial marcado. El administrador repite la consulta después. | Diseño |

## 3. Consistencia entre servicios

Cada servicio confirma sus cambios con transacciones locales sobre su propia base. No existe una transacción única entre bases (`02`, sección "Límites del diseño"). Se usan dos términos:

- **Consistencia fuerte:** toda lectura ve el último valor confirmado.
- **Consistencia eventual:** durante un tiempo dos servicios pueden tener estados distintos de una misma operación y después convergen.

### 3.1 Operaciones que cruzan servicios

| ID | Operación | Servicios y datos | Consistencia que exige | Mecanismo | Estado intermedio aceptado | Respaldo |
|---|---|---|---|---|---|---|
| CO-01 | Autorizar e iniciar una inspección | MS-02 (inspección y clave) con MS-01 (turno, pago y autorización) | Fuerte dentro de cada base. Eventual entre las dos. Reglas que no se pueden violar: no hay inspección sin autorización válida (RNF-03) y no hay duplicados (RNF-04). | Transacción local en cada servicio. Clave de idempotencia guardada antes de llamar. Restricciones de unicidad por turno, autorización y clave. MS-02 guarda el inicio solo después de validar la autorización. | Autorización guardada en MS-01 sin inicio en MS-02, recuperable con la misma clave. | E1 probado (RNF-02, RNF-03 y RNF-04) |
| CO-02 | Tarifa de un trámite | MS-01 usa la categoría que calcula MS-04 | Lectura actual al calcular. El valor ya aplicado es un dato histórico. | Consulta síncrona a MS-04. MS-01 obtiene el `fichaId` del trámite y guarda la categoría, su fecha y el `versionFichaId` usados (COM-07). | Si la ficha cambia después, el importe ya calculado no se recalcula solo. Qué ocurre depende de reglas de vigencia y cancelación aún sin acordar. | Diseño |
| CO-03 | Asignación de línea | MS-02 usa la categoría de MS-04 | Lectura vigente al asignar. La categoría se recalcula antes de asignar (RF-06, criterio 4). | Consulta síncrona a MS-04. MS-02 guarda el `fichaId` y el `versionFichaId` usados (COM-08). Sin categoría válida no asigna línea (RF-10, criterio 3). | La ficha puede cambiar después de asignar. La referencia guardada permite detectar el desfase, y una edición posterior de la ficha no altera la versión histórica (`02`). | Diseño |
| CO-04 | Impronta por cambio de uso (RF-07) | MS-04 compara el uso actual con el de la revisión anterior, que identifica MS-02 | Lectura de datos históricos. No exige el último valor. | MS-04 pide a MS-02 la última revisión por `vehiculoId` y recibe `inspeccionId`, `fichaId` y `versionFichaId` (COM-09). Con esa pareja consulta su propio historial y compara el uso. | Si MS-02 no responde, o la versión de una revisión existente no se puede resolver, la impronta no se puede determinar. Se marca como pendiente y no se asume; `04` lo distingue de un historial vacío (`existeRevision: false`). | Diseño |
| CO-05 | Permisos vigentes (RF-02 y RF-03) | MS-03 es la fuente. MS-01, MS-02 y MS-04 los verifican en cada solicitud protegida (COM-05). | Consistencia fuerte. RF-02, criterio 4: el cambio rige desde la siguiente solicitud protegida. | Consulta síncrona a MS-03 en cada solicitud (`04`, COM-05). No se usa copia local de permisos ni se confía solo en la firma del token del Gateway. | Ninguno. Si MS-03 no responde, la operación se rechaza con `503` y no se ejecuta (FR-10). | Diseño |
| CO-06 | Consulta de auditoría (RF-04) | MS-03 combina los registros de MS-03 y MS-04 | Cada registro es atómico con su cambio, porque se guarda en la misma transacción local. La consulta unificada es una lectura sin transacción entre servicios. | Cada servicio guarda su registro. MS-03 consulta las fuentes y marca el resultado como incompleto si una no responde. | Resultado parcial si falta una fuente. La hora de cada registro la pone su servicio, así que el orden entre servicios por fecha no es exacto. | Diseño |

### 3.2 Hueco en CO-01: el cliente pierde la clave

El contrato v0.2 define que otra clave para un turno ya autorizado se rechaza con `409 TURNO_YA_AUTORIZADO`, y que el reintento debe usar la misma clave. No describe qué ocurre si el cliente pierde esa clave.

Escenario: MS-01 guarda la autorización con la clave K1 y el cliente se cae antes de recibir la respuesta. Al volver, genera una clave nueva K2 para el mismo turno. MS-01 responde `409`, MS-02 no puede completar el inicio y el turno queda con autorización pero sin inspección, sin un camino desde el cliente para terminarlo.

El código de E1 (commit `f675258`, leído y no ejecutado) confirma el comportamiento:
- `Inspecciones.iniciar` busca una inspección existente solo por clave (`select * from inspecciones where clave = ?`) y no por turno.
- `Autorizaciones.autorizarTurno` responde `409 TURNO_YA_AUTORIZADO` si el turno ya tiene una autorización con una clave distinta.
- `Claves.reservar` asocia cada clave a un turno. Una clave nueva para el mismo turno es válida y llega a MS-01, que la rechaza.

Si K1 ya produjo una inspección, el cliente con K2 recibe `409` y no tiene una consulta para recuperarla. Si MS-02 no llegó a guardar el inicio (FR-04), el turno queda sin camino de recuperación. E1 prueba que otra clave recibe `409` (`pruebas/integracion.mjs`), pero no ofrece una forma de recuperar el turno. Hay tres salidas, todas con costo:

| Opción | Cambio que exige | Costo |
|---|---|---|
| A. Derivar la clave del turno, mientras exista un solo inicio por turno | El contrato: la clave deja de ser aleatoria | Cambia la cabecera que conserva el Gateway y la regla de unicidad |
| B. MS-02 busca por turno una clave pendiente y la reutiliza | La lógica de MS-02, que ya guarda la clave con el turno antes de llamar | Más lógica en un servicio, sin cambio de contrato externo |
| C. Conciliación manual del turno bloqueado | Un procedimiento operativo | Depende de una persona y no escala |

El documento `04` (sección 3) reconoce el mismo caso: dice que E1 no recupera una clave anterior por turno, que resolverlo exige una decisión de contrato y su ADR, y que no lo cubre el reintento actual. Si el equipo adopta A o B, la decisión cambia el contrato de E1 y se registra como ADR en `docs/adr/`. Mientras no se decida, el caso queda abierto y declarado.

### 3.3 Por qué no se usan 2PC, Saga ni outbox

- **Commit en dos fases (2PC)** es un protocolo donde un coordinador pide a todas las bases confirmar o abortar a la vez. Exige que las bases estén disponibles al mismo tiempo y las acopla. `02` descarta presuponer una transacción única entre bases.
- **Saga** es una secuencia de transacciones locales con pasos de compensación para deshacerlas. El único flujo que escribe en dos servicios (CO-01) se completa hacia adelante con la misma clave, y no necesita deshacer una autorización mientras la revocación siga sin acordarse en el alcance (`01`). Si el equipo incorpora cancelación o revocación de autorizaciones, aparece la necesidad de compensar y hay que revisar esta decisión con un ADR.
- **Outbox transaccional** es guardar el evento en la misma transacción local que el cambio y publicarlo después. Solo tendría sentido si el equipo adopta mensajería, por ejemplo para la auditoría con un servicio aparte (alternativa B de `01`). Con REST síncrono no se necesita. `04` evalúa Kafka y RabbitMQ y los descarta para estos canales, por lo que no se analizan escenarios de caída del broker.

### 3.4 Cuándo es aceptable la consistencia eventual

**Aceptable**, porque hay un camino de recuperación:
- Una autorización guardada sin el inicio de inspección correspondiente (CO-01).
- Categorías o importes guardados como dato histórico (CO-02 y CO-03).
- Una consulta de auditoría que muestra un resultado parcial marcado como tal (CO-06).

**No aceptable**, porque rompe una regla de negocio o de seguridad:
- Iniciar una inspección sin autorización válida (RNF-03).
- Duplicar una autorización o una inspección (RNF-04).
- Operar con permisos ya revocados (RF-02, RF-03 y `02`).
- Guardar una placa duplicada en el mismo período (RF-05). Esta regla es local de MS-04, así que se resuelve con una restricción en su propia base.

### 3.5 Relación con la matriz de comunicaciones y los diagramas

Cada operación de la sección 3.1 se apoya en canales de `04`. Los diagramas C4 nivel 2 y 3 de [docs/diagramas](../diagramas/) rotulan sus relaciones con los mismos identificadores COM-XX.

| Operación | Canales de `04` y de los diagramas | Escenarios |
|---|---|---|
| CO-01 Autorizar e iniciar | COM-01, COM-02 | FR-01 a FR-05, FR-08 |
| CO-02 Tarifa | COM-07 | FR-11 |
| CO-03 Asignación de línea | COM-08, COM-14 | FR-11 |
| CO-04 Impronta | COM-09 | FR-11 |
| CO-05 Permisos vigentes | COM-05 | FR-10 |
| CO-06 Auditoría | COM-10, COM-11 | FR-12 |

Los canales COM-03, COM-04 y COM-12 a COM-14 escriben en un solo servicio. No generan consistencia entre bases, pero `04` aclara que COM-13 y COM-14 no heredan la idempotencia de COM-01. La unicidad del comprobante de COM-13 debe controlarse localmente en MS-01.

## 4. Costos que introduce la separación

- **Más saltos por operación:** el inicio de inspección pasa por el Gateway, MS-02 y MS-01. RNF-01 (p95 de 100 ms) se midió sin Gateway y con dos servicios. El costo del recorrido ampliado no está medido (`03`, sección 4).
- **Más puntos de fallo:** cada dependencia síncrona entre servicios añade un escenario de la sección 2.
- **Verificación de permisos en cada solicitud:** COM-05 añade una llamada síncrona a MS-03 antes de cada operación protegida. El inicio de inspección pasa así por dos dependencias síncronas, MS-03 y MS-01, y MS-03 se vuelve crítico para todo el sistema (FR-10).
- **Consultas entre servicios** para datos que en una aplicación única serían una consulta a una sola base: la categoría de MS-04 (CO-02 y CO-03) y la auditoría unificada (CO-06).
- **Idempotencia en toda la cadena:** la clave debe conservarse en el cliente, el Gateway y los dos servicios, y el hueco de la sección 3.2 muestra que no es trivial.
- **Alternativa descartada en E1:** una aplicación modular única, con menor costo de coordinación local. E1 eligió separar para demostrar límites e intercambio distribuido (sección 12 de la referencia).

## 5. Decisiones que afectan este análisis

| Decisión | Escenarios | Estado |
|---|---|---|
| Espera del Gateway para COM-01. Criterio en `03` y `04`: espera del Gateway > verificación de permisos previa en MS-02 + 3 s de COM-02 + procesamiento local + margen. Las llamadas internas respetan el tiempo restante. No hay medición del recorrido con Gateway (RNF-01 se midió sin él), así que el número se fija con pruebas | FR-02, FR-08 | Criterio acordado; valor pendiente de medir |
| Cómo reciben los servicios los permisos vigentes | FR-10, CO-05 | Decidido en `03` y `04`: consulta a MS-03 en cada solicitud (COM-05). Costo: MS-03 es dependencia crítica |
| Una instancia de PostgreSQL con varias bases o una instancia por servicio. La separación aumenta el costo de infraestructura y se decide al contenerizar en PE-U2 | FR-06 | Pendiente, los cuatro |
| Tiempos de espera de COM-05 y de COM-07 a COM-11. Solo los 3 s de COM-02 tienen respaldo en RNF-02. `04` los deja para validar con pruebas | FR-10, FR-11 | Pendiente de medir |
| Qué hacer si el cliente pierde la clave de idempotencia (opciones A, B o C de la sección 3.2) | CO-01 | Decidido en `01`, `02` y `04`: queda fuera del alcance del taller y se conserva el contrato de E1. El `409` no cancela la autorización existente ni permite duplicar la inspección. Limitación declarada, sin ADR |
| Cómo se relaciona el trámite con `fichaId` o `vehiculoId` y cómo se identifica la versión de ficha | CO-03, CO-04 | Decidido en `02` y aplicado en `04` (COM-07 a COM-09): `vehiculoId`, `fichaId` y `versionFichaId` |
| Mensajería (Kafka o RabbitMQ) | Ninguno | Descartada para estos canales en `04` |

## 6. Límites de este análisis

- Los escenarios y operaciones con respaldo "Diseño" son razonamiento sobre la propuesta del equipo. No hay pruebas que los confirmen.
- El p95 de 100 ms de RNF-01 se midió en el entorno local de E1, sin Gateway. El costo del recorrido ampliado no está medido.
- La revocación de autorizaciones, la cancelación de turnos y las reinspecciones no están en el alcance de E1 y siguen pendientes de acuerdo en el taller, así que no se analizan aquí.
- El hueco de la sección 3.2 se confirmó leyendo el código de E1 en el commit del contrato. No se ejecutó ni se probó con una repetición real.
- Los nombres y los IDs de los servicios siguen la propuesta de `01`. Si el equipo los cambia, se actualizan los escenarios.
- Las rutas, códigos de respuesta y tiempos que se citan de `04` y `03` son propuesta de Kevin, salvo COM-01 y COM-02, que proceden del contrato de E1. Si esos documentos cambian, se revisan FR-08, FR-10 y la sección 3.5.
- `01` justifica MS-01 y MS-02 con necesidades del PFC (autorizar y registrar el inicio de una inspección) y no les asigna un RF numerado. Este documento no depende de esa decisión.
