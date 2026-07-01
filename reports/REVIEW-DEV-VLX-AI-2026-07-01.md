# Review dev.vlx.ai — Seguimiento post-auditoría pre-migración
**Fecha:** 2026-07-01 · **Auditora:** Laura Ceballos / Software Craft CR
**Objeto:** Nueva versión del sitio en `https://dev.vlx.ai` (protegido con AWS ALB + Amazon Cognito)
**Baseline de comparación:** [`vlx-auditoria-pre-migracion-master-es-2026-06-19.md`](../auditoria-pre-migracion-2026-06-19/reportes/vlx-auditoria-pre-migracion-master-es-2026-06-19.md) (score 81/100, "Adelante con condiciones")

---

## ⚠️ Nota metodológica — leer antes que el resto

1. **Acceso:** `dev.vlx.ai` está protegido por AWS ALB + Cognito (login OAuth2 por email/password, no auth básica). El navegador ya tenía sesión activa — no llegué a probar las credenciales que me diste (`laura.ceballos@softwarecraftcr.com`), no hizo falta.
2. **Herramientas externas bloqueadas:** PageSpeed Insights, Lighthouse remoto, CrUX y cualquier crawler público **no pueden auditar esta URL** — no tienen la cookie de sesión de Cognito. Todo lo de performance en este reporte viene de Navigation Timing API medido manualmente en el navegador autenticado, no de Lighthouse.
3. **GSC/GA4:** `dev.vlx.ai` no es una propiedad verificada — no aplica baseline de tráfico real todavía (es pre-lanzamiento).
4. **Backlinks / Keywords / ASO / GMB:** sin cambios desde la auditoría del 19-jun (no dependen del código del sitio) — no se repitieron en esta pasada.
5. **🔴 Duda para validar con Hans/Abe:** el H1 de homepage ahora dice *"The AI-Native Enterprise Platform for [Inspections/...]"* (animación tipo typewriter) — un posicionamiento de marca distinto al que vimos el 19-jun (*"Run Your Inspection Business on VLX"*). Puede ser evolución normal de la rama `Hans-review`, o puede ser una rama/ambiente distinto. **Confirmar con el equipo qué rama está desplegada en `dev.vlx.ai`** antes de dar por buena esta comparación 1:1.

Método usado: crawler propio ejecutado dentro del navegador autenticado (`fetch()` desde la página, hereda la cookie de sesión) sobre las **185 URLs del sitemap** + pruebas puntuales de URLs antiguas de WordPress.

---

## 1. Estado de los 7 arreglos obligatorios (must-fix del 19-jun)

| # | Arreglo | Estado | Evidencia |
|---|---|---|---|
| 1 | LCP del hub 7.9s | 🟡 **Probablemente resuelto** (no verificable con precisión) | Navigation Timing en `/digital-inspections-software/`: TTFB 82ms, load event 354ms — órdenes de magnitud mejor. No pude capturar el LCP exacto (PerformanceObserver no devolvió entradas vía automatización) ni correr PSI (bloqueado por Cognito). |
| 2 | `aggregateRating` 4.9/20 hardcodeado | ✅ **Resuelto** | Barrido completo de las 185 páginas: `aggregateRating` no aparece en ninguna. Lo quitaron en vez de arreglarlo — opción más segura. |
| 3 | 5 URLs `/blog/category/*` con 308 en sitemap | ✅ **Resuelto** (parcialmente reemplazado por un hallazgo nuevo, ver abajo) | Las 8 URLs de categoría en el sitemap devuelven `200` directo, no redirigen. |
| 4 | 2 redirects `/use-case/` con 404 | ✅ **Resuelto** | `/use-case/a-guide-to-prevent-insurance-fraud` → `/digital-inspections-software/insurance/` (200); `/use-case/strengthening-trust-in-financial-lending-...` → `/digital-inspections-software/financial-services/` (200). Exactamente los destinos recomendados. |
| 5 | Redirects de landing pages pagas (Capterra/G2/LinkedIn) | ❌ **NO resuelto** | Probadas las 13 LPs sin redirect de `AUDIT-MIGRATIONS-EN-2026-05-28.md` → **las 13 siguen dando 404** (`/landing-inspection-software-capterra-a/`, `/landing-inspection-software-g2/`, `/landing-inspections-software-linkedin/`, etc.). Las 3 que sí redirigían siguen apuntando a páginas genéricas, no a `/lp/[campaña]/`. Y las rutas `/lp/capterra/`, `/lp/g2/`, `/lp/linkedin/` **tampoco existen** (404) — la carpeta `/lp/` está en robots.txt (disallow) pero no tiene contenido todavía. |
| 6 | H1 homepage "VerificationInspections" sin espacio | ✅ **Resuelto** | HTML actual: `"The AI-Native Enterprise Platform for Inspections"` — espaciado correcto, sin concatenación. |
| 7 | SSR blog + paginación rastreable | 🟡 **Parcial — con regresión nueva** | Ver hallazgo nuevo abajo: la paginación de **categorías** de blog quedó bien, pero la paginación **principal** del blog se rompió de otra forma. |

**Resumen: 4 resueltos, 1 no resuelto, 2 parciales.**

---

## 2. Recomendados (no bloqueantes) — estado

| Ítem | Estado | Evidencia |
|---|---|---|
| 4 H1 de industria copywriting (construction/safety/hospitality/mining) | ✅ **Resuelto** | Los 4 ahora son keyword-led: *"Construction Inspection Software for Jobsite Safety & QA"*, *"Safety Inspection Software for EHS Teams"*, *"Hospitality Inspection Software for Brand-Standard Audits"*, *"Mining Inspection Software for Safety & Compliance"*. 12/12 industrias con H1 orientado a keyword (antes 8/12). |
| 6 industrias faltantes en menú/footer | ✅ **Resuelto** | Footer + mega-menú ahora enlazan las 18 industrias + KYPiT (antes faltaban vehicle, construction, safety, hospitality, mining, inspection-companies). |
| Hub → body links a clusters | ✅ **Mejorado** | El hub ahora enlaza las 19 páginas hijas (18 industrias + KYPiT) desde el cuerpo — antes 18. |
| Imágenes OG propias | 🟡 **Sigue pendiente** | **46 de 185 páginas** siguen sin `og:image`: las 12 páginas de industria/app, 11 posts de blog, y 22 páginas más (pricing, integrations, demo, checklists index, whats-new, security, developers, etc.). |
| Testimonios aprobados por cliente | ⚪ No verificable desde el crawl (requiere confirmación del cliente, no del código) |

---

## 3. Hallazgos NUEVOS (no estaban en la auditoría del 19-jun)

### 🔴 NUEVO-1 — Paginación principal del blog: sitemap dice una cosa, el sitio hace otra
El sitemap lista `/blog/page/2/` … `/blog/page/10/` (9 URLs, formato de ruta limpio — lo que se pidió). Pero al visitarlas:
- **Redirigen** a `/blog/?page=N` (parámetro de consulta)
- Esa página de destino tiene **`noindex, nofollow`** y **canonical → `/blog/`** (página 1)

Es decir: Google va a rastrear la URL del sitemap, lo van a rebotar a una URL con parámetro, y esa URL le va a decir "no me indexes, mi canonical es otra página". Señal contradictoria + presupuesto de rastreo desperdiciado en 9 URLs.

**Contraste:** la paginación de **categorías** (`/blog/category/industry-trends/page/2/`) sí funciona bien — 200 directo, `index, follow`, canonical auto-referenciado. Es decir, arreglaron un tipo de paginación y rompieron el otro.

### 🟡 NUEVO-2 — Dos slugs viejos de WordPress sin redirect
`/oil-and-gas/` y `/financial-institutions/` (nombres pre-cambio de slug) devuelven **404** en vez de redirigir a `/digital-inspections-software/oil-gas/` y `/digital-inspections-software/financial-services/`. Tráfico histórico bajo, pero es higiene de migración pendiente.

### 🟡 NUEVO-3 — Títulos y meta descripciones fuera de rango
- **39/185 páginas** con title vacío o >60 caracteres
- **47/185 páginas** con meta description vacía o >160 caracteres

No se había medido esto explícitamente el 19-jun; vale la pena una pasada de limpieza antes de lanzar.

### 🟢 NUEVO-4 — Compare pages ganaron schema básico
`/compare/vlx-vs-safetyculture/` y `/compare/vlx-vs-goaudits/` pasaron de **0 schema** a `Organization + WebSite + BreadcrumbList`. Sigue faltando un schema específico de comparación/producto, pero es mejora neta.

### ⚪ NUEVO-5 — Iconos decorativos sin `alt` (severidad baja)
32/37 imágenes en el hub sin atributo `alt` — son íconos SVG pequeños junto a los links de industria (ej. `/images/icons/construction.svg`), redundantes con el texto del link. Baja prioridad; lo ideal es `alt=""` explícito en vez de omitir el atributo, pero no es el mismo problema que el de "Trusted customer" genérico en logos de clientes reportado antes (eso no se volvió a probar en esta pasada).

### ⚪ NUEVO-6 — Sitemap creció de 175 → 185 URLs
10 URLs nuevas desde el inventario del 19-jun (no se investigó el detalle línea por línea, solo el conteo).

---

## 4. Lo que se confirma sólido (sin cambios para mal)

- Sitemap 100% canonicalizado a `vlx.ai` (dominio de producción) aunque se sirve desde `dev.vlx.ai` — correcto para cuando lance, solo hay que recordar apuntar las herramientas de auditoría al host de dev mientras tanto.
- robots.txt con postura de IA deliberada intacta (Perplexity/OAI-SearchBot/Applebot permitidos, GPTBot/ClaudeBot/CCBot/Amazonbot bloqueados).
- SSR funcionando: todas las páginas devuelven HTML completo con H1/schema/links sin necesitar JS (confirmado vía `fetch()` crudo, que es exactamente lo que ve Googlebot).

---

## 5. Limitaciones de esta pasada (a diferencia de la auditoría de 12 dimensiones del 19-jun)

No se repitieron por falta de acceso de herramientas o porque no aplican a un entorno pre-lanzamiento sin dominio verificado:
- **Performance real (Lighthouse/PSI/CrUX)** — bloqueado por el muro de Cognito
- **GSC/GA4 baseline** — `dev.vlx.ai` no es propiedad verificada
- **Backlinks / Keywords / SERP competidores** — sin cambios desde el 19-jun
- **ASO / GMB** — no dependen del código del sitio
- **GEO/IA (prompts en ChatGPT/Perplexity)** — no se repitió, `llms.txt` sigue presente (5.9KB) pero no se auditó contenido

---

## 6. Próximos pasos recomendados

1. **Confirmar con Hans/Abe** qué rama/ambiente es exactamente `dev.vlx.ai` (el copy de homepage cambió de forma material vs. lo visto el 19-jun).
2. **Arreglar las 13 landing pages pagas** — sigue siendo el único bloqueador crítico sin resolver de la lista original; con inversión activa en Capterra/G2/LinkedIn esto es dinero cayendo en 404.
3. **Corregir `/blog/page/N/`** — o quitarlas del sitemap, o hacer que sirvan contenido real en esa ruta (no redirigir a `?page=N` con noindex).
4. **Agregar los 2 redirects viejos** (`/oil-and-gas/`, `/financial-institutions/`).
5. **Rellenar los 46 `og:image` faltantes**, priorizando las 12 páginas de industria (son las de mayor intención comercial).
6. **Pasada de títulos/meta descriptions** en las ~40-47 páginas fuera de rango.
7. Una vez el sitio esté accesible sin Cognito (o me den una forma de bypass), correr PSI/Lighthouse real sobre el hub para confirmar el LCP.
