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
║  Task Scheduler · wrapper cmd · cada 5 min · 4 pasos:           ║
║                                                                 ║
║   [1] sync_solicitudes_kv_to_pg.py                              ║
║       KV → Postgres (status='recibida')                         ║
║                                                                 ║
║   [2] auto_procesar_pendientes.py                               ║
║       OCR (Mistral) + regex cadena + valida plazo               ║
║       Cierra automático fuera_de_plazo / ilegible / incompletos ║
║       Marca esperando_humano si está en plazo                   ║
║       → Notifica al solicitante (Outlook COM)                   ║
║                                                                 ║
║   [3] notificar_operador_pendientes.ps1                         ║
║       Si hay esperando_humano nuevos, manda UN email batch      ║
║       (no spam — usa columna notificacion_operador_at)          ║
║                                                                 ║
║   [4] sync_postgres_to_kv_index.py                              ║
║       PG → KV: rebuild índice por email + sube CFDIs            ║
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
| **Triaje automático** | Python · regex + reglas | `~/IA-WARP/MES/auto_procesar_pendientes.py` | Detección de cadena + plazo |
| **Datastore** | Postgres 15.13 | `10.0.10.121:5432/datastore01` | Source-of-truth de todas las solicitudes |
| **Notif solicitante** | PowerShell + Outlook COM | `notificar_solicitante.ps1` | Email branded GILM al usuario que pidió la factura |
| **Notif operador** | PowerShell + Outlook COM | `notificar_operador_pendientes.ps1` | Email batch con tabla de pendientes |
| **Subagente facturador** | Claude Code subagent | `~/.claude/agents/facturador.md` | Navegación de portales con Playwright + CAPTCHA humano |
| **Agente Tier 2** (en pausa) | Claude Agent SDK + Playwright | `~/IA-WARP/MES/tier2/` | Versión autónoma futura, pendiente API key |

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
| `esperando_humano` | Triaje | En plazo, lista para Tier 2 | Operador entra a Claude Code |
| `timbrada` | Tier 2 | ✅ CFDI generado y entregado | Cierre exitoso |
| `fuera_de_plazo` | Triaje | El ticket excede el plazo de la cadena | Solicitante notificado |
| `datos_incompletos` | Triaje | El archivo no es un ticket válido (ej. ya es un CFDI) | Solicitante notificado |
| `ticket_ilegible` | Triaje | OCR no extrajo fecha o datos clave | Solicitante notificado |
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
gilmdb -f ~/IA-WARP/MES/schema_solicitudes_facturacion.sql
gilmdb -f ~/IA-WARP/MES/add_email_solicitante.sql
gilmdb -f ~/IA-WARP/MES/add_notif_operador.sql
```

### Variables de entorno (User scope en Windows)
```powershell
[Environment]::SetEnvironmentVariable("MISTRAL_API_KEY", "<key>", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "<key>", "User")  # opcional Tier 2
```

### Task Scheduler
```powershell
schtasks /Create /TN "GILM_SyncSolicitudesKVtoPG" `
  /TR 'cmd /c "C:\Users\...\IA-WARP\MES\sync_solicitudes_wrapper.cmd"' `
  /SC MINUTE /MO 5 /RL LIMITED
```

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
- Logs del pipeline: `~/IA-WARP/MES/logs/sync_solicitudes_YYYYMMDD.log`
- Estado del Task Scheduler: `Get-ScheduledTaskInfo "GILM_SyncSolicitudesKVtoPG"`
- KV en Cloudflare: `cd ~/IA-WARP/facturas-intake-worker && wrangler kv key list --binding TICKETS --remote`
- Worker logs: dashboard de Cloudflare

## Roadmap

### Próximo (cuando se quiera)
- **Tier 2 autónomo**: el código está listo en `~/IA-WARP/MES/tier2/`, sólo falta API key y testing. Una vez activado, el agente Claude Agent SDK navegará portales solo y sólo escalará a humano para CAPTCHAs.

### Futuro (si crece el volumen)
- **Worker Cron Triggers** para liberar la PC: requiere migrar el datastore a cloud (Neon/Supabase) o exponer Postgres via Cloudflare Tunnel.
- **Servicio de anti-CAPTCHA** (2Captcha API, ~$3/1000) para reducir intervención humana a 0.
- **Dashboard interno** con métricas: tasa de éxito por cadena, tiempo medio de procesamiento, $ recuperados, etc.

---

## Cadenas catalogadas (a hoy)

| Cadena | Plazo | Portal |
|---|---|---|
| 🟢 7-Eleven | Mes + 5 días | e7-eleven.com.mx |
| 🟢 Walmart | Mismo mes | factura.walmart.com.mx |
| 🟢 OXXO | Mismo mes | factura.oxxo.com |
| 🟢 Costco | Mismo mes | costco.com.mx/facturacion |
| 🟢 Home Depot | Mismo mes | homedepot.com.mx |
| 🟢 Soriana | Mismo mes | facturacion.soriana.com |
| 🟢 H-E-B | Mismo mes | facturacion.heb.com.mx |
| 🟢 Soft Restaurant (El Frison, etc.) | 7 días | mefacturo.mx/{slug} |
| 🟡 Farmacias Benavides | 30 días naturales | e-facturate.com/benavides |
| 🟡 Farmacias del Ahorro | 30 días naturales | l.fdah.mx |
| 🔴 Farmacia Guadalajara | **72 horas** (¡ojo!) | farmaciasguadalajara.com/facturacion-electronica |

---

_Proyecto interno GILM · 2026_
