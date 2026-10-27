# Experimento: conversación con y sin historial (tarea 0)

- **Fecha / horario:** 2026-10-22, corridas entre 16:39 y 16:41
- **Tarea:** 0 — Prueba de concepto con Ollama
- **Herramienta:** `Taller/Script Ollama/scriptOllama.py` (script de consola; modo 1 = sin memoria, modo 2 = con memoria). El script fue generado con asistencia de IA (Claude); la ejecución de las pruebas es propia.

## Condiciones

| Campo | Valor |
| --- | --- |
| Versión del código | Script propio (no BatLLM). Biblioteca `ollama` 0.6.3 (venv de BatLLM) |
| Servidor | Ollama 0.40.8, local (`localhost:11434`) |
| Modelo | `llama3.2:3b` |
| Temperatura / otras opciones | `temperature: 0`; resto por defecto |
| Mensaje `system` | Ninguno enviado por el script (la plantilla del modelo agrega uno propio, ver más abajo) |
| Condición cambiada | Si se reenvía o no el historial en cada consulta |

## Hipótesis / comportamiento esperado

- **Sin memoria:** el modelo no puede saber un dato dado en un mensaje anterior, porque cada consulta contiene solo el último mensaje.
- **Con memoria:** el modelo puede responder usando datos de mensajes anteriores, porque se reenvían en cada consulta. Los tokens de entrada deberían crecer turno a turno.

## Prueba A — sin memoria

Salida cruda: [`../salidas/tarea0_2026-10-22_16-39-51_sin-memoria_llama3.2-3b.json`](../salidas/tarea0_2026-10-22_16-39-51_sin-memoria_llama3.2-3b.json)

| Turno | Mensajes enviados | Respuesta (resumen) | Tokens entrada / salida | Duración |
| --- | --- | --- | --- | --- |
| 1 | 1 — `Hola, mi nombre es Guillermo. ¿Cómo estás?` | Saluda por el nombre; aclara que es una IA sin emociones. | 38 / 52 | 4,83 s |
| 2 | 1 — `¿Podrías decirme cuál es mi nombre?` | "No tengo acceso a información personal sobre ti… no tengo la capacidad de recordar…" | 37 / 87 | 1,08 s |

## Prueba B — con memoria

Salida cruda: [`../salidas/tarea0_2026-10-22_16-41-52_con-memoria_llama3.2-3b.json`](../salidas/tarea0_2026-10-22_16-41-52_con-memoria_llama3.2-3b.json)

| Turno | Mensajes enviados | Respuesta (resumen) | Tokens entrada / salida | Duración |
| --- | --- | --- | --- | --- |
| 1 | 1 — `Hola. Mi nombre es Guillermo. ¿Cómo estás?` | Saluda por el nombre; aclara que es una IA sin emociones. | 38 / 51 | 0,85 s |
| 2 | 3 — historial + `¿Podrías decirme cuál es mi nombre?` | "Disculpa la confusión anterior, Guillermo. Tu nombre es Guillermo, que me dijiste al principio." | 110 / 48 | 0,66 s |
| 3 | 5 — historial + `¿Puedes decirme qué día es hoy?` | No tiene acceso a la fecha actual; "mi conocimiento se basa en la fecha de corte… diciembre de 2023"; sugiere alternativas. | 178 / 141 | 1,55 s |
| 4 | 7 — historial + `¿Podrías decirme cuál es mi nombre y qué estoy interesado en saber?` | Responde el nombre y que el interés es "saber cuál es el día actual". | 347 / 101 | 1,25 s |

## Observado

1. **Sin memoria**, el modelo no supo el nombre dado en el turno anterior. No inventó un nombre: respondió que no tiene acceso a esa información.
2. **Con memoria**, respondió correctamente el nombre (turno 2) y, en el turno 4, infirió el "interés" del usuario a partir de la pregunta del turno 3.
3. **Los tokens de entrada crecen con el historial:** 38 → 110 → 178 → 347. Cada consulta incluye todos los mensajes anteriores, incluidas las respuestas del modelo. Por ejemplo, la entrada del turno 4 (347) es aproximadamente la entrada del turno 3 (178) + la respuesta del turno 3 (141) + el nuevo mensaje y sus marcas de plantilla (~28).
4. En el turno 2 de la prueba B, el modelo empezó con **"Disculpa la confusión anterior"**, aunque en el historial no hubo ninguna confusión.
5. En el turno 3, el modelo mencionó una **fecha de corte de diciembre de 2023**. La plantilla de chat de `llama3.2:3b` (`ollama show llama3.2:3b --template`) agrega siempre un bloque `system` con `Cutting Knowledge Date: December 2023`, aunque el script no envíe ningún mensaje `system`.
6. La primera consulta de la sesión tardó 4,83 s; las siguientes, entre 0,66 y 1,55 s.

## Interpretación provisional

- El modelo no guarda estado entre consultas: la "memoria" de una conversación depende únicamente de que el cliente reenvíe el historial. (Obs. 1 y 2)
- El costo de la memoria crece con cada turno, porque se reenvían también las respuestas del modelo; en una conversación larga se terminaría alcanzando la ventana de contexto. (Obs. 3)
- La "confusión anterior" parece una fórmula aprendida en el entrenamiento, no algo presente en la entrada. (Obs. 4)
- La fecha de corte que menciona el modelo podría venir del bloque que agrega la plantilla, de su entrenamiento, o de ambos; con esta prueba no se puede distinguir. (Obs. 5)
- La demora de la primera consulta probablemente corresponde a la carga del modelo en la GPU. (Obs. 6)

## Discrepancias y limitaciones

- El mensaje del turno 1 **no fue idéntico** en las dos pruebas: `Hola, mi nombre es…` (A) vs. `Hola. Mi nombre es…` (B). Ambos ocupan 38 tokens y la diferencia no parece afectar el resultado, pero no se cumplió del todo "cambiar una sola condición".
- Solo se probó un modelo (`llama3.2:3b`), con una corrida por modo.
- La prueba A tuvo 2 turnos y la B, 4.

## Pendiente

- [ ] Repetir A y B con `smollm2` y `qwen2.5:7b`, con mensajes idénticos.
- [ ] Repetir cada caso al menos 2 veces para ver si los resultados se mantienen.
- [ ] Probar con un mensaje `system` propio y ver si cambia la mención a la fecha de corte.
- [ ] (Opcional) Repetir con modelito en lugar de la biblioteca `ollama`.
