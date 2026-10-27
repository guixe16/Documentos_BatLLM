# Experimento: primera partida de BatLLM

- **Fecha / horario:** 2026-10-26, partida entre 22:22 y 22:26 (ronda 1: 22:24:11–22:24:57; ronda 2: 22:25:52–22:26:33)
- **Tareas:** 1 (jugar y explorar) y 4 (primeros experimentos)
- **Sesión guardada:** [`../sesiones/primera_prueba-2026-10-26-22-26-57.json`](../sesiones/primera_prueba-2026-10-26-22-26-57.json)
- **Herramientas de análisis:** Game Analyzer de BatLLM; reconstrucción de los turnos con `replay_engine.replay_turn` (realizada con asistencia de IA).

## Condiciones

| Campo | Valor |
| --- | --- |
| Versión del código | BatLLM commit `d86ae05` (versión de la app: 0.3.6) |
| Biblioteca de conexión | modelito 1.4.6 |
| Modelo | `llama3.2:3b` (Ollama 0.40.8, local) |
| Temperatura | No configurada (valor por defecto del modelo) |
| `num_ctx` | 4096 |
| Contextos | Independientes (cada bot con su propio historial) |
| Prompt augmentation | Activada (el modelo recibe el estado del juego en cada consulta) |
| Reglas | 2 rondas de 8 turnos; vida inicial 30; daño por bala 5; escudo inicial levantado; tamaño del escudo 70° |
| Estado inicial | Bot 1 en (0.16, 0.57), rotación 245°; Bot 2 en (0.92, 0.40), rotación 56°; ambos con escudo levantado y 30 de vida |
| Prompts | Ambos jugadores usaron el mismo prompt en cada ronda |

**Prompt de la ronda 1:** *"Responde sólamente con comandos para poder controlar al robot. Ataca y defiéndete de la manera más óptima."*

**Prompt de la ronda 2:** *"Acercate lo más que puedas y concentrate en atacar."*

## Hipótesis / comportamiento esperado

<!-- Completar: qué esperaba que hicieran los bots con cada prompt, escrito antes de jugar. -->

## Resumen general

- 16 turnos, 32 jugadas (2 por turno).
- Ningún bot recibió daño: ambos terminaron con 30 de vida y no hubo ganador (`winner: null`).
- La reconstrucción de los 16 turnos con las reglas guardadas coincide con el estado registrado: **0 de 16 turnos con diferencias**.

## Observación 1: el modelo respondió siempre con comandos válidos

**Observado:**
- Al indicarle en el prompt que respondiera solamente con comandos, el modelo lo hizo: las 16 respuestas de la ronda 1 fueron comandos válidos, sin texto adicional.
- En la ronda 2, cuyo prompt no lo pedía, las 16 respuestas también fueron comandos válidos. En total, **32 de 32**; el Analyzer no marca ningún error (`[ERR]`).
- El código normaliza algunas respuestas antes de ejecutarlas: `A90` → `A90.0`, `C155` → `C155.0`, `A6` → `A6.0`.

**Interpretación provisional:**
- El modelo respetó el formato de comandos. Además del prompt del jugador, las instrucciones de sistema de BatLLM (`src/assets/system_instructions/augmented_independent_1.txt`) ya le exigen responder solo con un comando ("Your output needs to strictly be a command"). Como en la ronda 2 también respondió solo comandos sin que el prompt lo pidiera, con esta partida no se puede atribuir el resultado únicamente a la instrucción del jugador.

## Observación 2: disparos bloqueados por el propio escudo

**Observado:** en 4 jugadas, el modelo respondió `B` (disparar) mientras el bot tenía el escudo levantado. El juego no disparó y registró *"Shot blocked because shield is ON"*.

| Ronda / turno | Bot | Respuesta | Comando | Resultado |
| --- | --- | --- | --- | --- |
| 1 / 1 | 1 | `B` | `B` | Disparo bloqueado (escudo levantado) |
| 1 / 3 | 1 | `B` | `B` | Disparo bloqueado (escudo levantado) |
| 1 / 5 | 2 | `B` | `B` | Disparo bloqueado (escudo levantado) |
| 1 / 8 | 2 | `B` | `B` | Disparo bloqueado (escudo levantado) |

**Interpretación provisional:**
- Con la *prompt augmentation* activada, el modelo recibe el estado del escudo en cada consulta, y las instrucciones de sistema explican que solo se puede disparar con el escudo bajo. Aun así, eligió disparar con el escudo levantado.
- El comando era válido en formato, pero no tenía efecto en ese estado: la validez sintáctica no garantiza una acción útil.

**Recorrido de una acción (ronda 1, turno 1, Bot 1):**
- **¿Qué pudo decidir el jugador?** La instrucción general: "Ataca y defiéndete de la manera más óptima".
- **¿Qué decidió el modelo?** Disparar (`B`), sin bajar antes el escudo.
- **¿Qué decidió el código?** Aceptar `B` como comando válido, verificar el estado del escudo y bloquear el disparo.

## Observación 3: "Acercate" hizo que los bots se alejaran (ronda 2)

**Observado:**
- Con el prompt *"Acercate lo más que puedas y concentrate en atacar"*, ambos bots respondieron `M` (avanzar) en los primeros turnos **sin girar antes hacia el enemigo**.
- **Bot 1**, orientado a ~149°, avanzó hacia la izquierda (x: 0.13 → 0.05 en los turnos 1 a 4), **alejándose** del Bot 2, que estaba a la derecha.
- **Bot 2**, orientado a ~0°, avanzó hacia la derecha hasta el borde (x = 0.95) y **quedó trabado contra el borde**: en los turnos 2 a 5 respondió `M` y su posición no cambió.
- Ningún bot llegó a acercarse al otro durante la ronda.

| Turno | Bot 1 (respuesta → posición x) | Bot 2 (respuesta → posición x) |
| --- | --- | --- |
| 1 | `M` → 0.13 a 0.11 | `M` → 0.92 a 0.95 |
| 2 | `M` → 0.11 a 0.08 | `M` → 0.95 a 0.95 (borde) |
| 3 | `M` → 0.08 a 0.05 | `M` → sin cambio (borde) |
| 4 | `M` → 0.05 a 0.05 (borde) | `M` → sin cambio (borde) |
| 5 | `A3` → gira a 146° | `M` → sin cambio (borde) |

**Interpretación provisional:**
- El modelo tradujo "acercate" directamente a "avanzar" (`M`), sin tener en cuenta hacia dónde apuntaba el bot. El comando `M` mueve en la dirección actual, así que el resultado depende de una orientación que el modelo no consideró.
- El resultado contradice la expectativa del prompt: es un caso donde la instrucción del jugador, la decisión del modelo y la ejecución del código no coinciden.

**Recorrido de una acción (ronda 2, turno 1, Bot 1):**
- **¿Qué pudo decidir el jugador?** El objetivo: "Acercate lo más que puedas".
- **¿Qué decidió el modelo?** Avanzar (`M`), sin girar hacia el enemigo.
- **¿Qué decidió el código?** Mover al bot en su dirección actual (~149°) una distancia fija, y limitar la posición al borde del tablero en los turnos siguientes.

## Discrepancias y limitaciones

- Los dos jugadores usaron el mismo prompt en cada ronda; las diferencias entre bots vienen del estado y de las respuestas del modelo, no de las instrucciones.
- Una sola partida con un solo modelo; la temperatura no estaba fijada, por lo que repetir la partida podría dar respuestas distintas.
- La sesión guardada registra el prompt del jugador y la respuesta del modelo, pero no la consulta completa que recibió el modelo (instrucciones de sistema, bloque `[GAME_STATE]` e historial).

## Variante: instrucción explícita de girar hacia el enemigo

- **Fecha / horario:** 2026-10-26, partida entre 22:44 y 22:50 (ronda 1: 22:49:05–22:49:43; ronda 2: 22:49:58–22:50:40)
- **Sesión guardada:** [`../sesiones/Segunda_partida-2026-10-26-22-51-15.json`](../sesiones/Segunda_partida-2026-10-26-22-51-15.json)
- **Condición cambiada:** el prompt de la ronda 2. Se conservó el prompt original completo y se agregó una instrucción al principio:
  - Original: *"Acercate lo más que puedas y concentrate en atacar."*
  - Variante: *"Primero girá hasta quedar mirando al enemigo. Después acercate lo más que puedas y concentrate en atacar."*
- **Constante:** mismo commit, modelo y configuración; el prompt de la ronda 1 se repitió sin cambios; ambos jugadores usaron el mismo prompt.
- **No controlado (azar del juego):**
  - La posición y la orientación inicial de los bots son aleatorias (`src/game/bot.py`, líneas 95–98). Estado inicial de esta partida: Bot 1 en (0.57, 0.56), rotación 265.5°; Bot 2 en (0.47, 0.47), rotación 150.3°.
  - El orden en que juegan los bots se sortea al comenzar cada ronda y se mantiene en todos sus turnos (`src/game/game_board.py`, líneas 460–461).
  - La temperatura no estaba fijada.
- **Pregunta:** ¿una instrucción que menciona explícitamente la orientación logra que los bots se acerquen?

### Comparación de la ronda 2

El "desvío" es el ángulo entre la orientación del bot y la dirección hacia el enemigo: 0° significa mirándolo de frente y 180°, de espaldas. Calculado a partir de las posiciones y rotaciones guardadas.

| | Partida 1 (original) | Partida 2 (variante) |
| --- | --- | --- |
| Distancia entre bots al empezar la ronda 2 | **0.81** | **0.14** |
| Primeras respuestas | Turnos 1 a 4: `M` en ambos bots | Turno 1: Bot 2 `A` (**inválido**), Bot 1 `C0` (giro de 0°). Turno 2: Bot 2 `A180`, Bot 1 `A60` |
| Desvío respecto del enemigo después del turno 2 | Bot 1: ~163°; Bot 2: ~165° | Bot 1: **~17°**; Bot 2: **~18°** |
| Evolución de la distancia | Aumentó: 0.81 → 0.94 | Disminuyó: 0.14 → 0.08 (turno 3) → 0.05 (turno 5); después volvió a 0.11 |
| Daño | Ninguno | Turno 4: ambos dispararon y ambos acertaron (−5 de vida cada uno) |
| Final de la ronda | Bot 2 trabado contra el borde | Desde el turno 5 ambos giraron `A180` y se alejaron |

**Observado:**
- Con la instrucción explícita, en el turno 2 ambos bots giraron hasta quedar casi de frente al enemigo (desvío de ~17° y ~18°), en el turno 3 avanzaron y se acercaron, y en el turno 4 se dispararon y ambos acertaron.
- El comportamiento no se mantuvo: desde el turno 5 los bots giraron 180° y se alejaron.
- En el turno 1, el Bot 2 respondió `A` sin ángulo: el código lo marcó como inválido (`ERR`) y el bot no actuó. Es el único comando inválido de las dos partidas (**31 de 32** válidos en esta partida).
- En el turno 1, el Bot 1 respondió `C0`: un comando válido que no produce ningún giro.

**Interpretación provisional:**
- El resultado es compatible con que la instrucción explícita ayude a que el modelo gire antes de avanzar.
- **No se puede atribuir la diferencia solo al prompt:** en la variante, los bots empezaron la ronda 2 mucho más cerca (0.14 contra 0.81), lo que facilita acercarse y acertar. Además, hay una sola partida de cada tipo y la temperatura no estaba fijada.
- Que el desvío haya quedado en ~17° puede deberse a que el modelo usó las coordenadas del bloque `[GAME_STATE]` o a una coincidencia; con estos datos no se puede distinguir.
- `A` sin ángulo y `C0` sugieren que "girá", sin indicar cuánto, es una instrucción que el modelo no siempre sabe traducir a un comando con un valor útil.

### Repetición de la ronda 1 (mismo prompt en ambas partidas)

- La observación 2 **no se repitió**: en la partida 2 ambos bots bajaron el escudo en el primer turno (`S0`) y después dispararon; no hubo disparos bloqueados.
- Hubo daño: en el turno 5, un disparo del Bot 2 alcanzó al Bot 1 (−5 de vida).
- Las 16 respuestas fueron comandos válidos, como en la partida 1.
- Con el mismo prompt, el modelo siguió estrategias distintas en cada partida: es un ejemplo de la variabilidad de las respuestas.

### Otras observaciones

- En ambas partidas aparece repetidamente el giro `A180`, a veces dejando al bot de espaldas al enemigo.
- La reconstrucción de los 16 turnos de la partida 2 coincide con el estado registrado (0 diferencias), igual que en la partida 1.
- **Convención de ángulos (verificada en el código):** en pantalla, el eje *y* crece hacia abajo (`src/view/normalized_canvas.py` invierte el eje). Por eso 0° apunta a la derecha, 90° hacia abajo, 180° a la izquierda y 270° hacia arriba. `C` suma grados (giro horario en pantalla) y `A` resta (antihorario).

## Pendiente / preguntas

- [ ] Completar la hipótesis previa.
- [ ] Repetir la variante 1 o 2 veces más, idealmente en partidas donde los bots empiecen lejos, para ver si el giro hacia el enemigo se mantiene.
- [x] ~~Investigar por qué en la ronda 1 juega primero el Bot 1 y en la ronda 2 el Bot 2.~~ El orden se sortea al comenzar cada ronda (`src/game/game_board.py`, líneas 460–461).
- [ ] Probar un prompt que indique cómo calcular el giro (por ejemplo, usando las coordenadas del estado) y ver si mejora la precisión.
