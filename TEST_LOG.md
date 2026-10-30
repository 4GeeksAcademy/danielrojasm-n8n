# Registro de pruebas — SnackCheck v1.0.1

**Fecha:** 2026-10-30 · **Flujo:** `snackcheck-workflow.json` v1.0.1 · **Instancia:** n8n Cloud (workflow `gXWJSAZZuCkmnkGr`) · **Modelo IA:** `openai/gpt-oss-20b` en Groq

## Cómo se ejecutaron estas pruebas

Las pruebas TC-001 a TC-022 se corrieron con un **simulador** que lee el archivo exportado del flujo y evalúa, nodo por nodo, **las mismas expresiones** que ejecutaría n8n (IF, Set, Code, Respond y el armado de los prompts). Se simularon tres cosas externas: la solicitud al webhook, la respuesta de Open Food Facts y el texto de Groq (sustituido por un marcador `[IA-CAUTELOSA]` / `[IA-ALENTADORA]` para verificar qué rama se usó).

Los datos de **Nutella** y **Coca-Cola** son los valores reales que muestran hoy sus fichas en Open Food Facts. Los demás productos son perfiles sintéticos, marcados «(prueba)», diseñados para ejercitar cada rama y cada umbral. Para comprobar que el simulador sí detecta fallos, se rompió a propósito el umbral de azúcar (`<=` → `<`) y TC-008 cambió de resultado, como se esperaba.

**Resultado: 22/22 casos simulados pasaron, y 9/9 pruebas en vivo en n8n (TC-023 a TC-031) pasaron** tras tres correcciones de configuración documentadas abajo.

> ⚠️ **Hallazgo:** el brief dice que Nutella tiene Nutri-Score `e`, pero su ficha actual en Open Food Facts lo muestra como **desconocido** (faltan datos para calcularlo). El flujo lo maneja con el respaldo de semáforo y el resultado sigue siendo «no saludable» (TC-001). TC-002 cubre el caso con `e`, por si la ficha vuelve a tenerlo. Como la ruta «Nutri-Score malo gana al semáforo» depende de esto, la prueba de referencia para ese requisito es la **Coca-Cola (TC-003)**, que sí tiene E.

## Pruebas funcionales

| ID | Descripción | Entrada | Resultado esperado | Resultado obtenido | Estado |
|---|---|---|---|---|---|
| TC-001 | Nutella (datos reales, Nutri-Score unknown hoy) | `{"barcode":"3017620422003"}` | 200 texto · 2 altos (azúcar, grasa) · Nutri-Score desconocido → solo semáforo → no saludable · tono cauteloso | 200 · text/plain · semáforo: no saludable, Nutri-Score: desconocido → **no saludable** · preocupación 2/3 · IA: cautelosa | ✅ Pasó |
| TC-002 | Nutella con Nutri-Score E (valor del brief) | `{"barcode":"3017620422999"}` | 200 texto · ambas lecturas «no saludable» → no saludable · tono cauteloso | 200 · text/plain · semáforo: no saludable, Nutri-Score: no saludable → **no saludable** · preocupación 2/3 · IA: cautelosa | ✅ Pasó |
| TC-003 | Coca-Cola (datos reales, Nutri-Score E) | `{"barcode":"5449000000996"}` | 200 texto · semáforo «saludable» pero Nutri-Score E → gana el peor: no saludable | 200 · text/plain · semáforo: saludable, Nutri-Score: no saludable → **no saludable** · preocupación 0/3 · IA: cautelosa | ✅ Pasó |
| TC-004 | Producto Nutri-Score A (sintético) | `{"barcode":"1000000000001"}` | 200 texto · las tres marcas 🟢 · Nutri-Score A → saludable · tono alentador | 200 · text/plain · semáforo: saludable, Nutri-Score: saludable → **saludable** · preocupación 0/3 · IA: alentadora | ✅ Pasó |
| TC-005 | Nutri-Score unknown + 1 alto → solo semáforo | `{"barcode":"1000000000002"}` | Nutri-Score unknown → solo semáforo · 1 alto (sal) → moderado | 200 · text/plain · semáforo: moderado, Nutri-Score: desconocido → **moderado** · preocupación 1/3 · IA: alentadora | ✅ Pasó |
| TC-006 | Sin campo nutriscore_grade y números como texto | `{"barcode":"1000000000003"}` | Falta nutriscore_grade → solo semáforo; los strings numéricos se convierten → saludable | 200 · text/plain · semáforo: saludable, Nutri-Score: desconocido → **saludable** · preocupación 0/3 · IA: alentadora | ✅ Pasó |
| TC-007 | Nutri-Score C y 0 altos → moderado | `{"barcode":"1000000000004"}` | Semáforo saludable, Nutri-Score C → moderado (gana el peor) | 200 · text/plain · semáforo: saludable, Nutri-Score: moderado → **moderado** · preocupación 0/3 · IA: alentadora | ✅ Pasó |
| TC-008 | Valores exactamente en los umbrales | `{"barcode":"1000000000005"}` | Azúcar 22.5 = medio, sal 1.5 y grasa 17.5 no son «alto» (la regla es «más de») → saludable | 200 · text/plain · semáforo: saludable, Nutri-Score: saludable → **saludable** · preocupación 0/3 · IA: alentadora | ✅ Pasó |
| TC-009 | 3 altos con Nutri-Score not-applicable | `{"barcode":"1000000000006"}` | 3 altos · not-applicable se trata como desconocido → no saludable · preocupación 3/3 | 200 · text/plain · semáforo: no saludable, Nutri-Score: desconocido → **no saludable** · preocupación 3/3 · IA: cautelosa | ✅ Pasó |

## Pruebas de integración

| ID | Descripción | Entrada | Resultado esperado | Resultado obtenido | Estado |
|---|---|---|---|---|---|
| TC-010 | Respuesta con forma plana (como describe el brief) | `{"barcode":"1000000000009"}` | El paso Unwrap tolera el objeto plano → se clasifica igual | 200 · text/plain · semáforo: saludable, Nutri-Score: saludable → **saludable** · preocupación 0/3 · IA: alentadora | ✅ Pasó |
| TC-011 | Código con espacios alrededor | `{"barcode":" 3017620422003 "}` | Los espacios se recortan → se consulta 3017620422003 → veredicto | 200 · text/plain · semáforo: no saludable, Nutri-Score: desconocido → **no saludable** · preocupación 2/3 · IA: cautelosa | ✅ Pasó |
| TC-022 | Código enviado como número JSON (no string) | `{"barcode":5449000000996}` | El número se convierte a string → veredicto de Coca-Cola | 200 · text/plain · semáforo: saludable, Nutri-Score: no saludable → **no saludable** · preocupación 0/3 · IA: cautelosa | ✅ Pasó |

## Pruebas de error

| ID | Descripción | Entrada | Resultado esperado | Resultado obtenido | Estado |
|---|---|---|---|---|---|
| TC-012 | Cuerpo vacío {} | `{}` | 200 JSON de autodocumentación | 200 · application/json · autodocumentación | ✅ Pasó |
| TC-013 | Cuerpo ausente (null) | _(sin cuerpo)_ | 200 JSON de autodocumentación | 200 · application/json · autodocumentación | ✅ Pasó |
| TC-014 | Falta el campo barcode | `{"codigo":"123"}` | 400 JSON invalid_barcode «Falta el campo barcode» + ejemplo | 400 · application/json · invalid_barcode | ✅ Pasó |
| TC-015 | barcode vacío | `{"barcode":""}` | 400 JSON invalid_barcode «Falta el campo» + ejemplo | 400 · application/json · invalid_barcode | ✅ Pasó |
| TC-016 | barcode no numérico | `{"barcode":"abc-123"}` | 400 JSON invalid_barcode «solo números», received = "abc-123" | 400 · application/json · invalid_barcode | ✅ Pasó |
| TC-017 | Producto inexistente (OFF 404, status 0) | `{"barcode":"0000000000000"}` | El flujo no se cae; 404 JSON product_not_found + ejemplo | 404 · application/json · product_not_found | ✅ Pasó |
| TC-018 | Producto sin nutriments (objeto vacío) | `{"barcode":"1000000000007"}` | 200 JSON insufficient_data; no se llama a la IA | 200 · application/json · insufficient_data (faltan: sugars_100g, salt_100g, fat_100g) | ✅ Pasó |
| TC-019 | Producto con datos parciales (solo azúcar) | `{"barcode":"1000000000008"}` | 200 JSON insufficient_data con salt_100g y fat_100g faltantes | 200 · application/json · insufficient_data (faltan: salt_100g, fat_100g) | ✅ Pasó |
| TC-020 | Producto sin clave nutriments | `{"barcode":"1000000000010"}` | 200 JSON insufficient_data (sin objeto nutriments) | 200 · application/json · insufficient_data (faltan: sugars_100g, salt_100g, fat_100g) | ✅ Pasó |
| TC-021 | Fallo de red / timeout de OFF | `{"barcode":"9999999999999"}` | El nodo HTTP continúa ante el error → 404 JSON (limitación documentada) | 404 · application/json · product_not_found | ✅ Pasó |

## Pruebas en vivo en n8n (ejecuciones reales)

Ejecutadas en modo manual sobre la instancia real, con llamadas reales a Open Food Facts y a Groq. El número entre paréntesis es el ID de ejecución en n8n (pestaña *Executions*).

| ID | Tipo | Descripción | Entrada | Resultado esperado | Resultado obtenido | Estado |
|---|---|---|---|---|---|---|
| TC-023 | Integración | Llamada real a Open Food Facts | `{"barcode":"5449000000996"}` | HTTP 200, `status: 1`, datos en `body.product` | (#1) HTTP 200, `status: 1`, «product found», datos dentro de `body.product` (confirma el paso Unwrap). API: 527 ms | ✅ Pasó |
| TC-024 | Funcional | Gaseosa con IA real | Coca-Cola `5449000000996` | Semáforo saludable + Nutri-Score E → no saludable · tono cauteloso | (#9) semáforo «saludable», Nutri-Score «no saludable» → **no saludable**, preocupación 0/3, rama Cautionary; párrafo completo que sugiere tomarla como gusto ocasional y propone agua con gas | ✅ Pasó |
| TC-025 | Funcional | Nutella con IA real (Nutri-Score unknown en vivo) | `3017620422003` | Solo semáforo, 2 altos → no saludable · energía 539 kcal por corchetes | (#10) `nutriscore_grade: unknown` confirmado; azúcar 🔴 56.3, grasa 🔴 30.9, sal 🟢; `energy_kcal_100g: 539`; **no saludable**, preocupación 2/3, tono cauteloso | ✅ Pasó |
| TC-026 | Funcional | Producto real con Nutri-Score A | Barilla Spaghetti n.º 5 `8076800195057` | Ambas lecturas saludables → saludable · tono alentador | (#11) Nutri-Score A en vivo; azúcar 3.5 · sal 0.01 · grasa 2.0, todo 🟢; **saludable**, rama Encouraging; titular ✅ y párrafo alentador | ✅ Pasó |
| TC-027 | Error | Cuerpo vacío | `{}` | 200 JSON autodocumentación | (#2) ruta `Respond - Self Documentation (200)` · 62 ms | ✅ Pasó |
| TC-028 | Error | barcode no numérico | `{"barcode":"abc-123"}` | 400 JSON | (#3) ruta `Respond - Validation Error (400)` · 67 ms | ✅ Pasó |
| TC-029 | Error | Producto inexistente | `{"barcode":"2999999999993"}` | OFF 404 + `status: 0` → 404 JSON propio, sin caída | (#5) OFF respondió **404** «product not found»; el flujo siguió y respondió con `Respond - Product Not Found (404)` · 236 ms | ✅ Pasó |
| TC-030 | Error | Código inválido para OFF | `{"barcode":"0000000000000"}` | 404 JSON propio | (#4) Hallazgo: OFF responde **200** con `status: 0` («no code or invalid code»), no 404. Como el IF exige ambas señales, igual se respondió 404 · 119 ms | ✅ Pasó |

### Rendimiento (TC-031)

Cinco ejecuciones de la Coca-Cola de extremo a extremo (OFF + Groq + formato), ejecuciones #12 a #16:

| Corrida | 1 (#12) | 2 (#13) | 3 (#14) | 4 (#15) | 5 (#16) | Promedio |
|---|---|---|---|---|---|---|
| Duración | 1.07 s | 1.13 s | 0.78 s | 0.81 s | 0.94 s | **0.95 s** |

**Resultado: ✅ Pasó** (objetivo < 4 s). Las rutas de error, que no llaman a la IA, respondieron entre 62 y 236 ms. La primera ejecución con IA real (#9) tardó 2.49 s, probablemente por arranque en frío del modelo; las siguientes se estabilizaron por debajo de 1.2 s. La llamada a Open Food Facts tomó entre 188 y 527 ms.

### Incidencias encontradas y corregidas durante las pruebas en vivo

| # | Ejecución | Problema | Causa | Corrección |
|---|---|---|---|---|
| 1 | #1, #6 | El flujo se detenía en la IA: «Node does not have any credentials set» | `Model - Groq (Cautionary)` sin credencial asignada | Se asignó la credencial de Groq en ambos nodos de modelo |
| 2 | #7 | Groq respondía `model_not_found` | El modelo `llama-3.3-70b-versatile` ya no está disponible | Se cambió a `openai/gpt-oss-20b` en ambos nodos |
| 3 | #8 | El párrafo de la IA se cortaba a mitad de frase | El modelo razona antes de responder y agotaba el tope de 300 tokens | `Maximum Number of Tokens` subido a 1000 en ambos nodos |

En ningún caso hubo que tocar la lógica del flujo; las tres incidencias fueron de configuración del modelo.

## Evidencia: salidas obtenidas en el simulador

**TC-003 — Coca-Cola (200, text/plain):**

```
⚠️ Mejor como gusto ocasional — Coca-Cola
Marca: COCA-COLA SERVICES SA/NV · Nutri-Score: E

Por cada 100 g
──────────────────────────────────
🟢 Azúcar      10.6 g    (medio)
🟢 Sal         0.00 g    (normal)
🟢 Grasa        0.0 g    (normal)
⚪ Energía       42 kcal 
──────────────────────────────────
Nutrientes en alto: 0/3

[IA-CAUTELOSA] párrafo de prueba.
```

**TC-004 — Producto Nutri-Score A (200, text/plain):**

```
✅ ¡Buena elección! — Spaghetti (prueba)
Marca: Marca Prueba · Nutri-Score: A

Por cada 100 g
──────────────────────────────────
🟢 Azúcar       3.5 g    (bajo)
🟢 Sal         0.01 g    (normal)
🟢 Grasa        2.0 g    (normal)
⚪ Energía      359 kcal 
──────────────────────────────────
Nutrientes en alto: 0/3

[IA-ALENTADORA] párrafo de prueba.
```

**TC-016 — barcode no numérico (400, application/json):**

```json
{
  "error": "invalid_barcode",
  "message": "El campo \"barcode\" debe contener solo números (sin letras, espacios ni guiones).",
  "received": "abc-123",
  "example": {
    "barcode": "3017620422003"
  }
}
```

**TC-017 — producto inexistente (404, application/json):**

```json
{
  "error": "product_not_found",
  "message": "No encontramos ese código de barras en Open Food Facts. Revisa que esté completo o prueba con otro producto.",
  "barcode": "0000000000000",
  "example": {
    "barcode": "3017620422003"
  }
}
```

**TC-019 — datos parciales (200, application/json):**

```json
{
  "status": "insufficient_data",
  "message": "Encontramos el producto, pero no tiene datos de nutrición suficientes (azúcar, sal y grasa por 100 g) para darte un veredicto confiable.",
  "product": {
    "barcode": "1000000000008",
    "name": "Parcial (prueba)",
    "brands": "Marca Prueba"
  },
  "missing_fields": [
    "salt_100g",
    "fat_100g"
  ],
  "tip": "Puedes completar los datos del producto en openfoodfacts.org."
}
```
