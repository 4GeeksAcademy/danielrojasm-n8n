# SnackCheck — Verificador de nutrición (flujo de n8n)

**Versión:** 1.0.1 · **Archivo del flujo:** `snackcheck-workflow.json` · **Diagrama:** `snackcheck-diagram.excalidraw`

## Propósito

SnackCheck es el motor detrás de una app de consumo: recibe el código de barras de un alimento empaquetado y devuelve, en lenguaje claro y amigable, si es una buena elección para el día a día o algo para dejar como gusto ocasional. Los datos de nutrición vienen de Open Food Facts (base de datos abierta, sin clave) y el párrafo final lo redacta un modelo de IA de Groq, con un tono que cambia según el resultado.

## Cómo funciona

El flujo sigue una sola dirección, de la entrada a la respuesta, en cinco etapas (que coinciden con las notas adhesivas del lienzo).

**1. Entrada y validación.** El nodo `Webhook - Receive Barcode` acepta `POST /nutrition-check`. Si el cuerpo llega vacío, `Validate - Request Body Empty` envía la autodocumentación (200, JSON). Si hay cuerpo pero `barcode` falta o no es numérico, `Validate - Barcode Present & Numeric` devuelve un error estructurado (400, JSON) con un ejemplo.

**2. Datos.** `API Call - Open Food Facts Product` consulta `GET /api/v2/product/{barcode}.json` pidiendo solo `product_name, brands, nutriscore_grade, nutriments`. Está configurado con *Full Response* y *Never Error*, así que un 404 no tumba el flujo. `Validate - Product Found` exige las dos señales (HTTP 200 **y** `status = 1`); si no, responde 404 en JSON con un código de ejemplo. Luego `Set - Unwrap Product Object` deja el producto plano y `Set - Extract Product Fields` crea variables planas, leyendo la energía con notación de corchetes (`nutriments['energy-kcal_100g']`). Los campos ausentes quedan en `null`, nunca en `0`. `Validate - Nutrient Data Present` exige azúcar, sal y grasa; si falta alguno, responde «datos insuficientes» (200, JSON) sin clasificar.

**3. Lógica.** `Set - Classify Traffic Lights` aplica los umbrales del fundador por 100 g (azúcar: <5 bajo, 5–22.5 medio, >22.5 alto; sal >1.5 alto; grasa >17.5 alto) y asigna una marca 🔴/🟢 a cada nutriente. `Code - Concern Score` suma cuántos quedaron en `alto` (0–3) en un solo ítem, sin bucles; es una señal informativa que no decide el nivel. `Set - Compute Readings` obtiene la lectura de semáforo (0 altos → saludable, 1 → moderado, 2–3 → no saludable) y la de Nutri-Score (a/b → saludable, c → moderado, d/e → no saludable). `Set - Verdict Level (Worst Of Two)` toma la peor de las dos; si el Nutri-Score es `unknown`, `not-applicable` o falta, usa solo el semáforo.

**4. IA con tono condicional.** `Route - Verdict Is Unhealthy` envía los productos «no saludable» a `AI - Health Verdict (Cautionary)` y los «saludable»/«moderado» a `AI - Health Verdict (Encouraging)`. Cada rama tiene su propio prompt de sistema y su propio modelo de Groq; no es un prompt fijo.

**5. Salida.** `Merge - Append AI Verdict` reúne ambas ramas, `Set - Format Verdict Message` arma el mensaje (titular con emoji, números alineados con 🔴/🟢, puntuación de preocupación y párrafo de la IA) y `Respond - Verdict Plain Text (200)` lo devuelve como `text/plain`.

## Configuración

Necesitas una instancia de n8n con los nodos de LangChain (incluidos por defecto en versiones 1.x recientes).

1. En n8n, abre un flujo nuevo, menú **⋯ → Import from File** y elige `snackcheck-workflow.json`.
2. Abre `Model - Groq (Cautionary)` y `Model - Groq (Encouraging)` y selecciona tu credencial de Groq existente. Es la única credencial del proyecto.
3. Ambos nodos usan el modelo `openai/gpt-oss-20b` con un máximo de 1000 tokens. Este modelo razona antes de responder, así que un límite bajo (por ejemplo 300) corta el párrafo a mitad de frase. Si cambias de modelo, usa uno de chat de propósito general (no variantes *safeguard* o *guard*, que son clasificadores de moderación).
4. Guarda y activa el flujo. La URL de producción será `https://TU-N8N/webhook/nutrition-check` (para pruebas, usa la URL de *test* con `/webhook-test/`).

Open Food Facts no requiere clave ni registro. El flujo envía un encabezado `User-Agent` identificándose, como pide la política de uso de la API.

## Uso

Veredicto para un producto (Nutella):

```bash
curl -X POST https://TU-N8N/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{"barcode": "3017620422003"}'
```

Respuesta (200, `text/plain`), con el párrafo variando según la IA:

```
⚠️ Mejor como gusto ocasional — Nutella
Marca: Nutella, Ferrero · Nutri-Score: sin dato

Por cada 100 g
──────────────────────────────────
🔴 Azúcar      56.3 g    (alto)
🟢 Sal         0.11 g    (normal)
🔴 Grasa       30.9 g    (alto)
⚪ Energía      539 kcal
──────────────────────────────────
Nutrientes en alto: 2/3

[párrafo cauteloso y delicado generado por la IA]
```

Autodocumentación (cuerpo vacío):

```bash
curl -X POST https://TU-N8N/webhook/nutrition-check
```

## Códigos de prueba verificados en vivo

Nutella `3017620422003` (no saludable, Nutri-Score desconocido hoy en OFF), Coca-Cola `5449000000996` (no saludable por Nutri-Score E), Barilla Spaghetti n.º 5 `8076800195057` (saludable, Nutri-Score A) y `2999999999993` (no existe, OFF responde 404).

## Manejo de errores

| Situación | Nodo que responde | Código | Formato |
|---|---|---|---|
| Cuerpo vacío | `Respond - Self Documentation (200)` | 200 | JSON con servicio, campo requerido y ejemplo |
| `barcode` ausente | `Respond - Validation Error (400)` | 400 | JSON `invalid_barcode` + ejemplo |
| `barcode` no numérico | `Respond - Validation Error (400)` | 400 | JSON `invalid_barcode` + valor recibido + ejemplo |
| Producto no existe (OFF responde 404, `status: 0`) | `Respond - Product Not Found (404)` | 404 | JSON `product_not_found` + ejemplo |
| Producto sin azúcar, sal o grasa | `Respond - Insufficient Data (200)` | 200 | JSON `insufficient_data` + campos faltantes |
| Veredicto normal | `Respond - Verdict Plain Text (200)` | 200 | `text/plain; charset=utf-8` |

Open Food Facts usa dos formas de decir «no encontrado»: un código válido que no existe devuelve HTTP 404 con `status: 0`, y un código inválido (como `0000000000000`) devuelve HTTP 200 con `status: 0`. Como el flujo exige HTTP 200 **y** `status = 1`, ambos casos terminan en la respuesta 404. Si Open Food Facts no responde (timeout de 10 s o error de red), el nodo HTTP continúa gracias a *Continue (regular output)* y la solicitud termina en la respuesta 404, en lugar de caerse.

## Limitaciones

Un fallo temporal de Open Food Facts (5xx o timeout) se informa como «no encontrado»; una versión futura podría distinguirlo con un 502. Si la app envía el código como número JSON en lugar de string, los ceros iniciales (comunes en UPC) se pierden, así que la app debe enviarlo como string. Basta con que falte uno de los tres nutrientes para responder «datos insuficientes»; es una decisión conservadora para que un campo vacío nunca cuente como «no alto». Las bebidas se miden por 100 ml con los mismos umbrales, por eso el Nutri-Score es imprescindible para detectarlas. El texto de la IA puede variar entre ejecuciones y depende de la disponibilidad de Groq. El servicio no da consejo médico ni considera alergias, porciones ni dietas especiales.
