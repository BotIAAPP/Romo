# Portal Facturador GILM

Sistema end-to-end de facturación CFDI 4.0 para el Grupo Industrial LM (Perfiles LM · Indalum · Galvasid · Kobrex). El portal público recibe tickets de compra, una pipeline automática los triagea, y un operador humano (o un agente Claude en evolución) genera las facturas en los portales de cada cadena.

> **URL en producción:** https://botiaapp.github.io/Romo/
> **Worker:** https://facturas-intake.aromoaliacer.workers.dev/health

---

## Tabla de contenidos

1. [El problema](#el-problema)
2. [La solución](#la-solución)
3. [Arquitectura](#arquitectura)
4. [Componentes](#componentes)
5. [Estados de una solicitud](#estados-de-una-solicitud)
6. [Setup desde cero](#setup-desde-cero)
7. [Operación día a día](#operación-día-a-día)
8. [Roadmap](#roadmap)

---

## El problema

GILM compra tickets en muchas cadenas (restaurantes, farmacias, gasolineras, supermercados). Cada una tiene:
- **Su propio portal de facturación** (URL, campos, lógica, CAPTCHAs distintos)
- **Su propio plazo** — desde **72 horas** (Farmacia Guadalajara) hasta **30 días** (Benavides) o **mismo mes** (Walmart, OXXO, H-E-B, Soriana)
- **Su propio formato de ticket** — algunos tienen códigos de barras, otros QR, otros nada

Antes de este proyecto, el flujo era 100% manual: alguien recolectaba los tickets, abría cada portal, llenaba a mano, y descargaba PDF+XML uno por uno. Resultado: muchos tickets **se vencían sin facturarse**, dinero perdido por IVA no recuperado.

## La solución

Un portal web público + pipeline automática + agente facturador. El usuario sube la foto del ticket en 4 pasos. El sistema:

1. **Recibe** la solicitud en Cloudflare (escala infinito, $0).
2. **Triagea automático** con Mistral OCR — detecta cadena, fecha, total, valida plazo.
3. **Cierra solo** los casos obvios: fuera de plazo, ticket ilegible, archivo que no es ticket.
4. **Escala al operador** los que sí son facturables (vía email + cola en Postgres).
5. **Operador factura** con Claude Code + Playwright (CAPTCHAs resueltos por humano).
6. **CFDI entregado** al solicitante por correo automático.
7. **Todo queda registrado** en Postgres (incluyendo PDF/XML como bytea).

Resultado actual en producción: **~87% de los tickets se procesan sin intervención humana** (todos los fuera-de-plazo y datos-incompletos se cierran automáticamente).

---

## Arquitectura

```
┌────────────────┐
│   Usuario      │   Portal GILM
│   (cualquiera) │   https://botiaapp.github.io/Romo/
└────────┬───────┘
         │ POST /submit  (FormData: RFC + email + ticket)
         ↓
╔════════════════════════════ CLOUDFLARE ═════════════════════════╗
║  Worker `facturas-intake`                                       ║
║  ├─ POST /submit       → guarda en KV, manda email vía Resend   ║
║  ├─ GET  /list?email=X → lista filtrada por email del cliente   ║
║  └─ GET  /download/... → URLs HMAC-firmadas, expiran en 30 días ║
║                                                                 ║
║  KV TICKETS                                                     ║
║  ├─ tickets/2026/05/FAC-XXX-name.jpeg  (foto original)          ║
║  ├─ cfdi/FAC-XXX.{pdf,xml}             (CFDI timbrado)          ║
║  └─ index/by_email/{email}.json        (consulta del cliente)   ║
╚═════════════════════════════════════════════════════════════════╝
         │ wrangler kv key list (cada 5 min)
         ↓
╔══════════════════════════ PC LOCAL (servidor) ══════════════════╗
║  Task Scheduler · wscript run_hidden.vbs · cada 5 min · 5 pasos:║
║  (consola oculta vía VBS launcher — sin parpadeo)               ║
║                                                                 ║
║   [1] sync_solicitudes_kv_to_pg.py                              ║
║       KV → Postgres (status='recibida')                         ║
║                                                                 ║
║   [2] auto_procesar_pendientes.py                               ║
║       OCR (Mistral) + regex cadena + valida plazo               ║
║       Cierra automático fuera_de_plazo / datos_incompletos      ║
║       Si OCR falla la fecha → esperando_humano (revisa humano)  ║
║       → Notifica al solicitante (Outlook COM)                   ║
║                                                                 ║
║   [3] notificar_operador_pendientes.ps1                         ║
║       Si hay esperando_humano nuevos, manda UN email batch      ║
║       (no spam — usa columna notificacion_operador_at)          ║
║                                                                 ║
║   [4] sync_postgres_to_kv_index.py                              ║
║       PG → KV: rebuild índice por email + sube CFDIs            ║
║                                                                 ║
║   [5] backfill_cfdi_pendientes.py                               ║
║       Para 'timbrada_pendiente_archivos': intenta descarga      ║
║       directa por UUID (portales que la soportan) o empareja    ║
║       archivos en cfdi-intake/ por UUID del XML. Cuando         ║
║       ambos bytes están, el trigger auto-promueve a 'timbrada'. ║
╚═════════════════════════════════════════════════════════════════╝
         ↓
╔═══════════════ Postgres on-prem · 10.0.10.121 ══════════════════╗
║  datastore01 · tabla solicitudes_facturacion                    ║
║  - 24 columnas incluyendo bytea para PDF/XML                    ║
║  - Trigger marca procesada_at en status terminal                ║
║  - Índices por status, rfc_alias, email, uuid                   ║
╚═════════════════════════════════════════════════════════════════╝
         ↓
  Operador (Adrián) abre Claude Code cuando recibe el email
  → "factura los pendientes" → agente facturador navega portal con Playwright
```

## Componentes

| Componente | Tech | Repo / Ubicación | Función |
|---|---|---|---|
| **Frontend portal** | HTML + CSS + JS vanilla | `BotIAAPP/Romo` (este repo) | 2 tabs: Solicitar / Mis solicitudes. Branded GILM. |
| **Worker intake** | Cloudflare Workers (JS) | `~/IA-WARP/facturas-intake-worker/` | API REST: submit, list, download. HMAC-signed URLs. |
| **KV** | Cloudflare KV | namespace `8a2e518c64154cf5bad9da76042ce861` | Storage: tickets + CFDIs + índices por email |
| **Email transactional** | Resend API | aromoaliacer@gmail.com | Avisos al operador desde el Worker |
| **OCR** | Mistral OCR (`mistral-ocr-latest`) | API · `leer_ticket_ocr.py` | Lectura de tickets, ~$1/1000 páginas |
| **Triaje automático** | Python · regex + reglas | `~/IA-WARP/Facturas/pipeline/auto_procesar_pendientes.py` | Detección de cadena + plazo |
| **Backfill CFDI** | Python | `~/IA-WARP/Facturas/pipeline/backfill_cfdi_pendientes.py` | Empareja PDF/XML por UUID y promueve a `timbrada` |
| **Helper wrangler timeout** | Python | `~/IA-WARP/Facturas/pipeline/_wrangler_util.py` | Timeout con `taskkill /T` que sí mata el árbol de procesos en Windows (fix del kill 0xC000013A) |
| **Lanzador oculto** | VBScript | `~/IA-WARP/Facturas/pipeline/run_hidden.vbs` | Wrapper invisible para que la consola no parpadee cada 5 min |
| **Datastore** | Postgres 15.13 | `10.0.10.121:5432/datastore01` | Source-of-truth de todas las solicitudes |
| **Notif solicitante** | PowerShell + Outlook COM (o Microsoft Graph si está configurado) | `notificar_solicitante.ps1` + `enviar_correo_graph.py` | Email branded GILM al usuario que pidió la factura |
| **Notif operador** | PowerShell + Outlook COM | `notificar_operador_pendientes.ps1` | Email batch con tabla de pendientes |
| **Subagente facturador** | Claude Code subagent | `~/.claude/agents/facturador.md` | Navegación de portales con Playwright + CAPTCHA humano |
| **Agente Tier 2** (en pausa) | Claude Agent SDK + Playwright | `~/IA-WARP/Facturas/pipeline/tier2/` | Versión autónoma futura, pendiente API key |

## Estados de una solicitud

```
                ┌─────────────┐
                │  recibida   │  ← Worker la guardó en KV
                └──────┬──────┘     sync KV→PG la metió en PG
                       │
                       │  triaje automático
                       │
        ┌──────────────┼────────────────────────────────┐
        ↓              ↓                                ↓
   ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
   │ timbrada │  │fuera_de_plazo│  │  ilegible /  │  │esperando_humano  │
   │    ✅    │  │      🚫      │  │ incompletos🚫│  │  ⏳ Tier 2       │
   └──────────┘  └──────────────┘  └──────────────┘  └────────┬─────────┘
        ↑                                                     │
        │ Tier 2 manual: Claude Code + Playwright             │
        └─────────────────────────────────────────────────────┘
```

| Status | Origen | Significado | Acción |
|---|---|---|---|
| `recibida` | Worker | Recién subida desde el portal | Triaje la procesa en máx 5 min |
| `en_proceso` | Triaje / Operador | Siendo trabajada | — |
| `esperando_humano` | Triaje | En plazo, lista para Tier 2 (incluye casos donde OCR no leyó la fecha — Claude con visión los revisa manualmente) | Operador entra a Claude Code |
| `timbrada` | Tier 2 | ✅ CFDI generado, entregado y con **PDF + XML guardados como `bytea`** | Cierre exitoso |
| `timbrada_pendiente_archivos` | Tier 2 / Trigger | CFDI ya timbrado, pero falta PDF o XML en `bytea`. El trigger de la tabla degrada aquí cuando se marca `timbrada` sin ambos archivos | Esperar correo del portal o intervención manual; backfill [5/5] los empareja por UUID |
| `ya_facturada` | Triaje / Operador | El ticket ya tenía CFDI emitido en una solicitud previa (duplicado) | Cierre informativo |
| `fuera_de_plazo` | Triaje | El ticket excede el plazo de la cadena | Solicitante notificado |
| `datos_incompletos` | Triaje | El archivo no es un ticket válido (ej. ya es un CFDI subido por error) | Solicitante notificado |
| `ticket_ilegible` | Operador | OCR + visión humana coinciden en que el ticket no es facturable (sin TC, sin RFC visible, etc.). **Nota:** desde 2026-05-22 el triaje ya NO lo auto-asigna en fallos de OCR — esos van a `esperando_humano` para que Claude con visión los lea | Solicitante notificado |
| `captcha_fallido` | Tier 2 | El portal rechazó por CAPTCHA tras varios intentos | Reintento manual |
| `portal_caido` | Tier 2 | El portal no respondió | Reintento manual |
| `error` | — | Falla inesperada | Investigación manual |

## Setup desde cero

> Para reconstruir el sistema en otra máquina o reproducir desarrollo local.

### Pre-requisitos
- **Python 3.10+** con `psycopg2-binary`, `mistralai`, `anthropic`, `playwright`
- **Node.js 18+** con `wrangler` (Cloudflare CLI) autenticado
- **PostgreSQL client 15+** (psql en PATH)
- **Outlook desktop** (para notificaciones via COM)
- Acceso a la red interna (Postgres `10.0.10.121`)

### Cloudflare Worker
```bash
cd ~/IA-WARP/facturas-intake-worker
wrangler login
wrangler kv namespace create TICKETS   # si no existe
wrangler secret put RESEND_API_KEY
wrangler secret put DOWNLOAD_SIGN_SECRET
wrangler deploy
```

### Postgres
```bash
gilmdb -f ~/IA-WARP/Facturas/pipeline/schema_solicitudes_facturacion.sql
gilmdb -f ~/IA-WARP/Facturas/pipeline/add_email_solicitante.sql
gilmdb -f ~/IA-WARP/Facturas/pipeline/add_notif_operador.sql
```
(El schema agrega columnas `bytea` para PDF/XML, el trigger de invariante `timbrada ⇔ archivos`, y los estados extra `timbrada_pendiente_archivos` y `ya_facturada`.)

### Variables de entorno (User scope en Windows)
```powershell
[Environment]::SetEnvironmentVariable("MISTRAL_API_KEY", "<key>", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "<key>", "User")  # opcional Tier 2
```

### Task Scheduler
```powershell
# La tarea ejecuta wscript.exe contra un .vbs que lanza el wrapper con la
# consola OCULTA (Run cmd,0,True) — sin parpadeo y siguen aplicando
# ExecutionTimeLimit (4 min) e IgnoreNew para que las corridas no se piquen.
schtasks /Create /TN "GILM_SyncSolicitudesKVtoPG" `
  /TR 'wscript.exe "C:\Users\...\IA-WARP\Facturas\pipeline\run_hidden.vbs"' `
  /SC MINUTE /MO 5 /RL LIMITED
# Settings recomendadas tras crear:
#   MultipleInstances = IgnoreNew    (evita pileup)
#   ExecutionTimeLimit = PT4M        (safety net por si algo cuelga)
#   LogonType = Interactive          (Outlook COM necesita la sesión real)
```

**Gotcha resuelto:** las llamadas a `wrangler` se hacen vía el helper
`_wrangler_util.py` (`run_wrangler`), no con `subprocess.run(..., timeout=T, shell=True)`.
Razón: en Windows, ese timeout mata sólo `cmd.exe` y deja `node` huérfano
sosteniendo los pipes → la llamada nunca regresa y el `ExecutionTimeLimit`
mata todo el wrapper (`0xC000013A`). El helper mata el árbol completo con
`taskkill /F /T /PID` cuando expira, así un atoramiento se acota a 90 s
en vez de 4 min y el pipeline auto-recupera al siguiente ciclo.

### Frontend (este repo)
- Push a `main` → GitHub Pages rebuildeará en ~30s
- Verificar que el `WORKER_URL` en `index.html` apunte al Worker correcto

## Operación día a día

### Cuando un usuario sube un ticket
1. **t+0:** usuario clickea Enviar en el portal
2. **t+0..5min:** Task Scheduler hace el siguiente tick → ticket entra a Postgres como `recibida`
3. **t+5..10min:** triaje procesa, decide status, manda notificaciones
4. Si quedó `esperando_humano` y es la primera vez: **email al operador**

### Cuando el operador recibe email "N solicitudes esperan facturación"
1. Abrir Claude Code en el directorio del proyecto
2. Decir: `factura los pendientes`
3. El agente (subagente facturador) los procesa uno por uno con Playwright
4. Para CAPTCHAs: el agente toma screenshot y pide al humano que lo lea

### Para consultar el estado de una solicitud
- **Usuario final:** abre el portal → pestaña "Mis solicitudes" → mete su email → ve la lista con botones de descarga
- **Operador:** `gilmdb -c "SELECT * FROM solicitudes_facturacion WHERE ref_id='FAC-...'"`

### Para diagnosticar problemas
- Logs del pipeline: `~/IA-WARP/Facturas/pipeline/logs/sync_solicitudes_YYYYMMDD.log`
  (busca `- fin (step1=...)` para cierres limpios; `^C` indica que esa corrida fue matada por el `ExecutionTimeLimit` y se reanuda en el siguiente ciclo)
- Estado del Task Scheduler: `(Get-ScheduledTask "GILM_SyncSolicitudesKVtoPG" | Get-ScheduledTaskInfo).LastTaskResult` (0 = OK, 0xC000013A = killed por timeout)
- KV en Cloudflare: `cd ~/IA-WARP/facturas-intake-worker && wrangler kv key list --namespace-id 8a2e518c64154cf5bad9da76042ce861 --remote`
- Worker logs: dashboard de Cloudflare
- Conteo por estado: `gilmdb -c "SELECT status, COUNT(*) FROM solicitudes_facturacion GROUP BY status ORDER BY 2 DESC"`

## Roadmap

### Próximo (cuando se quiera)
- **Tier 2 autónomo**: el código está listo en `~/IA-WARP/Facturas/pipeline/tier2/`, sólo falta API key y testing. Una vez activado, el agente Claude Agent SDK navegará portales solo y sólo escalará a humano para CAPTCHAs.
- **Buzón de intake automático**: hoy, cuando un portal entrega el CFDI solo por correo (Walmart, iPark, etc.), hay que depositar los adjuntos en `~/IA-WARP/Facturas/cfdi-intake/` a mano. Falta el auto-ingest de un buzón IMAP/Graph que haga eso solo.
- **Sanear `kv_key` en el Worker**: filenames con `..` (ej. `Walmart..pdf`) rompen la URL del API de Cloudflare KV (`403` por path-traversal). El Worker debería reemplazar `..` por `_` al construir el key. Hoy se rompe el `wrangler kv key get` para esos archivos.

### Futuro (si crece el volumen)
- **Worker Cron Triggers** para liberar la PC: requiere migrar el datastore a cloud (Neon/Supabase) o exponer Postgres via Cloudflare Tunnel.
- **Servicio de anti-CAPTCHA** (2Captcha API, ~$3/1000) para reducir intervención humana a 0.
- **Dashboard interno** con métricas: tasa de éxito por cadena, tiempo medio de procesamiento, $ recuperados, etc.

---

## Cadenas catalogadas (a hoy)

Catálogo en `~/.claude/skills/generar-factura/portales.json`. Cada vez que se factura una cadena nueva, agregar su flujo ahí.

| Cadena | Plazo | Portal | Notas |
|---|---|---|---|
| 🟢 7-Eleven | Mes + 5 días | e7-eleven.com.mx | |
| 🟢 Walmart | **30 días naturales** | **facturacion.walmartmexico.com.mx** | URL "vieja" `factura.walmart.com.mx` ya no resuelve DNS. Entrada canónica: footer de walmart.com.mx (pero el sitio principal tiene challenge Akamai). El XML viene **solo por correo** — el portal sólo descarga PDF. |
| 🟢 OXXO | Mismo mes | factura.oxxo.com | Solo tickets > $30 MXN |
| 🟢 Costco | Mismo mes | costco.com.mx/facturacion | Requiere membresía |
| 🟢 Home Depot | Mismo mes | homedepot.com.mx/facturacion | |
| 🟢 Soriana | Mismo mes | facturacion.soriana.com | |
| 🟢 H-E-B | Mismo mes | facturacion.heb.com.mx | |
| 🟢 Soft Restaurant (El Frison, La Vicenta, etc.) | 7 días | mefacturo.mx/{slug} | |
| 🟢 Boston's Pizza | 30 días | facturacion.bostons.com.mx | |
| 🟢 Anderson's | Variable | xetux-e.com/facturacion/webFact | Permite descarga directa de XML por UUID |
| 🟢 iPark (Embia Parking) | Mes calendario | ipark.mx/facturacion-express | CFDI entregado **solo por correo** (`entrega_cfdi: solo_correo`); backfill espera adjuntos en `cfdi-intake/` |
| 🟡 Farmacias Benavides | 30 días naturales | e-facturate.com/benavides | |
| 🟡 Farmacias del Ahorro | 30 días naturales | l.fdah.mx/factura | |
| 🔴 Farmacia Guadalajara | **72 horas** (¡ojo!) | farmaciasguadalajara.com/facturacion-electronica | Plazo más corto del catálogo |
| ⚠️ Carnes Ramos | **Mismo día** | carnesramos.ddns.net:8301 | Backend con bug conocido (`Undefined offset:1` en `FacturarController.php:342`) que tumba la facturación incluso en plazo. Para tickets afectados: escalar por correo a `facturacionelectronica@carnesramos.com.mx` para emisión manual. |

---

_Proyecto interno GILM · 2026_
