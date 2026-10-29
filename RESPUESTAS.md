# Respuestas · Laboratorio 4
Alumno: 20210888
Presupuesto: 13 min  Métrica DORA: change failure rate

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: smoke, originalmente en la etapa deploy.
- Problema que causa en el pipeline: smoke dependía de build-push, igual que deploy. Por ello, ambos podían ejecutarse en paralelo y smoke podía validar la versión anterior de Lambda antes de que terminara el despliegue de la nueva imagen.
- Link al commit: https://github.com/Kto-PAPS/INF384-lab4-dev/commit/5820c2d44cca76ebc3c656a39251be19e31dc6e3
- Corrección: mover de etapa. Se cambió needs: build-push por needs: deploy, ubicando smoke en post-deploy.
- Evidencia: En el run inicial del 29/10, smoke y deploy comenzaron a las 21:24:06 UTC, lo que confirma que no existía una dependencia entre ambos. Run: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40957257154
### Defecto 2
- Job y etapa: load, etapa post-deploy.
- Problema que causa en el pipeline: La prueba de carga aceptaba respuestas HTTP 200 y 400 como correctas. Esto podía ocultar transferencias rechazadas por saldo insuficiente y presentar una tasa de errores engañosa.
- Link al commit: https://github.com/Kto-PAPS/INF384-lab4-dev/commit/f3403f4961c4d02b20d76fb8420d512ccca84f9a
- Corrección: modificar tests. Se configuró k6 para aceptar únicamente HTTP 200, se actualizó el check de transferencias y se redujo el monto de prueba a S/ 0.01 para disminuir rechazos por agotamiento de saldo.
- Evidencia: En el run inicial, k6 registró 34274 checks exitosos (100 %) y 0 % de http_req_failed. Sin embargo, el check original aceptaba explícitamente respuestas 200 o 400. Esto demuestra que el criterio permitía considerar válidas transferencias rechazadas, aunque el resumen no permite determinar cuántos HTTP 400 ocurrieron. Run: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40957257154/job/123741466537
### Defecto 3
- Job y etapa: smoke, etapa post-deploy.
- Problema que causa en el pipeline: El script de smoke incluía una verificación del commit desplegado, pero el workflow no configuraba EXPECTED_COMMIT. Por ello, podía aprobar una Lambda que respondiera correctamente aunque ejecutara una versión diferente.
- Link al commit: https://github.com/Kto-PAPS/INF384-lab4-dev/commit/a4c60e970a34f344c80bc896e9f7edd4231f4e9c
- Corrección: ajustar configuración. Se agregó EXPECTED_COMMIT con github.sha al paso Smoke del workflow.
- Evidencia: En el primer run, el log de smoke mostró la respuesta de /salud con el commit 49ba49ebee84f4c9f8b6943031595970311a3bb6, pero no ejecutó la verificación opcional de EXPECTED_COMMIT. El archivo original no definía esta variable. Run: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40957257154/job/123741182030
### Defecto 4
- Job y etapa:
- Problema que causa en el pipeline:
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 5
- Job y etapa:
- Problema que causa en el pipeline:
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
- Evidencia (dato observado, con link al run o fragmento de la salida local):

## Parte 2 · Implementación
- Link al run en verde en main:
- Link al pull request:
- Defecto de la aplicación introducido (archivo y descripción):
- Job que debería detenerlo:
- Link al run fallido:

## Parte 3 · Decisión bajo restricción
Link al run medido:

| Job | Duración (min:seg) |
|---|---|
| unit | |
| integration | |
| sast | |
| sca | |
| build-push | |
| deploy | |
| smoke | |
| e2e | |
| load | |
| **Total** | |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | |
| p95 | |
| Máximo | |
| Tasa de error | |
| Requests por segundo | |

Conclusión sobre el comportamiento bajo carga:

SLO propuesto (percentil, valor, ventana) y justificación:

SLA propuesto y diferencia con el SLO:

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|

Total resultante: [ ] min
Link al run en main con los cambios:

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:

## Declaración de uso de IA
Herramientas usadas y para qué:
