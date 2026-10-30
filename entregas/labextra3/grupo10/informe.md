# Informe — Lab Extra 3: Defendé PhantomCorp

- **Grupo:** 10
- **Integrantes:** Belén Rocío Ramello (`@belramello`), Carolina Paula Corazza (`@carocor1`), Chiara Agustina Kordek (`@chiarakordek`) y Juan Pablo Saby (`@JuanSaby`).
- **Fecha:** 30/10/2026
**Declaración de uso de IA:** completar por el grupo antes de entregar, según la consigna.

## 1. Tabla de defensas

| Defensa | Ataque que funcionaba | Fix aplicado | Verificación |
|---|---|---|---|
| D1 | La respuesta HTTP revelaba `PhantomServer/2.4.1-debug`. | El servidor ya no anuncia nombre ni versión en el encabezado `Server`. | El verificador recibió `Server: ''`; D1 quedó aprobada. |
| D2 | `GET /api/status` respondía 200 con `debug`, ruta de base de datos y PID. | Se deshabilitó el endpoint de debug; responde 404. | `GET /api/status -> 404`; D2 quedó aprobada. |
| D3 | `robots.txt` revelaba `/backup` y esa ruta devolvía información sin autorización. | Se quitó la ruta del archivo de robots y `/backup` ahora responde 404. | El verificador confirmó 404 y que `robots.txt` no nombra `/backup`; D3 quedó aprobada. |
| D4 | Datos de login concatenados al SQL permitían los dos bypass del verificador. | Se usa `?` con los valores enlazados como parámetros de SQLite. | Ambos bypass quedaron bloqueados y el login válido siguió funcionando; D4 quedó aprobada. |
| D5 | `archivo=../../../../etc/passwd` podía leerse fuera del directorio de descargas. | La ruta se canonicaliza y solo se sirve si permanece dentro de la raíz permitida y es un archivo. | El verificador devolvió 404 para la ruta externa; la descarga de `manual.txt` funcionó; D5 quedó aprobada. |

### Nota de verificación en Windows

El verificador oficial usa `/etc/passwd`, que no es un archivo de Windows. Para que la comprobación D5 no aprobara por ausencia del archivo, se preparó una fixture local controlada con el marcador `root:lab-extra-fixture` fuera de la carpeta de descargas. La copia vulnerable leyó esa fixture mediante el mismo payload y devolvió 200; la copia endurecida rechazó el acceso con 404. No se leyó ningún archivo real del sistema.

## 2. Regresión funcional

Después de los parches se verificaron estas funciones:

- `GET /`: HTTP 200.
- Login válido de la consigna: HTTP 200.
- Login inválido: HTTP 401.
- Descarga de `manual.txt`: HTTP 200 y contenido esperado.
- Intento de traversal a la fixture externa: HTTP 404, sin revelar el marcador.

La salida cruda está en `evidencia/salidas/02-regresion-funcional.txt`; el script reproducible es `evidencia/verificar-regresion.py`.

## 3. Evidencia de progreso

- El objetivo original obtuvo 0/5 defensas; salida en `evidencia/salidas/00-verificacion-inicial.txt`.
- La copia del grupo obtuvo 5/5; salida en `evidencia/salidas/01-verificacion-final.txt`.
- `./ctf status extra3` registró las cinco defensas; salida en `evidencia/salidas/03-status-extra3.txt`.
- Los comandos se enumeran en `evidencia/comandos.txt`.
- Captura real de `./ctf status extra3`: `evidencia/status-extra3.png`.

## 4. Preguntas P1–P5

### P1 — Antes y después

Antes del cambio, `BaseHTTPRequestHandler` enviaba `Server: PhantomServer/2.4.1-debug`. El verificador reconocía el producto y la versión. Se neutralizó el banner configurando `server_version` y `sys_version` sin datos identificatorios; la petición de prueba recibió un encabezado `Server` vacío y D1 pasó. La evidencia del ataque inicial y la salida final está en los archivos de verificación de la sección anterior.

### P2 — ¿Por qué no alcanza con quitar `/backup` de `robots.txt`?

**Pendiente para Belén:** explicar la diferencia entre no anunciar la ruta y controlar el acceso a su recurso, y vincularlo con el resultado observado en D3.

### P3 — Seguridad por oscuridad

D1 reduce la información pasiva que el encabezado revela, pero no corrige las vulnerabilidades de la aplicación. Ocultar producto y versión puede dificultar una identificación rápida, aunque un atacante todavía puede inferir tecnología con otras respuestas y pruebas. Por eso el cambio es una medida complementaria de reducción de exposición, no una barrera de acceso ni un sustituto de corregir D2–D5.

### P4 — ¿Por qué no conviene quitar comillas con `replace`?

**Pendiente para Carolina:** explicar por qué el filtrado manual depende de los detalles del motor SQL y cómo la consulta parametrizada implementada en D4 mantiene los valores separados de la sintaxis.

### P5 — Ranking de gravedad

**Pendiente para Juan:** ordenar D1–D5 según impacto y justificar la primera posición con la evidencia del laboratorio, aclarando qué demuestra y qué no demuestra cada prueba.

## 5. Conclusión

La consigna invierte la lógica de los laboratorios ofensivos: ya no alcanza con encontrar una entrada, sino que hay que cerrar cada fallo y comprobar que la aplicación conserve sus funciones legítimas. En esta entrega, las cinco defensas pasaron el verificador y las funciones principales no presentaron regresiones. El grupo completará las respuestas pendientes y revisará el informe antes de entregar.
