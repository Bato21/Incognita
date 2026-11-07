# Plan · decisiones de la entrevista (28 oct 2026)

Complementa docs/plan-mvp.md. Si algo choca, manda este archivo.

## Polo norte (borrador de 00-norte.md)

**Para qué:** una app publicada en App Store y Google Play, construida por mí solo con agentes de IA, que mis amigos usan y que entiendo al 100%. Es una pieza de portafolio.

**Para quién, primero:** Para futuros usuarios, jovenes de 18 a 40 años, personas viajeras que quieran dejar plasmado sus viajes y ver a lo largo del tiempo que partes  del mundo han descubierto.

**Desempate:** entender todo no se negocia. Si algo no alcanza, cede el plazo, no el entendimiento.

**Éxito**
- Publicada en ambas tiendas.
- Al menos 200 personas la usan un mes después de publicar. 
- Puedo explicar sin mirar cualquier parte: arquitectura, RLS, antitrampa y subida de fotos.

**Plazo:** por hitos, sin fecha. Las semanas del plan son estimaciones a ~8 h por semana; mis horas son variables.

**Principios del producto**
- P1. Desbloquear exige estar ahí: una lectura al abrir la app, sin GPS en segundo plano.
- P2. Nunca se guardan coordenadas: solo usuario, zona y fecha.
- P3. Privado por defecto; lo público se muestra por mes y año, nunca en tiempo real.
- P4. Los recuerdos se ven distintos y no cuentan en las estadísticas verificadas.
- P5. Bilingüe es/en desde el día 1.
- P6. Gratis de operar hasta tener usuarios reales. La app debe ser gratuita, de ninguna manera puede ser obligatorio ver anuncios o una suscripción. 

**Principios del proceso**
- P7. Nada entra a main si no lo puedo explicar.
- P8. Toda decisión cara de revertir pasa por /debatir y queda en un ADR.
- P9. Una etapa a la vez; termina cuando cumple su "listo cuando".
- P10. Repo público: cero secretos en git.

**No-objetivos del MVP:** grupos, comentarios, mensajes, pueblos y localidades, perfil web, monetización.

## Decisiones nuevas (cada una será un ADR)

| # | Decisión | Por qué |
|---|---|---|
| D1 | Modo recuerdo en cualquier momento, con estilo propio y fuera de las estadísticas verificadas | Evita el mapa vacío y cubre cuando olvidé tocar el botón |
| D2 | Código en inglés; app en es/en con i18n desde E0; cerebro en español | Las tiendas y los permisos piden inglés igual |
| D3 | Repo público desde el primer commit | Portafolio desde el día 1 |
| D4 | Diseño en Claude Design: sistema de diseño + 5 pantallas; Claude Code lo implementa | Diseño resuelto antes de codificar pantallas |
| D5 | iOS: build de desarrollo con Apple ID gratis en el Mac de un amigo hasta E5; Apple Developer (US$99) en E5 | Ver "iOS sin Mac propio" |
| D6 | Modelo opusplan: Opus en plan mode, Sonnet al implementar | Plan Pro: estirar límites compartidos con claude.ai |
| D7 | Supabase: proyecto dev ahora, prod en E6; el esquema cambia solo con migraciones; MCP solo contra dev y en solo lectura | Nunca exponer datos reales al agente |

Pendientes para /debatir: licencia del repo (E0, primer debate) y nombre final + bundle id (antes de E5). Nombre en clave: Around The World.

## iOS sin Mac propio

- **Día a día:** emulador Android en mi PC, más la app de desarrollo en mi iPhone, que carga el JavaScript desde el PC por Wi-Fi. Los cambios de JS no necesitan Mac.
- **Cuándo voy al Mac de un amigo:** cada 7 días para reinstalar, y cuando cambia una dependencia nativa. Por eso, todas las librerías nativas del MVP se instalan juntas al inicio de E2.
- **Firma:** con mi Apple ID en su Xcode; cierro sesión al terminar.
- **Build gratis:** se hace con FREE_PROVISIONING=1, que saca de app.config.ts el login con Apple y las push (la firma gratis no las admite).
- **Kit para el amigo (E2):** scripts/ios-dev-build.sh + docs/mac-kit.md con requisitos, pasos y cómo devolverme el resultado. Sirve también para QA en su simulador.
- **E5:** pago Apple Developer; desde ahí, builds internos por EAS y TestFlight, sin Mac.

## Cómo trabajo con el agente

- **Estilo mentor:** output style propio, basado en Explanatory. Explica el porqué de cada decisión y patrón, no es complaciente, discute cuando mi razonamiento es débil y me manda a fuentes primarias.
- **/debatir:** obligatorio en decisiones que van a ADR; a pedido en el resto. Protocolo:
  1. Claude plantea la decisión y qué está en juego.
  2. Doy mi postura antes de ver la suya.
  3. Claude presenta 2–3 alternativas con fuentes y hace de abogado del diablo.
  4. Investigo y defiendo.
  5. Se decide. El ADR lleva el porqué en mis palabras y qué nos haría cambiar de opinión.
- **Sesión:** un ítem por sesión. Plan mode → implementar → evidencia → guardian-norte → /cerrar-sesion.
- **Git:** una rama y un PR por ítem; desde E4, CI y revisión de Claude en cada PR; yo apruebo y mergeo.

## Etapas: cambios y técnica de IA

| Etapa | Cambios al plan | Técnica de IA |
|---|---|---|
| E0 | i18n es/en; diseño en Claude Design; secret scanning y push protection en GitHub; primer /debatir: licencia | Context engineering: cerebro, hook de sesión, estilo mentor, /debatir, guardian-norte |
| E1 | MCP de Supabase solo contra dev, en solo lectura | Subagentes de investigación; TDD: los 20 puntos se escriben antes que zona_en |
| E2 | Login con email y Google (Apple pasa a E5); dependencias nativas juntas; primer build en el Mac de un amigo; emulador Android; ubicación simulada solo en desarrollo | Verificación determinista: hook Stop que no deja cerrar con tests en rojo; kit del Mac para tus amigos |
| E3 | Sin cambios | Mi primera skill (pipeline de fotos: comprimir, quitar EXIF, verificar) evaluada con claude plugin eval contra una línea base sin ella |
| E4 | Push probadas en el emulador Android; lista de 15–20 testers Android con su correo de Google | GitHub Actions: CI y revisión de Claude en cada PR con el token de mi plan Pro |
| E5 | Pagar Apple y Google Play; Sign in with Apple y push en el iPhone; nombre final y bundle id | Evals del guardián: diffs con trampas sembradas (guardar coordenadas, tabla sin RLS, GPS en segundo plano) y medir cuántas detecta; subagente de seguridad |
| E6 | TestFlight y prueba cerrada en Play; proyecto Supabase prod | Workflow programado que resume el feedback de testers y abre issues |
| E7 | Sin cambios | Retro: podar instrucciones con aprendizajes-ia.md y /doctor prompt-audit |

## Costos y riesgos que cambian

**Costos**
- Claude Pro ya lo pago. Claude Code, la revisión en GitHub y las evals gastan el mismo límite que el chat; revisar /usage.
- Apple (US$99) y Google Play (US$25) en E5, como en el plan.

**Riesgos**
- Sin Mac disponible, o con un Xcode desactualizado: el emulador Android es el día a día; tener dos amigos con Mac.
- La app de desarrollo caduca cada 7 días: visita semanal, o semanas solo con emulador.
- Faltan testers Android: reclutar 15–20 desde E4.
- El debate frena demasiado: solo es obligatorio en decisiones que van a ADR.
- Repo público con un agente en CI: la acción solo se dispara con mis PRs y comentarios; los forks no reciben secretos.