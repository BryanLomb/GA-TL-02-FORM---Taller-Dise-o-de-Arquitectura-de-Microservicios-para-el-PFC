# API Gateway del sistema RTV

**Responsable:** GARCIA BAZURTO KEVIN LEONEL  
**Actividad:** GA-TL-02  
**Estado:** diseño preliminar para revisión del equipo

## 1. Ubicación y funciones

El API Gateway será la entrada común de los clientes web y móvil del sistema de Revisión Técnica Vehicular. Recibirá las solicitudes por HTTPS, validará las credenciales y las enviará al servicio responsable. Cada servicio conservará sus reglas de negocio y su base de datos privada.

| Función | Aplicación en RTV |
|---|---|
| Enrutamiento | Dirigir cada método y ruta al servicio que administra la operación |
| Autenticación | Validar la credencial del usuario antes de reenviar una solicitud protegida |
| Limitación de solicitudes | Aplicar cuotas para controlar el tráfico y responder ante un exceso de peticiones |

## 2. Enrutamiento

Se propone conservar REST con JSON para el inicio de inspecciones. Este flujo tiene un contrato en E1 y requiere recibir una autorización antes de confirmar el inicio. El Gateway mantendrá la ruta `POST /inspecciones` y reenviará la petición al servicio de Inspecciones.

```text
Cliente web o móvil
        |
        | POST /inspecciones
        v
API Gateway
        |
        | Validación de acceso y enrutamiento
        v
Inspecciones
        |
        | POST /turnos/{turnoId}/autorizaciones-inspeccion
        v
Turnos y Pagos
```

Inspecciones solicita la autorización por la red interna de servicios. Turnos y Pagos comprueba el turno, su habilitación y el pago, y devuelve la autorización. Inspecciones guarda el inicio en su base y responde al cliente a través del Gateway.

Las rutas de acceso, ficha vehicular y demás operaciones se incorporarán cuando el equipo apruebe el catálogo de servicios. Cada ruta registrará método, destino, permiso requerido y RF atendido.

## 3. Autenticación y permisos

La aplicación RTV de origen utiliza JWT. Se propone mantener ese mecanismo para el acceso de usuarios y validar en el Gateway la firma, el algoritmo permitido, el emisor, la audiencia y la expiración del token. El equipo definirá el responsable de emitirlo, la distribución de claves y el tratamiento de sesiones revocadas.

El inicio de sesión tendrá una ruta pública para presentar las credenciales. Las operaciones protegidas exigirán un token válido. Una credencial ausente o inválida producirá `401 Unauthorized`, con la cabecera `WWW-Authenticate` correspondiente.

El servicio destino comprobará los permisos sobre la operación y el recurso. Cuando el usuario carezca del permiso, responderá `403 Forbidden`. Este control atiende RF-01, RF-02 y RF-03 sobre acceso, roles y restricciones de funciones.

El Gateway eliminará las cabeceras de identidad interna enviadas por el cliente y transmitirá contexto verificable al destino. Para la autorización entre Inspecciones y Turnos y Pagos se conserva la credencial de servicio del contrato E1, vinculada a RNF-06.

## 4. Limitación de solicitudes

Se propone una cuota por usuario autenticado y grupo de operaciones. El inicio de sesión tendrá límites por origen y cuenta para controlar intentos repetidos.

El limitador podrá utilizar token bucket, con tasa de reposición y capacidad de ráfaga configurables. Los valores se fijarán con pruebas de carga y el volumen de uso esperado. Al agotar la cuota, el Gateway responderá `429 Too Many Requests` con `Retry-After`.

Esta política debe evaluarse junto con RNF-01. El criterio publicado de p95 ≤ 100 ms corresponde al inicio de inspección en el entorno local de E1. Se medirá nuevamente el recorrido con Gateway para conocer su costo adicional y comprobar el criterio en el entorno acordado.

## 5. Cabeceras y recuperación ante fallos

| Elemento | Tratamiento |
|---|---|
| `Authorization` | Validar según la política de acceso de la ruta |
| `Content-Type` | Conservar `application/json` en el inicio de inspección |
| `Idempotency-Key` | Conservar la clave durante el recorrido y los reintentos de la operación |
| `X-Request-Id` | Generar un identificador de correlación y propagarlo a los servicios y a la respuesta |

La llamada de Inspecciones a Turnos y Pagos tiene una espera inicial de 3 segundos, definida por RNF-02. La espera del Gateway deberá permitir que Inspecciones complete su procesamiento o devuelva el error previsto por el contrato. El valor se acordará con Eduardo.

Si el servicio destino está temporalmente indisponible, el Gateway responderá `503 Service Unavailable`. Si vence su espera, responderá `504 Gateway Timeout`. Los errores de negocio conservarán los códigos acordados en el contrato.

Una respuesta perdida puede ocurrir después de guardar la autorización. El cliente repetirá la misma operación con la misma clave de idempotencia. El Gateway la conservará y los servicios recuperarán sus registros según el contrato, protegiendo la integridad y el control de duplicados de RNF-03 y RNF-04.

Se registrarán ruta, destino, duración, código de respuesta e identificador de correlación. Los registros omitirán contraseñas y tokens.

## 6. Pendientes de integración

- Completar las rutas y su relación con RF a partir del catálogo aprobado por el equipo.
- Definir el responsable de identidad y las políticas de vigencia de tokens y permisos.
- Acordar cuotas y tiempos de espera con Eduardo.
- Incorporar las rutas a la matriz de comunicaciones y entregar las etiquetas del Gateway a Carlos.

## Referencias

- Requisitos y condiciones de las pruebas. [Informe E1 de RTV](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1/blob/f675258e151c9458494707b4ae239afb9131636a/docs/informe.tex)
- Inicio, autorización, idempotencia y errores. [Contrato REST v0.2](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1/blob/f675258e151c9458494707b4ae239afb9131636a/docs/contratos/contrato-inspecciones-turnos-pagos.md)
- Acceso de usuarios en la aplicación de origen. [AuthController](https://github.com/NixonPiguave/AppWebRevisionVehicular/blob/77d1ff6c38488fc02ab431472621ff4edb59f576/backend/src/main/java/com/revisionvehicular/backend/controllers/AuthController.java)
- Errores HTTP. [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html)
- Exceso de solicitudes. [RFC 6585](https://www.rfc-editor.org/rfc/rfc6585.html#section-4)
- Validación de JWT. [RFC 8725](https://www.rfc-editor.org/rfc/rfc8725.html)
