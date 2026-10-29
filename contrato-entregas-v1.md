# Contrato de entregas v1

Guía para quien desarrolla un sistema que recibe eventos del motor. El motor
define un único contrato; todo sistema destino se adapta a él. El motor no
conoce sistemas por nombre ni aplica reglas de negocio por destino: entrega el
evento tal como sale de la base de datos y el receptor decide qué hacer con él.

## Resumen

- El motor hace `POST` a la URL registrada del sistema, con `Authorization: Bearer <token>`.
- Cada evento se entrega **al menos una vez**. Tolerá duplicados con `Idempotency-Key`.
- Respondé `2xx` cuando el evento quedó aceptado. Cualquier otra respuesta se reintenta o falla según la tabla de más abajo.
- El motor envía además un ping de verificación al registrar o cambiar la URL o el token de tu sistema.
- Tu sistema se registra solo, por API, y se administra con un token de acceso propio (sección "Registro y rotación de tokens").
- La URL de entrega no puede apuntar a loopback, a metadata de nube ni al propio motor (sección "Restricciones de la URL de entrega").

## Petición de evento

```http
POST {url de entrega del sistema}
Content-Type: application/json
Authorization: Bearer <token de entrega del sistema>
Idempotency-Key: <ENVIRONMENT_CODE>:erp:<id_empresa>:<event_id>:<nombre del sistema>
X-Event-Id: <event_id>
X-Delivery-Attempt: <n>
X-Delivery-Type: event
X-Contract-Version: 1
```

| Header | Significado |
|---|---|
| `Authorization` | `Bearer` con el token de entrega de tu sistema. El motor lo descifra justo antes de enviar. |
| `Idempotency-Key` | Identifica la entrega (evento por sistema). Es estable entre reintentos y reprocesos. `<ENVIRONMENT_CODE>` es el código del entorno del motor (`DEV`, `PROD`…), por ejemplo `PROD:erp:3249:918273:mi-sistema`. Tratalo como un string opaco. |
| `X-Event-Id` | Id del evento en el origen. El mismo evento lleva el mismo id en todos los sistemas. |
| `X-Delivery-Attempt` | Número de intento de esta entrega (1 en el primer intento). |
| `X-Delivery-Type` | `event` para eventos. |
| `X-Contract-Version` | `1`. |

### Body

```json
{
  "id_company": 3249,
  "id_branch": 0,
  "event": {
    "type": "customer_update",
    "data": { "ref": 4978 }
  }
}
```

- El body es el objeto del evento tal como sale de la base de datos. No se renombran ni normalizan campos; si el evento trae campos adicionales, llegan intactos.
- `id_company`: entero positivo. Si el evento de origen no lo trae, el motor lo completa con la empresa del evento.
- `id_branch`: se envía **tal cual viene** (`0` o `null` significan "sin sucursal"). Si el evento no lo trae, no se agrega.
- `event.type`: nombre del evento. Es el único lugar donde viaja el tipo.
- `event.data`: objeto con los datos del evento.

## Petición de ping

```http
POST {url de entrega del sistema}
Content-Type: application/json
Authorization: Bearer <token de entrega>
X-Delivery-Type: ping
X-Contract-Version: 1

{}
```

Respondé `2xx` sin procesar nada. El motor envía el ping al dar de alta o
cambiar la URL o el token de un sistema, y solo guarda el cambio si respondés
`2xx`. El ping sale siempre desde el host del motor, que es el que alcanza las
URLs internas.

## Respuestas y qué hace el motor

| Respuesta | Clasificación | Qué hace el motor |
|---|---|---|
| `2xx` | Éxito | La entrega queda completada y no se reenvía. El body de la respuesta se ignora. |
| `401`, `403` | Reintentable | Reintenta. Cubre la ventana de rotación de token. |
| `408`, `429`, `5xx` | Reintentable | Reintenta. |
| Timeout, error de red, DNS o TLS | Reintentable | Reintenta. |
| Otro `4xx` o cualquier `3xx` | Permanente | Va directo a la cola `.failed`, sin consumir reintentos. |

Notas:

- Las redirecciones no se siguen: una respuesta `3xx` no es `2xx` y se trata como permanente. Registrá la URL final.
- Antes de cada envío el motor vuelve a comprobar que la suscripción siga activa y que tu sistema no esté revocado. Si no, omite la entrega (no te llega nada) y no es un error.
- `Retry-After` se ignora; los reintentos siguen el calendario del motor (`DELIVERY_RETRY_DELAY_MS`, `MAX_RETRIES`).
- Con los valores por defecto (`MAX_RETRIES` = 5, `DELIVERY_RETRY_DELAY_MS` = 30000) una entrega que siempre falla de forma reintentable se envía hasta 6 veces (el primer envío y 5 reintentos, uno cada 30 s) y después queda en `.failed`; un operador decide si reenviarla. No se reenvía sola.
- El body de tu respuesta nunca se lee, y el timeout cuenta hasta recibir los headers.

## Obligaciones del receptor

1. **Idempotencia.** La entrega es al menos una vez: puede repetirse tras un reintento, un reinicio o una respuesta perdida. Guardá el `Idempotency-Key` de lo que ya procesaste y, si llega de nuevo, respondé `2xx` sin repetir el efecto.
2. **Responder a tiempo.** El timeout por petición es `WEBHOOK_REQUEST_TIMEOUT_MS` (20000 ms por defecto, global para todos los sistemas). Si el trabajo es largo, aceptá con `2xx` y procesá en segundo plano.
3. **Responder `2xx` al ping**, sin procesar nada.
4. **Rotar el token sin corte.** Ver la sección siguiente.
5. **Eventos que no querés.** El motor no decide por reglas de negocio qué evento le sirve a cada sistema. Si recibís uno que no vas a procesar, respondé `2xx` e ignoralo. Un `4xx` (salvo los reintentables) lo manda a `.failed`.

## Restricciones de la URL de entrega

Un sistema registrado no puede convertir al motor en un proxy hacia la red
interna. La URL de entrega se comprueba al registrar el sistema, al cambiarla
con `PATCH /systems/me/delivery`, en cada ping y **en cada entrega** (con las
direcciones IP a las que resuelve el host en ese momento).

Se rechaza si el host (o cualquiera de sus IP resueltas) es:

- loopback (`127.0.0.0/8`, `::1`), `0.0.0.0/8` o `::`;
- link-local (`169.254.0.0/16`, que incluye la metadata de nube `169.254.169.254`; `fe80::/10`), y las direcciones de metadata `100.100.100.200` y `fd00:ec2::254`;
- multicast o reservadas (desde `224.0.0.0` en adelante, incluida `255.255.255.255`; `ff00::/8`);
- una IPv4 prohibida embebida en una forma IPv6 (`::ffff:127.0.0.1`, NAT64 `64:ff9b::/96`, 6to4 `2002::/16`), o un rango IPv6 de transición bloqueado entero (IPv4-compatible `::/96`, NAT64 local `64:ff9b:1::/48`);
- una dirección de las interfaces de red del propio motor.

También se rechazan URLs con usuario o contraseña y protocolos distintos de `http:`/`https:`.

Se **permiten** los rangos privados (RFC 1918: `10/8`, `172.16/12`, `192.168/16`), `100.64.0.0/10` y las direcciones ULA de IPv6 (`fc00::/7`), porque hay receptores internos legítimos.

Si el administrador definió `DELIVERY_ALLOWED_HOSTS`, solo valen los hosts de esa lista (comparación exacta del nombre de host, sin puerto).

Si la URL de un sistema deja de ser válida después del registro (por ejemplo, su DNS pasa a resolver a una IP prohibida), la entrega falla de forma permanente y el job va a la cola `.failed`; un fallo de DNS es reintentable.

Límites que conviene conocer:

- Queda una ventana pequeña de DNS rebinding entre la comprobación y la conexión: el motor no fija la conexión a la IP comprobada.
- La IP pública o de NAT del motor no se detecta como "propia" (solo se comparan las direcciones de sus interfaces).
- Las redirecciones nunca se siguen: un `3xx` cuenta como fallo permanente.

## Registro y rotación de tokens

Cada sistema se registra por su cuenta y se administra con un **token de
acceso** propio. No lo confundas con el **token de entrega**: este último es el
que el motor envía en `Authorization` a tu receptor; el de acceso es el que
vos enviás al motor. El sistema se deduce del token de acceso: nunca va en el
body y no podés ver ni tocar los datos de otro sistema.

En los ejemplos, `https://motor.example` es la URL base del motor (la da el
equipo del motor), y `https://tu-sistema.example/hook` es tu receptor.

### 1. Pedí una llave de registro

Para registrar un sistema necesitás una **llave de registro** (`rk_…`). La
crea el administrador del motor con `POST /admin/registration-key` y te la
pasa por un canal seguro. Dura 24 horas por defecto y, mientras viva, sirve
para registrar varios sistemas; si la crean de nuevo, la anterior deja de
valer. No existe forma de generarla vos mismo. El resto de las llamadas
(`/systems/me/*`, `/eventos`) no la usan: van con tu token de acceso.

### 2. Registrar el sistema

```bash
curl --request POST 'https://motor.example/systems/register' \
  --header 'Content-Type: application/json' \
  --data '{
    "registration_key": "rk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "name": "mi-sistema",
    "password": "una-contraseña-larga-y-única",
    "delivery": {
      "url": "https://tu-sistema.example/hook",
      "token": "token-que-el-motor-enviara-a-tu-receptor"
    }
  }'
```

- `registration_key`: la llave que te dio el administrador; obligatoria.
- `name`: entre 3 y 50 caracteres, solo `a-z`, `0-9`, `_` y `-`. No se puede cambiar después.
- `password`: de 12 a 128 caracteres. Guardala: sirve para recuperar el acceso si perdés el token.
- `delivery.url`: `http` o `https`, hasta 2048 caracteres; ver "Restricciones de la URL de entrega". `delivery.token`: de 1 a 4096 caracteres, sin la palabra `Bearer` (el motor la antepone); usá ASCII imprimible, porque viaja en un header.
- El body es estricto: cualquier campo adicional da `400`.
- Antes de guardar, el motor hace un ping a tu URL con ese token (ver "Petición de ping") y solo registra el sistema si respondés `2xx`. El nombre se comprueba **después** del ping: con un nombre ocupado tu receptor recibe igual el ping y luego ves `409`.

Respuesta `201`:

```json
{ "name": "mi-sistema", "access_token": "cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
```

**El token de acceso se muestra una sola vez**: el motor solo guarda su hash.
Guardalo en tu gestor de secretos. Con la contraseña podés pedir otro (ver
"Recuperar el acceso").

| Código | Cuándo | Qué hacer |
|---|---|---|
| `400` | `{"message":"Body inválido"}`: JSON mal formado, falta `Content-Type: application/json`, nombre o contraseña fuera de formato, URL inválida, falta algún campo (incluida `registration_key`) o sobra uno | Corregí el body |
| `401` | `Llave de registro inválida o vencida` | Pedí una llave nueva al administrador |
| `413` | `Body demasiado grande` (más de 100 KB) | Reducí el body |
| `409` | `El nombre ya está en uso` (esto revela que el nombre existe; es intencional) | Elegí otro nombre |
| `422` | `No se pudo verificar el sistema de entrega`, con `motivo`: `La URL de entrega no está permitida`, o el ping falló (`http://tu-sistema.example responded 500`, `Request to http://tu-sistema.example failed (TypeError)` si no hubo conexión, `… (AbortError)` si hubo timeout) | Revisá la URL y que tu receptor responda `2xx` al ping |
| `429` | `Demasiados intentos` | Esperá: el límite es por IP y por hora (`AUTH_RATE_LIMIT_PER_HOUR`, 30 por defecto; cuenta todas las llamadas a `register` y `recover`) |
| `503` | `No se pudo resolver el host de entrega, reintente más tarde` (DNS) o `No se pudo procesar la solicitud` (también si falla Mongo al verificar la llave) | Reintentá más tarde. Si el segundo mensaje llegó al registrar, el sistema pudo haber quedado creado: usá `recover` |

### 3. Usar el token de acceso

Todas las demás llamadas llevan `Authorization: Bearer cev_…`:

```bash
curl 'https://motor.example/systems/me' \
  --header 'Authorization: Bearer cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

`GET /systems/me` devuelve tu nombre, estado, el **origen** de tu URL de
entrega (nunca el token) y tus tokens de acceso activos (solo prefijo y
vencimiento; `expiresAt` es `null` si no vence):

```json
{
  "name": "mi-sistema",
  "status": "active",
  "delivery": { "origin": "https://tu-sistema.example" },
  "tokens": [
    { "prefix": "cev_ab12cd34", "createdAt": "2026-10-26T15:00:00.000Z", "expiresAt": null }
  ]
}
```

Errores de autenticación, comunes a todas las rutas con Bearer: `401`
`{"message":"Token inválido"}` (ausente, mal formado, desconocido o vencido),
`403` `Sistema revocado` y `503` `Autenticación no disponible`.

### 4. Activar y desactivar eventos

`POST /eventos`, `PATCH /eventos/estado` y `GET /eventos` usan el mismo
Bearer. **El body no lleva `system`** (un body con `system` o cualquier campo
desconocido da `400`): las suscripciones son siempre las de tu sistema.

```bash
curl --request POST 'https://motor.example/eventos' \
  --header 'Authorization: Bearer cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "evento": "payment_create", "es_activo": true }'

curl --request PATCH 'https://motor.example/eventos/estado' \
  --header 'Authorization: Bearer cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "evento": "payment_create", "es_activo": false }'

curl 'https://motor.example/eventos?id_empresa=123' \
  --header 'Authorization: Bearer cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

`GET /eventos` lista tus suscripciones (activas o no) como
`{ "suscripciones": [{ id_empresa, evento, system, es_activo, created_at, updated_at }] }`;
`id_empresa` es opcional y, si lo mandás, debe ser un entero positivo (`400` si no).
Los detalles de activación (cuerpos, errores y límites) están en la sección 7 de la [guía del desarrollador](guia-desarrollador.md#7-suscripciones-a-eventos).

### 5. Rotar el token de acceso

```bash
curl --request POST 'https://motor.example/systems/me/tokens' \
  --header 'Authorization: Bearer cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

Responde `201 { "access_token": "cev_…" }` (se muestra una sola vez). El token
con el que llamaste **sigue valiendo durante la gracia**
(`ACCESS_TOKEN_GRACE_MS`, 24 horas por defecto) y después vence. En ese lapso
valen ambos, así que podés actualizar tu configuración sin corte. Como máximo
hay **dos tokens activos**: si ya hay dos, responde `409`
`Ya hay dos tokens de acceso activos`; esperá a que venza el anterior o usá
"Recuperar el acceso". Si el token con el que llamás fue invalidado mientras
tanto (por una rotación o un `recover` concurrente), responde `401`.

### 6. Recuperar el acceso

Si perdiste el token, o sospechás que se filtró:

```bash
curl --request POST 'https://motor.example/systems/token/recover' \
  --header 'Content-Type: application/json' \
  --data '{ "name": "mi-sistema", "password": "una-contraseña-larga-y-única" }'
```

Responde `200 { "access_token": "cev_…" }` y **revoca de inmediato todos los
tokens anteriores**. Cualquier fallo (nombre inexistente, contraseña errónea,
sistema sin contraseña o revocado) da el mismo `401`
`{"message":"Credenciales inválidas"}`; un sistema revocado no puede recuperar
acceso. Comparte con `register` el límite por IP (`429`).

### 7. Cambiar la URL o el token de entrega

```bash
curl --request PATCH 'https://motor.example/systems/me/delivery' \
  --header 'Authorization: Bearer cev_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' \
  --header 'Content-Type: application/json' \
  --data '{ "token": "nuevo-token-de-entrega" }'
```

Body: `url` y/o `token` (al menos uno; estricto). El motor comprueba la URL, hace
ping con los valores resultantes y **solo guarda si responde `2xx`**. Respuesta
`204` sin body. Errores: `400` (body inválido), `422` (URL no permitida o ping
sin `2xx`, igual que en el registro), `503` (DNS u otro fallo). Los eventos en
vuelo usan el valor nuevo en su siguiente intento. Para no cortar entregas,
seguí los pasos de "Rotación de token sin corte".

Límites: el rate limit de `/systems/register` y `/systems/token/recover`
es independiente del de `/admin`; las rutas con Bearer no tienen ese límite.

## Rotación de token sin corte

El motor lee el token de entrega al momento de cada envío, así que un cambio
aplica de inmediato, también a los reintentos de eventos en vuelo. Para rotar
sin perder eventos:

1. Hacé que tu receptor acepte **el token viejo y el nuevo** a la vez.
2. Actualizá el token de entrega de tu sistema en el motor con `PATCH /systems/me/delivery` (ver "Registro y rotación de tokens"); el motor hace ping con el token nuevo antes de guardarlo.
3. Cuando ya no lleguen peticiones con el token viejo, retiralo de tu receptor.

Los `401`/`403` son reintentables justamente para absorber el desfase entre
cambiar el token en el motor y en tu receptor, pero no dependas de eso: aceptar
ambos tokens evita reintentos innecesarios.

## Ejemplo: evento `customer_update`

Petición que recibe tu endpoint (token de ejemplo):

```http
POST /hooks/cuenti HTTP/1.1
Host: receptor.example.com
Content-Type: application/json
Authorization: Bearer <token>
Idempotency-Key: PROD:erp:3249:918273:mi-sistema
X-Event-Id: 918273
X-Delivery-Attempt: 1
X-Delivery-Type: event
X-Contract-Version: 1

{"id_company":3249,"id_branch":0,"event":{"type":"customer_update","data":{"ref":4978}}}
```

Respuesta esperada:

```http
HTTP/1.1 200 OK
```

Cualquier `2xx` sirve; el body se ignora. Si el mismo `Idempotency-Key` llega
otra vez (por ejemplo `X-Delivery-Attempt: 2`), respondé `200` sin repetir el
efecto.
