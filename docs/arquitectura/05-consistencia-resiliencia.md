# Consistencia y resiliencia del sistema RTV

**Responsable:** REINOSO VELEZ EDUARDO DAVID  
**Actividad:** GA-TL-02  
**Estado:** borrador. Esta versión recoge los escenarios de fallo. El análisis de consistencia entre servicios se añade en una versión posterior.

**Base de trabajo:** [01-dominio-servicios.md](01-dominio-servicios.md), [02-propiedad-datos.md](02-propiedad-datos.md), [03-api-gateway.md](03-api-gateway.md), las secciones 6, 10 y 11 de `PFC-RTV-referencia-completa-para-GLMR.md` y el [contrato REST v0.2 de E1](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1/blob/f675258e151c9458494707b4ae239afb9131636a/docs/contratos/contrato-inspecciones-turnos-pagos.md).

## 1. Alcance y respaldo de cada escenario

Los servicios MS-01 a MS-04 son la propuesta de `01`, pendiente de aprobación del equipo. De todo el diseño, solo la llamada de Inspecciones (MS-02) a Turnos y Pagos (MS-01) está implementada y probada en E1, con REST síncrono. La columna **Respaldo** de cada escenario indica cuál de estos casos aplica:

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
| FR-06 | Cae la instalación de PostgreSQL que aloja varias bases | Si MS-01 y MS-02 comparten instalación, como en la demostración de E1, pierden la persistencia a la vez. Se pierde el aislamiento de fallos entre sus bases. | Los procesos de los servicios siguen vivos, pero no pueden escribir ni leer datos. | Los servicios responden `503 PERSISTENCIA_NO_DISPONIBLE`. Al volver la instalación, los registros confirmados se conservan con sus identificadores y estados (RNF-05, probado con reinicio de servicios y PostgreSQL). La decisión de usar instancias separadas está pendiente (sección 3). | E1 probado (recuperación); Diseño (aislamiento) |
| FR-07 | Cae el Gateway | Los clientes web y móvil no pueden acceder a ninguna ruta externa. | Los servicios siguen activos. La llamada interna de MS-02 a MS-01 no pasa por el Gateway, así que lo que estaba en curso entre ellos continúa. | Detección por una comprobación de estado (health check) del Gateway y por la falta de solicitudes nuevas en los registros de los servicios. Propuesta: que el Gateway no guarde estado de negocio, para poder desplegar más de una instancia. No se afirma disponibilidad. | Diseño |
| FR-08 | El Gateway espera menos que la suma de las esperas internas | El cliente recibe `504` aunque MS-02 habría devuelto el error previsto del contrato o habría completado el inicio. | El inicio puede quedar completado en MS-02 sin que el cliente lo sepa. | El valor de espera del Gateway debe ser mayor que los 3 s de MS-02 a MS-01 más el procesamiento local de MS-02. El cliente reintenta con la misma `Idempotency-Key`. Valor por acordar con Kevin. | Diseño |
| FR-09 | Un cliente supera la cuota del Gateway | El Gateway responde `429 Too Many Requests` con `Retry-After`. | Los servicios quedan protegidos del exceso de tráfico. | Si la cuota es demasiado baja, también rechaza usuarios legítimos en horas de más uso. Los valores se fijan con pruebas de carga, según `03-api-gateway.md`. | Diseño |
| FR-10 | Cae MS-03 (Identidad y Acceso) | El inicio de sesión (RF-01) deja de estar disponible. Los cambios de roles (RF-02) no se aplican y la regla "los permisos cambian desde la siguiente solicitud" no se puede cumplir mientras dure la caída. | Si el Gateway valida el token JWT por su firma y expiración, las sesiones ya emitidas siguen operando hasta que el token expira. | Hay una tensión sin resolver. Si cada servicio consulta permisos a MS-03 en cada solicitud, la caída bloquea todas las operaciones protegidas. Si usan una copia de los permisos, `02` prohíbe operar con permisos revocados. Decisión pendiente (sección 3). | Diseño |
| FR-11 | Cae MS-04 (Ficha y Clasificación Vehicular) | No se registra ni consulta la ficha (RF-05 a RF-10). MS-01 no puede calcular una tarifa nueva y MS-02 no puede asignar línea, porque necesitan la categoría vigente. | La autorización y el inicio de inspección. Según `01`, MS-01 resuelve la elegibilidad con datos propios y no consulta a MS-04. | Los consumidores pueden conservar la categoría como dato histórico, pero según `02` no sirve para una decisión nueva. Las operaciones que dependen de la categoría responden con error y se reintentan. | Diseño |
| FR-12 | Una fuente no responde durante la consulta de auditoría | La consulta unificada de MS-03 no puede combinar los registros de MS-04. El resultado no se debe presentar como completo y debe indicar qué fuente falló (RF-04). | Los registros de auditoría no se pierden: cada servicio los guarda en su propia base, en la misma transacción local que el cambio de negocio. | MS-03 detecta la fuente caída y devuelve el resultado parcial marcado. El administrador repite la consulta después. | Diseño |

## 3. Decisiones pendientes que afectan estos escenarios

| Decisión | Escenarios | Con quién se acuerda |
|---|---|---|
| Tiempo de espera del Gateway frente a los 3 s de MS-02 a MS-01 | FR-02, FR-08 | Kevin |
| Cómo reciben los servicios los permisos vigentes: consulta a MS-03 en cada solicitud o datos dentro de un token con expiración corta | FR-10 | Bryan y Kevin |
| Una instancia de PostgreSQL con varias bases o una instancia por servicio. La separación aumenta el costo de infraestructura y se decide al contenerizar en PE-U2 | FR-06 | Los cuatro |
| Tiempos de espera de MS-02 a MS-04 y de MS-01 a MS-04. Solo los 3 s de MS-02 a MS-01 tienen respaldo en RNF-02 | FR-11 | Kevin |
| Si Kevin propone mensajería (Kafka o RabbitMQ), añadir los escenarios de caída del broker y mensajes duplicados | Nuevos | Kevin |

## 4. Límites de este análisis

- Los escenarios con respaldo "Diseño" son razonamiento sobre la propuesta del equipo. No hay pruebas que los confirmen.
- El p95 de 100 ms de RNF-01 se midió en el entorno local de E1, sin Gateway. El costo del recorrido ampliado no está medido.
- La revocación de autorizaciones, la cancelación de turnos y las reinspecciones están fuera del alcance de E1 y no se analizan aquí.
- Los nombres y los IDs de los servicios siguen la propuesta de `01`. Si el equipo los cambia, se actualizan los escenarios.
