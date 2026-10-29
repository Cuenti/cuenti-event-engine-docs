# Guía del desarrollador: integrar un sistema con el motor de eventos

> Contrato de entregas: **v1** (`X-Contract-Version: 1`).
> Esta guía describe el comportamiento del motor tal como está implementado hoy. Los valores marcados "por defecto" los puede cambiar el administrador del motor; si dependés de alguno, confirmalo con él.

## Índice

- [0. Portada](#0-portada)
- [1. Conceptos y glosario](#1-conceptos-y-glosario)
- [2. Visión general del flujo](#2-visión-general-del-flujo)
- [3. Antes de empezar](#3-antes-de-empezar)
  - [3.1 Llave de registro](#31-llave-de-registro)
  - [3.2 TLS](#32-tls)
  - [3.3 Tu receptor](#33-tu-receptor)
  - [3.4 Qué URL de entrega se acepta](#34-qué-url-de-entrega-se-acepta)
  - [3.5 Tamaños y tiempos](#35-tamaños-y-tiempos)
- [4. Autenticación y ciclo de vida de los tokens](#4-autenticación-y-ciclo-de-vida-de-los-tokens)
- [5. Registro del sistema](#5-registro-del-sistema)
- [6. Gestión de la cuenta](#6-gestión-de-la-cuenta)
- [7. Suscripciones a eventos](#7-suscripciones-a-eventos)
- [8. Recibir entregas](#8-recibir-entregas)
- [9. Tabla maestra de errores](#9-tabla-maestra-de-errores)
- [10. Límites y cuotas](#10-límites-y-cuotas)
- [11. Escenarios y preguntas frecuentes](#11-escenarios-y-preguntas-frecuentes)
- [12. Puesta en marcha y recetas de prueba](#12-puesta-en-marcha-y-recetas-de-prueba)
- [13. Apéndices](#13-apéndices)
  - [Apéndice A. Referencia rápida de endpoints](#apéndice-a-referencia-rápida-de-endpoints)
  - [Apéndice B. Esquemas JSON](#apéndice-b-esquemas-json)
  - [Apéndice C. Historial y compatibilidad](#apéndice-c-historial-y-compatibilidad)
- [14. Dónde pedir ayuda](#14-dónde-pedir-ayuda)

---

## 0. Portada

### Qué es esta guía

Es la referencia completa para quien desarrolla un **sistema destino**: un servicio que recibe eventos del motor por webhook. Explica cómo registrar tu sistema, cómo administrar tus tokens, cómo activar y desactivar suscripciones a eventos y cómo construir un receptor que cumpla el contrato de entregas v1.

### A quién va dirigida

A desarrolladores que conocen HTTP y JSON pero no conocen el funcionamiento interno del motor. No necesitás saber nada de su base de datos, de sus colas ni de su infraestructura: todo lo que te afecta está acá.

### Convenciones

| Elemento | Valor que usa esta guía |
|---|---|
| URL base del motor | `https://engine.example.com` (la URL real te la da el administrador del motor) |
| Token de acceso | `cev_xxxxxxxx…` |
| URL de tu receptor | `https://tu-sistema.example/hooks/eventos` |
| Token de entrega | `tu-token-de-entrega` |
| Nombre de sistema | `mi-sistema` |
| IP de ejemplo | `203.0.113.10` (IPv4) y `2001:db8::10` (IPv6), rangos reservados para documentación |

Los mensajes de error se citan **textualmente**, tal como los devuelve el motor (en español, con tildes).

### Cómo leerla

- Si es tu primera integración, seguí el orden: secciones 1 a 8 y después la checklist de la [sección 12](#12-puesta-en-marcha-y-recetas-de-prueba).
- Si ya estás integrado y algo falla, andá directo a la [tabla maestra de errores](#9-tabla-maestra-de-errores) o a los [escenarios](#11-escenarios-y-preguntas-frecuentes).

### El recorrido completo en 10 líneas

1. Pedile al administrador una **llave de registro** (`rk_…`).
2. Levantá un receptor HTTP(S) público que responda `2xx` a un `POST`.
3. Registrá tu sistema con `POST /systems/register` (llave de registro, nombre, contraseña, URL y token de entrega).
4. El motor hace un *ping* a tu receptor; si responde `2xx`, te devuelve tu **token de acceso** `cev_…`, una sola vez.
5. Guardá el token de acceso y la contraseña en tu gestor de secretos.
6. Activá los eventos que te interesan con `POST /eventos`, por empresa (`id_empresa`).
7. El motor te entrega cada evento con un `POST` a tu receptor, con tu token de entrega en `Authorization: Bearer`.
8. Respondé `2xx` rápido y deduplicá con `Idempotency-Key`: la entrega es "al menos una vez".
9. Rotá el token de acceso con `POST /systems/me/tokens` y el de entrega con `PATCH /systems/me/delivery`, sin corte.
10. Si perdés el token de acceso o se filtra, usá `POST /systems/token/recover` con tu contraseña.

---

## 1. Conceptos y glosario

| Término | Qué es |
|---|---|
| **Motor** | El servicio que lee eventos del origen (el ERP) y los entrega a los sistemas suscritos. Expone el API HTTP de esta guía. |
| **Sistema** | Tu integración, identificada por un **nombre** único e inmutable (por ejemplo `mi-sistema`). Tiene una URL de entrega, un token de entrega, una contraseña y uno o dos tokens de acceso. |
| **Estado del sistema** | `active` (normal) o `revoked` (revocado por el equipo del motor). Un sistema revocado no puede usar el API ni recibe entregas. |
| **Evento** | Algo que ocurrió en el origen, identificado por su tipo (`evento`, por ejemplo `payment_create`). El nombre es texto libre; tiene que coincidir exactamente con el tipo que emite el origen. |
| **Empresa (`id_empresa`)** | Identificador numérico positivo de la empresa en el origen. Las suscripciones son por empresa. En el body de las entregas viaja como `id_company`. |
| **Suscripción** | La combinación `id_empresa + evento + sistema`, con un estado `es_activo` (`true`/`false`). Solo existen suscripciones de tu sistema para vos: no ves ni tocás las de otros. |
| **Token de acceso** | Credencial **tuya hacia el motor**. Formato `cev_` + 43 caracteres. Se envía en `Authorization: Bearer cev_…` en todas las llamadas de gestión. El motor lo genera y lo muestra una sola vez. |
| **Token de entrega** | Credencial **del motor hacia tu receptor**. La elegís vos al registrarte. El motor la envía en `Authorization: Bearer <token de entrega>` en cada entrega y en cada ping, para que tu receptor sepa que la petición viene del motor. |
| **Contraseña** | Secreto de 12 a 128 caracteres que elegís al registrarte. Sirve solo para recuperar el acceso (`POST /systems/token/recover`). No hay endpoint para cambiarla. |
| **Gracia** | Lapso durante el cual el token de acceso anterior sigue valiendo después de una rotación. 24 horas por defecto. |
| **Ping** | `POST` de verificación que el motor envía a tu receptor al registrarte y al cambiar la URL o el token de entrega. Lleva `X-Delivery-Type: ping` y body `{}`. Solo se guarda el cambio si respondés `2xx`. |
| **Entrega** | `POST` que el motor hace a tu receptor con un evento. Lleva `X-Delivery-Type: event`. |
| **Intento** | Cada envío de una misma entrega. El primero es el 1; los siguientes son reintentos. |
| **Cola de fallidos (`.failed`)** | Donde queda una entrega cuando falla de forma permanente o agota los reintentos. No se reenvía sola: un operador del motor decide si reprocesarla. |
| **Llave de registro** | Credencial `rk_…` que crea el administrador y que exigís una vez, en el body de `POST /systems/register`. Dura 24 horas por defecto, sirve para registrar varios sistemas y solo hay una vigente: al crear otra, la anterior deja de valer. |
| **Contrato v1** | El formato único de las peticiones que el motor envía a los receptores: método, headers, body, ping y clasificación de respuestas. Ver [sección 8](#8-recibir-entregas). |

### Token de acceso vs. token de entrega

Son dos secretos distintos que viajan en sentidos opuestos. No los confundas:

| | Token de acceso | Token de entrega |
|---|---|---|
| Dirección | Tu código → motor | Motor → tu receptor |
| Quién lo genera | El motor | Vos |
| Formato | `cev_` + 43 caracteres `[A-Za-z0-9_-]` | Texto libre de 1 a 4096 caracteres |
| Dónde viaja | Header `Authorization: Bearer cev_…` de tus llamadas al API | Header `Authorization: Bearer <token>` de las entregas y pings |
| Cuántos | Hasta 2 activos a la vez | Uno solo |
| Cómo se cambia | `POST /systems/me/tokens` (rotación) o `POST /systems/token/recover` | `PATCH /systems/me/delivery` |
| Se puede volver a ver | No (se muestra una sola vez) | No (el motor nunca lo devuelve) |

---

## 2. Visión general del flujo

```mermaid
flowchart TD
    A["Pedir al administrador una llave de registro"] --> B["Levantar tu receptor HTTP(S)"]
    B --> C["POST /systems/register"]
    C --> D{"Ping a tu receptor responde 2xx?"}
    D -- No --> E["422: corregí URL o receptor y reintentá"]
    E --> C
    D -- Sí --> F["201: guardá el token de acceso cev_"]
    F --> G["POST /eventos: activar suscripciones"]
    G --> H["El origen emite eventos"]
    H --> I["El motor hace POST a tu receptor"]
    I --> J{"Tu respuesta"}
    J -- 2xx --> K["Entrega completada"]
    J -- "401, 403, 408, 429, 5xx, timeout o error de red" --> L["Reintento cada 30 s por defecto"]
    L --> I
    J -- "Otro 4xx o 3xx" --> M["Cola de fallidos"]
    L -- "Reintentos agotados" --> M
    F --> N["Rotar tokens sin corte"]
    N --> O["POST /systems/me/tokens"]
    N --> P["PATCH /systems/me/delivery"]
```

Cada llamada al API atraviesa estos controles, en este orden. Conocerlo te ayuda a interpretar qué error aparece primero:

| Orden | Control | Si falla |
|---|---|---|
| 1 | Lectura del body JSON (máx. 100 KB) | `400` `Body inválido` o `413` `Body demasiado grande` |
| 2 | Ruta | `404` `No encontrado` si la ruta o el método no existen |
| 3 | Límite de intentos (solo `register` y `recover`) | `429` `Demasiados intentos` |
| 4 | Token Bearer (rutas con autenticación) | `401` `Token inválido`, `403` `Sistema revocado`, `503` `Autenticación no disponible` |
| 5 | Validación del body de la ruta | `400` |
| 6 | Llave de registro (solo `register`) | `401` `Llave de registro inválida o vencida`, `503` `No se pudo procesar la solicitud` |
| 7 | Lógica de la operación | `404`, `409`, `422`, `503`, según la ruta |

Consecuencias prácticas:

- El motor no filtra por IP: el control de acceso es la llave de registro (para el alta) y tu token de acceso (para lo demás). Los chequeos de salud `GET /health` y `GET /ready` no llevan autenticación (solo devuelven estado) y las rutas `/admin` son del administrador y se protegen con su clave.
- Un fallo interno inesperado fuera de las rutas de esta guía responde `500` `{"message":"Error interno"}`.
- Un JSON mal formado responde `400` antes de mirar tu token.
- En `register` y `recover`, el límite de intentos se cuenta **antes** de validar el body: un body con campos incorrectos también consume un intento. Un JSON mal formado no lo consume, porque se rechaza antes.

---

## 3. Antes de empezar

### 3.1 Llave de registro

Para registrar tu sistema necesitás una **llave de registro**. Esto vale solo para `POST /systems/register`: el resto de las rutas de esta guía usan tu token de acceso. No existe forma de generarla vos mismo.

**Cómo conseguirla.** Escribile al administrador del motor. Él la crea con `POST /admin/registration-key` y te la pasa por un canal seguro. Es un texto que empieza con `rk_`.

| Dato | Detalle |
|---|---|
| Vigencia | 24 horas desde que el administrador la crea (valor por defecto) |
| Uso | Mientras esté vigente sirve para registrar varios sistemas (por ejemplo, staging y producción) |
| Una sola vigente | Si el administrador crea otra, la anterior deja de valer aunque no haya vencido |
| Dónde va | En el body de `register`, campo `registration_key` |

**Qué ves si falla.** Sin el campo, `400` `Body inválido`. Con una llave equivocada, vencida o que el administrador reemplazó, `401`:

```json
{
  "message": "Llave de registro inválida o vencida"
}
```

Pedí una llave nueva al administrador. Si el motor no pudo verificarla por un fallo interno, responde `503` `No se pudo procesar la solicitud`: reintentá. Una vez registrado, la llave no se vuelve a usar: no hace falta guardarla.

### 3.2 TLS

- **API del motor.** Tu contraseña y tu token de acceso viajan en el body y en los headers. Usá siempre la URL `https://` que te dé el administrador. Si solo te dan una URL `http://`, tus credenciales viajan en claro: avisale y tratá esas credenciales como expuestas en cualquier red no confiable.
- **Tu receptor.** El motor acepta URLs `http://` y `https://`. Usá `https://`: tu token de entrega viaja en el header `Authorization`. El certificado tiene que ser válido para el motor (cadena de confianza estándar); con un certificado inválido o autofirmado la conexión falla y el ping responde `422` (ver [5.6](#56-el-ping-de-verificación)).

### 3.3 Tu receptor

Antes de registrarte necesitás un receptor funcionando, porque el registro hace un ping y solo se completa si respondés `2xx`. Requisitos mínimos:

1. Acepta `POST` en la URL que vas a registrar, con `Content-Type: application/json`.
2. Es alcanzable **desde el motor** (no desde tu máquina): DNS resoluble, firewall abierto hacia el motor.
3. Responde `2xx` al ping (`X-Delivery-Type: ping`, body `{}`) sin procesar nada.
4. Responde en menos de 20 segundos (por defecto).
5. Valida el header `Authorization: Bearer <tu token de entrega>`.
6. No redirige: un `3xx` cuenta como fallo. Registrá la URL final.

### 3.4 Qué URL de entrega se acepta

El motor comprueba la URL al registrarte, al cambiarla con `PATCH /systems/me/delivery`, antes de cada ping y **antes de cada entrega**, contra las IP a las que resuelve el host en ese momento.

**Formato:**

| Regla | Detalle |
|---|---|
| Protocolo | Solo `http:` o `https:` |
| Longitud | Máximo 2048 caracteres |
| Credenciales en la URL | Prohibidas (`https://usuario:clave@host/...` se rechaza) |
| Puerto | Cualquiera |
| Host | Nombre DNS, IPv4 o IPv6 entre corchetes (`https://[2001:db8::10]/hooks`) |
| Lista de hosts permitidos | Si el administrador configuró una, solo valen esos nombres de host (comparación exacta, sin distinguir mayúsculas, sin puerto). Por defecto no hay lista. Preguntale si aplica. |

**Direcciones rechazadas** (el host, o cualquiera de las IP a las que resuelve):

| Tipo | Rangos |
|---|---|
| Loopback y "esta red" | `127.0.0.0/8`, `0.0.0.0/8`, `::1`, `::` |
| Link-local (incluye metadata de nube) | `169.254.0.0/16` (incluye `169.254.169.254`), `fe80::/10` |
| Metadata de nube específica | `100.100.100.200`, `fd00:ec2::254` |
| Multicast y reservadas | `224.0.0.0` en adelante (incluye `255.255.255.255`), `ff00::/8` |
| IPv4 prohibida dentro de IPv6 | IPv4 mapeada (`::ffff:127.0.0.1`), IPv4 traducida (`::ffff:0:0:0/96`), NAT64 `64:ff9b::/96` con IPv4 prohibida, 6to4 `2002::/16` con IPv4 prohibida |
| Rangos IPv6 bloqueados enteros | IPv4-compatible `::/96`, NAT64 local `64:ff9b:1::/48` |
| El propio motor | Cualquier dirección de las interfaces de red del motor |
| No interpretable | Una dirección resuelta que no se puede clasificar |

Un receptor en el mismo host que el motor no se admite. Un nombre como `localhost` se rechaza porque resuelve a loopback. Si el host resuelve a varias IP, alcanza con que **una** esté prohibida para rechazar la URL.

**Direcciones permitidas** (además de las públicas):

| Tipo | Rangos |
|---|---|
| Privadas RFC 1918 | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| CGNAT | `100.64.0.0/10` (salvo `100.100.100.200`) |
| ULA IPv6 | `fc00::/7` (salvo `fd00:ec2::254`) |

**Qué respuesta da cada problema:**

| Situación | Al registrarte o con `PATCH /systems/me/delivery` | Durante una entrega |
|---|---|---|
| URL con protocolo distinto de http/https o mal formada | `400` `Body inválido` (falla la validación del body) | No aplica |
| Host o IP rechazada, credenciales en la URL, host fuera de la lista | `422` con `motivo` `La URL de entrega no está permitida` | Fallo permanente: la entrega va a la cola de fallidos |
| El DNS no resuelve | `503` `No se pudo resolver el host de entrega, reintente más tarde` | Fallo reintentable |

### 3.5 Tamaños y tiempos

| Concepto | Valor |
|---|---|
| Body máximo de tus llamadas al API | 100 KB; más grande responde `413` |
| Timeout del motor hacia tu receptor (ping y entregas) | 20 s por defecto, igual para todos los sistemas |
| Tamaño del body de una entrega | Depende del evento; el motor no le impone un límite propio (no verificado: tamaño máximo real de los eventos del origen) |

---

## 4. Autenticación y ciclo de vida de los tokens

### 4.1 Formato y uso

- Formato exacto: `cev_` seguido de 43 caracteres del alfabeto `A-Z`, `a-z`, `0-9`, `_`, `-` (47 caracteres en total).
- Header: `Authorization: Bearer cev_…`. La palabra `Bearer` no distingue mayúsculas y puede ir seguida de uno o más espacios.
- Las rutas que lo exigen: `GET /systems/me`, `POST /systems/me/tokens`, `PATCH /systems/me/delivery`, `POST /eventos`, `PATCH /eventos/estado` y `GET /eventos`.
- El sistema se deduce del token. Nunca mandes el nombre del sistema en el body.
- El prefijo visible del token son sus primeros 12 caracteres (`cev_` + 8). Es lo que muestra `GET /systems/me` para que identifiques cuál es cuál.

### 4.2 Reglas del ciclo de vida

| Regla | Detalle |
|---|---|
| Se muestra una sola vez | El motor guarda solo un hash. Si lo perdés, no se puede recuperar: se emite otro. |
| Máximo dos activos | Un sistema puede tener a lo sumo 2 tokens activos (sin vencer). Rotar con dos activos responde `409` `Ya hay dos tokens de acceso activos`. |
| Token nuevo | Nace sin vencimiento (`expiresAt: null`). |
| Rotación | Al rotar, cada token activo sin vencimiento pasa a vencer al final de la gracia (24 h por defecto). El nuevo no vence. |
| Recuperación | `recover` emite un token nuevo y **vence de inmediato** todos los anteriores. |
| Vencido | Un token vencido responde `401` como uno desconocido. |
| Sistema revocado | Cualquier token del sistema responde `403` `Sistema revocado`. |

```mermaid
stateDiagram-v2
    [*] --> Activo: register, rotación o recover
    Activo --> EnGracia: rotación con POST /systems/me/tokens
    EnGracia --> Vencido: termina la gracia
    Activo --> Vencido: recover
    EnGracia --> Vencido: recover
    Vencido --> [*]
    note right of Activo
        Sin vencimiento, expiresAt null
    end note
    note right of EnGracia
        Vale hasta expiresAt, 24 h por defecto
    end note
```

### 4.3 Qué responde el motor ante un problema de autenticación

| Situación | Código | Body |
|---|---|---|
| Falta el header `Authorization` | `401` | `{"message":"Token inválido"}` |
| Esquema distinto de `Bearer` | `401` | `{"message":"Token inválido"}` |
| Token con formato distinto de `cev_` + 43 caracteres | `401` | `{"message":"Token inválido"}` |
| Token desconocido | `401` | `{"message":"Token inválido"}` |
| Token vencido (gracia terminada o anulado por `recover`) | `401` | `{"message":"Token inválido"}` |
| Token de un sistema que ya no existe | `401` | `{"message":"Token inválido"}` |
| Sistema revocado | `403` | `{"message":"Sistema revocado"}` |
| El motor no puede consultar sus credenciales | `503` | `{"message":"Autenticación no disponible"}` |

Todas las causas de `401` dan la misma respuesta a propósito: no hay forma de distinguir un token vencido de uno inventado.

### 4.4 Cómo guardarlo

- Guardá el token de acceso y la contraseña en un gestor de secretos, nunca en el repositorio ni en logs.
- No compartas el token entre entornos: registrá un sistema por entorno (ver [escenario 11.14](#11-escenarios-y-preguntas-frecuentes)).
- Si tu configuración admite dos valores, usá eso durante las rotaciones: el viejo y el nuevo valen a la vez durante la gracia.

---

## 5. Registro del sistema

### 5.1 Resumen

| | |
|---|---|
| Método y ruta | `POST /systems/register` |
| Autenticación | Llave de registro en el body (`registration_key`) |
| Límite | Comparte con `recover` un límite por IP y por hora (30 por defecto) |
| Respuesta de éxito | `201` con `name` y `access_token` |

### 5.2 Campos del body

El body es **estricto**: cualquier campo no listado responde `400`.

| Campo | Tipo | Obligatorio | Reglas exactas |
|---|---|---|---|
| `registration_key` | string | Sí | La llave que te dio el administrador ([3.1](#31-llave-de-registro)). Entre 1 y 256 caracteres; no se guarda como parte de tu sistema. |
| `name` | string | Sí | Patrón `^[a-z0-9_-]{3,50}$`: de 3 a 50 caracteres, solo minúsculas sin tilde, dígitos, `_` y `-`. Inmutable: no se puede cambiar después. |
| `password` | string | Sí | De 12 a 128 caracteres. |
| `delivery` | objeto | Sí | Estricto: solo `url` y `token`. |
| `delivery.url` | string | Sí | URL `http:` o `https:`, máximo 2048 caracteres. Además tiene que pasar las reglas de [3.4](#34-qué-url-de-entrega-se-acepta). |
| `delivery.token` | string | Sí | De 1 a 4096 caracteres. Registralo **sin** la palabra `Bearer`: el motor la antepone (ver [8.3](#83-autenticar-al-motor-en-tu-receptor)). Viaja como valor de un header HTTP: usá solo caracteres ASCII imprimibles, sin saltos de línea (un salto de línea hace fallar el ping con `Request to … failed (TypeError)`). |

Ejemplos de `name`: `mi-sistema` y `erp_sync2` son válidos; `Mi-Sistema` (mayúsculas), `ab` (corto), `mi sistema` (espacio) y `mi.sistema` (punto) no.

### 5.3 Petición de ejemplo

```bash
curl --request POST 'https://engine.example.com/systems/register' \
  --header 'Content-Type: application/json' \
  --data '{
    "registration_key": "rk_xxxxxxxx…",
    "name": "mi-sistema",
    "password": "una-contraseña-larga-y-única",
    "delivery": {
      "url": "https://tu-sistema.example/hooks/eventos",
      "token": "tu-token-de-entrega"
    }
  }'
```

```json
{
  "registration_key": "rk_xxxxxxxx…",
  "name": "mi-sistema",
  "password": "una-contraseña-larga-y-única",
  "delivery": {
    "url": "https://tu-sistema.example/hooks/eventos",
    "token": "tu-token-de-entrega"
  }
}
```

### 5.4 Respuesta de éxito

Código `201`:

```json
{
  "name": "mi-sistema",
  "access_token": "cev_xxxxxxxx…"
}
```

**El `access_token` se muestra una sola vez.** Guardalo antes de hacer cualquier otra cosa.

### 5.5 Qué hace el motor, en orden

```mermaid
sequenceDiagram
    participant Dev as Tu código
    participant Motor
    participant Rec as Tu receptor
    Dev->>Motor: POST /systems/register
    Motor->>Motor: 1. Lectura del JSON
    Motor->>Motor: 2. Cuenta un intento del límite por IP
    Motor->>Motor: 3. Valida el body
    Motor->>Motor: 4. Verifica la llave de registro
    Motor->>Motor: 5. Comprueba la URL y resuelve el DNS
    Motor->>Rec: 6. Ping POST con body vacío y X-Delivery-Type ping
    Rec-->>Motor: 2xx
    Motor->>Motor: 7. Crea el sistema si el nombre está libre
    Motor->>Motor: 8. Emite el token de acceso
    Motor-->>Dev: 201 con name y access_token
```

| Paso | Qué pasa | Si falla |
|---|---|---|
| 1 | Lectura del JSON | `400` `Body inválido` / `413` `Body demasiado grande` (no cuenta intento) |
| 2 | Se cuenta un intento para tu IP | `429` `Demasiados intentos` |
| 3 | Validación del body (incluye que venga `registration_key`) | `400` `Body inválido` (el intento ya se contó) |
| 4 | Verificación de la llave de registro | `401` `Llave de registro inválida o vencida` / `503` `No se pudo procesar la solicitud` si falla la base de datos |
| 5 | Guarda de la URL (formato, lista de hosts, IP resueltas) | `422` URL no permitida / `503` DNS |
| 6 | Ping a tu receptor con tu token de entrega | `422` con el motivo del ping |
| 7 | Creación del sistema | `409` `El nombre ya está en uso` |
| 8 | Emisión del token de acceso | `503` `No se pudo procesar la solicitud` (el sistema ya quedó creado: ver [5.8](#58-casos-límite)) |

Dos consecuencias importantes del orden:

- **El nombre se comprueba después del ping.** Si el nombre está ocupado, tu receptor recibe igual el ping y después ves `409`. Y si tu URL falla, ves `422` aunque el nombre esté ocupado.
- **Nada se guarda si el ping falla.** Podés corregir y repetir con el mismo nombre.

### 5.6 El ping de verificación

```http
POST /hooks/eventos HTTP/1.1
Host: tu-sistema.example
Content-Type: application/json
Authorization: Bearer tu-token-de-entrega
X-Delivery-Type: ping
X-Contract-Version: 1

{}
```

| Aspecto | Comportamiento |
|---|---|
| Método | `POST` |
| Headers del contrato | `Content-Type: application/json`, `Authorization: Bearer <token de entrega>`, `X-Delivery-Type: ping`, `X-Contract-Version: 1`. No lleva `Idempotency-Key`, `X-Event-Id` ni `X-Delivery-Attempt`. |
| Body | `{}` |
| Éxito | Cualquier `2xx`. El body de tu respuesta se ignora. |
| Fallo | Cualquier otro código, incluidos `401`, `403` y `3xx` |
| Redirecciones | No se siguen: un `3xx` es fallo |
| Timeout | 20 s por defecto, hasta recibir los headers de tu respuesta |
| Reintentos | Ninguno: un solo intento por llamada |
| Origen | Sale desde el host del motor |
| Cuándo se envía | Al registrarte y en cada `PATCH /systems/me/delivery`. No hay pings periódicos. |

Formato del `motivo` del `422` cuando el ping falla:

| Causa | `motivo` |
|---|---|
| Tu receptor respondió un código distinto de `2xx` | `https://tu-sistema.example responded 500` (origen de la URL y código) |
| Timeout | `Request to https://tu-sistema.example failed (AbortError)` |
| Conexión rechazada, TLS inválido u otro error de red | `Request to https://tu-sistema.example failed (TypeError)` |

El motivo nunca incluye la ruta, la query ni el token: solo el origen (`esquema://host:puerto`).

### 5.7 Respuestas de error

| Código | Body exacto | Causa | Qué hacer |
|---|---|---|---|
| `400` | `{"message":"Body inválido"}` | JSON mal formado; falta un campo (incluido `registration_key`); sobra un campo; `name` fuera del patrón; `password` fuera de 12 a 128; URL no `http`/`https` o de más de 2048; token de entrega vacío o de más de 4096; falta `Content-Type: application/json` | Corregí el body. Salvo el JSON mal formado, consume un intento del límite. |
| `401` | `{"message":"Llave de registro inválida o vencida"}` | La llave es incorrecta, venció (24 h por defecto) o el administrador creó otra | Pedí una llave nueva al administrador ([3.1](#31-llave-de-registro)) |
| `409` | `{"message":"El nombre ya está en uso"}` | Otro sistema tiene ese nombre (esto revela que existe, a propósito) | Elegí otro nombre. Si el sistema es tuyo y perdiste el token, usá `recover`. |
| `413` | `{"message":"Body demasiado grande"}` | Body de más de 100 KB | Reducí el body |
| `422` | `{"message":"No se pudo verificar el sistema de entrega","motivo":"La URL de entrega no está permitida"}` | La URL apunta a una dirección prohibida, lleva usuario o contraseña, o su host no está en la lista de hosts permitidos | Usá una URL pública o privada permitida ([3.4](#34-qué-url-de-entrega-se-acepta)) |
| `422` | `{"message":"No se pudo verificar el sistema de entrega","motivo":"https://tu-sistema.example responded 500"}` | El ping no obtuvo `2xx` (el motivo varía, ver [5.6](#56-el-ping-de-verificación)) | Hacé que tu receptor responda `2xx` al ping y reintentá |
| `429` | `{"message":"Demasiados intentos"}` | Superaste el límite por IP y por hora | Esperá a que se cumpla la hora desde el primer intento contado ([10.1](#101-límite-de-intentos)) |
| `503` | `{"message":"No se pudo resolver el host de entrega, reintente más tarde"}` | El DNS de tu host no resolvió | Revisá tu DNS y reintentá |
| `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno del motor (base de datos, cifrado, emisión del token) | Reintentá más tarde; si persiste, ver [5.8](#58-casos-límite) |
| `500` | `{"message":"Error interno"}` | Error inesperado del motor | Reintentá más tarde; si persiste, ver [sección 14](#14-dónde-pedir-ayuda) |

### 5.8 Casos límite

- **Alta a medias.** Si el motor creó el sistema pero falló al emitir el token, respondés `503` `No se pudo procesar la solicitud` y el sistema ya existe con tu contraseña. Un nuevo `register` con el mismo nombre dará `409`. Usá `POST /systems/token/recover` con el mismo nombre y contraseña para obtener tu token.
- **Dos registros simultáneos con el mismo nombre.** Gana uno; el otro recibe `409`.
- **URL con IPv6.** Escribila entre corchetes: `https://[2001:db8::10]/hooks/eventos`. Se aplica la misma lista de rangos prohibidos.
- **URL privada.** `http://10.20.30.40:8080/hooks` se acepta si el motor la alcanza. Es útil para receptores internos.
- **Loopback.** `http://127.0.0.1/...`, `http://localhost/...`, `http://[::1]/...` y `http://[::ffff:127.0.0.1]/...` se rechazan con `422`.
- **Nombre con mayúsculas.** `Mi-Sistema` responde `400`: el nombre no se normaliza.
- **El ping llega a tu receptor aunque después veas `409`.** Respondé `2xx` igual; no tiene efecto.

---

## 6. Gestión de la cuenta

### 6.1 Ver tu sistema: `GET /systems/me`

```bash
curl 'https://engine.example.com/systems/me' \
  --header 'Authorization: Bearer cev_xxxxxxxx…'
```

Respuesta `200`:

```json
{
  "name": "mi-sistema",
  "status": "active",
  "delivery": {
    "origin": "https://tu-sistema.example"
  },
  "tokens": [
    {
      "prefix": "cev_xxxxxxxx",
      "createdAt": "2026-09-01T12:00:00.000Z",
      "expiresAt": "2026-10-27T15:00:00.000Z"
    },
    {
      "prefix": "cev_yyyyyyyy",
      "createdAt": "2026-10-26T15:00:00.000Z",
      "expiresAt": null
    }
  ]
}
```

| Campo | Significado |
|---|---|
| `name` | Nombre de tu sistema |
| `status` | `active`. Un sistema revocado no llega acá: responde `403`. |
| `delivery.origin` | Solo el origen de tu URL de entrega (esquema, host y puerto). Si la URL guardada no se puede interpretar, vale `invalid-url`. |
| `tokens` | Tokens de acceso activos, del más viejo al más nuevo: prefijo (12 caracteres), creación y vencimiento (`null` = no vence). En el ejemplo, el primero está en gracia porque se rotó el 26 a las 15:00. |

**Lo que nunca devuelve:** la ruta completa de tu URL de entrega, tu token de entrega, tus tokens de acceso completos, tu contraseña.

Errores: los de autenticación ([4.3](#43-qué-responde-el-motor-ante-un-problema-de-autenticación)) y `503` `{"message":"No se pudo procesar la solicitud"}`.

### 6.2 Rotar el token de acceso: `POST /systems/me/tokens`

```bash
curl --request POST 'https://engine.example.com/systems/me/tokens' \
  --header 'Authorization: Bearer cev_xxxxxxxx…'
```

No lleva body. Respuesta `201`:

```json
{
  "access_token": "cev_yyyyyyyy…"
}
```

**Rotación sin corte, paso a paso:**

1. Comprobá con `GET /systems/me` que tenés un solo token activo. Si hay dos, esperá a que venza el que está en gracia (mirá su `expiresAt`).
2. Llamá a `POST /systems/me/tokens` con el token actual.
3. Guardá el token nuevo en tu gestor de secretos.
4. Desplegá tu configuración con el token nuevo. Durante la gracia (24 h por defecto) valen los dos.
5. Verificá con `GET /systems/me` usando el token nuevo.
6. Al terminar la gracia, el viejo vence solo y responde `401`.

| Código | Body | Causa | Qué hacer |
|---|---|---|---|
| `201` | `{"access_token":"cev_…"}` | Token nuevo emitido | Guardalo |
| `409` | `{"message":"Ya hay dos tokens de acceso activos"}` | Ya tenés dos activos (por ejemplo, rotaste hace menos de 24 h) | Esperá a que venza el anterior o usá `recover` |
| `401` | `{"message":"Token inválido"}` | El token con el que llamaste ya no está activo (un `recover` simultáneo lo anuló o su gracia terminó mientras esperaba turno), o cualquier causa de [4.3](#43-qué-responde-el-motor-ante-un-problema-de-autenticación) | Usá el token vigente |
| `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno | Reintentá |

Notas:

- El token con el que rotás tiene que seguir activo en el momento de rotar.
- Rotar con un token que ya está en gracia no extiende su vencimiento.
- Si hacés dos rotaciones a la vez con el mismo token y las atiende el mismo proceso del motor, una sola tiene éxito; la otra recibe `409`. Con varias instancias del motor esto no está garantizado: no rotes en paralelo.

### 6.3 Recuperar el acceso: `POST /systems/token/recover`

Para cuando perdiste el token o sospechás que se filtró. **Vence de inmediato todos los tokens anteriores**.

```bash
curl --request POST 'https://engine.example.com/systems/token/recover' \
  --header 'Content-Type: application/json' \
  --data '{ "name": "mi-sistema", "password": "una-contraseña-larga-y-única" }'
```

| Campo | Tipo | Obligatorio | Reglas |
|---|---|---|---|
| `name` | string | Sí | Máximo 100 caracteres |
| `password` | string | Sí | Máximo 128 caracteres |

Body estricto: cualquier otro campo responde `400`.

Respuesta `200`:

```json
{
  "access_token": "cev_zzzzzzzz…"
}
```

| Código | Body | Causa |
|---|---|---|
| `200` | `{"access_token":"cev_…"}` | Token nuevo; todos los anteriores vencidos |
| `400` | `{"message":"Body inválido"}` | Falta un campo, sobra uno, tipo incorrecto o excede el largo |
| `401` | `{"message":"Credenciales inválidas"}` | Nombre inexistente, contraseña incorrecta, sistema sin contraseña o sistema revocado. La respuesta es idéntica en todos los casos. |
| `429` | `{"message":"Demasiados intentos"}` | Límite por IP compartido con `register` |
| `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno |

Después de un `recover`, actualizá **todas** las instancias que usaban el token anterior: dejan de funcionar al instante.

### 6.4 Cambiar la URL o el token de entrega: `PATCH /systems/me/delivery`

| Body | Efecto |
|---|---|
| `{"url":"https://nuevo.example/hooks"}` | Cambia solo la URL; se conserva el token de entrega actual |
| `{"token":"nuevo-token"}` | Cambia solo el token; se conserva la URL actual |
| `{"url":"…","token":"…"}` | Cambia ambos |

```bash
curl --request PATCH 'https://engine.example.com/systems/me/delivery' \
  --header 'Authorization: Bearer cev_xxxxxxxx…' \
  --header 'Content-Type: application/json' \
  --data '{ "token": "nuevo-token-de-entrega" }'
```

El motor comprueba la URL resultante, hace un **ping con la combinación resultante** (URL nueva o actual, token nuevo o actual) y **solo guarda si el ping responde `2xx`**. Si algo falla, no cambia nada.

| Código | Body | Causa |
|---|---|---|
| `204` | (sin body) | Cambio guardado |
| `400` | `{"message":"Body inválido"}` | Body vacío `{}`, campo no permitido (por ejemplo `system`), URL inválida (solo `http:` o `https:`), token vacío o de más de 4096 caracteres |
| `401`, `403`, `503` | Ver [4.3](#43-qué-responde-el-motor-ante-un-problema-de-autenticación) | Autenticación |
| `422` | `{"message":"No se pudo verificar el sistema de entrega","motivo":"…"}` | URL no permitida o ping sin `2xx`, igual que en el registro |
| `503` | `{"message":"No se pudo resolver el host de entrega, reintente más tarde"}` | DNS |
| `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno |

**Efecto sobre las entregas en curso.** El motor lee la URL y el token en el momento de cada envío. Apenas el cambio se guarda, el próximo intento de cualquier entrega, incluidos los reintentos pendientes, usa los valores nuevos.

Este endpoint no tiene límite de intentos por hora.

**Rotar el token de entrega sin corte:**

1. Hacé que tu receptor acepte el token viejo **y** el nuevo.
2. Llamá a `PATCH /systems/me/delivery` con `{"token":"nuevo"}`. El ping llega con el token nuevo.
3. Cuando ya no lleguen peticiones con el token viejo, retiralo de tu receptor.

**Cambiar la URL sin corte:**

1. Levantá el receptor nuevo y verificá que responde `2xx` al ping.
2. Mantené el viejo funcionando.
3. Llamá a `PATCH /systems/me/delivery` con `{"url":"nueva"}`.
4. Apagá el viejo cuando deje de recibir tráfico (una entrega que ya estaba en vuelo hacia la URL vieja puede terminar ahí).

### 6.5 Situaciones de cuenta

| Situación | Qué hacer |
|---|---|
| Perdiste la contraseña | No hay recuperación ni cambio de contraseña por API. Pedile al administrador del motor que te reemita las credenciales: recibís una contraseña y un token nuevos, y los tokens anteriores quedan anulados. |
| Perdiste todos los tokens pero tenés la contraseña | `POST /systems/token/recover` |
| Perdiste los tokens y la contraseña | Pedile al administrador que te reemita las credenciales |
| Se filtró tu token de acceso | `POST /systems/token/recover` de inmediato (anula todos), desplegá el token nuevo y revisá tus suscripciones con `GET /eventos` |
| Se filtró tu token de entrega | Rotalo con `PATCH /systems/me/delivery` siguiendo los pasos de 6.4 y dejá de aceptar el viejo en tu receptor |
| Se filtró tu contraseña | Pedile al administrador que te reemita las credenciales (no podés cambiarla vos) y, mientras tanto, no dependas de que la contraseña siga siendo secreta: quien la tenga puede hacer `recover` |
| Tu sistema fue revocado | Todas tus llamadas con Bearer responden `403` `Sistema revocado`, `recover` responde `401` y no recibís entregas. Hablá con el equipo del motor: la revocación y la reactivación no se hacen por API. |
| Querés cambiar el nombre | No se puede. Registrá un sistema nuevo y activá sus suscripciones. |
| Querés borrar tu sistema | No hay endpoint de baja. Desactivá tus suscripciones y pedile al equipo del motor que lo revoque. |

---

## 7. Suscripciones a eventos

### 7.1 Modelo

- Una suscripción es `id_empresa + evento + tu sistema`, con `es_activo`.
- Solo existen para vos las suscripciones de tu sistema. Activar o desactivar la tuya no afecta las de otros sistemas.
- Desactivar nunca borra: la suscripción queda con `es_activo: false` y se puede reactivar.
- Un mismo evento se entrega una vez a **cada** sistema con suscripción activa para esa empresa y ese evento, de forma independiente.

### 7.2 Endpoints

| Método y ruta | Para qué | `es_activo` |
|---|---|---|
| `POST /eventos` | Activar (o fijar el estado de) uno o varios eventos | Opcional, por defecto `true` |
| `PATCH /eventos/estado` | Desactivar o reactivar uno o varios eventos | Obligatorio |
| `GET /eventos` | Listar tus suscripciones | No aplica |

`POST /eventos` y `PATCH /eventos/estado` hacen la misma operación; la única diferencia es que en `PATCH` el campo `es_activo` es obligatorio. En los dos:

| Estado previo | `es_activo: true` | `es_activo: false` |
|---|---|---|
| No existe | La crea activa (`200`) | `404` `La suscripción no existe` |
| Existe activa | Sigue activa (`200`, actualiza `updated_at`) | La desactiva (`200`) |
| Existe inactiva | La reactiva (`200`) | Sigue inactiva (`200`) |

### 7.3 Reglas de validación del body

El body es **estricto**: cualquier campo no listado responde `400`, en particular `system` y `delivery`.

| Campo | Tipo | Obligatorio | Reglas |
|---|---|---|---|
| `id_empresa` | número o string | Sí | Número entero positivo seguro (`123`), o string de dígitos sin cero inicial (`"123"`). Se aceptan espacios alrededor del string (`" 123 "`). No valen `0`, negativos, decimales, `"0123"` ni `"12a"`. En las respuestas siempre vuelve como string. |
| `evento` | string | Uno de `evento` o `eventos` | Se recortan los espacios de los extremos; no puede quedar vacío. Distingue mayúsculas. Sin largo máximo propio. |
| `eventos` | array de strings | Uno de `evento` o `eventos` | Al menos 1 elemento; cada uno con las reglas de `evento`. Sin máximo de elementos (solo el límite de 100 KB del body). |
| `es_activo` | boolean | En `PATCH` sí; en `POST` no | `true` o `false` literales (`"true"` como string da `400`) |

Podés mandar `evento` y `eventos` a la vez: se combinan. Los duplicados se eliminan, manteniendo el primer orden de aparición (`evento` primero, después `eventos`).

El nombre del evento tiene que coincidir **exactamente** con el tipo que emite el origen (`event.type` en las entregas). Si activás `Payment_Create` y el origen emite `payment_create`, no vas a recibir nada. Consultá con el equipo del motor la lista de tipos disponibles.

### 7.4 Respuesta según la cantidad de eventos

La cantidad se cuenta **después** de eliminar duplicados.

| Caso | Código global | Body |
|---|---|---|
| 1 evento, éxito | `200` | El registro de la suscripción |
| 1 evento, no existe y pediste desactivar | `404` | `{id_empresa, evento, system, message: "La suscripción no existe"}` |
| 1 evento, fallo del motor | `503` | `{id_empresa, evento, system, message: "No se pudo confirmar la configuración; puede reintentar la solicitud"}` |
| Varios eventos, todos con éxito | `200` | `{"resultados":[…]}` |
| Varios eventos, al menos uno falló | `207` | `{"resultados":[…]}` |

Cada elemento de `resultados` tiene `status` (`200`, `404` o `503`) y `body` (el mismo body que tendría la respuesta de ese evento solo), en el mismo orden que los eventos del pedido. Cada evento es una escritura independiente: **no es todo o nada**. Un `207` puede tener todos sus elementos fallidos.

Ojo: `{"eventos":["a"]}` y `{"eventos":["a","a"]}` son un solo evento y responden con el formato de evento único, sin `resultados`.

### 7.5 Activar un evento

```bash
curl --request POST 'https://engine.example.com/eventos' \
  --header 'Authorization: Bearer cev_xxxxxxxx…' \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "evento": "payment_create" }'
```

Respuesta `200`:

```json
{
  "id_empresa": "123",
  "evento": "payment_create",
  "system": "mi-sistema",
  "es_activo": true,
  "created_at": "2026-10-26T15:00:00.000Z",
  "updated_at": "2026-10-26T15:00:00.000Z"
}
```

`created_at` se fija al crear la suscripción y no cambia; `updated_at` cambia en cada escritura, aunque el estado sea el mismo. Ignorá cualquier campo adicional que pudiera aparecer.

### 7.6 Activar varios eventos

```bash
curl --request POST 'https://engine.example.com/eventos' \
  --header 'Authorization: Bearer cev_xxxxxxxx…' \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": 123, "eventos": ["payment_create", "invoice_create"] }'
```

Respuesta `200`:

```json
{
  "resultados": [
    {
      "status": 200,
      "body": {
        "id_empresa": "123",
        "evento": "payment_create",
        "system": "mi-sistema",
        "es_activo": true,
        "created_at": "2026-10-26T15:00:00.000Z",
        "updated_at": "2026-10-26T15:00:00.000Z"
      }
    },
    {
      "status": 200,
      "body": {
        "id_empresa": "123",
        "evento": "invoice_create",
        "system": "mi-sistema",
        "es_activo": true,
        "created_at": "2026-10-26T15:00:00.000Z",
        "updated_at": "2026-10-26T15:00:00.000Z"
      }
    }
  ]
}
```

### 7.7 Desactivar y reactivar

```bash
curl --request PATCH 'https://engine.example.com/eventos/estado' \
  --header 'Authorization: Bearer cev_xxxxxxxx…' \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "evento": "payment_create", "es_activo": false }'
```

Respuesta `200`:

```json
{
  "id_empresa": "123",
  "evento": "payment_create",
  "system": "mi-sistema",
  "es_activo": false,
  "created_at": "2026-10-26T15:00:00.000Z",
  "updated_at": "2026-10-26T16:30:00.000Z"
}
```

Para reactivar, el mismo pedido con `"es_activo": true`.

Desactivar uno que no existe, respuesta `404`:

```json
{
  "id_empresa": "123",
  "evento": "evento_inexistente",
  "system": "mi-sistema",
  "message": "La suscripción no existe"
}
```

Desactivar varios con uno inexistente, respuesta `207`:

```json
{
  "resultados": [
    {
      "status": 200,
      "body": {
        "id_empresa": "123",
        "evento": "payment_create",
        "system": "mi-sistema",
        "es_activo": false,
        "created_at": "2026-10-26T15:00:00.000Z",
        "updated_at": "2026-10-26T16:30:00.000Z"
      }
    },
    {
      "status": 404,
      "body": {
        "id_empresa": "123",
        "evento": "evento_inexistente",
        "system": "mi-sistema",
        "message": "La suscripción no existe"
      }
    }
  ]
}
```

**Efecto de desactivar sobre entregas pendientes.** El motor comprueba la suscripción antes de cada intento. Si desactivás mientras una entrega espera un reintento, ese reintento no se envía y la entrega se descarta. Reactivar después no reenvía los eventos descartados ni los que ocurrieron mientras estaba inactiva.

### 7.8 Listar: `GET /eventos`

```bash
curl 'https://engine.example.com/eventos' \
  --header 'Authorization: Bearer cev_xxxxxxxx…'

curl 'https://engine.example.com/eventos?id_empresa=123' \
  --header 'Authorization: Bearer cev_xxxxxxxx…'
```

| Parámetro | Obligatorio | Reglas |
|---|---|---|
| `id_empresa` | No | Si va, tiene que cumplir `^[1-9][0-9]*$` exactamente (sin espacios). Repetido (`?id_empresa=1&id_empresa=2`) da `400`. |

Respuesta `200`, con tus suscripciones activas e inactivas, ordenadas por empresa (numéricamente) y después por evento:

```json
{
  "suscripciones": [
    {
      "id_empresa": "123",
      "evento": "invoice_create",
      "system": "mi-sistema",
      "es_activo": true,
      "created_at": "2026-10-26T15:00:00.000Z",
      "updated_at": "2026-10-26T15:00:00.000Z"
    },
    {
      "id_empresa": "123",
      "evento": "payment_create",
      "system": "mi-sistema",
      "es_activo": false,
      "created_at": "2026-10-26T15:00:00.000Z",
      "updated_at": "2026-10-26T16:30:00.000Z"
    }
  ]
}
```

Cada elemento trae exactamente esos seis campos. Sin suscripciones, la respuesta es `{"suscripciones":[]}`. No hay paginación.

| Código | Body | Causa |
|---|---|---|
| `400` | `{"message":"id_empresa debe ser un número positivo"}` | `id_empresa` inválido |
| `503` | `{"message":"No se pudieron consultar las suscripciones"}` | Fallo interno; reintentá |

### 7.9 Errores de validación (`400`)

Un JSON mal formado responde `{"message":"Body inválido"}`. Un JSON válido que no cumple las reglas responde:

```json
{
  "message": "Configuración de suscripción inválida",
  "errors": [
    {
      "code": "unrecognized_keys",
      "keys": ["system"],
      "path": [],
      "message": "Unrecognized key: \"system\""
    }
  ]
}
```

`errors` es la lista de problemas de validación, en inglés y con formato de la librería de validación. Ejemplos reales de elementos:

| Pedido | Elemento de `errors` |
|---|---|
| Sin `evento` ni `eventos` | `{"code":"custom","path":[],"message":"evento or eventos is required"}` |
| `PATCH` sin `es_activo` | `{"code":"custom","path":[],"message":"es_activo is required"}` |
| Con `system` o `delivery` | `{"code":"unrecognized_keys","keys":["system"],"path":[],"message":"Unrecognized key: \"system\""}` |
| `"es_activo":"true"` | `{"expected":"boolean","code":"invalid_type","path":["es_activo"],"message":"Invalid input: expected boolean, received string"}` |
| `"evento":"   "` | `{"origin":"string","code":"too_small","minimum":1,"inclusive":true,"path":["evento"],"message":"Too small: expected string to have >=1 characters"}` |
| `"eventos":[]` | `{"origin":"array","code":"too_small","minimum":1,"inclusive":true,"path":["eventos"],"message":"Too small: expected array to have >=1 items"}` |
| `"id_empresa":"0123"` | `{"origin":"string","code":"invalid_format","format":"regex","pattern":"/^[1-9]\\d*$/","path":["id_empresa"],"message":"Invalid string: must match pattern /^[1-9]\\d*$/"}` |
| `"id_empresa":0` o sin `id_empresa` | Un elemento con `"code":"invalid_union"` y `"path":["id_empresa"]` |
| Sin body o sin `Content-Type: application/json` | `{"expected":"object","code":"invalid_type","path":[],"message":"Invalid input: expected object, received undefined"}` |

Usá `path` y `code` para diagnosticar; no dependas del texto de `message` dentro de `errors`, que viene de la librería.

### 7.10 La primera activación de un evento para una empresa

Cuando una suscripción todavía no existe y **ningún** sistema tuvo nunca una suscripción para esa `id_empresa` y ese `evento` (activa o no), el motor le pide al origen que empiece a emitir ese evento para esa empresa, y espera su confirmación (hasta 20 s por defecto).

- Si el origen confirma, la suscripción se guarda y respondés `200`.
- Si no confirma, no responde a tiempo o el servicio de activación no está configurado, la suscripción **no** se guarda y ese evento responde `503` `No se pudo confirmar la configuración; puede reintentar la solicitud`.

Repetir exactamente el mismo pedido es seguro. Las activaciones siguientes de la misma empresa y evento (tuyas o de otro sistema) no consultan al origen. Desactivar nunca le pide al origen que deje de emitir.

### 7.11 Idempotencia y aislamiento

- Repetir un `POST` o `PATCH` con el mismo body deja el mismo estado. Solo cambia `updated_at`.
- Dos sistemas pueden tener la misma `id_empresa + evento`; cada uno ve y modifica solo la suya. Si otro sistema intenta desactivar "tu" suscripción, recibe `404`, porque para él no existe.

---

## 8. Recibir entregas

### 8.1 La petición de evento

```http
POST /hooks/eventos HTTP/1.1
Host: tu-sistema.example
Content-Type: application/json
Authorization: Bearer tu-token-de-entrega
Idempotency-Key: PROD:erp:3249:918273:mi-sistema
X-Event-Id: 918273
X-Delivery-Attempt: 1
X-Delivery-Type: event
X-Contract-Version: 1

{"id_company":3249,"id_branch":0,"event":{"type":"customer_update","data":{"ref":4978}}}
```

| Header | Valor | Uso |
|---|---|---|
| `Content-Type` | `application/json` | Siempre JSON |
| `Authorization` | `Bearer <tu token de entrega>` | Verificá que la petición viene del motor |
| `Idempotency-Key` | `<entorno>:erp:<id_empresa>:<id del evento>:<nombre del sistema>` | Clave única de esta entrega (evento por sistema). Es estable entre reintentos y reprocesos. Usala para deduplicar. Tratala como texto opaco. |
| `X-Event-Id` | Id numérico del evento en el origen | El mismo evento lleva el mismo id en todos los sistemas |
| `X-Delivery-Attempt` | `1`, `2`, … | Número de intento. Empieza en 1 y crece en cada intento; puede saltar números (por ejemplo, si un intento falló antes de llegar a tu receptor). |
| `X-Delivery-Type` | `event` | En el ping vale `ping` |
| `X-Contract-Version` | `1` | Versión del contrato |

La petición puede llevar otros headers estándar de HTTP (`Host`, `Content-Length` y los que agregue el cliente HTTP del motor). No forman parte del contrato: no dependas de ellos.

### 8.2 El body

El body es el objeto del evento **tal como lo emite el origen**. El motor no renombra campos y, si el evento trae campos adicionales, llegan intactos. Solo hace dos ajustes: agrega `id_company` (tomado de la empresa del evento) si el evento no lo trae, y quita los espacios de los extremos de `event.type`.

```json
{
  "id_company": 3249,
  "id_branch": 0,
  "event": {
    "type": "customer_update",
    "data": {
      "ref": 4978
    }
  }
}
```

| Campo | Tipo | Garantía |
|---|---|---|
| `id_company` | entero positivo | Siempre presente. Es la `id_empresa` de tu suscripción, como número. |
| `event` | objeto | Siempre presente |
| `event.type` | string no vacío | Siempre presente. Es el nombre del evento al que te suscribiste. Es el único lugar donde viaja el tipo. |
| `event.data` | objeto | Siempre presente. Contenido propio de cada tipo de evento. |
| `id_branch` | según el origen | Se envía tal cual viene (`0` o `null` significan "sin sucursal"). Si el evento no lo trae, no se agrega. |
| Otros campos | cualquiera | Pueden aparecer; ignorá los que no conozcas |

El esquema de `event.data` depende de cada tipo de evento y lo define el origen, no el motor (no verificado: no hay catálogo de esquemas en el motor).

### 8.3 Autenticar al motor en tu receptor

El motor arma el header así: `Authorization: Bearer ` + el token de entrega que registraste. Por eso:

- Registrá el token **sin** la palabra `Bearer`. Si registrás `Bearer abc`, vas a recibir `Authorization: Bearer Bearer abc`.
- En tu receptor, compará el valor después de `Bearer ` con tu token, preferentemente con una comparación de tiempo constante.
- Usá un token de caracteres imprimibles ASCII sin espacios: los caracteres no válidos en un header HTTP hacen fallar el envío.
- El ping lleva el mismo header: validalo también ahí.

### 8.4 Cómo responder

Respondé `2xx` cuando el evento quedó aceptado. El body de tu respuesta se ignora siempre (podés responder vacío).

| Tu respuesta | Clasificación | Qué hace el motor |
|---|---|---|
| `200` a `299` | Éxito | La entrega queda completada y no se reenvía |
| `401`, `403` | Reintentable | Reintenta (cubre el desfase durante una rotación del token de entrega) |
| `408`, `429` | Reintentable | Reintenta |
| `500` a `599` | Reintentable | Reintenta |
| Timeout (20 s por defecto) | Reintentable | Reintenta |
| Error de red, conexión rechazada, TLS, DNS | Reintentable | Reintenta |
| `3xx` (cualquier redirección) | Permanente | No sigue la redirección; va a la cola de fallidos sin reintentar |
| Cualquier otro `4xx` (`400`, `404`, `409`, `410`, `422`, …) | Permanente | Va a la cola de fallidos sin reintentar |
| URL que pasó a apuntar a una dirección prohibida | Permanente | Va a la cola de fallidos |

Detalles:

- El header `Retry-After` se ignora: los reintentos siguen el calendario del motor.
- El timeout cuenta hasta que llegan los headers de tu respuesta. Si tu proceso es largo, respondé `2xx` enseguida y procesá en segundo plano.
- Elegí los códigos con intención: un `400` o `404` por un error transitorio tuyo hace que pierdas el evento (queda en la cola de fallidos). Si querés que el motor reintente, respondé `503`.

### 8.5 Reintentos y cola de fallidos

```mermaid
flowchart TD
    A["Intento de entrega"] --> B{"Suscripción activa?"}
    B -- No --> C["Descartada, no se envía"]
    B -- Sí --> D{"Ya entregada antes con 2xx?"}
    D -- Sí --> E["No se reenvía"]
    D -- No --> R{"Sistema revocado?"}
    R -- Sí --> C
    R -- No --> F["POST a tu receptor"]
    F --> G{"Resultado"}
    G -- 2xx --> H["Completada"]
    G -- Permanente --> I["Cola de fallidos"]
    G -- Reintentable --> J{"Quedan reintentos?"}
    J -- Sí --> K["Espera 30 s por defecto"]
    K --> A
    J -- No --> I
    I --> L["Un operador decide si reprocesar"]
```

| Parámetro | Valor por defecto |
|---|---|
| Reintentos después del primer intento | 5 (hasta 6 intentos en total) |
| Espera entre intentos | 30 s, fija (sin crecimiento exponencial); puede ser algo mayor si hay cola |
| Timeout por intento | 20 s |
| Ventana aproximada total | Unos 2,5 minutos de esperas, más la duración de cada intento |
| Después de agotar los reintentos | La entrega queda en la cola de fallidos de tu sistema |
| Fallo permanente | Va a la cola de fallidos en el primer intento, sin consumir reintentos |

La cola de fallidos no se reprocesa sola y no tenés acceso a ella. Si necesitás que te reenvíen eventos, pedíselo al equipo del motor con los datos de la [sección 14](#14-dónde-pedir-ayuda) (los `X-Event-Id` o `Idempotency-Key` y el rango de horas).

**Cambios durante los reintentos:**

| Cambio | Efecto en el próximo intento |
|---|---|
| Cambiás la URL o el token de entrega | Usa los valores nuevos |
| Desactivás la suscripción | No se envía; la entrega se descarta |
| Tu sistema es revocado | No se envía; la entrega se descarta |
| Tu DNS pasa a resolver a una dirección prohibida | Fallo permanente, cola de fallidos |
| Tu DNS deja de resolver | Fallo reintentable |

**Cuándo empezás a recibir.** Después de registrarte, el motor empieza a consumir las entregas de tu sistema en hasta 30 segundos (por defecto). Los eventos de ese lapso no se pierden: esperan en tu cola.

### 8.6 Idempotencia y duplicados

La entrega es **al menos una vez**. El motor evita reenviar una entrega que ya recibió `2xx`, pero un mismo evento puede llegarte más de una vez si, por ejemplo:

- procesaste el evento pero tu respuesta llegó después del timeout;
- el motor no alcanzó a registrar tu `2xx` (reinicio o fallo interno);
- un operador reprocesó entregas.

Cómo deduplicar:

1. Usá `Idempotency-Key` como clave única. Es distinta por sistema y estable entre reintentos.
2. Guardá la clave de lo que ya procesaste. Si llega de nuevo, respondé `2xx` sin repetir el efecto.
3. `X-Event-Id` sirve para correlacionar el mismo evento entre sistemas, pero no es único por sí solo entre entornos: preferí `Idempotency-Key`.

### 8.7 Orden

**No hay garantía de orden.** El motor procesa eventos en paralelo y un reintento sale después de eventos más nuevos. Si el orden importa, usalo a partir de los datos del evento (por ejemplo, fechas o versiones dentro de `event.data`) y no del orden de llegada.

### 8.8 Concurrencia

Con la configuración por defecto, cada proceso del motor envía las entregas de un mismo sistema de a una por vez. Un receptor lento reduce tu ritmo de entrega, no el de otros sistemas. La cantidad de procesos del motor depende del despliegue (no verificado).

### 8.9 El ping

Ver [5.6](#56-el-ping-de-verificación). Es un `POST` a la misma URL, con `Content-Type: application/json`, `Authorization: Bearer <tu token de entrega>`, `X-Delivery-Type: ping`, `X-Contract-Version: 1` y el body `{}`. **No** lleva `Idempotency-Key`, `X-Event-Id` ni `X-Delivery-Attempt`. Distinguilo por `X-Delivery-Type: ping`, no por el body. Validá el token, respondé `2xx` sin procesar nada y sin exigir `Idempotency-Key`. Si respondés otra cosa, el registro (o el `PATCH /systems/me/delivery`) falla con `422`.

### 8.10 Eventos que no te interesan

El motor no filtra eventos por reglas de negocio. Si recibís uno que no vas a procesar, respondé `2xx` e ignoralo. Un `4xx` no reintentable lo manda a la cola de fallidos y le genera trabajo a un operador. Si no querés un tipo de evento nunca más, desactivá la suscripción.

### 8.11 Receptor de referencia

Receptor mínimo en Node.js con Express (Node 18 o superior; `npm install express`). Guardalo como `receptor.mjs` y arrancalo con `DELIVERY_TOKEN=<tu token de entrega> node receptor.mjs`. Registrá la URL `https://tu-sistema.example/hooks/eventos` (la ruta es `/hooks/eventos`). Hace cuatro cosas: valida el token, atiende el ping, deduplica por `Idempotency-Key` y responde `2xx` enseguida.

```js
import { createHash, timingSafeEqual } from 'node:crypto'
import express from 'express'

const DELIVERY_TOKEN = process.env.DELIVERY_TOKEN // el token de entrega que registraste (sin "Bearer")
const PORT = Number(process.env.PORT ?? 3000)
if (!DELIVERY_TOKEN) throw new Error('Falta DELIVERY_TOKEN')

// Comparación en tiempo constante (se hashea para igualar los largos).
const digest = (value) => createHash('sha256').update(value).digest()
const tokenValido = (header) => {
  const match = /^Bearer (.+)$/.exec(header ?? '')
  return match !== null && timingSafeEqual(digest(match[1]), digest(DELIVERY_TOKEN))
}

// Demo en memoria. En producción usá una tabla con restricción UNIQUE sobre la clave.
const procesadas = new Set()

const app = express()

app.post('/hooks/eventos', express.json({ limit: '1mb' }), (req, res) => {
  // 401 es reintentable para el motor: cubre el desfase de una rotación del token.
  if (!tokenValido(req.get('Authorization'))) return res.sendStatus(401)

  // Ping de verificación (registro y PATCH /systems/me/delivery): no se procesa.
  if (req.get('X-Delivery-Type') === 'ping') return res.sendStatus(204)

  const clave = req.get('Idempotency-Key')
  if (!clave) return res.sendStatus(400)
  if (procesadas.has(clave)) return res.sendStatus(200) // duplicado: ya lo tenías

  const { id_company, id_branch, event } = req.body
  procesadas.add(clave)

  // Respondé enseguida y procesá en segundo plano.
  setImmediate(() => {
    console.log(
      `evento ${req.get('X-Event-Id')} (intento ${req.get('X-Delivery-Attempt')})`,
      event.type, 'empresa', id_company, 'sucursal', id_branch, event.data
    )
  })
  res.sendStatus(202)
})

// Body que no es JSON válido: 400 (permanente) en vez de un 500 que el motor reintentaría.
app.use((err, _req, res, _next) => res.sendStatus(err.status ?? 500))

app.listen(PORT, () => console.log(`Receptor escuchando en :${PORT}`))
```

Probalo localmente antes de registrarte (`DELIVERY_TOKEN=mi-token node receptor.mjs`):

```bash
# Ping: 204
curl -i -X POST http://localhost:3000/hooks/eventos \
  -H 'Authorization: Bearer mi-token' -H 'Content-Type: application/json' \
  -H 'X-Delivery-Type: ping' -d '{}'

# Evento: 202 la primera vez, 200 si repetís la misma Idempotency-Key
curl -i -X POST http://localhost:3000/hooks/eventos \
  -H 'Authorization: Bearer mi-token' -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: PROD:erp:3249:918273:mi-sistema' -H 'X-Event-Id: 918273' \
  -H 'X-Delivery-Attempt: 1' -H 'X-Delivery-Type: event' -H 'X-Contract-Version: 1' \
  -d '{"id_company":3249,"id_branch":0,"event":{"type":"customer_update","data":{"ref":4978}}}'

# Token incorrecto: 401
curl -i -X POST http://localhost:3000/hooks/eventos -H 'Authorization: Bearer otro'
```

Detalles del ejemplo:

- Para que el motor te alcance, tu receptor tiene que estar detrás de una URL que cumpla las reglas de [3.4](#34-qué-url-de-entrega-se-acepta).
- `app.use(... express.json())` solo parsea si llega `Content-Type: application/json`; el motor siempre lo envía.
- Si tu proceso tarda, no hagas el trabajo dentro del manejador: encolalo (base de datos, cola de mensajes) y respondé `2xx`. Guardá la clave **antes** de responder, o bien en la misma transacción que el efecto.

---

## 9. Tabla maestra de errores

### 9.1 Por endpoint

| Endpoint | Código | Body exacto | Causa | Acción |
|---|---|---|---|---|
| Cualquiera con body | `400` | `{"message":"Body inválido"}` | JSON mal formado | Corregí el JSON |
| Cualquiera con body | `413` | `{"message":"Body demasiado grande"}` | Body de más de 100 KB | Reducí el body o partí el pedido |
| Ruta o método inexistente | `404` | `{"message":"No encontrado"}` | Ruta o método que no existe | Revisá método y ruta |
| Cualquiera | `500` | `{"message":"Error interno"}` | Error inesperado | Reintentá y reportalo |
| `POST /systems/register` | `400` | `{"message":"Body inválido"}` | Body fuera de las reglas | Corregí el body |
| `POST /systems/register` | `409` | `{"message":"El nombre ya está en uso"}` | Nombre ocupado | Elegí otro o usá `recover` |
| `POST /systems/register` | `422` | `{"message":"No se pudo verificar el sistema de entrega","motivo":"La URL de entrega no está permitida"}` | URL prohibida | Cambiá la URL |
| `POST /systems/register` | `422` | `{"message":"No se pudo verificar el sistema de entrega","motivo":"<origen> responded <código>"}` o `"Request to <origen> failed (<error>)"` | Ping sin `2xx` | Arreglá el receptor |
| `POST /systems/register` | `429` | `{"message":"Demasiados intentos"}` | Límite por IP y hora | Esperá |
| `POST /systems/register` | `503` | `{"message":"No se pudo resolver el host de entrega, reintente más tarde"}` | DNS | Revisá el DNS y reintentá |
| `POST /systems/register` | `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno | Reintentá; si `register` da `409` después, usá `recover` |
| `POST /systems/token/recover` | `400` | `{"message":"Body inválido"}` | Body fuera de las reglas | Corregí el body |
| `POST /systems/token/recover` | `401` | `{"message":"Credenciales inválidas"}` | Nombre, contraseña, sistema sin contraseña o revocado | Revisá los datos; si no tenés la contraseña, hablá con el administrador |
| `POST /systems/token/recover` | `429` | `{"message":"Demasiados intentos"}` | Límite por IP y hora | Esperá |
| `POST /systems/token/recover` | `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno | Reintentá |
| Rutas con Bearer | `401` | `{"message":"Token inválido"}` | Token ausente, mal formado, desconocido, vencido | Revisá el token o usá `recover` |
| Rutas con Bearer | `403` | `{"message":"Sistema revocado"}` | Sistema revocado | Hablá con el equipo del motor |
| Rutas con Bearer | `503` | `{"message":"Autenticación no disponible"}` | El motor no pudo verificar el token | Reintentá |
| `GET /systems/me` | `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno | Reintentá |
| `POST /systems/me/tokens` | `409` | `{"message":"Ya hay dos tokens de acceso activos"}` | Dos tokens activos | Esperá el vencimiento o usá `recover` |
| `POST /systems/me/tokens` | `401` | `{"message":"Token inválido"}` | Tu token fue anulado mientras rotabas | Usá el token vigente |
| `POST /systems/me/tokens` | `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno | Reintentá |
| `PATCH /systems/me/delivery` | `400` | `{"message":"Body inválido"}` | Body vacío, campo extra, URL o token inválidos | Corregí el body |
| `PATCH /systems/me/delivery` | `422` | `{"message":"No se pudo verificar el sistema de entrega","motivo":"…"}` | URL prohibida o ping sin `2xx` | Igual que en el registro |
| `PATCH /systems/me/delivery` | `503` | `{"message":"No se pudo resolver el host de entrega, reintente más tarde"}` | DNS | Reintentá |
| `PATCH /systems/me/delivery` | `503` | `{"message":"No se pudo procesar la solicitud"}` | Fallo interno | Reintentá |
| `POST /eventos`, `PATCH /eventos/estado` | `400` | `{"message":"Configuración de suscripción inválida","errors":[…]}` | Body fuera de las reglas | Revisá `errors` |
| `POST /eventos`, `PATCH /eventos/estado` | `404` | `{"id_empresa":"…","evento":"…","system":"…","message":"La suscripción no existe"}` | Desactivar algo que no existe | Activalo primero, o ignoralo |
| `POST /eventos`, `PATCH /eventos/estado` | `503` | `{"id_empresa":"…","evento":"…","system":"…","message":"No se pudo confirmar la configuración; puede reintentar la solicitud"}` | Fallo al guardar o el origen no confirmó la primera activación | Repetí el mismo pedido |
| `POST /eventos`, `PATCH /eventos/estado` | `207` | `{"resultados":[…]}` | Varios eventos con algún fallo | Reintentá solo los fallidos |
| `GET /eventos` | `400` | `{"message":"id_empresa debe ser un número positivo"}` | Query inválida | Corregí `id_empresa` |
| `GET /eventos` | `503` | `{"message":"No se pudieron consultar las suscripciones"}` | Fallo interno | Reintentá |

### 9.2 Por código HTTP

**`400` Bad Request**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Body inválido"}` | Cualquier ruta | JSON mal formado, o un JSON cuyo primer nivel no es objeto ni arreglo |
| `{"message":"Body inválido"}` | `/systems/*` | El body no cumple el esquema estricto de la ruta (incluye body ausente o sin `Content-Type: application/json`) |
| `{"message":"Configuración de suscripción inválida","errors":[…]}` | `POST /eventos`, `PATCH /eventos/estado` | El body no cumple las reglas de [7.3](#73-reglas-de-validación-del-body), incluido traer `system` o `delivery` |
| `{"message":"id_empresa debe ser un número positivo"}` | `GET /eventos` | Query `id_empresa` inválida o repetida |

**`401` Unauthorized**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Token inválido"}` | Rutas con Bearer | Token ausente, esquema distinto de Bearer, formato inválido, desconocido, vencido, de un sistema inexistente, o anulado por un `recover` mientras rotabas |
| `{"message":"Credenciales inválidas"}` | `POST /systems/token/recover` | Cualquier fallo de credenciales |

**`403` Forbidden**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Sistema revocado"}` | Rutas con Bearer | Tu sistema está revocado |

**`404` Not Found**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"No encontrado"}` | Cualquier ruta o método inexistente | Por ejemplo `GET /systems/register` o `DELETE /eventos` |
| `{…,"message":"La suscripción no existe"}` | `POST /eventos`, `PATCH /eventos/estado` | `es_activo: false` sobre una suscripción que no existe |

**`409` Conflict**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"El nombre ya está en uso"}` | `POST /systems/register` | Nombre ocupado |
| `{"message":"Ya hay dos tokens de acceso activos"}` | `POST /systems/me/tokens` | Límite de tokens |

**`413` Payload Too Large**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Body demasiado grande"}` | Cualquier ruta con body | Más de 100 KB |

**`422` Unprocessable Content**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"No se pudo verificar el sistema de entrega","motivo":"La URL de entrega no está permitida"}` | `register`, `PATCH /systems/me/delivery` | URL prohibida |
| `{"message":"No se pudo verificar el sistema de entrega","motivo":"<detalle del ping>"}` | `register`, `PATCH /systems/me/delivery` | Ping sin `2xx` |

**`429` Too Many Requests**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Demasiados intentos"}` | `register`, `recover` | Límite por IP y hora. La respuesta no trae `Retry-After`. |

**`500` Internal Server Error**

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Error interno"}` | Cualquiera | Error inesperado no clasificado |

**`503` Service Unavailable** (siempre reintentable)

| Body | Dónde | Causa |
|---|---|---|
| `{"message":"Autenticación no disponible"}` | Rutas con Bearer | El motor no pudo verificar el token |
| `{"message":"No se pudo procesar la solicitud"}` | `/systems/*` | Fallo interno |
| `{"message":"No se pudo resolver el host de entrega, reintente más tarde"}` | `register`, `PATCH /systems/me/delivery` | DNS |
| `{…,"message":"No se pudo confirmar la configuración; puede reintentar la solicitud"}` | `POST /eventos`, `PATCH /eventos/estado` | Fallo al guardar o primera activación no confirmada |
| `{"message":"No se pudieron consultar las suscripciones"}` | `GET /eventos` | Fallo interno |

---

## 10. Límites y cuotas

### 10.1 Límite de intentos

| Aspecto | Valor |
|---|---|
| Rutas afectadas | `POST /systems/register` y `POST /systems/token/recover`, con un **mismo contador** compartido |
| Clave | Tu IP, tal como la ve el motor |
| Máximo | 30 por hora, por defecto |
| Ventana | Fija de 1 hora desde el primer intento contado; al cumplirse la hora, el contador vuelve a cero |
| Qué cuenta | Toda llamada que llega a la ruta: éxitos, `400` de validación, `401` (incluida la llave de registro inválida), `409`, `422`, `503` y también las rechazadas con `429` |
| Qué no cuenta | Llamadas rechazadas antes: JSON mal formado (`400`), body demasiado grande (`413`) |
| Rutas sin límite | Todas las rutas con Bearer, incluida `PATCH /systems/me/delivery` |
| Alcance | El contador vive en la memoria de cada proceso del motor: se reinicia si el proceso se reinicia y, con varias instancias, cada una cuenta por separado |

Si varios sistemas comparten una IP, comparten también este contador.

### 10.2 Tamaños y longitudes

| Elemento | Mínimo | Máximo |
|---|---|---|
| Body de cualquier llamada | | 100 KB |
| `name` | 3 | 50, patrón `^[a-z0-9_-]{3,50}$` |
| `password` (registro) | 12 | 128 |
| `name` (recover) | | 100 |
| `password` (recover) | | 128 |
| `delivery.url` / `url` | | 2048 |
| `delivery.token` / `token` | 1 | 4096 |
| `evento` | 1 (sin contar espacios de los extremos) | Sin máximo propio |
| `eventos` | 1 elemento | Sin máximo propio (lo limita el body) |
| `id_empresa` (número) | 1 | Entero seguro de JavaScript (9007199254740991) |

### 10.3 Tokens y tiempos

| Elemento | Valor por defecto |
|---|---|
| Tokens de acceso activos por sistema | 2 (fijo) |
| Gracia tras rotar | 24 horas |
| Timeout del motor hacia tu receptor | 20 segundos |
| Reintentos de entrega | 5, cada 30 segundos |
| Espera de confirmación del origen en la primera activación | 20 segundos |
| Inicio de entregas tras el registro | Hasta 30 segundos |

---

## 11. Escenarios y preguntas frecuentes

**11.1 Me da `401` `Llave de registro inválida o vencida`.** La llave que usaste es incorrecta, venció (dura 24 horas por defecto) o el administrador creó otra, que anula la anterior. Pedile una llave nueva y reintentá el `register`.

**11.2 Mi IP cambió o salgo por IPv6.** No pasa nada: el motor no filtra por IP. Solo el límite de intentos de `register` y `recover` se cuenta por IP.

**11.3 Estoy detrás de un proxy o NAT.** El motor ve la IP de salida, no la de tu máquina. Esa IP solo se usa para el límite de intentos ([10.1](#101-límite-de-intentos)); si varios sistemas salen por la misma IP, comparten el contador.

**11.4 Mi receptor está caído.** Cada entrega se reintenta hasta 5 veces, cada 30 s (unos 2,5 minutos en total). Pasado eso, queda en la cola de fallidos y no se reenvía sola. Si la caída fue larga, pedile al equipo del motor que reprocese con los datos de la [sección 14](#14-dónde-pedir-ayuda).

**11.5 Mi receptor responde `401` o `403`.** Se reintenta, igual que un `5xx`. Si es por un token mal configurado, corregilo antes de que se agoten los reintentos. Revisá que no hayas registrado el token con la palabra `Bearer`.

**11.6 Mi receptor responde `404`.** Es permanente: la entrega va directo a la cola de fallidos. Revisá la ruta de la URL registrada (`GET /systems/me` solo muestra el origen: si dudás, volvé a fijarla con `PATCH /systems/me/delivery`).

**11.7 Mi receptor responde `500` o `503`.** Se reintenta.

**11.8 Mi receptor responde `429`.** Se reintenta a los 30 s; tu `Retry-After` se ignora.

**11.9 Mi receptor responde `302` u otra redirección.** Es permanente y la redirección no se sigue. En el registro o en el `PATCH`, el ping falla con `422` y `motivo` `<origen> responded 302`. Registrá la URL final (por ejemplo, con `https://` y la barra final correcta).

**11.10 Mi receptor tarda más de 20 s.** Es timeout y se reintenta: probablemente recibas el mismo evento varias veces. Respondé `2xx` enseguida y procesá en segundo plano.

**11.11 Cambio la URL de mi receptor.** `PATCH /systems/me/delivery` con `{"url":"…"}`. El motor hace ping a la URL nueva con tu token actual y solo guarda si responde `2xx` (`204`). Desde ese momento, todo intento nuevo va a la URL nueva. Ver los pasos en [6.4](#64-cambiar-la-url-o-el-token-de-entrega-patch-systemsmedelivery).

**11.12 Quiero rotar el token de acceso sin downtime.** `POST /systems/me/tokens`, desplegá el nuevo y dejá que el viejo venza en 24 h. Ver [6.2](#62-rotar-el-token-de-acceso-post-systemsmetokens).

**11.13 Se filtró mi token de acceso.** `POST /systems/token/recover` de inmediato: anula todos tus tokens. Después revisá tus suscripciones con `GET /eventos` y tu URL con `GET /systems/me` por si alguien las cambió.

**11.14 Quiero dos entornos (staging y producción).** Registrá un sistema por entorno, con nombres distintos (por ejemplo `mi-sistema-staging` y `mi-sistema`), cada uno con su receptor, su token de entrega y su contraseña. Si cada entorno tiene su propio motor, la URL base y la llave de registro se gestionan por separado en cada uno.

**11.15 Perdí la contraseña.** No hay recuperación por API. Pedile al administrador que te reemita las credenciales.

**11.16 Me da `429`.** Superaste los intentos de `register` y `recover` desde tu IP en la última hora (30 por defecto, compartidos). Esperá a que se cumpla la hora desde el primer intento. No reintentes en bucle: cada intento rechazado también cuenta y la respuesta no trae `Retry-After`.

**11.17 El registro dio `422` por el ping.** Mirá el `motivo`: `responded <código>` te dice qué respondió tu receptor; `failed (AbortError)` es timeout; `failed (TypeError)` es conexión rechazada, TLS inválido o error de red. Corregí y repetí: no quedó nada guardado.

**11.18 El nombre está ocupado.** `409` `El nombre ya está en uso`. Si es tuyo, usá `recover`; si no, elegí otro. Recordá que el ping se envía igual antes del `409`.

**11.19 Quiero pausar todos los eventos.** No hay un interruptor global. Listá con `GET /eventos` y desactivá con `PATCH /eventos/estado` usando `eventos` por cada `id_empresa`. Las entregas pendientes de reintento se descartan. Alternativa sin perder eventos: responder `503` desde tu receptor durante una pausa corta (menos de unos 2,5 minutos); pasado ese tiempo, los eventos van a la cola de fallidos.

**11.20 Activo un evento que ya estaba activo.** `200` con el registro: el estado no cambia y solo se actualiza `updated_at`. No consulta al origen.

**11.21 Desactivo uno que no existe.** `404` `La suscripción no existe` (o ese resultado dentro de un `207` si mandaste varios). No crea nada.

**11.22 Mando 100 eventos en el arreglo.** Se aceptan, siempre que el body no pase de 100 KB. Cada evento se procesa por separado y en paralelo; vas a recibir `200` si todos salen bien o `207` con el detalle. Los duplicados se eliminan antes.

**11.23 Envío la empresa como texto.** `"123"` y `123` son equivalentes. `" 123 "` también se acepta en el body (no en la query de `GET /eventos`). `"0123"`, `"12a"`, `0` y `1.5` dan `400`. En las respuestas vuelve siempre como string.

**11.24 Mis eventos llegan duplicados.** Es esperable: la entrega es al menos una vez. Deduplicá por `Idempotency-Key`. Si los duplicados son muchos, revisá que respondas antes del timeout.

**11.25 Mis eventos llegan desordenados.** No hay garantía de orden. Ordená por los datos del evento.

**11.26 Mi sistema fue revocado.** `403` `Sistema revocado` en las rutas con Bearer, `401` en `recover`, y no recibís entregas (las pendientes se descartan). Hablá con el equipo del motor.

**11.27 El motor responde `503`.** Es reintentable en todos los casos. Reintentá con espera creciente (por ejemplo 1 s, 2 s, 4 s…). Mirá el `message` para saber si es DNS de tu receptor, autenticación, la verificación de la llave de registro o un fallo interno.

**11.28 La base de datos o la mensajería del motor no están disponibles.** Vas a ver `503` (`Autenticación no disponible`, `No se pudo procesar la solicitud`, `No se pudieron consultar las suscripciones`, o el `503` por evento). Reintentá más tarde.

**11.29 Recibí un `207`. ¿Qué hago?** Recorré `resultados`: cada elemento trae su `status`. Los `200` ya quedaron aplicados. Reintentá solo los `503`. Los `404` en una desactivación significan que esa suscripción no existía.

**11.30 ¿Es seguro reintentar una llamada fallida?**

| Llamada | ¿Repetir es seguro? |
|---|---|
| `POST /eventos`, `PATCH /eventos/estado` | Sí: dejan el mismo estado |
| `GET /eventos`, `GET /systems/me` | Sí |
| `PATCH /systems/me/delivery` | Sí: el resultado es el mismo y hace otro ping |
| `POST /systems/register` | Sí, pero si el primer intento llegó a crear el sistema, el segundo da `409`: usá `recover` |
| `POST /systems/me/tokens` | Con cuidado: si el primero tuvo éxito y perdiste la respuesta, el segundo da `409` y el token emitido se perdió. Usá `recover` si necesitás recuperar el control. |
| `POST /systems/token/recover` | Sí, pero cada éxito anula el token del anterior: quedate con el último |

**11.31 Mi URL de entrega apunta a una IP privada.** Se permite (`10/8`, `172.16/12`, `192.168/16`, `100.64/10`, `fc00::/7`) si el motor la alcanza.

**11.32 Mi receptor está en el mismo servidor que el motor.** No se admite: loopback y las direcciones del propio motor se rechazan con `422`.

**11.33 Recibí un `POST` con body `{}`.** Es un ping (`X-Delivery-Type: ping`). Respondé `2xx`.

**11.34 Activé un evento pero no recibo nada.** Revisá en orden: (1) `GET /eventos` muestra la suscripción con `es_activo: true` para la `id_empresa` correcta; (2) el nombre del evento coincide exactamente con el tipo que emite el origen; (3) el origen efectivamente generó eventos de ese tipo para esa empresa; (4) tu receptor no está devolviendo `4xx` (que mandan las entregas a la cola de fallidos); (5) si te registraste hace segundos, esperá hasta 30 s.

**11.35 Desactivé y reactivé: ¿recibo lo que pasó en el medio?** No. Los eventos ocurridos mientras estaba inactiva no se entregan, y las entregas que esperaban reintento al desactivar se descartaron.

**11.36 Mando `system` en el body para elegir el sistema.** `400`. El sistema siempre es el dueño del token. Para operar otro sistema, usá su token.

**11.37 Llamo con el método equivocado** (por ejemplo `GET /systems/register` o `PUT /eventos`). `404` `{"message":"No encontrado"}`.

**11.38 Mi token de entrega tiene espacios o caracteres especiales.** Evitalo: usá caracteres ASCII imprimibles sin espacios. Los caracteres que no son válidos en un header HTTP hacen fallar el envío (en el ping, `422` con `failed (TypeError)`).

**11.39 Quiero saber qué token estoy usando.** `GET /systems/me` lista los prefijos (primeros 12 caracteres) de tus tokens activos; comparalos con el principio de tu token.

**11.40 Llamo al API desde un navegador.** El API está pensado para llamarse desde servidores: tu token de acceso no debe exponerse en un navegador. Las respuestas de preflight CORS no llevan credenciales, pero las llamadas reales exigen tu token (o la llave de registro en `register`).

**11.41 Tabla de diagnóstico rápido: "me pasa X".**

| Me pasa | Causa más probable | Qué hacer |
|---|---|---|
| `404` `No encontrado` | Método o ruta equivocados (por ejemplo `GET /systems/register` o `PUT /eventos`), o un prefijo de más en la URL base | Compará con el [Apéndice A](#apéndice-a-referencia-rápida-de-endpoints) |
| `400` `Body inválido` en `/systems/*` | JSON mal formado, falta `Content-Type: application/json`, campo desconocido (el esquema es estricto), `name` fuera de `^[a-z0-9_-]{3,50}$`, contraseña de menos de 12 caracteres, URL que no es `http`/`https` | Corregí el body con el [Apéndice B](#apéndice-b-esquemas-json); el motor no dice cuál campo falló |
| `400` `Configuración de suscripción inválida` | Falta `evento`/`eventos`, falta `es_activo` en `PATCH`, `id_empresa` no es un entero positivo, o mandaste `system`/`delivery` | Revisá el arreglo `errors` ([7.9](#79-errores-de-validación-400)) |
| `400` `Body inválido` en `/eventos` | JSON que ni siquiera se puede leer (el JSON válido pero con campos incorrectos da el otro `400`) | Corregí el JSON |
| `401` `Token inválido` | Token ausente, sin el prefijo `Bearer `, mal copiado, vencido (el viejo tras 24 h de la rotación), o anulado por un `recover` | Verificá el header con `GET /systems/me`; si lo perdiste, `POST /systems/token/recover` |
| `401` `Credenciales inválidas` en `recover` | Nombre o contraseña incorrectos, sistema sin contraseña o revocado (siempre el mismo mensaje) | Revisá los datos; sin la contraseña, pedí al administrador que te reemita las credenciales |
| `403` `Sistema revocado` | El administrador revocó tu sistema | Hablá con el equipo del motor |
| `409` `El nombre ya está en uso` | Otro sistema (o el tuyo, de un intento anterior) tiene ese nombre | Elegí otro; si es tuyo, `recover` |
| `409` `Ya hay dos tokens de acceso activos` | Rotaste hace menos de 24 h | Esperá el vencimiento del token viejo o usá `recover` |
| `413` `Body demasiado grande` | Body de más de 100 KB | Partí el pedido en varios |
| `422` con `motivo` `La URL de entrega no está permitida` | La URL no es `http(s)`, trae usuario/contraseña, apunta a loopback, a `169.254.x.x` o a una IP del propio motor, o no está en la lista de hosts permitidos | Usá una URL pública o privada permitida, no el mismo servidor del motor ([3.4](#34-qué-url-de-entrega-se-acepta)) |
| `422` con `motivo` `<origen> responded <código>` | Tu receptor no respondió `2xx` al ping (incluye `3xx`, `401` por token mal configurado, `404` por ruta mal escrita) | Probá el ping a mano ([12.2](#122-recetas-con-curl)) y corregí el receptor |
| `422` con `motivo` `Request to <origen> failed (...)` | Timeout (`AbortError`), conexión rechazada, TLS inválido o red inalcanzable (`TypeError`) | Verificá que el receptor es alcanzable desde el motor, con certificado válido, y responde en menos de 20 s |
| `429` `Demasiados intentos` | Más de 30 llamadas por hora (valor por defecto) a `register`/`recover` desde tu IP, con un contador compartido | Esperá a que se cumpla la hora; no reintentes en bucle (11.16) |
| `503` `No se pudo resolver el host de entrega, reintente más tarde` | El DNS del host de tu receptor no resuelve (o falla temporalmente) | Revisá el DNS y reintentá |
| `503` en otras rutas | Falla temporal del motor o de su base de datos | Reintentá con espera creciente (11.27) |
| `201` en el registro pero no llega ningún evento | No activaste suscripciones, el nombre del evento no coincide, o pasaron menos de 30 s desde el registro | Seguí la lista de 11.34 |
| Me llegan eventos repetidos | Reintentos por timeout o por una respuesta que no fue `2xx` | Deduplicá por `Idempotency-Key` (11.24) |
| Dejaron de llegar eventos que antes llegaban | Tu receptor devolvió un `4xx` permanente (el evento fue a la cola de fallidos), cambió tu token de entrega, o tu sistema fue revocado | Revisá los logs de tu receptor y `GET /systems/me`; pedí el reproceso con los datos de la [sección 14](#14-dónde-pedir-ayuda) |

`GET /health` y `GET /ready` existen en el motor, pero son para el monitoreo de quien lo opera: no los uses como prueba de tu integración ni para saber si tu sistema está bien registrado (no dicen nada de tu sistema).

---

## 12. Puesta en marcha y recetas de prueba

### 12.1 Checklist ordenada

1. [ ] Pedí al administrador la URL base del motor (puede cambiar durante la migración: configurala como variable, no la fijes en el código) y una llave de registro vigente (dura 24 horas por defecto: pedila cuando vayas a registrar).
2. [ ] Confirmá con el administrador si hay lista de hosts permitidos para las URL de entrega y qué tipos de evento emite el origen.
3. [ ] Generá un token de entrega aleatorio y largo (por ejemplo 32 bytes en base64url), sin la palabra `Bearer`.
4. [ ] Generá una contraseña de 12 a 128 caracteres.
5. [ ] Guardá ambos en tu gestor de secretos.
6. [ ] Tené tu receptor listo, desplegado o local con un túnel ([12.4](#124-desarrollar-con-un-túnel)): responde `2xx` al ping, valida `Authorization`, deduplica por `Idempotency-Key`, responde en menos de 20 s, no redirige.
7. [ ] Verificá que tu receptor es alcanzable desde fuera de tu red, con certificado válido.
8. [ ] Llamá a `POST /systems/register`.
9. [ ] Guardá el `access_token` de la respuesta (se muestra una sola vez).
10. [ ] Verificá con `GET /systems/me`.
11. [ ] Activá tus eventos con `POST /eventos` por cada `id_empresa`.
12. [ ] Verificá con `GET /eventos`.
13. [ ] Provocá el evento en el ERP y revisá en tu receptor (o en la interfaz de tu túnel) los headers y el body.
14. [ ] Agendá la rotación periódica del token de acceso y del token de entrega.
15. [ ] Documentá en tu equipo cómo usar `recover` en caso de filtración.

### 12.2 Recetas con curl

Definí las variables una vez:

```bash
export BASE='https://engine.example.com'
export TOKEN='cev_xxxxxxxx…'
```

Comprobar que llegás al motor (sin token responde `401` `Token inválido`):

```bash
curl -i "$BASE/eventos"
```

Registrar:

```bash
curl -i --request POST "$BASE/systems/register" \
  --header 'Content-Type: application/json' \
  --data '{
    "registration_key": "rk_xxxxxxxx…",
    "name": "mi-sistema",
    "password": "una-contraseña-larga-y-única",
    "delivery": { "url": "https://tu-sistema.example/hooks/eventos", "token": "tu-token-de-entrega" }
  }'
```

Ver tu sistema:

```bash
curl -i "$BASE/systems/me" --header "Authorization: Bearer $TOKEN"
```

Rotar el token de acceso:

```bash
curl -i --request POST "$BASE/systems/me/tokens" --header "Authorization: Bearer $TOKEN"
```

Recuperar el acceso:

```bash
curl -i --request POST "$BASE/systems/token/recover" \
  --header 'Content-Type: application/json' \
  --data '{ "name": "mi-sistema", "password": "una-contraseña-larga-y-única" }'
```

Cambiar la URL de entrega:

```bash
curl -i --request PATCH "$BASE/systems/me/delivery" \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{ "url": "https://tu-sistema.example/hooks/eventos-v2" }'
```

Cambiar el token de entrega:

```bash
curl -i --request PATCH "$BASE/systems/me/delivery" \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{ "token": "nuevo-token-de-entrega" }'
```

Activar un evento:

```bash
curl -i --request POST "$BASE/eventos" \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "evento": "payment_create" }'
```

Activar varios:

```bash
curl -i --request POST "$BASE/eventos" \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "eventos": ["payment_create", "invoice_create"] }'
```

Desactivar:

```bash
curl -i --request PATCH "$BASE/eventos/estado" \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{ "id_empresa": "123", "evento": "payment_create", "es_activo": false }'
```

Listar:

```bash
curl -i "$BASE/eventos" --header "Authorization: Bearer $TOKEN"
curl -i "$BASE/eventos?id_empresa=123" --header "Authorization: Bearer $TOKEN"
```

Simular el ping contra tu propio receptor, antes de registrarte:

```bash
curl -i --request POST 'https://tu-sistema.example/hooks/eventos' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer tu-token-de-entrega' \
  --header 'X-Delivery-Type: ping' \
  --header 'X-Contract-Version: 1' \
  --max-time 20 \
  --data '{}'
```

Tiene que responder `2xx` sin redirigir. Tené en cuenta que esta prueba sale desde tu máquina; el ping real sale desde el motor.

Simular una entrega contra tu receptor:

```bash
curl -i --request POST 'https://tu-sistema.example/hooks/eventos' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer tu-token-de-entrega' \
  --header 'Idempotency-Key: TEST:erp:123:1:mi-sistema' \
  --header 'X-Event-Id: 1' \
  --header 'X-Delivery-Attempt: 1' \
  --header 'X-Delivery-Type: event' \
  --header 'X-Contract-Version: 1' \
  --data '{"id_company":123,"id_branch":0,"event":{"type":"payment_create","data":{"ref":1}}}'
```

Repetí la misma petición: tu receptor tiene que responder `2xx` sin repetir el efecto.

### 12.3 Colección de Postman

En `docs/postman/` del repositorio del motor hay:

| Archivo | Para quién | Qué es |
|---|---|---|
| `cuenti-event-engine-integracion.postman_collection.json` | **Vos (dev de un sistema)** | Colección de integración: probar tu receptor, registrar el sistema, administrar tu cuenta y tus suscripciones. Solo usa tu token de acceso. |
| `event-engine-prod.postman_environment.json` | **Vos** | Environment con las variables de la colección de integración. |
| `cuenti-event-engine-fase2.postman_collection.json` y `event-engine-local.postman_environment.json` | Equipo del motor | Pruebas de QA y administración (llave de registro; necesitan la clave de administrador). **No las uses.** |

**Preparación (una vez):**

1. Importá la colección de integración y el environment `Event Engine · prod`, y seleccionalo.
2. Completá las variables del environment:

| Variable | Qué poner |
|---|---|
| `baseUrl` | La URL base del motor que te dio el administrador. Puede cambiar (por ejemplo de IP a dominio) cuando termine la migración: es la única variable que tocás en ese caso. |
| `registrationKey` | La llave de registro (`rk_…`) que te dio el administrador. Solo la usa `2.1`; vence a las 24 horas por defecto. |
| `systemName` | El nombre de tu sistema: minúsculas, números, `-` y `_`, de 3 a 50 caracteres. |
| `systemPassword` | De 12 a 128 caracteres. Guardala en tu gestor de secretos. |
| `deliveryUrl` | La URL **pública** de tu receptor o de tu túnel (ver [12.4](#124-desarrollar-con-un-túnel)). |
| `deliveryToken` | El token que tu receptor exige en `Authorization: Bearer`, sin la palabra `Bearer`. |
| `idEmpresa`, `evento`, `evento2` | La empresa y los tipos de evento que querés recibir. Los tipos válidos te los da el administrador. |

`accessToken` y `accessTokenPrevious` los llena la colección sola; no los toques.

**Orden de uso:**

1. **Carpeta 1** (antes de registrar): `1.1` y `1.2` envían un ping y un evento de ejemplo a tu receptor para comprobar que responde `2xx`, sin redirigir. Si fallan acá, el registro va a fallar.
2. **`2.1 Registrar mi sistema`**: el motor hace un ping a `deliveryUrl` y, si responde `2xx`, devuelve el `access_token` (una sola vez); la colección lo guarda en `accessToken`.
3. **`4.1 Activar un evento`** (o `4.2`): suscribe tu sistema a `evento` para `idEmpresa`.
4. Provocá el evento en el ERP y mirá la petición llegar a tu receptor o a la interfaz de tu túnel.

Después, las carpetas **3** (tu sistema: ver, cambiar URL o token de entrega, rotar el token de acceso) y **4** (listar, desactivar, reactivar) son las operaciones del día a día. Si perdés el token de acceso, `2.2 Recuperar acceso`.

**Qué tener en cuenta:**

- Si falta una variable, la petición no sale y Postman muestra `Completá en el environment: …`.
- `2.1` y `2.2` cuentan contra el límite de 30 llamadas por hora y por IP ([10.1](#101-límite-de-intentos)).
- Los sistemas no se pueden borrar por la API: elegí un `systemName` que quieras conservar. Si querés probar con uno descartable, usá un nombre claro (por ejemplo `mi-sistema-prueba`) y pedile al administrador que lo dé de baja.
- `2.1` necesita la variable `registrationKey` (la llave que te da el administrador, [3.1](#31-llave-de-registro)); si venció, pedí una nueva.
- Los tests de la colección verifican los códigos de respuesta; si uno falla, abrí la consola de Postman (`View → Show Postman Console`) para ver el cuerpo de la respuesta.

### 12.4 Desarrollar con un túnel

No necesitás desplegar nada ni levantar el motor: el motor ya está en un servidor y le entrega los eventos a una URL pública. Para desarrollar en tu máquina, exponé tu receptor local con un túnel (por ejemplo ngrok, Cloudflare Tunnel u otro que te dé una URL pública con HTTPS).

**Flujo completo:**

1. Levantá tu receptor local (el de [8.11](#811-receptor-de-referencia) sirve de base) en un puerto, por ejemplo `3000`.
2. Abrí el túnel hacia ese puerto y copiá la URL pública `https://…` que te da.
3. Probalo desde afuera con la carpeta 1 de Postman o con la receta de ping de [12.2](#122-recetas-con-curl), usando la URL del túnel como `deliveryUrl`.
4. Registrá tu sistema (`2.1`) con esa URL y activá tus eventos (`4.1`).
5. En el ERP, ejecutá la acción que genera el evento, para la empresa (`idEmpresa`) que activaste. Los tipos de evento y la acción del ERP que los dispara te los confirma el administrador del motor.
6. Mirá la entrega llegar: en la consola de tu receptor y en la interfaz de inspección de tu túnel (muestra headers y body de cada petición y permite reenviarla).

**Qué cuidar con túneles:**

- **El túnel tiene que estar encendido al registrar y al cambiar la URL**: el motor hace un ping antes de guardar. Si no responde `2xx`, recibís `422`.
- **Si el túnel cambia de URL al reiniciarlo** (es lo habitual en los planes gratuitos), ejecutá `PATCH /systems/me/delivery` (`3.2` en Postman) con la nueva. No hace falta volver a registrarte.
- **Si el túnel está apagado cuando llega un evento**, el proveedor suele responder con un error propio. Un `4xx` distinto de `401`, `403`, `408` y `429` (por ejemplo `404`) es un fallo **permanente** para el motor: el evento no se reintenta y va a la cola de fallidos. Un `5xx` sí se reintenta. Mantené el túnel encendido mientras esperás el evento y confirmá con el administrador cómo reenviar un evento fallido.
- **Sin redirecciones**: usá la URL `https://` final del túnel. Un `3xx` (por ejemplo de `http` a `https`) cuenta como fallo permanente.
- **Respondé rápido**: el motor espera hasta 20 s por defecto, contando la latencia del túnel. Respondé `2xx` y procesá en segundo plano.
- **Deduplicá por `Idempotency-Key`**: una entrega puede llegar más de una vez.
- **El token de entrega** que pongas en `deliveryToken` es el que tu receptor local tiene que validar; no lo reutilices en producción.
- **Datos reales**: lo que llega es información del ERP. No dejes la URL del túnel ni sus logs a la vista de terceros.

---

## 13. Apéndices

### Apéndice A. Referencia rápida de endpoints

| Método | Ruta | Auth | Body | Éxito | Errores propios |
|---|---|---|---|---|---|
| `POST` | `/systems/register` | Llave de registro | `{registration_key, name, password, delivery:{url, token}}` | `201 {name, access_token}` | `400`, `401`, `409`, `413`, `422`, `429`, `503` |
| `POST` | `/systems/token/recover` | Nombre + contraseña | `{name, password}` | `200 {access_token}` | `400`, `401`, `429`, `503` |
| `GET` | `/systems/me` | Bearer | | `200 {name, status, delivery:{origin}, tokens:[…]}` | `503` |
| `POST` | `/systems/me/tokens` | Bearer | | `201 {access_token}` | `401`, `409`, `503` |
| `PATCH` | `/systems/me/delivery` | Bearer | `{url?, token?}` (al menos uno) | `204` | `400`, `422`, `503` |
| `POST` | `/eventos` | Bearer | `{id_empresa, evento?, eventos?, es_activo?}` | `200` o `207` | `400`, `404`, `503` |
| `PATCH` | `/eventos/estado` | Bearer | `{id_empresa, evento?, eventos?, es_activo}` | `200` o `207` | `400`, `404`, `503` |
| `GET` | `/eventos?id_empresa=` | Bearer | | `200 {suscripciones:[…]}` | `400`, `503` |

Comunes a todas: `400`/`413` por JSON, `404` ruta, `500`. Comunes a las de Bearer: `401`, `403` revocado, `503` autenticación.

Todos los bodies de petición y respuesta son JSON (`Content-Type: application/json`). Las respuestas de error siempre llevan `message`.

### Apéndice B. Esquemas JSON

Los esquemas siguen JSON Schema (draft 2020-12). Son descriptivos: la autoridad es el comportamiento documentado en las secciones anteriores.

**`POST /systems/register`, request:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["registration_key", "name", "password", "delivery"],
  "properties": {
    "registration_key": { "type": "string", "minLength": 1, "maxLength": 256 },
    "name": { "type": "string", "pattern": "^[a-z0-9_-]{3,50}$" },
    "password": { "type": "string", "minLength": 12, "maxLength": 128 },
    "delivery": {
      "type": "object",
      "additionalProperties": false,
      "required": ["url", "token"],
      "properties": {
        "url": { "type": "string", "maxLength": 2048, "format": "uri", "pattern": "^[hH][tT][tT][pP][sS]?:" },
        "token": { "type": "string", "minLength": 1, "maxLength": 4096 }
      }
    }
  }
}
```

**`POST /systems/register`, response `201`:**

```json
{
  "type": "object",
  "required": ["name", "access_token"],
  "properties": {
    "name": { "type": "string" },
    "access_token": { "type": "string", "pattern": "^cev_[A-Za-z0-9_-]{43}$" }
  }
}
```

**`POST /systems/token/recover`, request:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["name", "password"],
  "properties": {
    "name": { "type": "string", "maxLength": 100 },
    "password": { "type": "string", "maxLength": 128 }
  }
}
```

**`POST /systems/token/recover` (`200`) y `POST /systems/me/tokens` (`201`), response:**

```json
{
  "type": "object",
  "required": ["access_token"],
  "properties": {
    "access_token": { "type": "string", "pattern": "^cev_[A-Za-z0-9_-]{43}$" }
  }
}
```

**`GET /systems/me`, response `200`:**

```json
{
  "type": "object",
  "required": ["name", "status", "delivery", "tokens"],
  "properties": {
    "name": { "type": "string" },
    "status": { "type": "string", "enum": ["active", "revoked"] },
    "delivery": {
      "type": "object",
      "required": ["origin"],
      "properties": { "origin": { "type": "string" } }
    },
    "tokens": {
      "type": "array",
      "maxItems": 2,
      "items": {
        "type": "object",
        "required": ["prefix", "createdAt", "expiresAt"],
        "properties": {
          "prefix": { "type": "string", "minLength": 12, "maxLength": 12 },
          "createdAt": { "type": "string", "format": "date-time" },
          "expiresAt": { "type": ["string", "null"], "format": "date-time" }
        }
      }
    }
  }
}
```

**`PATCH /systems/me/delivery`, request:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "minProperties": 1,
  "properties": {
    "url": { "type": "string", "maxLength": 2048, "format": "uri", "pattern": "^[hH][tT][tT][pP][sS]?:" },
    "token": { "type": "string", "minLength": 1, "maxLength": 4096 }
  }
}
```

**`POST /eventos` y `PATCH /eventos/estado`, request** (en `PATCH`, `es_activo` es obligatorio):

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["id_empresa"],
  "anyOf": [{ "required": ["evento"] }, { "required": ["eventos"] }],
  "properties": {
    "id_empresa": {
      "oneOf": [
        { "type": "integer", "minimum": 1, "maximum": 9007199254740991 },
        { "type": "string", "pattern": "^\\s*[1-9][0-9]*\\s*$" }
      ]
    },
    "evento": { "type": "string", "pattern": "\\S" },
    "eventos": { "type": "array", "minItems": 1, "items": { "type": "string", "pattern": "\\S" } },
    "es_activo": { "type": "boolean", "default": true }
  }
}
```

**Registro de suscripción** (respuesta de un evento y elementos de `suscripciones`):

```json
{
  "type": "object",
  "required": ["id_empresa", "evento", "system", "es_activo", "created_at", "updated_at"],
  "properties": {
    "id_empresa": { "type": "string", "pattern": "^[1-9][0-9]*$" },
    "evento": { "type": "string" },
    "system": { "type": "string" },
    "es_activo": { "type": "boolean" },
    "created_at": { "type": "string", "format": "date-time" },
    "updated_at": { "type": "string", "format": "date-time" }
  }
}
```

**Resultado por evento fallido** (`404` o `503`, solo o dentro de `resultados`):

```json
{
  "type": "object",
  "required": ["id_empresa", "evento", "system", "message"],
  "properties": {
    "id_empresa": { "type": "string" },
    "evento": { "type": "string" },
    "system": { "type": "string" },
    "message": {
      "type": "string",
      "enum": [
        "La suscripción no existe",
        "No se pudo confirmar la configuración; puede reintentar la solicitud"
      ]
    }
  }
}
```

**Respuesta de varios eventos (`200` o `207`):**

```json
{
  "type": "object",
  "required": ["resultados"],
  "properties": {
    "resultados": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["status", "body"],
        "properties": {
          "status": { "type": "integer", "enum": [200, 404, 503] },
          "body": { "type": "object" }
        }
      }
    }
  }
}
```

**Body de una entrega (lo que recibe tu receptor):**

```json
{
  "type": "object",
  "required": ["id_company", "event"],
  "additionalProperties": true,
  "properties": {
    "id_company": { "type": "integer", "minimum": 1 },
    "id_branch": {},
    "event": {
      "type": "object",
      "required": ["type", "data"],
      "additionalProperties": true,
      "properties": {
        "type": { "type": "string", "minLength": 1 },
        "data": { "type": "object" }
      }
    }
  }
}
```

**Respuestas de error generales** (todas las de error salvo los resultados por evento `404`/`503` de `/eventos`):

```json
{
  "type": "object",
  "required": ["message"],
  "properties": {
    "message": { "type": "string" },
    "motivo": { "type": "string" },
    "errors": { "type": "array" }
  }
}
```

`motivo` solo aparece en los `422` y `errors` solo en el `400` de `/eventos`.

### Apéndice C. Historial y compatibilidad

**Contrato v1.** Es la única versión del contrato de entregas. Todas las entregas y pings llevan `X-Contract-Version: 1`. Si en el futuro cambia el contrato, el valor de ese header lo va a indicar.

**Si venís de la versión anterior del motor:**

| Antes | Ahora |
|---|---|
| Los sistemas los cargaba el equipo del motor | Cada sistema se registra solo con `POST /systems/register`. Los sistemas existentes recibieron credenciales del administrador. |
| `/eventos` se autorizaba por IP y el body llevaba `system` | `/eventos` exige `Authorization: Bearer cev_…` y un body con `system` responde `400` |
| Rutas `/failsafe/*` con el header `X-Failsafe-Key` | Retiradas: responden `404` . `failsafe` y `failsafe_sesion` son sistemas comunes que usan `/eventos` con su propio token. |
| `GET /failsafe/suscripciones` | `GET /eventos` |
| Lista de IPs en variables de entorno del motor | Ya no hay filtro por IP: para registrarte necesitás una llave de registro que crea el administrador ([3.1](#31-llave-de-registro)) |
| Suscripción con metadatos `delivery` | Sin `delivery`: un body con `delivery` responde `400`. Todos los sistemas reciben el mismo contrato. |

---

## 14. Dónde pedir ayuda

Escribile al administrador o al equipo del motor por el canal que te hayan indicado.

**Incluí siempre:**

| Dato | Ejemplo |
|---|---|
| Nombre de tu sistema | `mi-sistema` |
| Entorno | producción |
| Método y ruta | `POST /eventos` |
| Código de estado | `503` |
| Body completo de la respuesta | `{"message":"Autenticación no disponible"}` |
| Fecha y hora con zona horaria | `2026-10-26 15:04:05 UTC` |
| Body que enviaste, sin secretos | `{"id_empresa":"123","evento":"payment_create"}` |
| Para problemas de entrega | `X-Event-Id`, `Idempotency-Key`, `X-Delivery-Attempt` y la respuesta que dio tu receptor |
| Prefijo del token, si es un problema de autenticación | `cev_xxxxxxxx` (los 12 caracteres que muestra `GET /systems/me`) |

**Nunca incluyas:** tu token de acceso completo, tu token de entrega, tu contraseña ni headers `Authorization` sin censurar. Si los compartiste por error, rotalos (`recover`, `PATCH /systems/me/delivery`) y avisá.
