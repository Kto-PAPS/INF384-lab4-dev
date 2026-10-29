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
- Job y etapa: smoke, etapa post-deploy.
- Problema que causa en el pipeline: La prueba consultaba /salud, pero no validaba el contenido de la respuesta. Un HTTP 200 con ok:false podía pasar inadvertido y permitir que continuaran las pruebas posteriores.
- Link al commit: https://github.com/Kto-PAPS/INF384-lab4-dev/commit/5d95b8c1a34d42899cd15fe2c30797089558e703
- Corrección: modificar tests. Se incorporó una validación con jq que exige ok:true y que version y commit sean cadenas de texto.
- Evidencia: En el run inicial, smoke imprimió la respuesta de /salud con ok:true, pero el script original no evaluaba ese campo. La corrección incorpora jq -e para producir un error ante una respuesta no saludable. Run original: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40957257154/job/123741182030
### Defecto 5
- Job y etapa: load, etapa post-deploy.
- Problema que causa en el pipeline: La prueba de carga completa se ejecutaba en cada pull request, aunque tiene un costo elevado y no es necesaria para obtener retroalimentación rápida de cada cambio. Esto retrasa la validación de los PR y consume recursos de manera innecesaria.
- Link al commit: https://github.com/Kto-PAPS/INF384-lab4-dev/commit/d3fb7a633a8ad01415299f200820c6d5db827632
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración. Se añadió una condición if para omitir load en los pull requests, conservando su ejecución en main y en las ejecuciones manuales.
- Evidencia (dato observado, con link al run o fragmento de la salida local): En el run inicial del 29/10, el job load tardó 12 minutos y 12 segundos. El workflow original tenía el evento pull_request habilitado y no incluía ninguna condición para omitir ese job. Por tanto, el mismo control costoso se programaba también para los PR. Run: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40957257154/job/123741466537

## Parte 2 · Implementación
- Link al run en verde en main: Run verde en main, con las cinco correcciones integradas: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40964636437
- Link al pull request: https://github.com/Kto-PAPS/INF384-lab4-dev/pull/2
- Defecto de la aplicación introducido (archivo y descripción): En src/util.js se modificó la condición de validación de montos de monto <= 0 a monto < 0. Esto permite incorrectamente transferencias de monto cero. El cambio se realizó exclusivamente en la rama lab4-defecto-controlado.
- Job que debería detenerlo: unit. El job falló como se esperaba: 26 pruebas aprobadas y 2 fallidas, correspondientes al rechazo del monto cero y a la transferencia con monto inválido. Como build-push depende de unit, se impidió el despliegue de la aplicación defectuosa.
- Link al run fallido: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40965261061

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40964636437

| Job | Duración (min:seg) |
|---|---|
| unit | 0:11 |
| integration | 0:10 |
| sast | 0:19 |
| sca | 0:19 |
| build-push | 0:38 |
| deploy | 0:18 |
| smoke | 0:13 |
| e2e | 1:02 |
| load | 12:13 |
| **Total** | 15:23 |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 68.63 ms |
| p95 | 70.96 ms |
| Máximo | 1.1 s |
| Tasa de error | 0.00 % |
| Requests por segundo | 48.72866 req/s |

Conclusión sobre el comportamiento bajo carga:

La prueba de carga ejecutó 35 118 solicitudes con un promedio de 48.73 solicitudes por segundo. El p95 fue de 70.96 ms y la tasa de errores HTTP reportada fue 0 %, cumpliendo los umbrales configurados.

El comportamiento observado fue favorable bajo esta carga, aunque hubo un máximo de 1.1 segundos. Los resultados no garantizan que el sistema mantenga el mismo rendimiento con una concurrencia mayor o durante periodos más largos.

SLO propuesto (percentil, valor, ventana) y justificación:

Se propone mantener un p95 de latencia inferior a 150 ms, medido en ventanas móviles de 30 días. En el run #4 se registró un p95 de 70.96 ms, por lo que el objetivo incorpora un margen para la variación normal del rendimiento. En cada prueba de k6 se evaluará el mismo umbral sobre las solicitudes realizadas durante esa ejecución.

SLA propuesto y diferencia con el SLO:

Se propone un SLA de p95 inferior a 250 ms, evaluado mensualmente para los clientes de Monedero Andino. A diferencia del SLO, que constituye un objetivo interno de operación, el SLA representa un compromiso contractual y podría incluir compensaciones por incumplimiento.

El SLA tiene un límite menos exigente para dejar un margen operativo entre el objetivo interno y el compromiso ofrecido a los clientes. Ambos valores son propuestas; el laboratorio no demuestra todavía su cumplimiento sostenido durante un mes.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---:|
| Reducir la carga constante de 10m a 4m | load | 6:04 |

Total resultante: 9:09 min
Link al run en main con los cambios: https://github.com/Kto-PAPS/INF384-lab4-dev/actions/runs/40965633430

El pipeline pasó de 15:23 a 9:09, con un ahorro total observado de 6:14. De ese ahorro, 6:04 corresponden directamente a load y 0:10 a diferencias de duración de otros jobs entre ejecuciones.

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Se redujo la prueba de carga de 12 a 6 minutos, manteniendo 4 minutos de carga constante. El riesgo aceptado es detectar con menor probabilidad una degradación que aparezca después de varios minutos de uso sostenido. Por ejemplo, una fuga de memoria introducida en el procesamiento de POST /transferir podría provocar que las transferencias se vuelvan progresivamente más lentas, sin superar el SLO durante los seis minutos de prueba.

Si ese defecto llegara a producción y provocara un incidente que requiriera intervención, el Change Failure Rate subiría, porque aumentaría la proporción de despliegues que ocasionan fallos. No se afirma que actualmente exista esa fuga ni que el CFR haya aumentado. Se mantiene el control de carga en main, junto con las pruebas unitarias, de integración, smoke, E2E y de seguridad, para limitar el riesgo.
Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:

Se mantiene sast en pre-build y como control bloqueante para hallazgos de severidad ERROR.

En el run #4, sast tardó 19 segundos, un costo pequeño frente al presupuesto de 13 minutos. Moverlo después del despliegue o convertirlo en informativo podría permitir la publicación de código con vulnerabilidades que el análisis sí detecta.

La decisión busca contener el riesgo de incidentes posteriores al despliegue y evitar un posible aumento del Change Failure Rate. No se atribuye una reducción cuantificada de CFR porque no se dispone de datos históricos para medirla.

## Declaración de uso de IA
Herramientas usadas y para qué:
Se utilizó ChatGPT como apoyo para revisar el enunciado, analizar la configuración de GitHub Actions, identificar problemas en el pipeline, plantear correcciones y organizar las respuestas. Los cambios se aplicaron en el repositorio personal y se contrastaron con los commits, ejecuciones y logs de GitHub Actions.
