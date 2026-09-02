# CLAUDE.md — cotizador-adhesivos (ADH)

## AYJ Maquinaria — Ecosistema de Cotizadores

Este repo es parte de un ecosistema de 5 cotizadores para AYJ Maquinaria S.A.S. (35+ años, Bogotá y Medellín). Cada uno es un **single-file HTML** (~5.000-6.000 líneas, HTML+CSS+JS vanilla, sin frameworks) que se despliega directo con GitHub Pages.

| Cotizador | Repo | Estado |
|---|---|---|
| MQ V1 — Máquinas (estable) | aristi01AYJ/cotizador-ayj | producción |
| MQ V2 — Máquinas (desarrollo) | aristi01AYJ/cotizador-ayj-v2 | desarrollo activo |
| SW — Software imos | aristi01AYJ/cotizador-software | producción |
| ADH — Adhesivos | aristi01AYJ/cotizador-adhesivos | producción |
| SVC — Servicios | aristi01AYJ/cotizador-service | producción |

**Regla de sincronización:** en Máquinas, V2 es donde se desarrolla y prueba; cuando un fix o feature está estable, se replica en V1 (mismo cambio, mismo archivo `index.html`).

### Stack técnico
- Frontend: HTML + CSS + JS vanilla, un solo archivo `index.html` por cotizador
- Auth: MSAL.js + Azure AD (Entra ID)
- Backend de datos: Microsoft Graph API v1.0 → listas de SharePoint Online
- Hosting: GitHub Pages
- TRM: BanRep (datos.gov.co), con fallback a open.er-api.com (ADH usa BanRep × 1.15)

### ⚠️ Credenciales — NUNCA hardcodear en este repo (es público)
Este repositorio es **público** (GitHub Pages gratis). El token de GitHub, el Azure AD App ID, los `siteId` de SharePoint y la clave del módulo MG viven en el documento de handoff privado del Proyecto de Claude **"APP PARA AYJ - COTIZADOR"** — pídeselos a Adolfo o consulta ese documento, nunca los pegues en código ni los subas a este repo. Si necesitas guardarlos localmente para trabajar, usa un `.env` o similar que esté en `.gitignore`.

### Workflow de edición de código
1. Editar `index.html` localmente (o vía Claude Code con el repo clonado).
2. **Validar el JS antes de cualquier commit**: extraer el bloque `<script>...</script>` principal y correr `node --check`. Nunca subir sin este paso.
3. Commit descriptivo, push a `main` (GitHub Pages se actualiza solo).
4. Si el fix aplica también a la otra versión de Máquinas (V1 ↔ V2), replicarlo ahí igual.
5. Actualizar el handoff del Proyecto de Claude con el fix aplicado (fecha, archivo, línea, qué cambió) para que la próxima sesión tenga contexto.

### Reglas críticas Graph API / SharePoint (aplican a todos)
- NUNCA usar `$select=fields` en queries → error 400. Usar `expand=fields&$top=500`.
- Campo NIT interno: `NIT_x002f_RUT` — leer con `.toString().trim()`.
- Campos tipo Hyperlink (LinkPDF, Ficha Técnica): `{Url:"...", Description:"..."}`.
- Clientes/Contactos: SIEMPRE en el `siteId` de Comercial (todos los cotizadores, incluso SVC).
- **SIEMPRE usar `graphGetAll()` (pagina con `@odata.nextLink`) para cualquier query que pueda superar 500 ítems** — especialmente `generarNumOferta()`. `graphGet()` con `$top=500` solo trae la primera página; una vez la lista de Cotizaciones supera 500 registros, el consecutivo se queda pegado para siempre porque deja de ver los ítems más nuevos (bug real, 31-ago-2026, afectó los 5 cotizadores).


## Este repo específico: Adhesivos

- Moneda: siempre **COP**. TRM = BanRep × 1.15 (no usar la TRM directa de BanRep como en los demás).
- Ficha Técnica (PDF) visible en catálogo (ícono), carrito y PDF final ("📄 Ver Ficha Técnica").
- Botón ✏️ de edición está en la tarjeta del catálogo, no en el carrito.
- Modal de edición: Nombre, Marca, Modelo, Tipología, Presentación, TARIFA, COSTO_AYJ, CostoTotal, Moneda, URL Ficha.
- Dos modos de guardado: "Solo en cotización" vs "Guardar + actualizar SP" (este último hace PATCH a `ADHESIVOS_LIST`).
- Campo Ficha Técnica en SharePoint: `Ficha_x0020_Tecnica`, tipo Hyperlink `{Url, Description}`.
- Pendiente: verificar el nombre interno en SharePoint de `Presentación` para la edición inline (puede tener sufijo tipo `_x00f3_n`). El PATCH de guardado (`ei_presentacion` → SP) solo escribe a `Presentaci_x00f3_n`; si en algún tenant el campo real es `Presentacion` a secas, ese guardado fallaría silenciosamente — revisar si se reporta.
- **Pedido a Fábrica** (botón en el menú de escritorio, junto a Liquidador): modal aislado del carrito de cotización para armar la orden de compra al proveedor. Busca ítem del catálogo (o texto libre) + cantidad → PDF agrupado por marca/proveedor, con costo AYJ (no la tarifa de venta al cliente) y la **TRM real de BanRep** (`trmBanRepPura`, sin el +15%). No guarda historial en SharePoint todavía — solo PDF descargable. Funciones: `abrirPedidoFabrica`, `agregarItemPedidoFab`, `generarPDFPedidoFabrica`, `cargarTRMBanRepPuraPF`.

### Fixes recientes
- **02-sep-2026 — Nueva feature: Pedido a Fábrica.** Ver sección arriba.
- **31-ago-2026 — PDF mostraba Descuento en USD en modo COP; edición de precio era en USD.** En `generarPDF()`, la fila "Descuento" no respetaba `d.modoCOP` (Subtotal e IVA sí). En `renderCarrito()`, el input de precio editable (🔓 Precio) siempre era en USD sin importar el modo de moneda — ahora sigue `body.modo-cop` y usa la nueva `editarPrecioItemCOP()`.
- **31-ago-2026 — Consecutivo de cotización pegado (COT-2026-435 repetido en 5 cotizaciones seguidas).** `generarNumOferta()` usaba `graphGet()` sin paginar; se cambió a `graphGetAll()`. Ver regla en la sección de Graph API arriba.
- **28-ago-2026 — Unidad de Medida (Presentación) no llegaba al Carrito ni a la Cotización.** Causa: `cargarMaquinas()` sí leía `presentacion` de SharePoint por ítem individual, pero `procesarCatalogo()` no la copiaba al agrupar en `grupos` (línea ~1371), y además faltaba en 3 sitios donde se hace `carrito.push(...)`: `toggleBP` (Bajo Pedido), `liqGetItem` (Liquidador) y `duplicarOferta` (cargar cotización guardada). Se agregó `presentacion` en los 4 puntos. También se cambió el badge de `📦 <valor>` a `U.M.: <valor>` en el carrito y en el PDF para que sea inequívoco que es la unidad de medida.
