# Gatillos CD Bávaro (herramientasPages)

Dashboard de una sola página que reemplaza el Excel "Herramientas de gestión CD Bavaro.xlsx" (AB InBev RD, DPO). Calcula el **gatillo** de cada indicador por mes y cruza los días que lo pasaron con los **Relatos de Anomalía** y **5XQ** (5 porqués) registrados en Microsoft Forms.

El usuario es analista de productividad del CD Bávaro y estudiante de Ingeniería de Sistemas: explica los cambios con lo básico, en español.

## Archivos

| Archivo | Para qué |
|---|---|
| `index.html` | La página completa (HTML + CSS + JS en un solo archivo). Se publica en GitHub Pages como los otros tableros (owdPages, accionesPages, montacargasPages). |
| `Plantilla_Gatillos_CD_Bavaro.xlsx` | Formato para subir datos. Hojas: Instrucciones, Indicadores, Datos, Gatillos fijos (la hoja Plan es opcional; la trae la plantilla que descarga la página). |
| `Plantilla_Plan_CD_Bavaro.xlsx` | Formato vacío del plan (Instrucciones, Plan, Listas) con listas desplegables. Lo generó un script de Python con la librería estándar; la página genera uno igual (sin listas desplegables) desde "Descargar plantillas". |
| `gatillos-api/` | Worker de Cloudflare + D1 (`gatillos-db`). No va en el repositorio (`.gitignore`). Publicar: `npx wrangler deploy` dentro de la carpeta. |

GitHub: `Jerez129/herramientasPages`, Pages desde `main` / raíz → https://jerez129.github.io/herramientasPages/

## Base de datos (Worker `https://gatillos-api.japf20092004.workers.dev`)

- Tablas: `datos(ind,f,v,o)`, `regs(t,data)`, `plan(id,data)`, `config(key,value)` con `ind`, `fijos`, `cfg`, `regMeta`. Esquema en `gatillos-api/schema.sql`.
- Rutas: `GET /api/state`; `PUT /api/config/:key`; `POST /api/datos {put,del}`; `POST /api/regs/:t {rows,first}` (first borra los de ese tipo); `POST /api/plan {put,del}`.
- Leer es libre. Guardar pide el secreto `EDIT_KEY` (configurado el 2026-11-03; solo el usuario conoce la clave). La página pide la clave (botón de estado de la base en el encabezado) y la guarda en localStorage `gatillos-bvr-clave`.
- En la página: `guardar()` guarda local y programa `sync()`, que compara `foto(S)` con `DB.snap` (lo que está en la base) y manda solo lo que cambió. `guardarUI()` es para cambios de vista (no van a la base). `pull()` al abrir: si la base está vacía sube lo local; si hay cambios locales sin subir (`gatillos-bvr-pend`) los sube (lo local gana); si no, toma la base.
- D1 permite 100 valores por consulta: el Worker arma INSERT de varias filas (`multi`) y la página manda en partes.

La constante `SEED` dentro de `index.html` (una línea JSON larga) son los datos iniciales migrados del Excel anterior: `{ind:[...], datos:[[indicador, fecha, valor]], fijos:[[indicador, "AAAA-MM", gatillo]]}`. No edites esa línea a mano salvo para regenerarla.

## Regla del gatillo (igual que las hojas mensuales del Excel)

- Gatillo relato = promedio ± k × desviación estándar **de muestra** (STDEV) de los datos de los `meses` anteriores (por defecto 3: el gatillo de noviembre usa agosto–octubre).
  - "Mayor peor" suma, "Menor peor" resta. k por indicador (1 Almacén, 2 Distribución en el Excel).
  - Se necesitan al menos 2 datos.
- Gatillo 5XQ (matinal / cambio de turno) = gatillo relato ajustado `margen`% hacia lo más exigente (5%). El Excel hacía ×0.95 siempre; aquí depende del sentido.
- Un gatillo fijo (`S.fijos[id]["AAAA-MM"]`) manda sobre el cálculo.
- Un día "pasa" si: Mayor peor → valor > gatillo; Menor peor → valor < gatillo. Nivel `R` (relato) o `5` (solo 5XQ).
- Un día está analizado si hay un registro del mismo indicador con fecha entre el día y `S.cfg.tol` días después. Nivel R exige relato; nivel 5 acepta relato o 5XQ.
- La vista Mes muestra el cumplimiento **separado**: relatos = `hechoR / R`, 5XQ = `hecho5 / c5` (tarjetas arriba y una columna de cada uno en la tabla).
- Relatos y 5XQ de enero a marzo 2026 vienen del Form anterior (`viejo.zip`), sumados directo a D1 con `wrangler d1 execute --file`; se procesaron con el mismo `cargarForms` de la página.
- Verificado: con los datos de la hoja Septiembre, el cálculo da exactamente los gatillos de septiembre de "2. Indicadores".

## Unidades internas

- Tiempo: fracción de día (como Excel). `fmt()` lo muestra h:mm:ss.
- Porcentaje: decimal (0.031 = 3.1%). `parseVal()` acepta "3.1%", "0.031" o "3.1".
- Número: tal cual.

## Estado (`S`, guardado en localStorage `gatillos-bvr-v1`)

```
S = { v:1,
  ind:   [{id, nombre, area, unidad, sentido, k, meses, margen, alias:[...], activo}],
  datos: { [indId]: { "AAAA-MM-DD": {v, o} } },   // o: 'x' Excel anterior, 'p' plantilla, 'm' escrito en la página
  fijos: { [indId]: { "AAAA-MM": valor } },
  reg:   [{t:'R'|'5', f, a, ind, resp, chofer, ruta, res, pq1, causa, acc, cron}],  // de los Forms
  regMeta: { R:{archivo,cargado,n,desde,hasta}, '5':{...} },
  plan:  [{id, ind, indN, f, t:'R'|'5', resp, lim, nota, hecho}],          // Plan de análisis
  cfg: { tol } }
```

## Plan de análisis (pestaña "Plan de análisis", `vPlan`)

- "Por planificar" = días del mes con estado `pend`/`parcial` en `resumen()` que no están en `S.plan` (clave indicador + día). Se agregan uno a uno o todos, con responsable y fecha límite.
- Estado (`planEstado`): `forms` si `regsDia()` trae el análisis (relato exige relato; 5XQ acepta cualquiera), `hecho` si se marcó a mano, `venc` si pasó `lim`, si no `pend`.
- `indN` guarda el nombre del indicador para reubicar el plan al volver a los datos del Excel anterior (los ids cambian).
- Solo los relatos llevan responsable; en los 5XQ `resp` queda vacío (`nuevoPlan`).
- Hoja **Plan**: con hoja Indicadores (respaldo completo) reemplaza el plan; sola (formato del plan) agrega o actualiza por indicador + día + tipo.
- Menú "Descargar plantillas": `plantillaDatos()` (sin hoja Indicadores, así que al cargarla solo agrega), `formatoPlan(conDias)`, `exportar()` (respaldo completo).

`GC` (caché de gatillos) y `RC` (índice de registros por indicador) se limpian con `invalidar()` después de cualquier cambio.

## Carga de archivos (`procesar`)

- Si el libro tiene hoja **Indicadores** → reemplaza catálogo, datos y gatillos fijos. Si solo trae **Datos** → agrega/actualiza.
- Si no, se trata como exportación de Forms: busca la fila de encabezados con "Indicador", detecta columnas por nombre (`cargarForms`), filtra centro Bávaro. Los registros se **suman por la columna ID** (número de respuesta de Forms, guardado en `reg[].id`): una exportación partida en varios archivos (5XQ viene en 1-5000 y 5001-7564) no borra la otra parte, y una respuesta repetida se actualiza. Sin columna ID, el archivo reemplaza todos los registros de su tipo. Es 5XQ si el nombre del archivo/hoja dice 5XQ / "5 por qué" o hay columnas de supervisor/conferente/chofer.
- Unión Forms ↔ indicador: `alias` del catálogo. `Área|Nombre` para nombres repetidos en varias áreas (ej. "Distribución|Roturas").
- Probado con las exportaciones reales de 2026 (encabezados en español: "Hora de inicio", "Centro", "Responsable-DOxx", "Indicador/2/3", "Resultado"): 297 relatos y 349 5XQ de Bávaro de 9.461 respuestas. Algunos indicadores de los Forms no tienen gatillo (Carga de Camiones, Productividad de Picking, TMA, ADF…): se quedan como registros sin gatillo.

## Convenciones (iguales a los otros tableros del usuario)

- Un solo archivo, sin build. SheetJS 0.18.5 desde cdnjs. Fuente IBM Plex Sans / Condensed.
- Colores como tokens en `:root`, con modo oscuro (`prefers-color-scheme` y `data-theme`). Estados: `--good`, `--warn`, `--crit`.
- Todo acceso a localStorage va en try/catch.
- Sin `alert`/`confirm`: las confirmaciones son en línea (`confirmar()`).
- Textos en español, frases cortas, sin tecnicismos.

## Probar

Abre `index.html` con doble clic (sin base: el Worker solo acepta GitHub Pages y localhost). Para probar con la base: `python -m http.server 8765` y abrir `http://127.0.0.1:8765/`; ojo, eso escribe en la base real. `?mes=2026-06` abre en ese mes. Con Playwright: interceptar `cdnjs` si no hay red y servir `xlsx.full.min.js` local.

## Fuentes de datos reales (en la carpeta, no van al repositorio)

- `tiempo.zip` → `tiempo.xlsx` (hoja DATA, una fila por ruta y día): columnas `TML` (25), `Tiempo en Ruta` (26), `Tiempo Interno` (30). El dato diario de la página = **promedio de todas las rutas del día**. Verificado: coincide exacto con los datos de agosto 2026 que venían del Excel.
- `Rutina_Cumplimiento_Resultados_<Mes>.xlsx`, hoja **Diaria logistica**: fila de encabezado con "INDICADOR" y "DISPARADOR" (= gatillo del mes) y fechas como números de Excel; una fila por KPI (DEVOLUCION, ROTURA [DE] DISTRIBUCION / ALMACEN, DIFERENCIA [DE] INVENTARIO, CRITICA, NO CARGADO, TIEMPO MTTO CORRECTIVO / TIEMPO MEDIO DE REPARACION, MODULACION). El archivo de Septiembre trae el encabezado con fechas de agosto: hay que tomar el mes del nombre del archivo y el día de la columna. Enero_2025 y Febrero_2025 son **de 2026** con el año mal escrito (su TML coincide con tiempo.xlsx de 2026 y los domingos vacíos son los de 2026).
- El usuario **no quiere nada de 2025**: el SEED ya no tiene datos de 2025 y la planilla empieza en enero 2026. Ojo con fechas con 2025 escrito por error en lugar de 2026: comprobarlo con otra fuente (día de la semana, tiempo.xlsx, hora de llenado de los Forms) antes de corregir o descartar. Las exportaciones de Forms de 2026 revisadas no tienen ese error.
- Rotura tiene dos filas en distinta unidad (unidades vs HL); Crítica de Rutina (cientos) no coincide con los datos depurados del Excel. En el Excel se quitaban a mano los días atípicos (los que pasaron el gatillo de su mes) antes de calcular.
- `Planilla_Datos_CD_Bavaro_2026-01_a_2026-10.xlsx`: planilla completa armada con esas fuentes (origen Plantilla = real, Excel anterior, Inventado = meses sin fuente, hasta ago 2026). Hoja Resumen: gatillo fijo vs calculado. Los scripts que la generaron estaban en el scratchpad de la sesión (no se guardaron en el proyecto).
- Origen `i` (Inventado): cuenta para el cálculo del gatillo, pero `resumen()` y la gráfica no lo marcan como día que pasó el gatillo.

## Pendiente / conocido

- En el Excel anterior los valores se ordenaron sin mover sus fechas: los datos con origen "Excel anterior" (junio–agosto 2026) tienen el conjunto de valores correcto pero el día no es confiable. Se marcan "aprox.".
- Septiembre y octubre 2026 ya tienen datos reales en la planilla completa (si el usuario la carga). Revisar el gatillo de noviembre de Crítica, Devolución y Tiempo interno: septiembre no se pudo depurar (el gatillo fijo no corresponde a la escala de los datos).
- Crítica y No cargado cambiaron de unidad en julio 2026 (de % a HL); los gatillos fijos de feb–jun quedaron en %.
- Cambiar la clave de edición: `npx.cmd wrangler secret put EDIT_KEY` dentro de `gatillos-api/` (en PowerShell `npx` sin `.cmd` está bloqueado).
