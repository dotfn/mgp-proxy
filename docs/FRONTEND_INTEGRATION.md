# Integración con el frontend (Next.js)

Guía para el equipo de frontend: cómo hablarle a `mgp-proxy`, qué esperar de cada
respuesta, y qué patrones de UI/UX conviene usar dado cómo se comporta el proxy por
dentro. No es documentación de referencia de la API de MGP en sí — es específicamente
"qué necesita saber el frontend para no pelearse con este proxy en particular".

## Por qué el proxy se comporta así (el resumen que importa para UI)

`mgp-proxy` no le pega a la Municipalidad en tiempo real en cada request. En el medio
hay: caché en memoria, un rate limiter hacia MGP, un circuit breaker, y — la parte que
más afecta a la UX — un **bridge con navegador headless** que resuelve un challenge de
Cloudflare y por el que pasan todas las requests que no están cacheadas.

Consecuencia directa para el frontend: **la latencia no es uniforme**. La mayoría de las
requests son cache HIT (rápidas, <50ms), pero una request que cae en cache MISS puede
tardar de <1s a varios segundos, y en el peor caso (justo después de que el proxy
arranca o renueva su sesión con Cloudflare) hasta ~10-25s. La UI tiene que tolerar eso
sin sentirse rota.

## Dos formas de pegarle

### `GET /mgp/:accion` (recomendado para el front)

```
GET /mgp/RecuperarProximosArribosW?identificadorParada=P3608&codigoLineaParada=93
```

Los query params son los mismos parámetros de la acción. Trae `Cache-Control` seteado
(útil si en algún punto se pone un CDN/edge cache delante), y `Access-Control-Allow-Origin: *`
en la respuesta.

### `POST /` (form-urlencoded)

```
POST /
Content-Type: application/x-www-form-urlencoded

accion=RecuperarProximosArribosW&identificadorParada=P3608&codigoLineaParada=93
```

Mismo resultado, distinto formato de request. Usar éste sólo si ya tenés ese código de
antes (viejo cliente V670); para código nuevo, `GET /mgp/:accion` es más simple de
cachear en el browser y de debuggear pegándole desde la barra de direcciones.

### Acciones disponibles

| Acción | Params | Frecuencia de cambio |
|---|---|---|
| `RecuperarLineaPorCuandoLlega` | — | Semi-estática (24h) |
| `RecuperarCallesPrincipalPorLinea` | `codLinea` | Semi-estática (24h) |
| `RecuperarInterseccionPorLineaYCalle` | `codLinea`, `codCalle` | Semi-estática (24h) |
| `RecuperarParadasConBanderaPorLineaCalleEInterseccion` | `codLinea`, `codCalle`, `codInterseccion` | Semi-estática (24h) |
| `RecuperarParadasConBanderaYDestinoPorLinea` | `codLinea`, `isSublinea` | Semi-estática (24h) |
| `RecuperarRecorridoParaMapaAbrevYAmpliPorEntidadYLinea` | `codLinea`, `isSublinea` | Semi-estática (24h) |
| `RecuperarBanderasAsociadasAParada` | `identificadorParada` | Semi-estática (24h) |
| `RecuperarProximosArribosW` | `identificadorParada`, `codigoLineaParada` | Dinámica (30s) |

**Ojo con `identificadorParada`**: es el campo `Identificador` de la parada (formato
`P3608`), **no** el `Codigo` numérico. Con el código numérico el WS responde
`CodigoEstado: -1` con `"Parada inexistente"` — no es un error del proxy, es que se
mandó el parámetro equivocado.

## Cómo leer la respuesta

### El body SIEMPRE es JSON, pero un HTTP 200 no significa "hay datos"

El WS de MGP (y por lo tanto el proxy) puede devolver `200 OK` con un error de negocio
adentro:

```json
{ "CodigoEstado": -1, "MensajeEstado": "La parada no corresponde a la linea" }
```

**Regla para el frontend: siempre revisar `CodigoEstado` antes de asumir que hay datos
útiles.** `0` = ok, cualquier otra cosa = error de negocio (parada/línea inválida,
inexistente, etc.) — mostrar `MensajeEstado` o un mensaje propio, no reintentar
automáticamente (reintentar no lo va a arreglar, el parámetro está mal).

Esto es distinto de un error HTTP (4xx/5xx), que sí es reintentable.

### Headers que importan

| Header | Valores | Qué decir en la UI |
|---|---|---|
| `X-Cache` | `HIT` / `MISS` / `STALE` | `STALE` = "esto es dato viejo, MGP no está respondiendo ahora". Vale la pena mostrar algo tipo *"última actualización hace un rato"* en vez de tratarlo como dato en tiempo real. |
| `X-Stale-Reason` | string libre, sólo si `X-Cache: STALE` | Motivo interno (para debug/soporte, no para mostrarle al usuario final tal cual) |
| `Cache-Control` | sólo en `GET /mgp/:accion` | `s-maxage`/`max-age` distintos según si la acción es semi-estática o dinámica |

### Errores HTTP y qué significan

| Status | Body (`error`) | Significa | Qué hacer en la UI |
|---|---|---|---|
| `400` | `empty_body` | Falta el body en el POST | Bug del cliente, no del usuario — no debería pasar con `GET /mgp/:accion` |
| `415` | `unsupported_content_type` | Content-Type mal puesto en el POST | Bug del cliente |
| `502` | `mgp_unavailable`, con `message` | Ver tabla de abajo — la request no se pudo resolver contra MGP | Depende del `message`, ver siguiente sección |

### Desglosando el 502 (`mgp_unavailable`)

El campo `message` de un 502 no es texto para mostrarle al usuario — es para decidir
**cómo** reintentar. Los prefijos que importan:

| `message` empieza con... | Qué pasó | Reintento sugerido |
|---|---|---|
| `circuit_open` | El circuit breaker está cortado: MGP viene fallando sostenido y el proxy dejó de pegarle por un rato (hasta 5 min, con backoff creciente) | **No reintentar en loop.** Mostrar "el servicio de arribos no está disponible, reintentando en unos minutos" y backear bastante (30-60s+) antes de reintentar |
| `bridge_busy` | El bridge interno está saturado (mucha cola en simultáneo). Es momentáneo, no indica que MGP esté caído | Reintentar rápido, en 1-3s — es la señal más "benigna" de las tres |
| cualquier otra cosa (`webWS.php devolvió 429`, `bridge_timeout`, etc.) | Falla puntual contra MGP o el bridge | Reintento con backoff normal (ver más abajo) |

En la práctica, lo más simple y suficientemente bueno es: **un único esquema de retry
con backoff exponencial para cualquier 502**, sin parsear el `message` — la tabla de
arriba es para cuando se quiera afinar la UX más (por ejemplo, no mostrar "reintentando"
agresivo si es `circuit_open`).

## Patrones de UI/UX recomendados

### 1. Loading state que tolere varios segundos

No asumir que la respuesta llega en <1s. En particular, la **primera request después de
que el proxy arrancó o renovó su sesión con Cloudflare** (pasa cada ~9 minutos, dura
~10s) puede tardar más de lo normal. Un spinner simple está bien, pero si tarda más de
~3-4s conviene un mensaje tipo *"buscando arribos..."* en vez de dejar el spinner solo
(la sensación de "esto está colgado" aparece rápido en apps de tiempo real).

### 2. No hacer polling más rápido que el TTL

`RecuperarProximosArribosW` cachea 30s (`fresh`). Pollear cada 5-10s no trae datos más
nuevos, sólo le pega más seguido a un endpoint que va a responder lo mismo desde caché.
**Recomendado: polling cada 20-30s** para pantallas de "próximo arribo" en vivo.

### 3. Retry con backoff exponencial, con techo

```
intento 1 → falla → esperar 2s
intento 2 → falla → esperar 4s
intento 3 → falla → esperar 8s
...techo en ~30s, no seguir subiendo
```

No reintentar en loop tight (sin espera) — contra un `circuit_open` eso no hace nada
más que quemar batería/requests hasta que el breaker cierre solo.

### 4. Distinguir "sin datos" de "dato viejo" de "error"

Con `CodigoEstado` + `X-Cache` + status HTTP alcanza para tres estados de UI distintos:

- **Dato fresco**: `200` + `CodigoEstado: 0` + `X-Cache: HIT` o `MISS` → mostrar normal.
- **Dato viejo pero hay algo**: `200` + `CodigoEstado: 0` + `X-Cache: STALE` → mostrar
  el dato con alguna indicación sutil de que puede no ser exacto ahora mismo.
- **Sin dato**: `502`, o `200` con `CodigoEstado != 0` → mensaje de error o vacío,
  según si es un error de negocio (parada mal puesta) o de disponibilidad (502).

### 5. Identificar al cliente (opcional, ayuda al análisis de uso)

El proxy acepta un header `x-client-id` (o `x-device-id`) para trackear analytics de
producto sin depender de IP/user-agent solamente. Si el front ya genera algún ID de
dispositivo/sesión, mandarlo no cuesta nada y mejora las métricas del lado del proxy.

## Ejemplo: wrapper de fetch en TypeScript

```typescript
const PROXY_BASE_URL = process.env.NEXT_PUBLIC_MGP_PROXY_URL!;

type MgpResponse<T> = { CodigoEstado: number; MensajeEstado?: string } & T;

class MgpBusinessError extends Error {
  constructor(public codigoEstado: number, mensaje: string) {
    super(mensaje);
  }
}

class MgpUnavailableError extends Error {
  constructor(message: string, public retriable: "fast" | "slow" | "normal") {
    super(message);
  }
}

async function fetchMgp<T>(
  accion: string,
  params: Record<string, string>,
): Promise<MgpResponse<T>> {
  const qs = new URLSearchParams(params).toString();
  const url = `${PROXY_BASE_URL}/mgp/${accion}?${qs}`;

  const res = await fetch(url, {
    headers: { "x-client-id": getOrCreateClientId() },
  });

  if (res.status === 502) {
    const body = await res.json().catch(() => ({ message: "" }));
    const message: string = body.message ?? "";
    const retriable = message.startsWith("circuit_open")
      ? "slow"
      : message.startsWith("bridge_busy")
        ? "fast"
        : "normal";
    throw new MgpUnavailableError(message, retriable);
  }
  if (!res.ok) {
    throw new Error(`mgp-proxy respondió ${res.status}`);
  }

  const data = (await res.json()) as MgpResponse<T>;
  if (data.CodigoEstado !== 0) {
    throw new MgpBusinessError(data.CodigoEstado, data.MensajeEstado ?? "Error desconocido");
  }
  return data;
}

async function fetchMgpWithRetry<T>(
  accion: string,
  params: Record<string, string>,
  maxAttempts = 4,
): Promise<MgpResponse<T>> {
  let attempt = 0;
  while (true) {
    try {
      return await fetchMgp<T>(accion, params);
    } catch (e) {
      attempt++;
      // Errores de negocio (parada/línea mal puesta): no tiene sentido reintentar.
      if (e instanceof MgpBusinessError) throw e;
      if (attempt >= maxAttempts) throw e;

      const base = e instanceof MgpUnavailableError && e.retriable === "fast" ? 1000 : 2000;
      const delay = Math.min(base * 2 ** (attempt - 1), 30_000);
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}
```

Uso:

```typescript
try {
  const { arribos } = await fetchMgpWithRetry<{ arribos: Arribo[] }>(
    "RecuperarProximosArribosW",
    { identificadorParada: "P3608", codigoLineaParada: "93" },
  );
  // mostrar arribos
} catch (e) {
  if (e instanceof MgpBusinessError) {
    // parada/línea inválida — mensaje de error de datos, no de "servicio caído"
  } else {
    // servicio no disponible tras reintentar — mensaje de "intentá más tarde"
  }
}
```

## CORS

El proxy sólo habilita CORS para los orígenes listados en `ALLOWED_ORIGINS` (env var del
lado del proxy, separada por comas). Si el front corre en un dominio nuevo (preview de
Vercel, dominio custom, etc.) y las requests fallan silenciosamente en el browser con un
error de CORS en la consola, es porque ese origen no está en la lista — avisar al equipo
que mantiene el proxy para agregarlo, no es algo que se arregle del lado del front.

## Para debug rápido sin pedirle nada al equipo de backend

- `GET /stats` — dashboard visual: requests, hit rate de caché, estado del circuit
  breaker, estado del bridge.
- `GET /stats/data` — el mismo estado en JSON, útil para ver por ejemplo
  `bridge.ready`/`bridge.queueDepth` o `queue.breakerState` sin abrir el dashboard.

Si algo anda raro (todo tarda mucho, muchos 502), mirar `/stats/data` antes de asumir
que es un bug del front — casi siempre la causa está ahí (`breakerState: "open"`,
`bridge.ready: false`, etc.).
