# Recorrido de código (tarea 2)

**Recorrido elegido:** desde HistoryManager hasta el Game Analyzer

- [ ] Prompt: `HomeScreen` → `OllamaConnector`
- [ ] Respuesta: `parse_model_response` → cambio de estado
- [x] Acción: `HistoryManager` → Game Analyzer

**Coordinado con:** _(quién cubre los otros recorridos)_
**Versión del código:** commit `d86ae05`

## Pasos del recorrido

| # | Archivo:línea | Función / método | Recibe | Transforma | Devuelve / produce | Luego |
| --- | --- | --- | --- | --- | --- | --- |
| 1 |src/game/bot.py:355 | process_llm_response | El texto crudo del modelo | El crudo en comando | El comando o ERR si no es válido| Ejecuta el comando, lo muestra en el panel de commands y termina el turno|
| 2 | src/game/game_board.py:543 |on_bot_llm_interaction_complete | La info de un bot | - | Registra la acción del bot en el historial | Si ya jugaron los dos bots, cierra el turno (end_turn) y agenda el siguiente |
| 3 | src/game/history_manager.py:347 | record_play | El bot (con su última respuesta y su último comando) | Arma el registro de la jugada | Agrega {bot_id, llm_response, cmd} a la lista plays del turno actual | Queda en memoria (self.games) hasta guardar la sesión |
| 4 | src/game/history_manager.py:366 | end_turn | El tablero de juego | Toma una foto del estado de todos los bots (_get_bots_state, línea 414) | Guarda post_state y end_time en el turno | El turno queda completo. Antes, start_turn (línea 315) había guardado pre_state y start_round (línea 178) las reglas congeladas y los prompts |
| 5 | src/view/home_screen.py:547 | _save_session_file | El nombre de archivo elegido en Save Session | Arma la ruta en la carpeta de sesiones (usa BATLLM_HOME) y controla que no se salga de esa carpeta | La ruta destino | Llama a save_session del HistoryManager (línea 587) |
| 6 | src/game/history_manager.py:485 | save_session | La ruta destino y todas las partidas en memoria | Filtra los turnos completos, arma el sobre v2 (build_session_payload, session_schema.py:23) y lo valida (validate_session_payload, session_schema.py:48) | El archivo JSON, escrito de forma atómica (temporal + renombrar) | La sesión queda en saved_sessions |
| 7 | src/view/analyzer_load_screen.py:153 | open_selected_session | La ruta del archivo elegido en el Analyzer | Lee el JSON y lo vuelve a validar (load_session_payload, session_schema.py:130) | Un AnalyzerSessionModel con la sesión | Lo pasa a la pantalla de revisión (load_session) |
| 8 | src/analyzer_model.py:129 | round_steps | Una ronda de la sesión guardada | Por cada turno llama a replay_turn y controla que el post_state de un turno coincida con el pre_state del siguiente | La lista de pasos (Turn start + una jugada por bot) con marcas [DMG], [ERR], [SHD] y [!] | La pantalla los muestra en el Session Tree |
| 9 | src/game/replay_engine.py:649 | replay_turn | pre_state, plays, reglas y el post_state guardado | Vuelve a parsear cada respuesta y compara con el cmd guardado (grounding); reaplica las jugadas en orden con apply_play (línea 461) y compara con el post_state (compare_state_maps, línea 623) | Un TurnReplay: estado reconstruido, eventos y si hay diferencias (mismatch) | Si algo no coincide, round_steps marca el turno con [!] |
| 10 | src/view/analyzer_review_screen.py:194 | _refresh_view | El modelo y el paso seleccionado | Formatea los textos (format_plays, analyzer_model.py:247; format_state_diff, analyzer_model.py:265) | El texto de las pestañas Plays y State Diff, y el tablero (set_replay_state, analyzer_board.py:38) | El usuario ve, por ejemplo, Bot 2 llm='A' cmd=ERR |

## Ejemplo concreto de ejecución

**Jugada:** sesión [`Segunda_partida-2026-10-26-22-51-15.json`](sesiones/Segunda_partida-2026-10-26-22-51-15.json), ronda 2, turno 1, **Bot 2**. El modelo respondió `A` (giro antihorario sin ángulo) y el juego lo marcó como `ERR`. Experimento: [2026-10-26-primera-partida.md](experimentos/2026-10-26-primera-partida.md), sección "Variante".

*Redactado con asistencia de IA (Claude) a partir del recorrido del código.*

### Parte 1: registrar la acción durante el juego

**1. `Bot.process_llm_response`** — `src/game/bot.py:355`

- **Recibe:** el texto crudo del modelo, `"A"`.
- **Hace:**
  - guarda `last_llm_response = "A"` (línea 363);
  - llama a `parse_model_response("A")` (línea 364, definida en `src/game/replay_engine.py:283`). La `A` indica un giro antihorario, pero no hay número, así que devuelve `normalized_cmd = "ERR"` y `valid = False`;
  - guarda `last_cmd = "ERR"` (línea 365). Como no es válido, **no ejecuta nada** y el estado del bot no cambia;
  - muestra `ERR` en rojo en el panel Commands.
- **Avisa:** llama a `board_widget.on_bot_llm_interaction_complete(self)`.

**2. `GameBoard.on_bot_llm_interaction_complete`** — `src/game/game_board.py:543`

- Llama a `history_manager.record_play(bot)` (línea 549).
- Si ya jugaron los dos bots, llama a `history_manager.end_turn(self)` (línea 555) y agenda el turno siguiente.

**3. `HistoryManager.record_play`** — `src/game/history_manager.py:347`

- **Recibe:** el objeto `Bot`.
- **Produce:** agrega a la lista `plays` del turno actual:

  ```json
  {"bot_id": 2, "llm_response": "A", "cmd": "ERR"}
  ```

- Las jugadas quedan **en el orden en que se ejecutaron** (el orden sorteado de la ronda). Ese orden importa después para la reconstrucción.

**4. El resto del ciclo de vida** — mismo archivo

- `start_round` (línea 178): guarda `initial_state`, las reglas congeladas (`gameplay_settings_snapshot`, línea 214) y los prompts.
- `start_turn` (línea 315): crea el turno y guarda `pre_state`, una foto de todos los bots.
- `end_turn` (línea 366): guarda `post_state` y `end_time`.
- `_get_bots_state` (línea 414): arma esas fotos con `id`, `health`, `x`, `y`, `rot`, `shield`, `current_prompt` y `last_llm_response`.

Todo se guarda **en memoria**, en `self.games`, hasta que el jugador guarda la sesión.

### Parte 2: guardar la sesión

**5. Botón Save Session** — `src/view/home_screen.py:547` (`_save_session_file`)

- Resuelve la carpeta con `resolve_saved_sessions_dir`, que usa `BATLLM_HOME`, y comprueba que el nombre no "escape" de esa carpeta.
- Llama a `history_manager.save_session(target_path)` (línea 587).

**6. `HistoryManager.save_session`** — `src/game/history_manager.py:485`

- **Filtra** (líneas 492–511): exporta solo los turnos completos, con `end_time`, `plays` y `post_state`. Un turno a medio jugar no se guarda.
- **Arma el sobre** (línea 516): `build_session_payload` (`src/game/session_schema.py:23`) agrega `schema_version: 2`, `session_type`, `app_version`, `saved_at` y `llm_metadata`, que incluye el modelo y `num_ctx`.
- **Valida** (línea 522): `validate_session_payload` (`session_schema.py:48`) controla la estructura antes de escribir.
- **Escribe de forma atómica** (líneas 526–532): primero escribe un archivo temporal y después lo renombra con `os.replace`. Si algo falla a mitad de camino, no queda un JSON corrupto.

### Parte 3: el Game Analyzer

**7. Abrir la sesión** — `src/view/analyzer_load_screen.py:153` (`open_selected_session`)

- `load_session_payload` (`session_schema.py:130`) lee el JSON y lo vuelve a validar. Si el formato no es v2, muestra un error de compatibilidad.
- Crea un `AnalyzerSessionModel(payload, path)` y se lo pasa a la pantalla de revisión (línea 169).

**8. `AnalyzerSessionModel.round_steps`** — `src/analyzer_model.py:129`

- Para cada turno de la ronda llama a `replay_turn(pre_state, plays, reglas, saved_post_state=post_state)` (línea 140).
- **Comprueba la continuidad** (línea 146): el `post_state` de un turno tiene que coincidir con el `pre_state` del siguiente.
- **Arma los pasos:** "Turn start" más una entrada por jugada, con sus marcas `[DMG]`, `[ERR]`, `[SHD]` y `[!]`.

**9. `replay_turn`** — `src/game/replay_engine.py:649`

Acá ocurre la verificación. Hace dos controles distintos:

- **Grounding** (líneas 662–663): vuelve a parsear la respuesta guardada (`"A"`) con el mismo `parse_model_response` del juego en vivo y comprueba que dé el comando guardado (`"ERR"`). Si no coincide, marca *"response-command disagreement"*.
- **Reproducción** (líneas 664–673 y 675): parte del `pre_state`, aplica cada jugada en orden con `apply_play` (línea 461) y compara el estado obtenido con el `post_state` guardado usando `compare_state_maps` (línea 623). Las posiciones y rotaciones se comparan con una tolerancia de 0.000001; la vida y el escudo, exactos.

> En el ejemplo, `"A"` vuelve a dar `ERR`, igual que lo guardado, y `apply_play` no cambia nada, igual que el estado guardado. Por eso el turno sale con `[ERR]` pero sin `[!]`. Son los dos predicados del paper: *grounding consistency* y *operative replay*.

**10. Mostrarlo en pantalla** — `src/view/analyzer_review_screen.py:194` (`_refresh_view`)

- **Pestaña Plays:** `model.format_plays()` (`analyzer_model.py:247`) arma la línea `Bot 2 llm='A' cmd=ERR` (línea 218 de la pantalla).
- **State Diff:** `format_state_diff()` (línea 219).
- **Tablero:** `analyzer_board.set_replay_state(...)` (línea 231) dibuja el estado **reconstruido**, no el guardado.

## Dudas para la clase

-
