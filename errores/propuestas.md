# Propuestas de contribución

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
- **Causa probable:** `modelito.ollama_service.list_local_models()` devuelve filas completas de `ollama list` en lugar de solo los nombres (verificado el 2026-10-22: `'qwen2.5:7b  845dbda0ea48  4.7 GB  20 hours ago'`). Si la pantalla de configuración guardó una de esas filas como modelo, se explica el valor corrupto. El commit `b4120b6` ("Use canonical Ollama model names for selection and readiness") parece corregirlo del lado de BatLLM, pero el archivo ya corrupto quedó en el repositorio.
- **Posible abordaje:** restaurar `llm.model: smollm2` (y revisar el resto de los valores cambiados en `9554dad`); evaluar si conviene una protección adicional contra subir configuración local.
- **Estado:** **resuelto por el mantenedor.**
  - Commit `fc680a2` ("Restore config.yaml", 2026-10-23): `llm.model` vuelve a ser `smollm2`.
  - Commit `98d0a24` ("fixed model selection: modelito returns more than just the name"): corrige la selección de modelos, en línea con la causa probable.
  - Verificado el 2026-10-26 en `d86ae05`: la suite completa sin Ollama (`pytest src/tests`) da **247 pasan, 2 se omiten, 0 fallan**.
  - Nota: con modelito 1.4.6, `list_local_models()` sigue devolviendo filas completas de `ollama list` (verificado el 2026-10-24); BatLLM lo compensa de su lado.

## 2. modelito no usa la API HTTP de Ollama: aplana los roles, ignora las opciones y corrompe el texto

- **Tipo:** error reproducible (en modelito 1.4.5, la versión que fija `requirements.txt`; es la última publicada en PyPI al 2026-10-22). Afecta a BatLLM.
- **Encontrado:** 2026-10-22, BatLLM commit `645189b`, modelito 1.4.5, Ollama 0.40.8, Windows 10, `llama3.2:3b`.
- **Cómo reproducirlo:**
  1. Con Ollama corriendo, ejecutar en el venv de BatLLM:
     ```python
     from modelito import Client, Message, OllamaProvider
     p = OllamaProvider(host="http://localhost", port=11434, model="llama3.2:3b")
     c = Client(provider=p, model="llama3.2:3b")
     print(ascii(c.summarize([Message(role="user", content="Escribí: canción")], settings={"temperature": 0})))
     ```
  2. Alternativa: el script de la tarea 0 (`Taller/Script Ollama/scriptOllama.py`) con la biblioteca `modelito`, comparado con la biblioteca `ollama`.
- **Qué pasa** (según el código de `modelito/ollama.py`, `OllamaProvider.summarize`, líneas 89–209, y verificado con pruebas):
  1. **La llamada HTTP siempre falla en silencio.** El pedido a `/api/chat` no incluye `"stream": false`, así que Ollama responde en *streaming* (varias líneas JSON; 8 en la prueba). `json_post` intenta leerlo como un único JSON, falla (`Extra data: line 2 column 1`) y el error se descarta (líneas 130–134).
  2. **Se usa la línea de comandos como alternativa** (líneas 163–204). Prueba `ollama run <modelo> --prompt …` (falla: `unknown flag: --prompt`), `ollama generate …` (falla: el comando no existe) y finalmente `ollama run <modelo> "<texto>"`, que funciona.
  3. **Se pierden los roles.** El texto enviado por la línea de comandos es el contenido de todos los mensajes unido con saltos de línea (líneas 106–109). `system`, `user` y `assistant` llegan al modelo como un único mensaje de usuario.
  4. **Se ignoran las opciones.** `settings` no se usa (`if settings is not None: pass`, líneas 103–104): `temperature`, `num_ctx`, `num_predict`, etc. no llegan a Ollama.
  5. **En Windows se corrompe el texto de la respuesta.** `run_ollama_command` (`ollama_service.py`, línea 1197) decodifica la salida con la codificación del sistema y no con UTF-8 (`"canción"` → `Melodía` se recibió como `MelodÃ­a`). Además, la respuesta incluye códigos de control de la terminal (`ESC[5D ESC[K`) y palabras duplicadas donde la terminal ajustó la línea (`favor…\nfavorito`).
  6. **Puede bloquearse.** `subprocess.run` no tiene límite de tiempo; una de las pruebas quedó colgada y hubo que interrumpirla.
  7. **Si la línea de comandos también falla, devuelve la entrada como si fuera la respuesta** (línea 209, "deterministic fallback"). Según el código; no se probó.
- **Efecto en BatLLM (verificado):** `src/game/ollama_connector.py` llama a `Client.summarize` para cada consulta del juego (líneas 81 y 467, con `stream=False`). Al invocar ese cliente directamente con un mensaje `system` y uno `user`, se registraron las tres llamadas a la línea de comandos descritas en el punto 2, con ambos mensajes unidos en un solo texto. Por lo tanto, en las partidas:
  - las instrucciones del juego no viajan con rol `system`;
  - el historial no conserva los roles;
  - la temperatura y `num_ctx` configuradas no se aplican;
  - en Windows, el historial que se reenvía puede contener texto corrupto.
  Esto afecta la interpretación de los experimentos del taller (contextos independientes/compartido, temperatura).
- **Qué debería pasar:** enviar `"stream": false` (o leer correctamente el *streaming*), pasar `settings` como `options` de Ollama, conservar los roles usando `/api/chat`, decodificar en UTF-8 y no ocultar los errores.
- **Por qué es relevante:** BatLLM es un instrumento de investigación; las condiciones experimentales (roles, contexto, temperatura) deben ser las que se configuran.
- **Posible abordaje:** corregirlo en modelito (repositorio `krahd/modelito`) y agregar una prueba contra un servidor que responda en *streaming*, como lo hace Ollama por defecto. Una prueba de integración en BatLLM podría verificar que los roles y las opciones llegan al servidor.
- **Estado:** **corregido en modelito 1.4.6** (tras comentarlo con Tom).
  - Verificado el 2026-10-24 con modelito 1.4.6, llamando al `Client` de `src/game/ollama_connector.py`:
    - el pedido HTTP incluye `stream: false` y `options` (`temperature`, `num_ctx`);
    - los roles llegan separados (`['system', 'user']`) y no se usa la línea de comandos (0 llamadas);
    - los acentos llegan bien (`'Canción.'`);
    - con el `system` "Respondé siempre con una sola palabra: PINGÜINO.", el modelo respondió `PINGÜINO.`.
  - `stream()` ya no devuelve la última línea (`"done": true`) como texto.
  - BatLLM fija `modelito==1.4.6` en `requirements.txt` desde el commit `7af76cf` (PR #62, 2026-10-26).


## Mi descripción del problema:

**Cómo debería funcionar:**  
BatLLM -> modelito -> POST a url/api/chat -> Ollama -> modelo de Ollama
Lo enviado debería ser  la lista de mensajes junto con sus roles y opciones como la temperatura.

**Cómo funciona:**

Por defecto /api/chat de Ollama responde en streaming. O sea que manda la respuesta de a pedazos a medida que la va generando, un JSON por pedazo.

[{"message":{"content":"¡"},"done":false}  
{"message":{"content":"Hola"},"done":false}  
{"message":{"content":"!"},"done":false}  
{"message":{"content":""},"done":true, ...}]

modelito, para leer la respuesta, hace json.loads(cuerpo) y ese método espera sólo un objeto JSON, pero recibe varios, porque la respuesta se devolvió en varios. Ese código
se atrapa con una Exception pero se descarta sin avisar. Luego, por cómo está programado, usa la terminal como alternativa para contactarse con Ollama y ahí se pierden los roles. El problema es que la terminal solo acepta un texto, entonces lo que antes era un JSON con un mensaje por rol, ahora es un mensaje separado por saltos de línea.

[{system: "Respondé siempre con una sola palabra: PINGÜINO."
user:   "Escribí la palabra canción."}]  
         ↓ Pasa a ser  
"Respondé siempre con una sola palabra: PINGÜINO.\nEscribí la palabra canción."

Y ese texto es lo que le llega al modelo como un único mensaje con rol user. El rol system desaparece y si quisieramos usar historial también desaparece la distinción entre lo que dijo el usuario y la respuesta del modelo. Además, temperature y otros parámetros desaparecen en el camino.

Además hay un problema con la codificación a la hora de mostrar los mensajes. Las acentuaciones generan problemas y el mensaje queda corrupto.

**Cómo se puede corregir:**

- Si al conectarse con Ollama se envía "stream": false entonces Ollama envía una respuesta final de una, no por partes. Esto igual cortaría la posibilidad de ir renderizando el mensaje generado a medida que va siendo devuelto.

- Usando el json.loads, leer lo recibido línea por línea. Ir tomando el message.content de cada línea, mostrarlo y acumularlo.  
  Cuando el mensaje en json.loads tenga "done": true, se frena.  
  Al final, unir los pedazos y ese texto es lo que se guarda en el historial con el rol "assistant".

## 3. La pantalla Settings no entra en una pantalla de 1920×1080: la barra de botones queda fuera

- **Tipo:** error reproducible de interfaz
- **Encontrado:** 2026-10-26, commit `d86ae05`, Windows 10, pantalla 1920×1080 (área de trabajo 1920×1040), escala de Windows al 100 %.
- **Cómo reproducirlo:** abrir BatLLM (ventana maximizada) e ir a **Settings**.
- **Qué pasa:**
  - Parte del contenido queda fuera de la ventana; en particular, la barra inferior de botones (**Cancel**, **Set as Defaults**, **Set Temporarily**, **Ollama Config**) no se ve.
  - Sin el botón **Ollama Config** no se puede elegir el modelo desde la interfaz.
- **Causa probable** (estimada a partir de `src/view/settings_screen.kv`; no medida en la app):
  - El contenedor principal tiene altura fija (`height: self.minimum_height`), no tiene `ScrollView` y está centrado verticalmente (`center_y: 0.5`).
  - Sumando las alturas declaradas (márgenes y separaciones ~155 px, títulos con 80 px de relleno arriba y abajo ~260 px, 6 deslizadores ~390 px, 6 casillas ~360 px, barra de botones 90 px), necesita **~1250 px**; la ventana maximizada ofrece ~1000 px.
  - Al estar centrado, el sobrante (~250 px) se corta mitad arriba y mitad abajo: unos 125 px quedan debajo del borde, más que los 90 px de la barra de botones.
- **Solución temporal:** lanzar BatLLM con `$env:KIVY_METRICS_DENSITY = "0.75"`, que reduce toda la interfaz al 75 %.
- **Qué debería pasar:** todos los controles de Settings deberían ser accesibles en resoluciones comunes de escritorio.
- **Por qué es relevante:** 1920×1080 es una de las resoluciones más comunes; la pantalla deja inaccesible la configuración de Ollama. Se relaciona con las pruebas de primer uso en Windows y con `docs/UI_UNIFICATION_PLAN_1_0.md`.
- **Posible abordaje:** poner los controles dentro de un `ScrollView` y dejar la barra de botones fija abajo; o reducir los rellenos de los títulos. Agregar una prueba que verifique que la barra de botones queda dentro de la ventana en 1920×1080.
- **Detalle adicional:** el título de la pantalla dice **"BattLLM"** (doble *t*), en `settings_screen.kv`, línea 27.
- **Estado:** idea. Pendiente: captura de pantalla como evidencia.

