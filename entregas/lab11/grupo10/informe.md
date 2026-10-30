# Informe forense — Incidente PhantomCorp (phantomcorp-web)

**Grupo:** 10  
**Investigadora:** Carolina Paula Corazza — @carocor1  
**Fecha:** Pendiente de entrega.  
**Sistema afectado:** phantomcorp-web (10.0.0.5) · **Tipo:** intrusión + exfiltración

## 0. Declaración de uso de IA

Se utilizó OpenCode como apoyo para explicar la verificación de hashes y preparar el script reproducible de cadena de custodia. La investigadora verificó los hashes contra `labs/lab11-forensia/caso/evidencia.sha256`, contrastó las referencias con los artefactos originales y revisó que el script normalice CRLF a LF solo en memoria, sin modificar la evidencia.

## 1. Resumen del incidente

Los registros muestran que una IP externa enumeró el sitio y luego obtuvo respuestas exitosas en el portal de RR. HH. y en una API que permitió ejecutar comandos. El atacante descargó y usó una webshell, elevó privilegios mediante un binario SUID y dejó indicios de persistencia en un cron y una cuenta agregada. Los artefactos muestran que se preparó un archivo con datos de clientes y se intentó enviarlo a un servidor externo, aunque no confirman que el destino lo haya recibido. La memoria capturada a las 02:30 UTC muestra una conexión C2 activa en ese momento; no permite afirmar el estado actual del sistema.

## 2. Cadena de custodia

Se comparó el SHA-256 de cada archivo listado en `evidencia.sha256` con el hash registrado al preservar la evidencia. Como el entorno Windows presenta finales de línea CRLF mientras que el repositorio conserva LF, el verificador convierte CRLF a LF solo en memoria antes de calcular el hash. No se escribieron cambios en los archivos originales del caso.

El resultado fue **8 archivos coincidentes, 1 discrepancia y 0 archivos faltantes**. `logs/auth.log` no coincide: el manifiesto registra `0000decafbad0000f1e2d3c4b5a6978800112233445566778899aabbccddeeff`, mientras que el hash calculado tras normalizar finales de línea es `7d18039d128d0df7a245ca02f469ea02cd1d4f205a24fb33c5bcf7041b66e9ab`. Los otros ocho archivos listados coinciden con sus hashes esperados.

La discrepancia de `auth.log` es consistente con las advertencias incluidas en el propio archivo (líneas 1 y 5–7), pero el hash por sí solo no establece quién lo modificó ni cuándo. Por cadena de custodia, se conserva el archivo tal como fue recibido, se documenta la discrepancia y se trata su contenido con cautela; no se intenta repararlo ni reemplazarlo.

La ejecución reproducible está en `verificar_custodia.py` y su salida completa en `verificacion-custodia.txt`. El reporte registra hashes esperados y calculados para cada archivo. El verificador termina con código 1 cuando encuentra una discrepancia o un archivo faltante; para esta ejecución eso señala el `MISMATCH` esperado de `logs/auth.log`, no una falla al calcular los hashes.

## 3. Línea de tiempo del ataque

_Pendiente de revisión y redacción por Belén._

## 4. Indicadores de compromiso (IOCs)

| Tipo | Valor | Evidencia y límite |
|---|---|---|
| IP atacante | `185.220.101.42` | Aparece en el sondeo, enumeración y solicitudes del incidente (`labs/lab11-forensia/caso/logs/access.log:6-22`). Es el origen observado en esos registros; no atribuye identidad o actor. |
| C2 saliente | `45.13.37.7:4444` | La captura de memoria muestra `nc` con proceso PID 1490 y conexión `ESTABLISHED` al puerto 4444 (`labs/lab11-forensia/caso/memoria/pslist.txt:9`; `labs/lab11-forensia/caso/memoria/netstat.txt:4`). Describe el estado en la captura del 05/09/2026 a las 02:30 UTC, no el estado actual del host. |
| Webshell | `/var/www/uploads/shell.php` | El log registra el comando de descarga y luego una solicitud a la ruta (`labs/lab11-forensia/caso/logs/access.log:20-21`); el listado del disco ubica el archivo (`labs/lab11-forensia/caso/disco/listado.txt:4`) y la evidencia lo describe defangeado (`labs/lab11-forensia/caso/disco/var/www/uploads/shell.php.evidencia.txt:1-10`). No reconstruir ni ejecutar la muestra neutralizada. |
| Usuario backdoor | `phantom-svc` | Figura como usuario agregado con UID/GID 0 (`labs/lab11-forensia/caso/disco/etc/passwd.snippet:3-4`). El caso no aporta evidencia de un inicio de sesión con esa cuenta ni de que haya sido utilizada. |
| Persistencia por cron | `/etc/cron.d/phantom-maint` | El listado registra el archivo (`labs/lab11-forensia/caso/disco/listado.txt:6`) y el comentario de la webshell indica relanzamiento cada cinco minutos (`labs/lab11-forensia/caso/disco/var/www/uploads/shell.php.evidencia.txt:9`). El contenido del cron no está incluido, así que no se afirma su línea de ejecución exacta. |

## 5. Alcance del daño

_Pendiente de revisión y redacción por Juan._

## 6. Contención y erradicación (recomendaciones)

_Pendiente de revisión y redacción por Juan._

## 7. Preguntas de análisis

### P2 — Integridad de la evidencia

Comparé el SHA-256 de `logs/auth.log` con el valor registrado para ese archivo en `evidencia.sha256`. El hash calculado, `7d18039d128d0df7a245ca02f469ea02cd1d4f205a24fb33c5bcf7041b66e9ab`, no coincide con el valor del manifiesto (`0000decafbad0000f1e2d3c4b5a6978800112233445566778899aabbccddeeff`). Esa discrepancia indica que el archivo recibido no coincide con la versión cuya huella figura en la custodia. Por eso conservé el archivo sin modificarlo y documenté la diferencia. Si no se registra, no es posible confiar en su integridad ni distinguir con claridad qué contenido pertenece a la evidencia preservada. El hash detecta una diferencia, pero no identifica por sí solo quién la causó ni cuándo. La comprobación está documentada en `verificacion-custodia.txt`.

### P3 — Correlación de artefactos

La descarga de la webshell puede correlacionarse en tres artefactos dentro de una ventana cercana a las 02:19 del 05/09/2026. `labs/lab11-forensia/caso/logs/access.log:20` registra a las 02:19:02 UTC una solicitud que ejecuta `wget` desde `45.13.37.7` para guardar `s.php` como `/var/www/uploads/shell.php`. `labs/lab11-forensia/caso/disco/listado.txt:4` ubica ese archivo con hora 02:19, con precisión de minuto y sin zona horaria indicada. `labs/lab11-forensia/caso/memoria/bash_history.txt:4` conserva el mismo comando y destino, pero no asigna hora individual a los comandos. En conjunto respaldan el despliegue de la webshell alrededor de ese momento; las marcas no son idénticas y no permiten fijar un segundo exacto para la creación del archivo o la ejecución del comando.

### P4 — Recuperación del historial

El comando `history -c` borra la lista del historial de comandos de la sesión de Bash, pero no elimina automáticamente otras copias que ya existan. En este caso, el encabezado de `memoria/bash_history.txt:1` indica que el historial fue recuperado desde memoria, aunque el atacante ejecutó `history -c` (`memoria/bash_history.txt:10`). Por eso se pudieron recuperar comandos como la descarga del webshell y la preparación del archivo para exfiltración (`memoria/bash_history.txt:4,8-9`). La evidencia demuestra que esos comandos estaban en el historial recuperado de memoria; no permite afirmar más sobre el mecanismo exacto de recuperación.
