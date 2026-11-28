# Matriz de comunicaciones del sistema RTV

**Responsable:** GARCIA BAZURTO KEVIN LEONEL

**Actividad:** GA-TL-02

**Estado:** propuesta para revisión del equipo.

## 1. Alcance y tecnología

La matriz sigue los cuatro servicios de [01-dominio-servicios.md](01-dominio-servicios.md) y la propiedad de datos de [02-propiedad-datos.md](02-propiedad-datos.md). El inicio de inspección y su autorización conservan el contrato de E1. Las demás rutas son propuestas de diseño.

Se emplea REST/JSON síncrono para confirmar la autorización antes del inicio (RNF-03), verificar permisos vigentes (RF-02) y obtener una categoría válida para tarifa y línea (RF-10). La auditoría se guarda con transacciones locales y se consulta por API.

Los clientes usan HTTPS hacia el Gateway. Para los canales internos se propone HTTPS y autenticación de servicio; E1 ya define una credencial Bearer para MS-02 → MS-01. Ningún canal permite acceder a la base de otro servicio.

## 2. Canales y contratos

Todas las filas utilizan REST/JSON síncrono. En COM-01, el contrato de `POST /inspecciones` procede de E1 y su paso por el Gateway es propuesto. COM-02 conserva el contrato interno de E1.

| ID | Origen → destino | Método y ruta | Información intercambiada | Requisito y justificación |
|---|---|---|---|---|
| COM-01 | Web/móvil → Gateway → MS-02 Inspecciones | `POST /inspecciones` | Entrada: `turnoId`, `Idempotency-Key`. Salida: `inspeccionId`, `turnoId`, `autorizacionId`, `estado`. | RNF-01 a RNF-04. El cliente necesita confirmar el inicio; el Gateway conserva la clave y el contrato. |
| COM-02 | MS-02 → MS-01 Turnos y Pagos | `POST /turnos/{turnoId}/autorizaciones-inspeccion` | Cuerpo `{}`; credencial de servicio, clave y correlación. Salida: `autorizacionId` y `turnoId`. | RNF-02, RNF-03 y RNF-06. MS-01 valida turno habilitado y pago confirmado antes de autorizar. |
| COM-03 | Web/móvil → Gateway → MS-03 Identidad y Acceso | `POST /sesiones` | Credenciales de usuario; devuelve token y vencimiento si la cuenta está activa. | RF-01. Ruta pública con limitación de intentos; las credenciales no se registran en logs. |
| COM-04 | Web/móvil → Gateway → MS-03 | `PUT /usuarios/{usuarioId}/roles` | Conjunto completo de identificadores de roles; resultado de la asignación. | RF-02 y RF-03. MS-03 valida usuario, roles y permiso del administrador; confirma todo o rechaza sin cambios. |
| COM-05 | Cada servicio que recibe una operación protegida → MS-03 | `POST /autorizaciones/verificar` | Sesión o token verificable, operación y recurso; devuelve actor y decisión de acceso vigente. | RF-02 y RF-03. Consultar en cada solicitud evita autorizar con roles revocados. El servicio destino aplica la decisión; validar la firma en el Gateway no sustituye este control. |
| COM-06 | Web/móvil → Gateway → MS-04 Ficha y Clasificación Vehicular | `POST /fichas`; `PATCH /fichas/{fichaId}`; `GET /fichas/{fichaId}` | Placa, período, tipo, uso, ejes y peso. Salida: ficha, categoría, fecha de clasificación e indicador de impronta. | RF-05 a RF-10. MS-04 valida y recalcula; los clientes no suministran una categoría como dato oficial. |
| COM-07 | MS-01 → MS-04 | `GET /fichas/{fichaId}/clasificacion` | `fichaId`, `versionFichaId`, categoría vigente y fecha de clasificación. | RF-10; CO-02 de `05`. La tarifa conserva la clasificación aplicada como dato histórico. |
| COM-08 | MS-02 → MS-04 | `GET /fichas/{fichaId}/clasificacion` | `fichaId`, `versionFichaId`, categoría vigente y fecha de clasificación. | RF-06 y RF-10; CO-03. Sin clasificación válida no se asigna una línea. Este canal no se añade al inicio mínimo de E1. |
| COM-09 | MS-04 → MS-02 | `GET /vehiculos/{vehiculoId}/ultima-revision` | Entrada: `vehiculoId`. Salida: `inspeccionId`, `fichaId` y `versionFichaId` de la revisión anterior; ausencia explícita si no existe. | RF-07; CO-04. MS-04 obtiene el uso histórico desde su propia ficha y lo compara con el actual para determinar la impronta. |
| COM-10 | Web/móvil → Gateway → MS-03 | `GET /auditoria?usuarioId=…&desde=…&hasta=…&accion=…` | Filtros, registros, indicador `completo` y fuentes no disponibles. | RF-04. Solo consulta quien tiene permiso; un resultado parcial debe reconocerse como tal. |
| COM-11 | MS-03 → MS-04 | `GET /auditoria?usuarioId=…&desde=…&hasta=…&accion=…` | Registros de alta de placa y cambios de uso, con identificador de origen. | RF-04 y RF-07; CO-06. MS-03 combina estos registros con su auditoría local de roles. MS-04 conserva la propiedad de sus registros. |
| COM-12 | Web/móvil → Gateway → MS-01 | `POST /turnos`; `GET /turnos/{turnoId}` | Crear: referencia de vehículo o ficha, período y datos del turno; devuelve `turnoId` y estado. Consultar: devuelve el turno y su estado administrativo. | Flujo administrativo de `01`; RNF-03 y RNF-05. MS-01 conserva el turno y su elegibilidad. |
| COM-13 | Web/móvil → Gateway → MS-01 | `POST /turnos/{turnoId}/confirmaciones-pago` | Referencia del comprobante y confirmación del personal autorizado; devuelve identificador y estado del pago. | RNF-03, RNF-05 y RF-03. Registrar una confirmación no significa integrar una pasarela bancaria ni emitir automáticamente la autorización. |
| COM-14 | Web/móvil → Gateway → MS-02 | `POST /inspecciones/{inspeccionId}/asignaciones-linea` | Línea solicitada y referencia de ficha; devuelve asignación y versión de clasificación utilizada. | RF-06 y RF-10. MS-02 comprueba compatibilidad con la categoría de COM-08 antes de confirmar. |

El inicio y la autorización se trazan a los RNF de E1. En COM-07, MS-01 obtiene `fichaId` del trámite y conserva `versionFichaId` con la categoría aplicada. En COM-08, MS-02 conserva `fichaId` y `versionFichaId` al asignar la línea. En COM-09, busca la revisión anterior por `vehiculoId`; MS-04 resuelve el uso histórico mediante la pareja `fichaId` y `versionFichaId`, conforme a [02-propiedad-datos.md](02-propiedad-datos.md). Una edición posterior no altera la versión histórica usada en la revisión.

### Elección del protocolo

| Opción | Evaluación para estos canales |
|---|---|
| REST/JSON | Conserva el contrato de E1, sirve a clientes web y móvil y permite responder dentro del ciclo de la solicitud. RNF-02 exige acotar la espera y RNF-03 exige comprobar la autorización antes del inicio. |
| gRPC | Permitiría contratos tipados entre servicios, pero exige otro contrato y herramientas sin una necesidad de rendimiento demostrada. RNF-01 requiere medir el recorrido; no demuestra que cambiar el protocolo mejore su resultado. |
| Kafka | Su publicación y consumo diferidos no sustituyen la autorización ni los permisos vigentes exigidos por RNF-03 y RF-02. No hay requisito actual de reproducción de un flujo de eventos que justifique incorporarlo. |
| RabbitMQ | Es útil para trabajos diferidos y entrega mediante colas; las decisiones descritas requieren respuesta inmediata y no existe un trabajo diferido identificado que justifique añadir un broker. |

### Respuestas de los contratos propuestos

| Canales | Respuesta de éxito y validaciones |
|---|---|
| COM-03 | `201`: token, tipo Bearer y vencimiento. `400` si faltan campos; `401` si las credenciales no permiten abrir sesión. |
| COM-04 | `200`: usuario y roles asignados. `400` para una asignación inválida; `404` si no existe la cuenta. Sin actualización parcial. |
| COM-05 | `200`: actor, decisión `permitido` y operación verificada. Una decisión negativa hace que el servicio de negocio responda `403`; una sesión inválida produce `401`. La credencial del servicio llamante también debe ser válida. |
| COM-06 | Alta `201`, edición y consulta `200`; incluyen identificador de ficha, datos validados y clasificación. `400` para campos inválidos, `404` si no existe y `409` por placa duplicada en el período. Si faltan datos para clasificar, la respuesta lo identifica y no ofrece una categoría vigente utilizable. |
| COM-07 y COM-08 | `200`: `fichaId`, `versionFichaId`, categoría vigente y fecha de clasificación. `404` si no existe la ficha; `409` si no tiene clasificación válida para la decisión solicitada. |
| COM-09 | `200`: `existeRevision: true`, `inspeccionId`, `fichaId` y `versionFichaId`; si no hay revisión previa, `existeRevision: false` sin referencias de revisión. Una revisión existente cuya versión no puede resolverse deja la impronta pendiente; no se interpreta como historial vacío. |
| COM-10 y COM-11 | `200`: registros filtrados, origen y estado de completitud; `400` si los filtros o fechas son inválidos. No ofrecen edición ni borrado. |
| COM-12 | Crear `201`; consultar `200`; `400` por datos inválidos y `404` por turno inexistente. El alta no supone pago confirmado. |
| COM-13 | `201`: confirmación registrada; `404` por turno inexistente y `409` ante un estado incompatible o comprobante ya aplicado. La unicidad del comprobante debe controlarse localmente. |
| COM-14 | `201`: asignación guardada; `404` por inspección o línea inexistente y `409` ante categoría inválida o estado incompatible. |

Los contratos propuestos de escritura distintos de COM-01 no heredan automáticamente su garantía de idempotencia. Para reintentos deben acordarse claves y restricciones propias; mientras tanto, el cliente consulta el resultado o solicita conciliación antes de repetir una escritura incierta.

## 3. Respuestas, esperas y recuperación

| Canal | Respuesta y tratamiento del fallo | Escenarios de `05` |
|---|---|---|
| COM-01 y COM-02 | E1 responde `201` al crear y `200` al recuperar con la misma clave. Mantiene `400`, `401`, `404`, `409` y `415` según el contrato. MS-02 traduce indisponibilidad de MS-01 a `503 TURNOS_NO_DISPONIBLE`, espera agotada a `504 TURNOS_TIEMPO_AGOTADO` y respuesta inválida a `502 RESPUESTA_TURNOS_INVALIDA`. Un fallo local de persistencia produce `503 PERSISTENCIA_NO_DISPONIBLE`. No se confirma un inicio sin autorización válida. | FR-01 a FR-05; CO-01. |
| COM-03 a COM-05 | Credencial inválida: `401`; permiso insuficiente: `403`. Si MS-03 no responde, una operación que requiere verificar permisos responde `503` y no ejecuta cambios. No se usa una copia de permisos para aceptar la operación. | FR-10 y CO-05; conserva RF-02. |
| COM-06 a COM-09 | Validaciones de negocio se devuelven al consumidor. Dependencia indisponible: `503`; espera agotada: `504`. No se reemplaza la categoría vigente por una histórica para calcular una tarifa o asignar una línea. Si no puede consultarse la revisión previa, la impronta queda pendiente de determinar. | FR-11; CO-02 a CO-04. |
| COM-10 y COM-11 | Si una fuente falla, la consulta puede devolver `200` con `completo: false` y su identificador en `fuentesNoDisponibles`. Los registros retornados conservan su origen. Un conjunto vacío de una fuente disponible no equivale a una fuente caída. | FR-12; CO-06. |
| Entrada al Gateway | Exceso de cuota: `429` con `Retry-After`. Destino indisponible: `503`; espera agotada: `504`. Conserva los errores de negocio del destino. | FR-07 a FR-09; documento `03`. |

La espera de COM-02 es de **3 segundos**, conforme a RNF-02. Para COM-01 se usa el siguiente presupuesto:

`espera del Gateway > verificación de permisos previa en MS-02 + 3 s de COM-02 + procesamiento local + margen`.

Los 3 segundos de COM-02 abarcan la respuesta completa de MS-01, incluida cualquier verificación de permisos que realice durante esa llamada. No se cuentan esas verificaciones por segunda vez. Las llamadas internas respetan el presupuesto restante; agotar una espera no implica cancelar una transacción ya iniciada en el receptor. Por eso un resultado incierto se recupera con la misma clave.

Los tiempos de COM-05 y COM-07 a COM-11 y el margen del Gateway requieren validación mediante pruebas. El p95 local de E1 no mide el recorrido con Gateway.

No se proponen reintentos automáticos de escrituras en el Gateway. Ante una respuesta incierta de COM-01, el cliente conserva y reutiliza la misma clave; MS-02 propaga esa clave en COM-02. El registro local puede quedar temporalmente como autorización sin inspección y completarse mediante reintento. Esto no constituye una transacción entre bases.

Se conserva el contrato de E1: el reintento usa el mismo turno y la misma clave. Si se pierde la clave y se envía otra para un turno ya autorizado, MS-01 rechaza con `409 TURNO_YA_AUTORIZADO`; el taller no incorpora recuperación por turno. Este rechazo no cancela la autorización existente ni permite duplicar la inspección.

## 4. Etiquetas para C4

Los diagramas [C4 nivel 2](../diagramas/c4-n2-microservicios.drawio) y [C4 nivel 3](../diagramas/c4-n3-microservicios.drawio) ya identifican los canales COM-01 a COM-14. Para cada relación se utiliza la etiqueta `COM-XX · propósito · REST/JSON · síncrona`. Los recorridos externos se dibujan en dos tramos: cliente → Gateway y Gateway → servicio. Las consultas internas van directamente al servicio propietario.

- COM-01: iniciar inspección; COM-02: autorizar inicio.
- COM-03: abrir sesión; COM-04: asignar roles; COM-05: verificar permisos vigentes.
- COM-06: gestionar ficha; COM-07: consultar categoría para tarifa; COM-08: consultar categoría para línea.
- COM-09: consultar revisión anterior; COM-10: consultar auditoría unificada; COM-11: consultar auditoría de ficha.
- COM-12: gestionar turno; COM-13: confirmar pago; COM-14: asignar línea.

La autorización de inicio COM-02 permanece interna. COM-05 corresponde a las llamadas de MS-01, MS-02 y MS-04 a MS-03; MS-03 verifica localmente el acceso a sus propias operaciones. Esta lectura coincide con FR-10 y CO-05 de [05-consistencia-resiliencia.md](05-consistencia-resiliencia.md).

## 5. Seguridad y observabilidad

El Gateway conserva `Content-Type`, `Idempotency-Key` y un `X-Request-Id` de correlación. Verifica el token de usuario y elimina cabeceras de identidad interna aportadas por el cliente. Cada servicio autentica a sus consumidores y restringe los endpoints internos; COM-05 no permite que un cliente elija libremente la identidad a verificar.

Las llamadas delegadas conservan el actor de forma verificable. MS-03 verifica permisos con sus datos vigentes, incluidos los de consultas internas cuando corresponda. Su verificación se resuelve localmente, sin invocar recursivamente COM-05. Las credenciales de servicio se configuran fuera del código.

Se registran identificador de correlación, canal, duración y resultado, sin contraseñas ni tokens completos. Los registros operativos no sustituyen la auditoría de RF-04: el servicio propietario guarda la acción y su auditoría en la misma transacción local.

## Fuentes

- [Catálogo de servicios](01-dominio-servicios.md), [propiedad de datos](02-propiedad-datos.md), [API Gateway](03-api-gateway.md) y [consistencia y resiliencia](05-consistencia-resiliencia.md).
- [Referencia del PFC RTV](../../PFC-RTV-referencia-completa-para-GLMR.md), requisitos RF-01 a RF-10 y RNF-01 a RNF-06.
- [Contrato REST v0.2 de E1](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1/blob/f675258e151c9458494707b4ae239afb9131636a/docs/contratos/contrato-inspecciones-turnos-pagos.md).
- [Controlador de Inspecciones de E1](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1/blob/f675258e151c9458494707b4ae239afb9131636a/src/inspecciones/src/main/java/Inspecciones.java).
