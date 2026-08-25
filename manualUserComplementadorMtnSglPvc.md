# Manual del usuario — complementadorMtnSglPvc

## Qué hace esta skill

`complementadorMtnSglPvc` recopila notificaciones de Marine Traffic y Skylight
Global desde el correo Gmail (vía un proyecto Google Apps Script) y fusiona esos
datos nuevos a un histórico maestro acumulativo por plataforma (TSV), evitando
duplicados. Incluye una etapa `--verify` de solo lectura para revisar qué
entraría antes de escribir cambios reales.

**Alcance:** esta skill empaqueta el flujo de fusión ya existente. No automatiza
por completo la lectura del correo hasta el archivo TSV final — hay un paso
manual intermedio (ver "Operación paso a paso", Paso 02).

## Instalación

1. Copia el bootstrap (`complementadorMtnSglPvcBootstrap.md`) a la raíz del
   proyecto donde quieres usar la skill.
2. Sigue los pasos del propio bootstrap: crea las carpetas indicadas, guarda el
   script de extracción embebido en `/tmp/extractComplementadorMtnSglPvc.py` y
   ejecútalo con `python3`. Esto escribe los archivos reales de la skill
   (`SKILL.md`, `scripts/*.py`, `scripts/*.js`, `requirements.txt`) en tu
   proyecto.
3. Instala la dependencia Python (en tu propio entorno virtual):
   ```bash
   pip install -r requirements.txt
   ```
4. Copia el contenido de `scripts/00runnersNotificationsMtnSglPvc.js` y
   `scripts/01notificationsMtnSglPvc.js` a un proyecto de Google Apps Script
   propio (no se despliega automáticamente — Apps Script vive en Google, no en
   tu sistema de archivos local).
5. Verifica que el número de archivos extraídos coincide con el reportado por el
   bootstrap.

## Cómo invocar

Activa esta skill cuando recibas un archivo nuevo de Marine Traffic o Skylight
Global y necesites fusionarlo al histórico maestro: "complementa el histórico de
Skylight", "fusiona el TSV de Marine Traffic", "corre el complementador MTN/SGL".

## Operación paso a paso

### Paso 00 — Configurar el proyecto Apps Script (una sola vez)

Antes del primer uso, edita 3 puntos en tu copia de Apps Script — no son bugs,
son configuración propia de cada organización:

| Función | Qué edita | Nota |
|:---|:---|:---|
| `getTargetDataSet` | `folderPath = ["[CARPETA_DRIVE_NIVEL1]", "[CARPETA_DRIVE_NIVEL2]", "[CARPETA_DRIVE_NIVEL3]"]` | Reemplaza los 3 placeholders por tu propia ruta de carpetas en Google Drive |
| `mailLabels` | diccionario numero→etiqueta Gmail | Reescribe completo con tus propias etiquetas de Gmail |
| `targetHojas` | diccionario clave→"archivo/hoja" | Reescribe completo con el nombre de tu propio archivo y hojas de Google Sheets |

### Paso 01 — Extraer notificaciones del correo

Corre `ejecutaSkyLight()`, `ejecutaMarine()` o `ejecutaAmbos()` desde el editor de
Apps Script (o automatiza con `setupSkylightJob()`/`setupMarineJob()`, que
registran un trigger recurrente vía `registerBatchJob`/`batchDispatcher`). Esto
vuelca las notificaciones nuevas en la hoja de cálculo de trabajo ("Wrk")
correspondiente y archiva los hilos de Gmail procesados.

### Paso 02 — Exportar a TSV/CSV (paso manual, no automatizado)

Descarga la hoja de cálculo de trabajo como TSV o CSV (Archivo → Descargar →
Valores separados por tabulaciones/comas, desde Google Sheets).

### Paso 03 — Depositar el archivo

Coloca el TSV/CSV exportado en la carpeta configurada como `complementos` (ver
Paso 04).

### Paso 04 — Configurar las rutas del complementador Python (una sola vez)

Edita el diccionario `RUTAS` al inicio de cada script:

| Script | Clave | Placeholder a reemplazar |
|:---|:---|:---|
| `02complementadorHistoricoSkylight.py` | `historico` | `[RUTA_HISTORICO_SKYLIGHT_TSV]` |
| `03complementadorHistoricoMarineTraffic.py` | `historico` | `[RUTA_HISTORICO_MARINETRAFFIC_TSV]` |
| ambos | `complementos` | `[RUTA_COMPLEMENTOS_DIR]` |
| ambos | `backup` | `[RUTA_BACKUP_DIR]` |

### Paso 05 — Verificación previa (sin cambios)

```bash
python3 scripts/02complementadorHistoricoSkylight.py --verify
python3 scripts/03complementadorHistoricoMarineTraffic.py --verify
```

Revisa el preview: cuántos registros nuevos detectó, cuántos duplicados descartó.

### Paso 06 — Fusión real

```bash
python3 scripts/02complementadorHistoricoSkylight.py
python3 scripts/03complementadorHistoricoMarineTraffic.py
```

Cada script crea un backup del histórico antes de escribir la fusión, en la
carpeta configurada como `backup`.

## Troubleshooting

- **El script Python no encuentra el histórico**: revisa que reemplazaste
  `[RUTA_HISTORICO_SKYLIGHT_TSV]` / `[RUTA_HISTORICO_MARINETRAFFIC_TSV]` por una
  ruta real, y que el archivo existe (aunque sea vacío con encabezados, en el
  primer uso).
- **`GmailApp.getUserLabelByName(...)` retorna `null` en Apps Script**: la
  etiqueta que pusiste en `mailLabels` no existe en tu Gmail — créala primero
  desde la interfaz de Gmail, con el nombre exacto (sensible a mayúsculas y a la
  jerarquía con `/`).
- **`getTargetDataSet` retorna `null`**: alguno de los 3 niveles de
  `folderPath` no existe en tu Drive, o el archivo/hoja de `targetHojas` no
  existe con ese nombre exacto dentro de esa carpeta.
- **Registros duplicados tras la fusión**: revisa `CONFIG.claveDedup` en cada
  script Python (`eventId` para Skylight, `urlEvento` para Marine Traffic) — si tu
  fuente de datos cambia de formato, esa clave puede dejar de ser única.
- **El trigger `batchDispatcher` no procesa nada**: verifica que hay trabajos
  registrados con `PropertiesService.getScriptProperties()` — si no hay
  `job_*` guardado, el dispatcher se limpia solo y no hace nada.

## Recomendaciones

- No subas los TSV históricos con datos reales de embarcaciones a un repositorio
  público — son datos operativos, no código.
- Reescribe `mailLabels` y `targetHojas` completos antes del primer uso; no dejes
  mezcladas claves de la organización original con las tuyas.
- Si tu equipo no usa Google Sheets como paso intermedio, considera que el
  Paso 02 (exportar a TSV) es manual por diseño en esta versión — automatizarlo
  (ej. con `SpreadsheetApp` exportando directo a Drive como TSV) es una mejora
  futura fuera del alcance de este paquete.
- Corre siempre `--verify` antes de la fusión real la primera vez que uses esta
  skill en un proyecto nuevo, para confirmar que las rutas y la deduplicación se
  comportan como esperas.
