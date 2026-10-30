# Changelog

Todos los cambios relevantes de este flujo se documentan aquí.
El formato sigue [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y el proyecto usa [Versionado Semántico](https://semver.org/lang/es/).

## [1.0.1] - 2026-10-30
### Fixed
- Modelo de IA cambiado de `llama-3.3-70b-versatile` (ya no disponible en Groq, error `model_not_found`) a `openai/gpt-oss-20b` en ambos nodos de modelo.
- Límite de tokens de salida subido de 300 a 1000: el nuevo modelo razona antes de responder y el párrafo se cortaba a mitad de frase.
- Credencial de Groq asignada a `Model - Groq (Cautionary)`, que había quedado sin credencial.

### Added
- Pruebas en vivo TC-023 a TC-031 en el registro de pruebas, incluida la medición de rendimiento (promedio 0.95 s).
- Documentación de las dos respuestas de «no encontrado» de Open Food Facts y de los códigos de prueba verificados.

## [1.0.0] - 2026-10-26
### Added
- Mensaje de veredicto con formato de varias secciones: titular con emoji, números alineados con marcadores 🔴/🟢 y párrafo de la IA, devuelto como `text/plain`.
- README del flujo, registro de pruebas TC-001 a TC-013 y documentación en línea en todos los nodos.

## [0.4.0] - 2026-10-26
### Added
- Paso de IA con Groq y enrutamiento condicional de tono: prompt cauteloso para «no saludable» y alentador para «saludable»/«moderado».
- Merge (Append) para reunir ambas ramas de IA.

## [0.3.0] - 2026-10-26
### Added
- Clasificación de semáforo para azúcar, sal y grasa con los umbrales del fundador.
- Puntuación de preocupación (0–3) en un nodo Code de un solo ítem.
- Lecturas de semáforo y Nutri-Score; nivel general = la peor de las dos, con respaldo a solo semáforo si el Nutri-Score es desconocido.

## [0.2.0] - 2026-10-26
### Added
- Llamada a Open Food Facts con el código inyectado en la URL.
- Respuesta 404 propia para productos no encontrados y respuesta 200 de «datos insuficientes».
- Extracción de campos planos, incluida la energía por notación de corchetes.

## [0.1.0] - 2026-10-26
### Added
- Diagrama del flujo en Excalidraw.
- Webhook `POST /nutrition-check`, autodocumentación para cuerpo vacío y validación 400 de código ausente o no numérico.
