# Cómo añadir noticias al launcher

Cada noticia es su propio archivo `.json` dentro de la carpeta `news/` del repositorio — no un
único archivo gigante con todas apiladas dentro. Para publicar una noticia nueva, basta con subir
un archivo nuevo a esa carpeta; para editar una existente, se edita su archivo; para quitarla, se
borra el archivo. El launcher lo descarga solo, igual que el `launcher-manifest.json` — **no hace
falta recompilar ni redistribuir nada** para que una noticia nueva llegue a todos los jugadores.

Si algún archivo de `news/` queda mal formado, esa noticia en concreto simplemente no aparece —
no rompe las demás noticias ni el resto del launcher.

## Cómo nombrar y subir el archivo

El nombre del archivo sí importa en un punto: la última parte antes de `.json` tiene que ser el
código de idioma de esa noticia — `es`, `en`, `fr`, `de` o `ru`. Por ejemplo:

```
news/2026-07-10-mazmorra-norte.es.json
news/2026-07-10-mazmorra-norte.en.json
```

El resto del nombre (`2026-07-10-mazmorra-norte`) es libre, solo para que os aclaréis vosotros
mismos en el repositorio. El launcher decide en qué idioma está cada noticia **únicamente por ese
sufijo del nombre**, nunca por el contenido de dentro del archivo — así que si un archivo no
termina en uno de esos códigos de idioma reconocidos, esa noticia se ignora por completo (no
aparece en ningún idioma).

No hace falta traducir cada noticia a los 5 idiomas — podéis subir solo la versión en español al
principio, y añadir `.en.json`/`.fr.json`/etc. más adelante si os apetece. Un jugador con el
idioma seleccionado en el launcher para el que no exista archivo, simplemente no verá esa
noticia; si no hay NINGUNA noticia en su idioma, el panel entero desaparece (no se queda vacío ni
roto, simplemente no se muestra).

## Estructura de cada noticia

```json
{
  "id": "mazmorra-norte",
  "date": "2026-07-10",
  "title": "Nueva mazmorra añadida",
  "image": "https://raw.githubusercontent.com/delayis/dlauncher/main/news/images/mazmorra.jpg",
  "summary": "Hemos añadido una mazmorra nueva al norte del spawn, con jefes y loot exclusivo.",
  "body": "Texto completo que se ve al abrir la noticia, puede ser bastante más largo que el resumen."
}
```

- **`id`**: identificador único de la noticia, solo interno (no se muestra). Basta con que no se
  repita entre noticias.
- **`date`**: en formato `AAAA-MM-DD` (por ejemplo `2026-07-10`). Además de ser lo que se muestra,
  **se usa para ordenar las noticias automáticamente** (ver más abajo) — si no respeta ese
  formato, el orden puede salir raro.
- **`title`**: título, se ve tanto en la vista rápida como al abrir la noticia.
- **`image`**: la URL completa de la imagen de portada. Ver la sección de abajo sobre cómo
  funciona esto exactamente.
- **`summary`**: texto corto para la vista rápida (1-2 líneas, se corta con "…" si es más largo).
- **`body`**: el texto completo que se ve al hacer clic y abrir la noticia.

## El orden ahora es automático

A diferencia de como funcionaba antes (una lista a mano donde había que acordarse de poner la
noticia nueva al principio), ahora el launcher junta todas las noticias de `news/` y las ordena
él solo por el campo `date`, de más reciente a más antigua. No hace falta preocuparse por dónde
la subes ni en qué orden — solo que el campo `date` sea correcto.

## Cómo funciona lo de la imagen

El campo `image` es una URL completa y directa a la imagen — el launcher **no descarga la imagen
ni la procesa de ninguna manera**: simplemente se la pasa tal cual a la ventana del launcher, que
la carga como cualquier página web carga una `<img>`. Eso quiere decir que:

- La imagen puede vivir literalmente donde quieras: en la misma carpeta `news/` del repo (como en
  el ejemplo de arriba), en Imgur, en un adjunto de Discord, en vuestra propia web... cualquier
  URL pública y directa a un archivo de imagen sirve.
- Si subes la imagen al mismo repositorio de GitHub, **no hace falta buscar ningún botón de
  "Raw" ni convertir la URL a mano**: pega tal cual el enlace que ves en la barra de direcciones
  al abrir la imagen en GitHub (`github.com/.../blob/...`) — el launcher detecta que es un enlace
  de GitHub y lo convierte él solo a la URL directa que necesita.
- Como la imagen la pide directamente la ventana del launcher (no pasa por nuestro código), un
  cambio en la imagen se ve tan rápido como lo permita quien la esté sirviendo — si usas GitHub
  raw, puede tardar hasta 5 minutos en refrescarse por su propio cache, distinto a los ~60
  segundos del resto de datos (ver la guía de migración para más detalle sobre esto).

## Qué tamaño de imagen encaja perfectamente

La imagen se ve en dos sitios con distinta forma:

- **Vista rápida (card)**: una caja de 320×180 píxeles — proporción **16:9**.
- **Vista completa (al abrir la noticia)**: hasta 460 de ancho × 220 de alto como máximo —
  proporción algo más ancha que 16:9.

En ambos casos, si subes una imagen que no tenga exactamente esa proporción, no se recorta ni se
deforma — se ve completa, con una franja oscura rellenando el hueco sobrante (como el "letterbox"
de una peli en pantalla panorámica). Pero si quieres que encaje sin ninguna franja en la vista
rápida, usa una imagen en **proporción 16:9**, que es justo la proporción de una captura de
Minecraft normal (1920×1080, 2560×1440, etc. son todas 16:9) — no hace falta que sea ese tamaño
exacto, cualquier imagen 16:9 encaja igual de bien porque se escala.

## Comprime la imagen antes de subirla

Aunque subas una captura a 2560×1440, en el launcher nunca se ve a ese tamaño real — se ve dentro
de una caja pequeña (320×180 en la vista rápida). Eso significa que estás haciendo descargar un
archivo de varios MB para acabar mostrándolo diminuto, lo que solo hace que tarde más en cargar.

Antes de subir una captura, redúcele el tamaño con cualquier editor de imágenes (Paint,
Photoshop, o una web como https://squoosh.app/) a algo parecido a **1280 píxeles de ancho**
(manteniendo la proporción 16:9, así queda en unos 1280×720). Se ve exactamente igual dentro del
launcher, pero el archivo pesa mucho menos.

## Resumen rápido antes de publicar una noticia

- ✅ Imagen en proporción 16:9 (o lo más parecido posible).
- ✅ Imagen redimensionada a ~1280px de ancho antes de subirla, no la captura original a máxima
  resolución.
- ✅ `date` en formato `AAAA-MM-DD` correcto (el orden depende de esto, ya no hay que colocarla a
  mano en ningún sitio).
- ✅ `id` distinto al de cualquier otra noticia existente (no hace falta que coincida entre las
  distintas versiones de idioma de la misma noticia, cada archivo es independiente).
- ✅ `image`: si la imagen está en GitHub, vale con pegar el enlace normal de la página que la
  muestra — el launcher lo convierte solo. Si está en otro sitio (Imgur, Discord...), tiene que
  ser ya un enlace directo al archivo.
- ✅ El archivo termina en `.es.json`, `.en.json`, `.fr.json`, `.de.json` o `.ru.json` — si no,
  esa noticia no aparecerá para nadie.
