# Gatillos y análisis de anomalías — CD Bávaro

Reemplaza el Excel "Herramientas de gestión CD Bavaro". Calcula el gatillo de cada indicador por mes (promedio ± k desviaciones de los meses anteriores), muestra qué días lo pasaron y si ya tienen relato o 5XQ, y permite planificar los análisis que faltan.

Página: https://jerez129.github.io/herramientasPages/

## Uso

1. Abre la página. Los datos se guardan en una base compartida (Cloudflare Worker `gatillos-api` + D1): todos ven lo mismo.
2. **Descargar plantillas** (arriba a la derecha):
   - *Datos diarios (vacía)*: para agregar datos y gatillos fijos.
   - *Plan de análisis (vacío / con días por planificar)*: para subir los relatos y 5XQ por hacer.
   - *Respaldo completo*: todo lo de la base. Al cargarlo reemplaza todo.
3. Arrastra o usa **Cargar Excel** con una plantilla o con las exportaciones de los Forms de **Relatos** y **5XQ**.
4. Pestaña **Plan de análisis**: días que pasaron un gatillo sin análisis, responsable (solo relatos), fecha límite y estado.

También puedes escribir o pegar datos en la pestaña **Datos**, y agregar o quitar indicadores en **Indicadores**.

> No subas exportaciones de los Forms al repositorio: tienen nombres de personas. El `.gitignore` excluye los `.xlsx` salvo las plantillas.
