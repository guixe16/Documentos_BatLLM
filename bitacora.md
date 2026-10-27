# Bitácora de trabajo

Una entrada por sesión de trabajo, de la más antigua a la más reciente.

**Total acumulado:** _5 h 0 min_ (actualizar en cada entrada)

---

<!--
PLANTILLA — copiar debajo para cada nueva sesión

## AAAA-MM-DD — HH:MM–HH:MM (X h YY min)

**Tarea(s):** 0 / 1 / 2 / 3 / 4 / 5

**Qué hice:**
-

**Resultados / evidencia:** (enlaces a experimentos, salidas, sesiones, commits)
-

**Dificultades e intentos de solución:**
-

**Pendiente / dudas para la clase:**
-

**Anotaciones**
-->

## 2026-10-21 — 19:30–21:00 (1:30hs)

**Tarea(s):** 1 (preparación), 0 (instalación de Ollama)

**Qué hice:**
- Cloné `krahd/BatLLM`, creé el fork https://github.com/guixe16/BatLLM y reconfiguré los remotes: `origin` → fork, `upstream` → original.
- Creé el entorno virtual `.venv_BatLLM` con Python 3.12.10 e instalé `requirements.txt` (Kivy 2.3.1, kivymd 1.2.0, modelito 1.4.5, ollama 0.6.3).
- Corrí la suite de tests sin Ollama (`pytest src/tests`) sobre el commit `645189b`: **192 pasan, 2 se omiten, 1 falla**.
- Instalé Ollama 0.40.8 y descargué `smollm2` (1,8 GB), `llama3.2:3b` (2,0 GB) y `qwen2.5:7b` (~4,7 GB).
- Hardware: NVIDIA RTX 4060 (8 GB de VRAM), 16 GB de RAM.

**Resultados / evidencia:**
- Test que falla: `test_shipped_llm_config_uses_smollm2_first_install_defaults` → ver [propuestas.md](propuestas.md).

**Dificultades e intentos de solución:**
- En Windows, `python` abre el alias de la Microsoft Store; se usa `py -3.12` para crear el venv.
- `python run_tests.py non-live` se detiene en el primer test que falla y no corre el resto; la suite completa se corrió con `pytest` directamente.
- Al ejecutar `ollama` sin argumentos se abre un menú para conectar integraciones (Claude Code, Codex, VS Code…). No hace falta para BatLLM; se sale con `Esc`.

**Pendiente / dudas para la clase:**
Ninguna

**Anotaciones:**

- Al probar la conexión con Ollama, al modelo le mandé el mensaje "Hola, ¿quién sos?" y me respondió "¡Hola! Me llamo Luna, pero puedo responder a preguntas y proporcionar información sobre una amplia variedad de temas. Soy un modelo de inteligencia artificial, diseñado para ayudar y aprender contigo. ¿En qué puedo ayudarte hoy?"

## 2026-10-22 — 9:00–11:00 (2hs)

**Tarea(s):** 0 (prueba de concepto con Ollama)

**Qué hice:**
- Creé el script `Taller/Script Ollama/scriptOllama.py` (con asistencia de IA): conversación por consola con un modelo de Ollama, con modo sin memoria (1) y con memoria (2), y registro en JSON de los mensajes enviados, la respuesta y los tokens de cada turno.
- Probé la conexión con `llama3.2:3b`.
- Ejecuté la prueba sin memoria y con memoria con `llama3.2:3b`, temperatura 0.
- Repasé los conceptos de tokens, ventana de contexto, roles de los mensajes y temperatura.
- Revisé la plantilla de chat del modelo (`ollama show llama3.2:3b --template`): convierte la lista de mensajes en un único texto con marcas de rol, y agrega siempre un bloque `system` con `Cutting Knowledge Date: December 2023`.

**Resultados / evidencia:**
- Experimento documentado: [experimentos/2026-10-22-tarea0-memoria.md](experimentos/2026-10-22-tarea0-memoria.md)
- Salidas crudas: [salidas/tarea0_2026-10-22_16-39-51_sin-memoria_llama3.2-3b.json](salidas/tarea0_2026-10-22_16-39-51_sin-memoria_llama3.2-3b.json), [salidas/tarea0_2026-10-22_16-41-52_con-memoria_llama3.2-3b.json](salidas/tarea0_2026-10-22_16-41-52_con-memoria_llama3.2-3b.json)
- Sin memoria, el modelo no supo el nombre dado en el turno anterior; con memoria, sí. Los tokens de entrada crecieron 38 → 110 → 178 → 347 en 4 turnos.

**Dificultades e intentos de solución:**
- La biblioteca `ollama` no está instalada en el Python global; el script se ejecuta con el Python del venv de BatLLM (activándolo con `Activate.ps1`).
- La ruta `Taller\Script Ollama` tiene un espacio, así que hay que escribirla entre comillas en la terminal.

**Pendiente / dudas para la clase:**
- Repetir las pruebas con `smollm2` y `qwen2.5:7b`, con mensajes idénticos en ambos modos.
- (Opcional) Hacer lo mismo con modelito.
- ¿La mención a la fecha de corte viene de la plantilla o del entrenamiento del modelo?

## 2026-10-24 — 9:00–10:30 (1:30hs)

**Tarea(s):** -

**Qué hice:**
- Actualicé la versión de modelito a la 1.4.6.
- Leí el paper

**Notas:**  
- BatLLM sigue usando modelito 1.4.5

## 2026-10-24 — 19:00–23:30 (4:30hs)

**Tarea(s):** -

**Qué hice:**
- Jugué BatLLM
- Documenté las partidas
- Recorrido del código
