# Anómala · Arquitectura de contenidos para WordPress

Documento de trabajo para desarrollar el tema a partir de la maqueta HTML. Define tipos de contenido, taxonomías, campos, URLs, plantillas y SEO.

**Criterio general:** todo en código, sin plugins. Las estructuras de contenido (tipos, taxonomías y campos) se registran en un *must-use plugin* propio (`wp-content/mu-plugins/anomala-core/`), y la presentación vive en el tema (`wp-content/themes/anomala/`). Así, si algún día se cambia el tema, el contenido y sus datos siguen funcionando. El mu-plugin es código del proyecto, no un plugin de terceros.

- Prefijo del proyecto: `anm_` (funciones, opciones, meta keys, handles).
- Prefijo del módulo SEO: `mgc_` (se mantiene el de MG SEO Core).
- Requisitos: WordPress 6.5+, PHP 8.1+.

Estado: **borrador**. Lo marcado como *Pendiente* falta decidirlo con el cliente.

---

## 1. Tipos de contenido

| Tipo | Slug | Qué es | Archivo | Plantillas |
|---|---|---|---|---|
| Nota | `post` (nativo) | Todo contenido editorial | — | `single.php` |
| Autor | `autor` (CPT) | Firmas: equipo y columnistas invitados | `/autores/` | `archive-autor.php`, `single-autor.php` |
| Evento | `evento` (CPT) | Actividades de la cartelera | `/cartelera/` | `archive-evento.php`, `single-evento.php` |
| Página | `page` (nativo) | Quiénes somos, Contacto, Transparencia, legales | — | `page.php`, `page-{slug}.php` |

**Por qué `autor` es un CPT y no un usuario:** los usuarios de WordPress son los suscriptores de los planes pagos (Conversa, Encuentro), y separarlos evita mezclar firmas con cuentas. Además, los columnistas invitados no necesitan una cuenta. Quien edita en el panel sigue siendo un usuario; quien firma es una ficha `autor`.

**Por qué los formatos no son CPT:** se evaluó y se descartó. Casi todos los listados mezclan formatos (Lo nuevo, Publicaciones, Cultura, Vida real, temas, series, autor, búsqueda, RSS). Como taxonomía, cada formato conserva su menú propio en el panel (ver §6) y sus campos específicos (ver §3) sin complicar las consultas.

### Registro del CPT `autor`

```php
register_post_type( 'autor', [
  'label'        => 'Autores',
  'public'       => true,
  'show_in_rest' => true,
  'has_archive'  => 'autores',
  'rewrite'      => [ 'slug' => 'autores', 'with_front' => false ],
  'supports'     => [ 'title', 'editor', 'thumbnail', 'excerpt' ], // título = nombre; editor = bio larga; excerpt = bio corta
  'menu_icon'    => 'dashicons-id',
] );
```

### Registro del CPT `evento`

```php
register_post_type( 'evento', [
  'label'        => 'Cartelera',
  'public'       => true,
  'show_in_rest' => true,
  'has_archive'  => 'cartelera',
  'rewrite'      => [ 'slug' => 'cartelera', 'with_front' => false ],
  'supports'     => [ 'title', 'editor', 'thumbnail', 'excerpt' ], // thumbnail = afiche vertical 3/4
  'menu_icon'    => 'dashicons-calendar-alt',
] );
```

---

## 2. Taxonomías

| Taxonomía | Slug | Aplica a | Valores | Por elemento | Editor | URL |
|---|---|---|---|---|---|---|
| Categoría | `category` (nativa) | post | Cultura · Vida real | **1, obligatoria** | Selector único propio | `/cultura/`, `/vida-real/` *(ver §5)* |
| Formato | `formato` | post | Columna · Entrevista · Reportaje · Crónica · Ensayo | **1, obligatoria** | Selector único propio | `/formato/entrevista/` |
| Serie | `serie` | post | Spoiler Alert! · Coleccionistas psiquiátricos · Nadie te pidió que escribieras | **0 o 1** | Selector único propio | `/serie/spoiler-alert/` |
| Temas | `post_tag` (nativa) | post | Abierta, se va sumando | Varios | Panel nativo de etiquetas | `/tema/musica/` |
| Medio | `medio` | post | Video · Audio | **Automático** | Oculta | `/medio/video/` |
| Tipo de autor | `tipo_autor` | autor | Permanente · Invitado | 1 | Selector único propio | No pública |
| Disciplina | `disciplina` | evento | Teatro · Música · Artes visuales · Cine · Danza… | 1 | Selector único propio | `/cartelera/disciplina/teatro/` |

**Reglas:**

- **Formato y Medio no se mezclan.** El formato es cómo está contada la nota; el medio dice si trae video o audio. Los nombres del acta del tipo «Entrevista audiovisual» se registran como Formato = Entrevista, y el medio Video lo asigna el sistema.
- **Una serie se suma al formato, no lo reemplaza.** Si una serie usa siempre el mismo formato (Nadie te pidió… = Entrevista), el editor puede preseleccionarlo al elegir la serie, pero el formato queda guardado igual.
- **Temas:** pueden crearlos solo Editor y Administrador, para evitar duplicados («Música», «música»). Los autores eligen entre los existentes: `assign_terms` para todos y `manage_terms` / `edit_terms` solo para editores.
- **Medio no tiene término «Texto».** «Texto e imagen» es la consulta de las notas sin Video ni Audio: `tax_query` con `operator => 'NOT EXISTS'`.

### Registro (ejemplo `formato`; `serie`, `medio`, `tipo_autor` y `disciplina` siguen el mismo patrón)

```php
register_taxonomy( 'formato', 'post', [
  'label'             => 'Formatos',
  'public'            => true,
  'hierarchical'      => true,          // lista cerrada, como categorías
  'show_in_rest'      => true,
  'show_admin_column' => true,
  'meta_box_cb'       => false,         // se reemplaza por el selector único (§2.2)
  'rewrite'           => [ 'slug' => 'formato', 'with_front' => false ],
  'capabilities'      => [ 'manage_terms' => 'manage_options', 'edit_terms' => 'manage_options',
                           'delete_terms' => 'manage_options', 'assign_terms' => 'edit_posts' ],
] );

register_taxonomy( 'medio', 'post', [
  'public'       => true,   // tiene archivo y conteo
  'show_ui'      => false,  // nadie la ve ni la marca
  'show_in_rest' => false,
  'rewrite'      => [ 'slug' => 'medio', 'with_front' => false ],
] );
```

### 2.1 Medio automático

Al guardar una nota se revisa el contenido y se asignan los términos. Si alguien agrega o quita un video, se corrige solo al volver a guardar.

```php
add_action( 'save_post_post', function ( $post_id ) {
  if ( wp_is_post_revision( $post_id ) || wp_is_post_autosave( $post_id ) ) return;
  $c = get_post_field( 'post_content', $post_id );
  $terms = [];
  if ( has_block( 'core/video', $c ) || preg_match( '/youtube\.com|youtu\.be|vimeo\.com/i', $c ) ) $terms[] = 'video';
  if ( has_block( 'core/audio', $c ) || preg_match( '/spotify\.com|soundcloud\.com/i', $c ) )      $terms[] = 'audio';
  wp_set_object_terms( $post_id, $terms, 'medio' );
} );
```

Desde el medio se resuelven: el ícono verde de las tarjetas, la etiqueta «Video» en la cabecera de la nota, los filtros «Todos / Texto e imagen / Video / Audio» y el bloque «Por medio» de Temas.

### 2.2 Selector único (sin plugins)

El panel nativo de taxonomías jerárquicas usa casillas y permite marcar varias. Para Categoría, Formato, Serie, Tipo de autor y Disciplina:

1. Se oculta el panel nativo: `meta_box_cb => false`, y para `category` se filtra `rest_prepare_taxonomy` o se quita el panel en el editor.
2. El tema registra un panel propio en la barra lateral del editor de bloques (`PluginDocumentSettingPanel`, un archivo JS del mu-plugin sin dependencias externas) con un `<select>` por taxonomía.
3. Se valida en el servidor en `save_post`: si llegan varios términos, queda solo uno, y si Categoría o Formato vienen vacíos, la nota no se puede publicar (vuelve a borrador con aviso en `admin_notices`).

### 2.3 Datos de los términos (term meta, sin plugins)

| Taxonomía | Campo | Meta key | Uso |
|---|---|---|---|
| serie | Lema | *descripción nativa del término* | «Secretos y accidentes de la cultura pop.» |
| serie | Resumen | `anm_resumen` | Texto de la tarjeta en Temas y en el índice de series |
| serie | Imagen | `anm_imagen` (ID de adjunto) | Foto de la tarjeta y OG del archivo |
| serie | Formato por defecto | `anm_formato_defecto` (term ID) | Preselección en el editor |
| category | Color | `anm_color` | Cultura = `--ui-success`, Vida real = `--ui-highlight` |
| formato, post_tag | Imagen | `anm_imagen` | Mosaicos de Temas |

Se editan con campos propios en `{taxonomy}_edit_form_fields` y se guardan en `edited_{taxonomy}`, registrados con `register_term_meta( ..., [ 'show_in_rest' => true ] )`.

---

## 3. Campos (post meta)

Todos se registran con `register_post_meta()` (`show_in_rest => true`, `sanitize_callback`, `auth_callback`) y se editan con *meta boxes* propias. Las meta boxes clásicas funcionan dentro del editor de bloques.

### Nota (`post`)

| Campo | Meta key | Tipo | Cuándo aparece |
|---|---|---|---|
| Autores | `anm_autor` | ID de `autor`, **una fila por autor** (`add_post_meta` repetido) | Siempre. Orden: `anm_autores_orden` (lista de IDs) |
| Destacado en portada | `anm_destacado` | bool + `anm_destacado_orden` (1–3) | Siempre |
| Persona entrevistada: nombre | `anm_entrevistado_nombre` | texto | Formato = Entrevista |
| Persona entrevistada: foto | `anm_entrevistado_foto` | ID de adjunto | Formato = Entrevista |
| Persona entrevistada: descripción | `anm_entrevistado_descripcion` | texto corto | Formato = Entrevista |
| Créditos adicionales | `anm_creditos` | JSON `[ { rol, nombre } ]` | Siempre (Fotografía, Video, Edición…) |
| Duración | `anm_duracion` | texto («12 min») | Si Medio = Video o Audio |

**Autores con una fila por autor:** así se puede consultar «notas de este autor» con un `meta_query` directo (`key => 'anm_autor', value => $id`), sin buscar dentro de un arreglo serializado.

**Campos condicionales sin ACF:** la meta box «Persona entrevistada» se registra siempre, y el JS del editor la muestra u oculta según el formato elegido en el selector único. La plantilla la imprime solo si `has_term( 'entrevista', 'formato' )` y hay nombre.

**Tiempo de lectura:** se calcula, no se escribe: `ceil( str_word_count( wp_strip_all_tags( $content ) ) / 200 )`.

### Autor (`autor`)

| Campo | Meta key / fuente |
|---|---|
| Nombre | `post_title` |
| Foto | imagen destacada. Sin foto se muestran las iniciales (`.avatar`) |
| Bio corta (tarjetas, firma) | `post_excerpt` |
| Bio larga (perfil) | `post_content` |
| Disciplina | `anm_disciplina` (texto) |
| Cargo o rol | `anm_cargo` («Editor de TurnoAM») |
| Redes | `anm_instagram`, `anm_x`, `anm_web`, `anm_linkedin` |
| Usuario vinculado (opcional) | `anm_usuario` (ID de usuario, si además edita en el panel) |

### Evento (`evento`)

| Campo | Meta key | Notas |
|---|---|---|
| Funciones | `anm_funciones` | JSON `[ { inicio: ISO8601, fin } ]`. La primera ordena la cartelera |
| Fecha de término | `anm_fin` | Para «Hasta 30 nov». El evento sale solo de la cartelera al pasar |
| Lugar | `anm_lugar` | Nombre del espacio |
| Dirección | `anm_direccion` | |
| Región / Comuna | `anm_region`, `anm_comuna` | Filtros de la cartelera |
| Precio | `anm_precio_desde` | Entero en CLP. 0 = gratis |
| Entradas / información | `anm_url` | Enlace externo |
| Tipo | `anm_tipo` | `propio` (Evento de Anómala) · `terceros` |
| Organizador | `anm_organizador` | Solo si es de terceros |
| Destacado | `anm_destacado` | Aparece en la portada |
| Invitaciones Encuentro | `anm_invitaciones` | bool + cupos, solo si es propio |

---

## 4. Relaciones

| Relación | Cómo |
|---|---|
| Nota → Autores | `anm_autor` en la nota (una fila por autor) |
| Autor → Notas | `WP_Query` con `meta_query` sobre `anm_autor`, en `single-autor.php` |
| Nota → Serie | Taxonomía `serie` |
| Serie → Notas | Archivo `taxonomy-serie.php` (más reciente primero) |
| Nota → Relacionadas («Sigue leyendo») | Misma serie, luego mismo tema y luego misma categoría, sin repetir la actual |

---

## 5. URLs

| Contenido | URL |
|---|---|
| Nota | `/%category%/%postname%/` → `/cultura/jodorowsky-dune/` |
| Categoría | `/cultura/`, `/vida-real/`. Se quita la base `category/` con reglas propias (`category_rewrite_rules`) |
| Formato | `/formato/columna/` |
| Serie | `/serie/spoiler-alert/` |
| Índice de series | `/series/` (página con plantilla `page-series.php`) |
| Tema | `/tema/musica/` (base de etiquetas en Ajustes → Enlaces permanentes) |
| Temas (índice) | `/temas/` (`page-temas.php`) |
| Medio | `/medio/video/`, `/medio/audio/` |
| Autores | `/autores/`, `/autores/isabel-agurto/` |
| Cartelera | `/cartelera/`, `/cartelera/nombre-del-evento/` |
| Búsqueda | `/?s=` (nativa) |

*Pendiente:* confirmar si se quiere la categoría en la URL de la nota. Si una nota cambia de categoría, la URL cambia y habría que redirigir (el gestor de redirecciones del módulo SEO lo cubre).

---

## 6. Panel de administración

- **Notas:** submenús filtrados bajo «Entradas»: Columnas, Entrevistas, Reportajes, Crónicas y Ensayos. Son enlaces a `edit.php?formato=columna`, etc., con `add_submenu_page`, y dan el mismo orden que tendrían los CPT.
- **Columnas del listado:** Formato, Serie, Categoría, Autores y un ícono de video o audio (`manage_posts_columns`), con filtro por formato y serie (`restrict_manage_posts`).
- **Autores:** columna Tipo (Permanente / Invitado) y filtro por tipo.
- **Cartelera:** columnas de próxima función, lugar, tipo y destacado, ordenadas por fecha.
- **Roles:** Administrador, Editor (crea temas, publica), Autor del equipo (escribe y envía a revisión) y Suscriptor (solo comenta según plan, ver §8).

---

## 7. Plantillas (jerarquía de WordPress → maqueta)

| Plantilla | Maqueta | Notas |
|---|---|---|
| `front-page.php` | `home-page.html` | Destacados (`anm_destacado`), Lo nuevo, Voces, Selección, Cartelera, Aliados |
| `home.php` / `archive.php` | `publicaciones.html` | Filtros por formato y medio como parámetros de consulta |
| `category.php` | `eje.html`, `vida-real.html` | Color desde `anm_color` |
| `taxonomy-formato.php` | — | Puede reutilizar `archive.php` |
| `taxonomy-serie.php` | `serie.html` | Lema, descripción, entregas, otras series |
| `taxonomy-medio.php` | — | Reutiliza `archive.php` |
| `tag.php` | `tema.html` | |
| `page-temas.php` | `temas.html` | Con más movimiento, Series, Todos los temas, Por formato, Por medio |
| `page-series.php` | `series.html` | `get_terms( 'serie' )` |
| `single.php` | `single*.html` | Partes por formato: `template-parts/perfil-entrevistado.php` (Entrevista), firma junto al título y aviso de opinión (Columna) |
| `archive-autor.php` | `autores.html` | Filtro por tipo de autor |
| `single-autor.php` | `autor.html` | Perfil + notas donde aparece |
| `archive-evento.php` | `cartelera.html` | Agrupada por mes; filtros por disciplina, fecha, región, comuna y precio |
| `single-evento.php` | `single-evento.html` | |
| `search.php` | `busqueda.html` | Busca en notas y en autores |
| `404.php` | `404.html` | |
| `page.php` | `page.html` y legales | |
| `comments.php` | Bloque «Conversación» | Ver §8 |

**Partes compartidas:** `template-parts/tarjeta.php` (la `.entry-card` única), `compartir.php`, `creditos.php`, `sobre-autores.php`, `mas-leido.php`. La clase `single` que WordPress pone en el `<body>` activa la tipografía de lectura (Inter) en `.cuerpo`.

---

## 8. Suscripción y comentarios (sin plugins)

Todo el contenido editorial es gratuito. Los planes pagos habilitan los comentarios (Conversa) y además las invitaciones a eventos propios (Encuentro).

- **Plan del usuario:** roles propios `anm_conversa` y `anm_encuentro`, o `anm_plan` en user meta con su vencimiento `anm_plan_vence`.
- **Comentarios:** `comments.php` revisa el plan **en el servidor**. A quien no tiene plan no se le envían los comentarios en el HTML. Además, el filtro `pre_comment_approved` y el permiso de `comment_form` bloquean el envío sin plan.
- **Pago recurrente:** integración propia con la pasarela vía API y webhooks que actualizan el plan. *Pendiente:* elegir la pasarela (Flow, Mercado Pago o Webpay Plus).
- **Newsletter:** se envía desde un proveedor de mailing externo, no desde WordPress. *Pendiente:* definir qué servicio usarán (ver §10).
  - **Formulario:** los formularios (portada, pie de las notas, panel Hazte Anómala) envían a un endpoint propio del tema. Ese endpoint llama a la API del proveedor con `wp_remote_post` y envía el correo, el origen y el consentimiento con su fecha. La clave de la API vive en `wp-config.php`, nunca en el navegador.
  - **Confirmación y bajas:** la doble confirmación, el registro del consentimiento y las bajas los maneja el proveedor. La persona vuelve a las páginas `newsletter-confirmar` y `newsletter-baja` del sitio.
  - **Armado del boletín:** a mano en el editor del proveedor o automático desde el RSS del sitio («RSS to email»), según lo que prefiera la redacción.
  - **Suscriptores pagos:** al activarse un plan, el mismo código agrega o actualiza el contacto con la etiqueta del plan y busca por correo para no duplicar. Al cancelar, se quita la etiqueta y la persona sigue en el newsletter gratuito.
- **Correo transaccional (SMTP):** cuenta, contraseña, pagos, invitaciones y respuestas a comentarios (los textos están en `correos.html`). Se envían por el **mismo servicio de mailing**, por SMTP autenticado o por su API. Se configura en código con `phpmailer_init` y las credenciales van en `wp-config.php`, sin plugin de SMTP. Se configuran SPF, DKIM y DMARC del dominio para ese servicio.
- **Cookies:** aviso propio. La analítica se carga solo después de aceptar, y la integración con el panel de Tracking del módulo SEO respeta esa elección.

---

## 9. SEO: MG SEO Core adaptado

Se parte de `circuloautocool/mg-seo-core` (v1.0.0) y se crea una versión para Anómala en `themes/anomala/mg-seo-core/`. Se carga con `require_once get_template_directory() . '/mg-seo-core/loader.php';`.

### 9.1 Qué se quita

- `admin/panel-whatsapp.php`, con su CPT `mgc_wa_click` y su endpoint AJAX público, y la línea correspondiente de `loader.php`.
- `core/seo-hreflang.php`: el sitio no es multilenguaje.
- En `admin/panel-business.php`: horarios, calificación de Google, geo/mapa, dirección y las referencias a «AutoRepair» y «Datos del Negocio».
- En la Organization de la portada (`seo-meta.php`): `contactPoint`, `geo`, `openingHoursSpecification` y `aggregateRating`.
- La salida de `meta keywords`, y el criterio de keyword focus del analizador.
- Textos de otros proyectos: «— Blog |» y «marketing digital» en `seo-pagination.php`, y los placeholders y comentarios de Autocool, `mg_servicio` y Quform.
- El ping a Google y Bing del sitemap: esos servicios ya no existen.

### 9.2 Qué se corrige

- **Duplicados con WordPress:**
  - quitar `rel_canonical` y `wp_robots` del núcleo, y desactivar `/wp-sitemap.xml` (`wp_sitemaps_enabled` → false);
  - dejar una sola línea Sitemap en robots.txt, un solo `<link rel=sitemap>` y una sola impresión del código personalizado del `<head>`.
- **Doble canonical en archivos paginados:** `seo-meta` no imprime canonical cuando `is_paged()`, y el título suma «Página N».
- **Schema por defecto:** `post` → tipo según formato (ver 9.3) sin depender de guardar el panel. Se quitan del selector los tipos sin generador (FAQPage, Course, JobPosting, RealEstateListing).
- **Archivos de taxonomía y CPT:** hoy imprimen el schema del primer post del loop (`global $post`); deben imprimir CollectionPage.
- **Seguridad:**
  - `mgc_custom_head_code` y `mgc_schema_json` solo para quien tiene `unfiltered_html`, con el JSON validado y sin `</script>`;
  - `$wpdb->esc_like` en la búsqueda de redirecciones, el nonce del enlace de edición y caché del mapa de redirecciones (hoy hace una consulta por visita);
  - los slugs de menú equivocados (`mg-redirects` → `mgc-redirects`, `mg-tracking-settings` → `mgc-tracking`).
- **Interfaz que falta:** SEO por término (título, descripción, OG, noindex) en `{taxonomy}_edit_form_fields` y opciones de archivo por CPT (`mgc_archive_*`).

### 9.3 Qué se agrega para un medio

**Organización y sitio**
- `NewsMediaOrganization` con `@id`, logo `ImageObject`, `sameAs` (Instagram…) y las páginas de principios: `publishingPrinciples` → Quiénes somos (Brújula), `ethicsPolicy`, `correctionsPolicy` → Transparencia. El panel «Datos del Negocio» pasa a «Datos del medio».
- `WebSite` con `SearchAction` (`/?s={search_term_string}`).

**Tipo de schema por formato** (filtro `mgc_schema_type`)

| Formato | Schema |
|---|---|
| Reportaje | `NewsArticle` (o `ReportageNewsArticle`) |
| Columna | `OpinionNewsArticle` |
| Crónica | `NewsArticle` |
| Entrevista | `NewsArticle` + `about` → `Person` (la persona entrevistada) |
| Ensayo | `Article` |

**Datos del artículo**
- `author`: arreglo de `Person` desde `anm_autor`, cada uno con `url` = enlace a la ficha del autor, `image` y `sameAs`. El filtro `mgc_schema_article_author` se amplía para devolver varios.
- `publisher` → `@id` de la organización.
- `articleSection` (categoría), `keywords` (temas), `inLanguage: es-CL`, `wordCount`, `mainEntityOfPage`, `isAccessibleForFree: true`.
- `VideoObject` / `AudioObject` cuando la nota tiene Medio = Video o Audio: nombre, miniatura, duración (`anm_duracion`) y `embedUrl`.

**Open Graph:** `og:type=article`, `article:published_time`, `article:modified_time`, `article:author`, `article:section` y `article:tag`.

**Páginas de autores, series y eventos**
- Autor: `ProfilePage` con `mainEntity` = `Person` (nombre, bio, foto, `jobTitle` = `anm_cargo`, `sameAs`).
- Serie: `CollectionPage` + `ItemList` de entregas. Las imágenes y descripciones salen del term meta.
- Evento: `Event` con `startDate`/`endDate` (de `anm_funciones`), `location` (`Place` + `PostalAddress` desde `anm_lugar`, `anm_direccion` y `anm_comuna`), `offers` (precio en CLP, `isAccessibleForFree` si es 0), `organizer` (Anómala o `anm_organizador`), `image` y `eventStatus`.

**Sitemaps**
- Índice con notas, páginas, autores, eventos vigentes y taxonomías (`category`, `formato`, `serie`, `post_tag`, `medio`).
- `lastmod` real: la nota más reciente de cada término.
- Sitemap de **Google News** (notas de las últimas 48 horas, espacio `news:`). *Pendiente:* postular a Google Publisher Center.

**Robots**
- `noindex, follow` en búsqueda, páginas de comentarios, `?replytocom`, Mi cuenta, Ingresar, pago, confirmaciones y archivos de fecha.
- Global: `max-image-preview:large, max-snippet:-1, max-video-preview:-1`.
- Los feeds quedan indexables.

**Migas de pan:** Inicio › Categoría › Nota; Inicio › Series › Serie; Inicio › Autores › Autor; Inicio › Cartelera › Evento.

**Qué se mantiene:**
- metabox SEO (título, descripción, OG, noindex) en notas, autores, eventos y páginas;
- gestor de redirecciones;
- tracking (GTM y GA4), respetando el consentimiento de cookies;
- conexión con Google (GA4 y Search Console) y el dashboard.

---

## 10. Pendientes con el cliente

1. Criterio entre **Cultura y Vida real**, y la categoría de cada nota del lanzamiento.
2. **Nombres de las series:** el acta indica que los propuso Alonso después de la reunión y no se acordaron en la sesión. También falta definir «Nadie te pidió que escribieras» frente a «Nadie te pidió que… crearas».
3. **Formatos:** dónde caben «artículo cultural» (Dune, hoy Reportaje) y «ensayo fotográfico y perfil» (Coleccionistas, hoy Ensayo).
4. **Disciplinas de la cartelera:** si son una lista aparte de los temas o comparten términos.
5. **Pasarela de pago.**
6. **Servicio de mailing** (newsletter y también SMTP de los correos del sitio):
   - qué servicio usarán; si ya tienen uno, cuál y cuántos contactos;
   - si el boletín se arma a mano o desde el RSS, y con qué frecuencia;
   - la dirección de envío (por ejemplo, hola@anomala.org).

   Si no tienen ninguno, la recomendación es Brevo o MailerLite. Substack no sirve porque no tiene una API abierta.
7. **Categoría en la URL** de las notas (§5).
8. Datos de los autores (fotos, bios, redes) y del primer coleccionista.
