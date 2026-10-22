# Bitácora de trabajo

Una entrada por sesión de trabajo, de la más antigua a la más reciente.

**Total acumulado:** _0 h 00 min_ (actualizar en cada entrada)

---

<!--
PLANTILLA — copiar debajo para cada nueva sesión

## AAAA-MM-DD — HH:MM–HH:MM (X h YY min)

**Participantes:**
**Tarea(s):** 0 / 1 / 2 / 3 / 4 / 5

**Qué hice:**
-

**Resultados / evidencia:** (enlaces a experimentos, salidas, sesiones, commits)
-

**Dificultades e intentos de solución:**
-

**Pendiente / dudas para la clase:**
-
-->

## 2026-10-21 — HH:MM–HH:MM (completar)

**Participantes:** _(completar)_
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
- Tarea 0: script de consultas con y sin historial.
- Definir `BATLLM_HOME` antes de abrir la app para que la configuración no se escriba en `src/configs/config.yaml`.
