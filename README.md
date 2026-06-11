# 🍕 La Vera Pizza Insights

Sistema de análisis de inteligencia comercial basado en IA para negocios de delivery. Procesa exportaciones de WhatsApp y genera un dashboard interactivo con métricas de ventas, perfiles de clientes, patrones de comportamiento y recomendaciones accionables.

**Demo en vivo:** [jhoeliel.github.io/pizza-insights](https://jhoeliel.github.io/pizza-insights)

---

## Índice

- [Arquitectura general](#arquitectura-general)
- [Stack tecnológico](#stack-tecnológico)
- [Flujo de datos](#flujo-de-datos)
- [Estructura del análisis IA](#estructura-del-análisis-ia)
- [Proxy Cloudflare Workers](#proxy-cloudflare-workers)
- [Indicadores generados](#indicadores-generados)
- [Cómo replicarlo](#cómo-replicarlo)
- [Variables y configuración](#variables-y-configuración)
- [Limitaciones conocidas](#limitaciones-conocidas)

---

## Arquitectura general

```
┌─────────────────────────────────────────────────────────────┐
│                     USUARIO (Navegador)                     │
│                                                             │
│  1. Sube archivos .txt (exports de WhatsApp)                │
│  2. App parsea mensajes localmente (JS puro)                │
│  3. Construye 4 prompts con muestras representativas        │
│  4. Envía a proxy Cloudflare → Anthropic Claude API         │
│  5. Renderiza dashboard con resultados                      │
└─────────────────────────┬───────────────────────────────────┘
                          │  HTTPS POST (JSON)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Cloudflare Worker (CORS Proxy)                 │
│                                                             │
│  claude-proxy.jhoelielpalma75.workers.dev                   │
│                                                             │
│  - Recibe requests desde GitHub Pages                       │
│  - Agrega headers CORS                                      │
│  - Reenvía a api.anthropic.com/v1/messages                  │
│  - Devuelve respuesta al navegador                          │
└─────────────────────────┬───────────────────────────────────┘
                          │  HTTPS POST (Anthropic API)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  Anthropic Claude API                       │
│                                                             │
│  Modelo: claude-sonnet-4-5                                  │
│  Max tokens output: 4000 por llamada                        │
│  Temperatura: 0.2 (respuestas consistentes)                 │
│  4 llamadas secuenciales con pausa de 5s entre cada una     │
└─────────────────────────────────────────────────────────────┘
```

---

## Stack tecnológico

| Capa | Tecnología | Justificación |
|---|---|---|
| Frontend | HTML + CSS + Vanilla JS | Sin dependencias, carga instantánea, compatible con GitHub Pages |
| Hosting | GitHub Pages | Gratis, deployment por commit, HTTPS automático |
| CORS Proxy | Cloudflare Workers | Gratis (100k req/día), latencia mínima, sin servidor propio |
| IA | Anthropic Claude Sonnet 4.5 | Balance óptimo entre costo y calidad de análisis |
| Gráficos | Canvas API nativo | Sin librerías externas, pie chart dibujado con arc() |
| Almacenamiento | localStorage del navegador | API key persiste entre sesiones sin backend |
| Exportación | window.print() + CSS @media print | PDF del reporte sin librerías |

**Por qué no React/Vue/Angular:** La app es un single-file HTML autocontenido. Esto facilita el deployment en GitHub Pages, elimina el paso de build, y permite que cualquier persona replique el proyecto con solo copiar un archivo.

**Por qué no almacenamiento en servidor:** Los chats de WhatsApp contienen datos privados de clientes. Todo el procesamiento ocurre en el navegador del usuario. Los mensajes viajan a la API de Claude únicamente para el análisis y no son persistidos.

---

## Flujo de datos

### 1. Ingesta de archivos

```javascript
// El usuario sube archivos .txt exportados de WhatsApp
// FileReader lee cada archivo en UTF-8
reader.readAsText(file, 'UTF-8');

// Cada archivo se almacena en memoria como:
{ name: "Chat de WhatsApp con +51 999 xxx xxx.txt", text: "contenido..." }
```

### 2. Parseo de mensajes

El parser usa regex para extraer mensajes del formato estándar de WhatsApp:

```
DD/MM/YYYY, HH:MM - Nombre/Número: Mensaje
```

```javascript
var m = line.match(
  /^(\d{1,2}\/\d{1,2}\/\d{2,4}),\s*(\d{1,2}:\d{2})(?::\d{2})?\s*[-–]\s*([^:]+):\s*(.+)$/
);
// Extrae: fecha, hora, remitente, mensaje
```

**Sanitización antes de enviar a la IA:**
```javascript
function cleanMsg(m) {
  var txt = m.msg
    .replace(/[\u0000-\u001F\u007F]/g, ' ')  // elimina control chars
    .replace(/\\/g, '')                         // elimina backslashes
    .substring(0, 120);                         // trunca a 120 chars
  return '[' + m.time + '] ' + m.sender.substring(0, 15) + ': ' + txt;
}
```

### 3. Construcción de muestras representativas

Los mensajes se dividen en 3 bloques tomando muestras distribuidas uniformemente:

```javascript
function sampleMsgs(arr, n) {
  if (arr.length <= n) return arr.map(cleanMsg);
  var result = [], step = arr.length / n;
  for (var i = 0; i < n; i++) {
    result.push(cleanMsg(arr[Math.floor(i * step)]));
  }
  return result;
}
// 45 mensajes por bloque = 135 mensajes representativos en total
// Distribuidos uniformemente para cubrir toda la línea de tiempo
```

### 4. Llamadas a la API

```javascript
async function claude(key, prompt, label) {
  var res = await fetch('https://claude-proxy.jhoelielpalma75.workers.dev', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': key,
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-5',
      max_tokens: 4000,
      messages: [{ role: 'user', content: prompt }]
    })
  });
  // Extrae JSON de la respuesta
  var raw = data.content.map(b => b.text || '').join('');
  var clean = raw.replace(/```json/g,'').replace(/```/g,'').trim();
  var s = clean.indexOf('{'), e = clean.lastIndexOf('}');
  return JSON.parse(clean.substring(s, e+1));
}
```

---

## Estructura del análisis IA

El análisis se divide en **4 llamadas secuenciales** con 5 segundos de pausa entre cada una para respetar el rate limit de 30,000 tokens/min del tier básico de Anthropic.

### Llamada 1 — Ventas y perfiles
**Bloques usados:** S1 + S2 (primer y segundo tercio de mensajes)

**Extrae:**
- Clasificación de conversaciones: cerradas / no cerradas / curiosos
- Abandono post-precio (cuántas se cortan tras ver el precio)
- Zonas geográficas mencionadas
- Recurrencia (clientes nuevos vs recurrentes)
- Objeciones que impidieron cierres
- Perfiles de clientes

**JSON esperado:**
```json
{
  "ventas": {
    "cerradas": 15,
    "no_cerradas": 30,
    "curiosos": 26,
    "total_conversaciones": 71,
    "tasa_conversion": "21%",
    "ticket_promedio": "S/55",
    "abandono_post_precio": 8,
    "abandono_post_precio_pct": "11%"
  },
  "zonas": [{ "zona": "Bellamar", "menciones": 12, "porcentaje": 17 }],
  "recurrencia": {
    "clientes_recurrentes": 10,
    "clientes_nuevos": 61,
    "pct_recurrentes": "14%",
    "pct_nuevos": "86%"
  },
  "objeciones_cierre": [{ "texto": "muy lejos el delivery", "frecuencia": "Alta", "impacto": "Alto" }],
  "perfiles_clientes": [{ "nombre": "Comprador nocturno", "icono": "🌙", "descripcion": "Pide entre 7-11pm", "porcentaje": 65, "tipo": "top" }]
}
```

### Llamada 2 — Productos y operaciones
**Bloques usados:** S2 + S3 (segundo y tercer tercio)

**Extrae:** Pizzas individuales, combos, tamaños, métodos de pago, competencia mencionada, horarios pico, sentimiento

### Llamada 3 — Estadísticas y frases
**Bloques usados:** S1 + S3 (primer y tercer tercio — máxima dispersión temporal)

**Extrae:** Nivel de satisfacción, tiempo de respuesta percibido, tiempo promedio de cierre, fuente de recomendación orgánica (TikTok/referidos/etc), conversaciones sin respuesta, frases textuales reales, objeciones generales

### Llamada 4 — Recomendaciones
**Input:** Resumen sintetizado de los resultados de llamadas 1-3 (no envía mensajes raw)

**Extrae:** Recomendaciones accionables priorizadas + indicadores adicionales sugeridos por la IA

**Ventaja de este diseño:** Cada llamada recibe bloques distintos del historial, maximizando la cobertura temporal sin exceder límites de tokens.

---

## Proxy Cloudflare Workers

### Por qué es necesario

GitHub Pages sirve archivos desde `github.io`. Los navegadores modernos bloquean llamadas a dominios externos desde páginas estáticas sin los headers CORS correctos. La API de Anthropic no incluye `Access-Control-Allow-Origin: *` en sus respuestas porque está diseñada para llamadas servidor-a-servidor.

### Código del worker

```javascript
export default {
  async fetch(request, env) {
    // Manejo de preflight CORS
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type, x-api-key',
        }
      });
    }

    const body = await request.json();
    const apiKey = request.headers.get('x-api-key');

    // Reenvío a Anthropic
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': apiKey,
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify(body)
    });

    const data = await response.json();

    return new Response(JSON.stringify(data), {
      status: response.status,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',   // ← esto resuelve el CORS
      }
    });
  }
};
```

### Seguridad

La API key viaja en el header `x-api-key` sobre HTTPS. El worker no almacena, loguea ni procesa las keys — solo las reenvía. Para mayor seguridad en producción se recomienda:
- Restringir el worker a tu dominio específico (reemplazar `*` por `https://jhoeliel.github.io`)
- Rotar la API key periódicamente desde `console.anthropic.com/settings/keys`

---

## Indicadores generados

### Resumen superior (12 métricas)
| Métrica | Descripción |
|---|---|
| Total mensajes | Suma de todos los mensajes parseados |
| Conversaciones | Número de chats únicos analizados |
| Ventas cerradas | Conversaciones que terminaron en pedido confirmado |
| No cerradas | Preguntaron pero no compraron |
| Curiosos | Solo consultas informativas sin intención de compra |
| Conversión | % de conversaciones que terminaron en venta |
| Ticket promedio | Estimado del monto promedio por pedido |
| Abandono x precio | Conversaciones cortadas tras ver el precio |
| Sin respuesta | Mensajes del cliente sin respuesta del negocio |
| Hora pico | Hora con mayor volumen de mensajes (calculado localmente) |
| Satisfacción | Nivel Alto/Medio/Bajo según tono de las conversaciones |
| Tiempo cierre | Estimado de mensajes/tiempo hasta completar una venta |

### Secciones del dashboard (16 secciones)
1. Distribución de conversaciones (pie chart)
2. Análisis de ventas y conversión
3. Objeciones que impidieron cierres
4. Pizzas más pedidas (sin combo)
5. Combos más pedidos
6. Tamaños más frecuentes
7. Métodos de pago preferidos
8. Clientes recurrentes vs nuevos
9. Zonas geográficas activas
10. Perfiles de clientes
11. Horarios pico
12. Competencia mencionada
13. ¿Cómo nos encontraron? (canal orgánico)
14. Sentimiento general
15. Objeciones frecuentes con respuestas sugeridas
16. Recomendaciones accionables + indicadores adicionales

---

## Cómo replicarlo

### Prerrequisitos
- Cuenta en GitHub
- Cuenta en Cloudflare (gratis)
- Cuenta en Anthropic con créditos cargados ([console.anthropic.com](https://console.anthropic.com))

### Paso 1 — Fork o copia del repositorio

```bash
# Opción A: clonar el repo
git clone https://github.com/jhoeliel/pizza-insights.git
cd pizza-insights

# Opción B: descargar solo el index.html y subirlo a un repo nuevo
```

### Paso 2 — Crear el proxy Cloudflare Workers

1. Crear cuenta en [workers.cloudflare.com](https://workers.cloudflare.com)
2. **Create application** → **Worker** → **Start with Hello World**
3. Reemplazar el código con el worker del archivo `claude_proxy_worker.js`
4. Hacer **Deploy**
5. Copiar la URL del worker (formato: `https://nombre.usuario.workers.dev`)

### Paso 3 — Actualizar la URL del proxy en el código

En `index.html`, línea ~430, reemplazar:
```javascript
// Antes:
var res = await fetch('https://claude-proxy.jhoelielpalma75.workers.dev', {

// Después (con tu URL):
var res = await fetch('https://tu-worker.tu-usuario.workers.dev', {
```

### Paso 4 — Activar GitHub Pages

1. Subir `index.html` a un repositorio público en GitHub
2. **Settings** → **Pages** → **Deploy from branch** → `main` → `/ (root)`
3. Esperar ~2 minutos → la app estará en `https://tu-usuario.github.io/nombre-repo`

### Paso 5 — Obtener API key de Anthropic

1. Ir a [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys)
2. **Create key** → copiar (solo se muestra una vez)
3. Cargar créditos mínimos ($5 USD) en **Billing**
4. Abrir la app desplegada → pegar la key → **Guardar**

### Costo estimado por análisis
- Análisis de 70 chats (~1900 mensajes): ~$0.08 USD
- El análisis hace 4 llamadas a Claude Sonnet 4.5
- ~4,000 tokens de entrada + ~2,000 tokens de salida por llamada
- Total: ~24,000 tokens input + ~8,000 tokens output ≈ $0.08 USD al precio actual

---

## Variables y configuración

### Parámetros ajustables en el código

```javascript
// Número de mensajes por bloque de muestra (línea ~370)
// Aumentar mejora precisión pero puede superar rate limits
var n = 45; // mensajes por bloque (3 bloques = 135 mensajes totales)

// Pausa entre llamadas a la API (línea ~420)
// Aumentar si tienes errores de rate limit (429)
await new Promise(x => setTimeout(x, 5000)); // 5000ms = 5 segundos

// Modelo de Claude (línea ~430)
model: 'claude-sonnet-4-5'
// Alternativas: 'claude-haiku-4-5' (más barato), 'claude-opus-4-5' (más potente)

// Tokens de salida por llamada (línea ~435)
max_tokens: 4000
// Reducir si hay errores de cuota, aumentar si el JSON se corta

// Temperatura (línea ~436)
temperature: 0.2
// 0.0 = más determinista, 1.0 = más creativo
// Para análisis de datos se recomienda 0.1-0.3
```

### Rate limits de Anthropic por tier

| Tier | Tokens/min | Requests/min | Cómo subir |
|---|---|---|---|
| Tier 1 (inicial) | 40,000 | 50 | Automático tras $100 gastados |
| Tier 2 | 80,000 | 100 | Automático tras $500 gastados |
| Tier 3 | 160,000 | 200 | Automático tras $1,000 gastados |

Con Tier 1, la app necesita las pausas de 5s entre llamadas. Con Tier 2+, se pueden eliminar.

---

## Limitaciones conocidas

**Formato de exportación de WhatsApp**
El parser está diseñado para el formato latinoamericano (`DD/MM/YYYY`). Exportaciones en inglés usan `MM/DD/YYYY` y pueden no parsearse correctamente. Para adaptar, modificar el regex en la función `parseAll()`.

**Representatividad de la muestra**
Con 100+ chats, se analizan 45 mensajes por bloque (135 en total). Conversaciones muy cortas (1-2 mensajes) tienen la misma probabilidad de aparecer en la muestra que conversaciones largas, lo que puede subestimar la frecuencia de ciertas interacciones.

**Clasificación de ventas**
La IA clasifica conversaciones como "cerrada/no cerrada/curiosa" basándose en el texto. Sin acceso al sistema de pedidos real, esta clasificación es una estimación. Conversaciones donde el cliente confirma verbalmente pero paga en efectivo pueden ser subcontadas.

**Memoria entre llamadas**
Las 4 llamadas son independientes. Claude no recuerda el contexto de la llamada 1 al hacer la llamada 2. Los bloques se solapan parcialmente (S2 aparece en llamadas 1 y 2) para mantener cierta coherencia.

**Privacidad**
Los mensajes de WhatsApp contienen datos personales (nombres, teléfonos, direcciones). Los mensajes viajan a la API de Anthropic para el análisis. Revisar la [política de privacidad de Anthropic](https://www.anthropic.com/privacy) antes de usar en producción con datos de clientes reales.

---

## Estructura del repositorio

```
pizza-insights/
├── index.html              # App completa (HTML + CSS + JS en un solo archivo)
└── README.md               # Este archivo
```

El proyecto intencionalmente mantiene todo en un único archivo `index.html` para simplificar el deployment y la portabilidad. No hay dependencias npm, no hay proceso de build, no hay archivos de configuración.

---

## Créditos

Desarrollado por **Jhoeliel Palma** — [github.com/jhoeliel](https://github.com/jhoeliel)

Construido con Claude AI (Anthropic) como herramienta de desarrollo y como motor de análisis.
