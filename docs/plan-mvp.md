# Plan MVP · Incógnita

Versión 2.3 · 7 nov 2026 · Baptiste Vial

Este archivo es el **qué** (producto, técnica, etapas). El **cómo trabajo** (polo norte, principios P1–P10, decisiones D1–D7, iOS sin Mac, /debatir, técnicas de IA por etapa) está en [plan-decisiones.md](plan-decisiones.md). Si algo choca, manda plan-decisiones.md.

Los puntos marcados **[Qn]** siguen abiertos y se cierran en la sección [Preguntas abiertas](#preguntas-abiertas).

**Identidad:** nombre **Incógnita** (de *terra incognita*, el mapa por descubrir), igual en es/en. Título en tiendas: "Incógnita: tu mapa de viajes" / "Incógnita: Travel Map", para no confundirse con el modo incógnito. Bundle id y package de Android: `io.github.bato21.incognita`. Esquema de deep links: `incognita://`. El repo también se llama `Incognita` (antes Around-the-World).

**Estado del repo:** solo documentación. La E0 no ha empezado: aún no existen la app Expo ni el cerebro (`docs/brain/`).

---

## 1. Resumen

App gratis para iOS y Android: desbloqueas lugares del mundo **estando ahí** y en cada lugar guardas fotos ordenadas por mes y año. Tu perfil es un globo con lo que has descubierto, más un álbum por lugar, y lo ven tus amigos u otros usuarios según tu privacidad.

- **Desbloquear:** al abrir la app se lee la ubicación una vez. Si estás en un lugar nuevo, aparece un aviso y lo confirmas con una foto. No hay GPS en segundo plano (P1).
- **Viajes pasados:** se marcan como recuerdo, a mano o a partir de la ubicación guardada en tus fotos. Se ven distintos y no cuentan en las estadísticas verificadas (P4, D1).
- **Modo de juego:** Viajero (países) o Explorador (regiones/estados) define qué se desbloquea en el globo. Se elige al registrarse y se cambia en el perfil.
- **Social:** perfil privado por defecto, seguidores con solicitud, feed y página pública de cada lugar.
- **Meta:** publicada en ambas tiendas, con al menos 200 usuarios activos un mes después de publicar, y que yo pueda explicar cada parte sin mirar.

## 2. Decisiones de producto

### 2.1 Desbloquear un lugar (flujo "estoy aquí")

1. Al abrir la app se pide ubicación **solo mientras se usa** y se toma una lectura.
2. El servidor responde qué país y región corresponden a ese punto (`zona_en`, una RPC de Postgres).
3. Si hay algo nuevo, aparece arriba del globo el aviso **"Lugar desbloqueable aquí"**, que se puede descartar.
4. Al tocar el aviso se abre un panel con lo que se desbloquearía ("país nuevo", "región nueva") y tres opciones: **Tomar foto**, **Elegir de la galería** o **Ahora no**. La foto es **opcional**: "Ahora no" desbloquea igual y la foto se puede agregar después.
5. **Confirmar zona:** muestra la foto, un minimapa y la zona detectada por niveles (país y región), indicando cuáles ya son tuyos y cuáles son nuevos.
6. **"No es aquí":** propone las zonas vecinas para corregir. Solo se aceptan las que limitan con la zona detectada y quedan dentro del margen de precisión de la lectura, así que no sirve para desbloquear lugares lejanos.
7. Al confirmar, el servidor vuelve a validar y guarda solo usuario, zona y fecha. Las coordenadas no se guardan (P2).
8. Desbloquear una región desbloquea también su país.
9. Pantalla de éxito con el botón "Ver en mi globo".

El botón **+** de la barra ofrece dos caminos: "Estoy aquí ahora" e "Importar viajes pasados desde mis fotos".

**Antitrampa.** Ninguna medida es perfecta; juntas bastan para un proyecto personal.
- Se rechazan las lecturas con precisión peor que unos 100 m.
- En Android se rechazan las ubicaciones simuladas, usando el campo `mocked` de expo-location.
- Se marcan los viajes imposibles comparando el centroide de la última zona con el de la nueva y el tiempo entre ambas, sin guardar coordenadas.
- Hay un límite de desbloqueos por hora.
- Si el usuario solo concede ubicación aproximada, solo se desbloquean las zonas que contienen el círculo de precisión completo; si toca un borde regional, solo el país.
- La ubicación simulada para probar existe solo en builds de desarrollo.

### 2.2 Modos de juego

El modo define la **unidad desbloqueable** del mapa.

| Modo | Unidad desbloqueable | Qué se ve | Contadores del perfil |
|---|---|---|---|
| Viajero | País | Las divisiones internas se dibujan como líneas, pero no son áreas por desbloquear | Países, continentes y fotos |
| Explorador (recomendado) | Primer nivel administrativo: regiones de Chile y Francia, estados de EE. UU. | Países y regiones/estados | Países, regiones y fotos |

- Se elige en el paso 2 de 3 del registro ("¿Cómo quieres explorar?") y se cambia en el perfil. Una etiqueta sobre el globo muestra el modo actual y lleva al perfil.
- **Implementación:** siempre se guarda el desbloqueo al nivel más fino disponible en ese punto, sea cual sea el modo. El modo decide qué unidad se pinta y se cuenta. Así, un Viajero que pasa a Explorador ya tiene sus regiones, y cambiar de modo nunca borra nada.
- **Argonauta** (máximo detalle: comunas, pueblos, islas) queda para después del MVP.

### 2.3 Zonas

- **MVP:** países y primer nivel administrativo (regiones/estados) de todo el mundo. Las comunas de Chile llegan con Argonauta, después del MVP.
- **Fuentes [Q8]:**
  - geoBoundaries, con licencia CC BY que permite uso comercial. GADM queda descartado porque no permite uso comercial sin permiso.
  - Natural Earth, de dominio público, que es la que usó el prototipo de diseño.
- **Buscar y dibujar van separados:**
  - En la base solo van las geometrías de búsqueda, partidas con `ST_Subdivide` en `zone_parts` con índice GIST.
  - Los contornos que se dibujan son archivos estáticos simplificados en R2, en dos niveles de detalle según el zoom (país y región).
  - La base solo devuelve los IDs desbloqueados. Así ya no hace falta un endpoint de contornos.
- **Después del MVP:** ciudades del mundo, pueblos, recintos e hitos, desde OpenStreetMap.

### 2.4 Recuerdos y viajes pasados

- **Modo recuerdo (D1):** se puede marcar en cualquier momento una zona del pasado. Se ve con otro estilo en el globo y no cuenta en las estadísticas verificadas. Evita empezar con el mapa vacío y cubre las veces en que olvidaste tocar el botón.
- **Importar desde fotos (entra al MVP):** se importan las fotos que el usuario **elige** con el selector del sistema, en selección múltiple, sin recorrer toda la galería, para cumplir la política de fotos de Play. Leer la ubicación de esas fotos se valida con un spike en la E0: en Android probablemente requiera `ACCESS_MEDIA_LOCATION` y en iOS el acceso al asset. El flujo ya diseñado se ajusta así:
  1. Un permiso que explica que solo se leen la ubicación y la fecha de las fotos, y que nada se desbloquea sin tu confirmación.
  2. Elegir las fotos (selección múltiple) y una barra de progreso mientras se revisan.
  3. Los resultados ("Encontramos 6 lugares nuevos"), con fotos, fecha y qué desbloquea cada uno; se pueden desmarcar.
  4. Las fotos que se suman a lugares que ya tienes, y las fotos sin ubicación, para asignarlas a mano.
  5. Un resumen final.
- La ubicación de cada foto se lee **en el teléfono** y solo se envía a `zona_en` para saber su zona; no se guarda (P2). Lo importado entra como **recuerdo** (`metodo = 'foto_exif'`), porque los EXIF se pueden editar.

### 2.5 Fotos por lugar y mes/año

- Solo se suben fotos a zonas desbloqueadas o marcadas como recuerdo.
- El mes y el año se sugieren desde la fecha de la foto y se pueden editar.
- Antes de subir, la foto se comprime a unos 1.600 px y se le **quitan los metadatos**, sobre todo el GPS. Se comprueba con exiftool.
- Se usa el selector de fotos del sistema, porque Google Play restringe el permiso de acceso a toda la galería. La importación también lo usa.
- Tope inicial de 10 fotos por zona y mes, para controlar costos.
- **Pin por foto:** cada foto subida deja un pin en el globo, en la ubicación de la foto (EXIF o la lectura al desbloquear). Al tocar el pin se abre el álbum de esa zona. La ubicación se extrae en el teléfono antes de quitar el EXIF del archivo y se guarda aparte en `photos.pin`. **La precisión y quién ve el pin exacto están abiertos [Q12]**, porque guardar esta ubicación cambia P2.

### 2.6 Social

- Perfil público o privado, con seguidores mediante solicitud.
- Feed simple de las personas que sigues, con sus desbloqueos y fotos nuevas, sobre la tabla `actividad`.
- Página pública de cada lugar con las fotos de todos, filtrables por mes y año.
- Solo "me gusta" y notificaciones push de solicitudes y me gusta.
- Tarjeta para compartir en stories.
- **Fuera del MVP:** grupos y globos compartidos (están en el prototipo de diseño, pero no entran) [Q5], comentarios, mensajes, perfil web y monetización.

### 2.7 Privacidad y cumplimiento

- La cuenta es privada por defecto. Al hacerla pública, cualquiera ve tu globo y tus fotos.
- Lo público se muestra por mes y año, nunca en tiempo real (P3).
- Desde la app se puede reportar, bloquear y borrar la cuenta; Apple lo exige. Para Play también hace falta una página web de borrado.
- Edad mínima de 16 años, según la Ley 21.719. Su vigencia es el 1 de dic de 2026, con un proyecto de ley en trámite para aplazarla a dic de 2027; hay que revisar su estado antes de la E5.
- La política de privacidad, los términos y el correo de contacto se publican en GitHub Pages.

## 3. Diseño

- **Herramienta (D4):** Claude Design. El lienzo ya existe con las filas Acceso, App, Desbloquear lugares y Niveles de detalle.
- **Pantallas diseñadas:**
  - Acceso: Bienvenida, Ingresar, Registro (3 pasos: cuenta, modo de juego y permisos), Permisos.
  - App: Main (globo personal), Lugar (detalle con filtros), Fotos, Perfil (con modo de juego y estadísticas por continente), Modo.
  - Desbloqueo: aviso, panel, Confirmar, Importar.
  - También Compartido y "Tus globos", que quedan fuera del MVP.
- **Globo:** 3D, gira solo, se arrastra y hace zoom (pellizco y doble toque), con dos niveles de detalle (país y región). [Q7]
- **Paleta: 2 neutros y 1 acento, en ambos temas.**

  | | Oscuro | Claro |
  |---|---|---|
  | Fondo | Carbón cálido #161513 | Papel crema #F6F3EC |
  | Texto | Blanco hueso #ECE8E1 | Tinta #1E1D1A |
  | Acento salvia | #9CC5A8 | #4E8A66 |
  | Botón principal | Blanco hueso con texto oscuro | Tinta con texto claro |

  El verde se usa casi solo en lo desbloqueado y en las selecciones. El globo no lleva halo y tiene meridianos y fronteras tenues.
- **Movimiento:** estilo sobrio con animaciones fluidas, hechas con react-native-reanimated en el hilo nativo. La fluidez real se logra en Claude Code, no en el prototipo HTML.
- **Pendiente en diseño:** la pantalla de feed, la tabla de modos con la nueva definición, el paso "elegir fotos" de la importación, los estados vacíos y de error, y pasar la paleta a un DESIGN.md con tokens que use la app.

## 4. Stack y arquitectura

| Capa | Herramienta | Nota |
|---|---|---|
| App | Expo (React Native + TypeScript, Expo Router), i18n es/en desde E0 | Builds en la nube; dev build en el Mac de un amigo hasta E5 (D5) |
| Backend | Supabase: Auth, Postgres + PostGIS, RLS | `zona_en` y `desbloquear` como **RPC de Postgres**, porque las llamadas a la API son ilimitadas |
| Edge Functions | Solo para la URL firmada de R2 y SafeSearch | |
| Fotos y contornos | Cloudflare R2 | No cobra la salida de datos |
| Globo y mapa | Por definir [Q7] | MapLibre + OpenFreeMap si sirve para el globo 3D |
| Moderación | Google Cloud Vision SafeSearch | Es el filtro automático que pide Apple |
| Errores | Expo Observe | |
| Respaldos | `supabase db dump` semanal con GitHub Actions hacia R2 | El plan Free no trae backups |

**Flujos clave**
- **Desbloqueo:** la app envía lat, lng, precisión y simulada a la RPC `desbloquear`. La RPC valida, inserta en `unlocks` y `actividad`, y descarta las coordenadas.
- **Subida de foto:**
  1. La app comprime la foto y le quita el EXIF.
  2. Una Edge Function entrega una URL firmada de R2.
  3. La app sube la foto directo a R2.
  4. La Edge Function corre SafeSearch y cambia el `estado` a visible u oculta.
- **Lectura del globo:** contornos estáticos desde R2 (con caché) e IDs desbloqueados desde la base.

## 5. Modelo de datos (Postgres)

| Tabla | Campos clave | Nota |
|---|---|---|
| profiles | id, username, nombre, avatar, es_privado, modo_juego, idioma | modo_juego: viajero o explorador |
| zones | id, nombre_es, nombre_en, nivel, padre_id, pais, centroide | nivel: pais o region (comuna después del MVP) |
| zone_parts | zone_id, geom | Viene de `ST_Subdivide`; índice GIST; solo para búsqueda |
| unlocks | user_id, zone_id, metodo, fecha | PK (user_id, zone_id); metodo: gps, recuerdo o foto_exif |
| photos | id, user_id, zone_id, periodo, key_r2, estado, likes_count, pin | periodo = primer día del mes; estado: pendiente, visible u oculta |
| follows | seguidor_id, seguido_id, estado | estado: pendiente o aceptado |
| likes | user_id, photo_id | |
| actividad | id, actor_id, tipo, zone_id, ref_id, fecha | Feed; índice (actor_id, fecha) |
| profile_stats | user_id, contadores por nivel y verificados/recuerdo | Se mantiene con triggers |
| blocks, reports | autor, objetivo, motivo, estado | Moderación |

**Funciones y reglas**
- `zona_en(lat, lng)` devuelve país y región con `ST_Contains` sobre `zone_parts`.
- `desbloquear(lat, lng, precision, simulada)` aplica las validaciones de 2.1 y no guarda coordenadas.
- `puede_ver(dueño)` es verdadera si el contenido es tuyo, si la cuenta es pública o si sigues al dueño con la solicitud aceptada, y nunca si hay un bloqueo. La usan **todas** las reglas RLS.
- `likes_count` y los contadores por nivel se mantienen con triggers.
- El esquema cambia solo con migraciones (D7).

**Medidas (Postgres 16, con datos de prueba e índices):** una foto ocupa unos 310 B, un desbloqueo unos 170 B y un me gusta unos 175 B. Eso da unos 9 KB por usuario activo al mes. Las geometrías se estimaron en 130 MB fijos; hay que **medirlas en la E1** con `pg_total_relation_size`.

## 6. Etapas

Las etapas avanzan por hitos, sin fecha. Las semanas son estimaciones a unas 8 h por semana. Cada etapa termina cuando cumple su "listo cuando" (P9). Las técnicas de IA de cada etapa están en plan-decisiones.md.

### E0 · Preparación (~1 sem)
- [ ] Cerrar las preguntas abiertas de este archivo
- [ ] Primer /debatir: licencia del repo [Q9]
- [ ] Proyecto Expo con TypeScript, Expo Router e i18n es/en; `name: Incógnita`, `slug: incognita`, `ios.bundleIdentifier` y `android.package` = `io.github.bato21.incognita`, `scheme: incognita`
- [ ] Cerebro `docs/brain/`, CLAUDE.md/AGENTS.md, hook de sesión, estilo mentor, /debatir y guardian-norte
- [ ] Secret scanning y push protection en GitHub
- [ ] Proyecto Supabase **dev** con PostGIS y bucket R2
- [ ] DESIGN.md con los tokens de la paleta
- [ ] Spike del globo [Q7] y spike de lectura de ubicación en las fotos elegidas (iOS y Android)

Listo cuando: la app vacía corre en el emulador Android y en mi iPhone, y los dos spikes tienen su conclusión escrita.

### E1 · Zonas (~2 sem)
- [ ] Descargar los límites, simplificarlos y cargar `zones` y `zone_parts` con índice GIST
- [ ] Generar los contornos estáticos por nivel y subirlos a R2
- [ ] TDD: escribir los 20 puntos de prueba (incluidos bordes entre regiones y entre países) **antes** que `zona_en`
- [ ] Medir el tamaño real de las geometrías
- [ ] MCP de Supabase solo contra dev, en solo lectura

Listo cuando: cualquier punto del mundo devuelve su país y región en menos de 0,5 s y las geometrías pesan menos de 200 MB.

### E2 · Desbloquear (~3 sem)
- [ ] Instalar juntas todas las dependencias nativas del MVP y hacer el primer build en el Mac de un amigo, con el kit del Mac
- [ ] Login con email y Google (Apple pasa a la E5)
- [ ] Registro de 3 pasos: cuenta, modo de juego y permisos
- [ ] Flujo completo de 2.1 (aviso, panel, confirmar y corregir) con antitrampa
- [ ] Globo con zonas pintadas según el modo y contadores
- [ ] Modo recuerdo manual

Listo cuando: desbloqueo 3 regiones reales sin falsos positivos (con ubicación simulada en dev para las lejanas) y cambiar de modo no pierde datos.

### E3 · Fotos (~3 sem)
- [ ] Selector del sistema y cámara; mes y año sugeridos
- [ ] Comprimir y quitar EXIF, verificado con exiftool (primera skill)
- [ ] URL firmada de R2, miniaturas y SafeSearch
- [ ] Página de lugar con fotos por mes y año; perfil-portafolio
- [ ] Importar viajes pasados desde las fotos elegidas, como recuerdo

Listo cuando: subo 30 fotos de 5 zonas, no queda GPS en ninguna, el portafolio carga rápido con datos móviles, y al importar 50 fotos con ubicación se detectan bien sus zonas en iOS y Android.

### E4 · Social (~3 sem)
- [ ] Perfil público o privado, seguir y solicitudes
- [ ] Feed simple sobre `actividad`
- [ ] Página pública de cada lugar, me gusta, push (probadas en Android) y tarjeta para stories
- [ ] CI y revisión de Claude en cada PR
- [ ] Lista de 15–20 testers Android

Listo cuando: 5 amigos la usan una semana y ven lo del resto.

### E5 · Seguridad y cumplimiento (~2 sem)
- [ ] Tests de RLS: nadie ve fotos privadas ajenas
- [ ] Reportar, bloquear y cola de moderación
- [ ] Borrado de cuenta en la app y en la web; política de privacidad, términos y contacto
- [ ] Edad mínima de 16 años
- [ ] Pagar Apple (US$99) y Google Play (US$25); Sign in with Apple y push en iPhone
- [ ] Revisar que no haya marca registrada "Incógnita" en la clase de apps (INAPI en Chile) antes de pagar las cuentas
- [ ] Revisar el estado de la Ley 21.719

Listo cuando: paso la checklist de la sección 7.

### E6 · Beta cerrada (~4 sem)
- [ ] Proyecto Supabase **prod**
- [ ] TestFlight
- [ ] Prueba cerrada en Play con al menos 12 testers durante 14 días seguidos, subiendo al menos una actualización en ese tiempo

Listo cuando: siguen inscritos 12 o más testers tras 14 días y no hay crashes graves.

### E7 · Publicación (~2 sem)
- [ ] Fichas en es/en, capturas e ícono
- [ ] Formularios de privacidad de Apple y de seguridad de datos de Play; clasificación por edad
- [ ] Cuenta demo con zonas y fotos ya cargadas, y notas para los revisores que expliquen el desbloqueo
- [ ] Solicitar acceso a producción en Play y enviar a revisión en App Store

Listo cuando: la app está publicada gratis en ambas tiendas.

## 7. Publicación: checklist antes de enviar

- [ ] Reportar, bloquear y SafeSearch funcionando
- [ ] Correo de soporte visible en la app y en la ficha
- [ ] Borrado de cuenta en la app y página web de borrado declarada en Play
- [ ] Política de privacidad publicada
- [ ] Textos de los permisos de ubicación y fotos en es/en
- [ ] Si se ofrece login con Google, también Sign in with Apple
- [ ] Cuenta demo lista: el revisor probablemente no está en Chile
- [ ] Capturas de la app en uso

| | App Store | Google Play |
|---|---|---|
| Cuenta | US$99 al año | US$25, pago único |
| Prueba previa | TestFlight, opcional | Obligatoria: 12 testers durante 14 días, más unos 7 días de revisión |
| Borrado de cuenta | En la app | En la app y en la web |

## 8. Costos

**MVP en planes gratis:** US$124 el primer año y US$99 desde el segundo, solo por las cuentas de las tiendas. Claude Pro ya lo pago.

**Escenarios** (costo del mes 12; supuestos: 20% de usuarios diarios, 6 fotos y 20 me gusta por usuario al mes):

| Usuarios activos al mes | 30 | 1.000 | 3.000 | 10.000 |
|---|---|---|---|---|
| Base de datos | 134 MB | 253 MB | 498 MB | 1,4 GB |
| Plan Supabase | Free | Free | Free | Pro (US$25) |
| R2 | US$0 | US$0,27 | US$1,12 | US$4,07 |
| SafeSearch | US$0 | US$7,50 | US$25,50 | US$88,50 |
| **Total** | **US$0** | **~US$8** | **~US$27** | **~US$118** |

- **Free** sirve para desarrollo, beta y lanzamiento. Ojo: se pausa tras una semana sin actividad.
- **Pasar a Pro** cuando la base supere ~400 MB o el egress ~4 GB al mes, sin esperar al límite, porque al pasarlo la base queda en solo lectura. Hay que dejar activado el tope de gasto.
- **El riesgo de costo es SafeSearch**, no la base. Si crece, conviene filtrar solo las fotos públicas o las de cuentas nuevas, revisando que siga cumpliendo con Apple. Como la foto es opcional al desbloquear, este costo no crece con cada desbloqueo.

## 9. Riesgos

| Riesgo | Mitigación |
|---|---|
| El alcance crece (modos, importar, grupos, hitos) | MVP congelado; cada idea nueva va a la sección 10 |
| El globo 3D no rinde en React Native | Spike en E0 [Q7]; plan B: mapa 2D con globo solo en el zoom lejano |
| No se puede leer la ubicación de las fotos elegidas | Spike en E0; si Android la redacta, pedir `ACCESS_MEDIA_LOCATION` y justificarlo en el formulario de Play |
| Ubicación falsa | Validaciones de 2.1 y reportes; aceptar que no será perfecto |
| Revelar dónde está alguien | Solo mes y año, sin coordenadas ni EXIF |
| Fotos inapropiadas | SafeSearch, reportes, bloqueo y respuesta a cada reporte en menos de 48 h |
| La base se llena por las geometrías | Medir en E1 y simplificar más si pasan de 200 MB |
| Supabase se pausa o no hay backups | Dump semanal y Pro al lanzar si hace falta |
| Sin Mac, o la app de desarrollo caduca cada 7 días | Emulador Android para el día a día y dos amigos con Mac |
| Faltan testers Android | Reclutar desde la E4 |
| Ley 21.719 | Consentimiento explícito, borrado real y edad mínima de 16; revisar la vigencia en E5 |
| Rechazo en las tiendas | Checklist y cuenta demo |

## 10. Después del MVP (en orden sugerido)

1. Grupos y globos compartidos, ya prototipados.
2. Modo Argonauta: comunas de Chile, segundo nivel administrativo del mundo y luego pueblos, islas, recintos e hitos desde OSM.
3. Comentarios con moderación.
4. Perfil web público.
5. Resumen anual para compartir.
6. Retos (16 regiones, 346 comunas) y alianzas con municipios.
7. Monetización opcional sin anuncios ni suscripción obligatoria (P6), por ejemplo pósters o libros impresos.

## Preguntas abiertas

Cerradas el 7 nov: Q10 (nombre: Incógnita), Q13 (bundle id: `io.github.bato21.incognita`), Q3 (sin Argonauta en el MVP), Q1 (foto opcional), Q2 (el modo define la unidad desbloqueable y se guarda siempre lo más fino), Q4 (importar entra al MVP, con fotos elegidas) y Q6 (feed simple en el MVP).

| # | Pregunta | Mi propuesta |
|---|---|---|
| Q5 | ¿Confirmas que los grupos y globos compartidos quedan fuera aunque estén diseñados? | Sí, fuera |
| Q7 | Globo 3D: ¿MapLibre (si su versión para React Native soporta globo), un globo propio con Skia o three.js, o 2D con globo solo de lejos? | Spike en E0 y decidir con /debatir (es difícil de revertir) |
| Q8 | ¿Fuente de límites del mundo: geoBoundaries (CC BY, exige atribución) o Natural Earth (dominio público)? | Natural Earth |
| Q9 | Licencias: del repo y de lo que usamos (datos, mapas, fuentes, librerías) y qué costo u obligación trae cada una | /debatir en la E0 |
| Q12 | Pin por foto: ¿qué precisión guardar y quién la ve? | El dueño ve el pin exacto; los demás lo ven redondeado a ~1 km, y nunca se muestra en fotos del día en curso. P2 pasa a "nunca se guarda tu ubicación en vivo; la de las fotos solo con tu consentimiento". Va a /debatir y ADR (P8) |
| Q11 | ¿Hace falta un mínimo de distancia o tiempo para volver a mostrar el aviso de un lugar descartado? | No volver a mostrarlo por esa zona en 24 h |

## Fuentes

- [App Review Guidelines de Apple](https://developer.apple.com/app-store/review/guidelines/)
- [Google Play: requisitos de prueba para cuentas personales nuevas](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en-GB)
- [Google Play: permisos de fotos y videos](https://support.google.com/googleplay/android-developer/answer/14115180)
- [Google Play: borrado de cuenta](https://support.google.com/googleplay/android-developer/answer/13327111)
- [Precios de Supabase](https://supabase.com/pricing) · [Tope de gasto](https://supabase.com/docs/guides/platform/billing-faq)
- [Precios de Cloudflare R2](https://developers.cloudflare.com/r2/pricing/)
- [Precios de Expo (EAS)](https://expo.dev/pricing)
- [Precios de Cloud Vision](https://cloud.google.com/vision/pricing)
- [expo-location: campo mocked](https://unpkg.com/expo-location@55.1.8/build/Location.types.d.ts)
- [geoBoundaries frente a GADM](https://atlas.co/data-sources/geo-boundaries/)
- [chilemapas: comunas con datos INE y SUBDERE](https://search.r-project.org/CRAN/packages/chilemapas/chilemapas.pdf)
- [Ley 21.719 (BCN)](https://www.bcn.cl/leychile/navegar?i=1209272)
- [Proyecto de prórroga de la Ley 21.719 (Diario Constitucional)](https://www.diarioconstitucional.cl/estudios-juridicos/alerta-legal-proyecto-posterga-entrada-en-vigencia-de-ley-de-proteccion-de-datos-personales-por-carla-illanes/)
- [Heatmap de Strava y bases militares (CNBC)](https://www.cnbc.com/2018/01/28/global-heat-map-for-joggers-exposing-sensitive-us-military-information.html)