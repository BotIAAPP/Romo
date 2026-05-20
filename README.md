# Portal Facturador · Aliacer

Portal web para solicitar facturas CFDI 4.0 a Perfiles LM o Galvasid.

**URL pública:** https://botiaapp.github.io/Romo/

## Flujo

1. Elige el RFC receptor (Perfiles LM o Galvasid)
2. Sube la foto/PDF/XML del ticket
3. Indica el correo destino
4. Confirma y envía

La solicitud llega vía Web3Forms al correo del operador, quien (o el agente `facturador` de Claude Code) procesa el ticket, navega el portal de la cadena, llena los datos fiscales del RFC elegido y devuelve el CFDI por correo.

## Stack

- HTML/CSS/JS estático, sin build, sin dependencias de servidor.
- Google Fonts: Saira, Saira Condensed, JetBrains Mono.
- Backend de intake: [Web3Forms](https://web3forms.com) (free tier, sin signup necesario).
- Hosting: GitHub Pages.

## Cambiar el `access_key`

Edita `index.html` y modifica la constante `WEB3FORMS_KEY` en el bloque `<script>`.
