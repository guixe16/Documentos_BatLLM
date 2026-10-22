# Propuestas de contribución (tarea 5)

Problemas encontrados e ideas de mejora. Para cada uno: qué pasa, por qué es relevante y cómo se podría abordar.

<!--
PLANTILLA

## N. <título>

- **Tipo:** error reproducible / ambigüedad de interfaz / prueba ausente / documentación
- **Encontrado:** AAAA-MM-DD, commit ``
- **Cómo reproducirlo:**
- **Qué pasa / qué debería pasar:**
- **Por qué es relevante:**
- **Posible abordaje:**
- **Estado:** idea / discutido en clase / issue # / PR #
-->

## 1. `config.yaml` del repositorio con un nombre de modelo corrupto

- **Tipo:** error reproducible
- **Encontrado:** 2026-10-21, commit `645189b`
- **Cómo reproducirlo:** en un clon limpio, `pytest src/tests` (o `python run_tests.py non-live`).
- **Qué pasa:** falla `test_shipped_llm_config_uses_smollm2_first_install_defaults`. En `src/configs/config.yaml`, la clave `llm.model` contiene una fila completa de `ollama list` (`mistral-small3.2:24b-instruct-2506-q4_K_M  5a408ab55df5  15 GB  2 months ago`) en lugar de `smollm2`. Entró en el commit `9554dad`.
- **Qué debería pasar:** la configuración incluida en el repo debería usar `smollm2` como modelo inicial, y el test debería pasar.
- **Por qué es relevante:**
  - Una instalación nueva arranca con un modelo que no existe.
  - `run_tests.py non-live` se detiene en ese primer test y no corre el resto de la suite.
  - Correr la app desde el código fuente sin `BATLLM_HOME` escribe la configuración en un archivo versionado, lo que facilita que estos cambios se suban por accidente.
- **Posible abordaje:** restaurar `llm.model: smollm2` (y revisar el resto de los valores cambiados en `9554dad`); evaluar si conviene una protección adicional contra subir configuración local.
- **Estado:** idea
