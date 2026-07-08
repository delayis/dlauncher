# Cómo funciona el sistema de noticias por dentro (y cómo migrarlo)

Esta guía explica cómo el launcher junta las noticias a partir de varios archivos sueltos, cómo
funciona exactamente lo de la imagen, y qué cambiaríamos si algún día dejamos de alojar esto en
GitHub. Es la contraparte técnica de [NEWS-GUIDE.md](NEWS-GUIDE.md), que explica cómo publicar
una noticia sin entrar en estos detalles.

## 1. La idea, en plata

En vez de un único archivo con todas las noticias apiladas dentro (que crecería sin parar y sería
fácil de romper al editarlo a mano), cada noticia es su propio archivo `.json` dentro de la
carpeta `news/` del repositorio. El launcher, al arrancar, pregunta "¿qué archivos hay ahora mismo
dentro de `news/`?", se descarga el contenido de cada uno, y los junta todos en una sola lista
ordenada por fecha para mostrarlos.

## 2. Cómo lo hacemos por dentro: listar la carpeta

A diferencia de `launcher-manifest.json` (que es un archivo suelto y ya sabemos su URL exacta de
antemano), con `news/` no sabemos cuántos archivos hay ni cómo se llaman — así que primero le
preguntamos a la API de GitHub por el **listado de la carpeta** (no por un archivo en concreto).
GitHub responde con algo así:

```json
[
  { "name": "2026-07-10-mazmorra-norte.es.json", "path": "news/2026-07-10-mazmorra-norte.es.json", "type": "file" },
  { "name": "2026-06-28-bienvenida.en.json", "path": "news/2026-06-28-bienvenida.en.json", "type": "file" }
]
```

Con esa lista, pedimos el contenido de cada archivo `.json` uno por uno (igual que pedimos
`launcher-manifest.json`), los juntamos todos en una sola lista, y los ordenamos por el campo
`date` de más reciente a más antigua — automático, sin que nadie tenga que colocarlos a mano en
ningún orden concreto.

Si un archivo en concreto está mal formado, simplemente se descarta esa noticia y seguimos con
las demás — un error de sintaxis en una no debe tirar abajo el panel entero.

Código relevante: `fetch_remote_news()` en [news.rs](src-tauri/src/news.rs).

## 3. Cómo se detecta el idioma de cada noticia

El idioma no viaja dentro del contenido del JSON — se decide **solo por el nombre del archivo**,
mirando el sufijo justo antes de `.json` (`detect_lang()` en [news.rs](src-tauri/src/news.rs)
compara ese sufijo contra la lista fija `KNOWN_LANGS` = `es`/`en`/`fr`/`de`/`ru`, los mismos
códigos que ya usa `src/i18n.ts`). Si el nombre no termina en uno de esos códigos, el archivo se
descarta directamente en la fase de listado, antes incluso de pedir su contenido.

Descargamos y cacheamos **todas las noticias de todos los idiomas de golpe**, cada una con su
`lang` ya detectado. El filtrado por el idioma que el jugador tenga seleccionado en Ajustes ocurre
después, en el frontend (`main.tsx`, filtrando el array `news` recibido por `lang === language`) —
no hace falta volver a pedir nada a GitHub cuando alguien cambia de idioma, el cambio es
instantáneo porque ya tenemos todo descargado.

## 4. Cómo funciona lo de la imagen (la parte que no hacemos nosotros)

Esta es la diferencia clave frente a los mods: un mod es un archivo que Minecraft necesita tener
en el disco para poder cargarlo, así que el launcher **sí** tiene que descargarlo de verdad a la
carpeta de mods. Una imagen de noticia, en cambio, solo se **muestra en pantalla** — así que el
campo `image` de cada noticia no es más que una URL de texto, y esa URL se le pasa tal cual a la
ventana del launcher (que por dentro es literalmente un navegador embebido). La ventana hace lo
mismo que haría cualquier página web al encontrarse una etiqueta `<img src="...">`: pide esa URL
por su cuenta y muestra lo que reciba.

En la práctica esto significa:

- **Nuestro código Rust nunca toca la imagen.** No la descarga, no la guarda en disco, no la
  procesa. Todo el trabajo de imágenes vive en el campo `image` (texto) + el navegador embebido.
- **La imagen puede estar en cualquier sitio público**, sin que tengamos que cambiar ni una línea
  de código: GitHub, Imgur, un adjunto de Discord, un bucket S3 público, vuestra propia web... La
  única condición es que sea una URL directa a un archivo de imagen (no una página HTML que la
  muestra dentro).
- **Migrar dónde viven las imágenes no tiene nada que ver con migrar dónde vive el JSON de las
  noticias.** Son dos cosas completamente independientes — podríamos mover el listado de noticias
  a otro sitio y dejar las imágenes en GitHub (o al revés) sin ningún problema.

Hay una excepción práctica a "el campo `image` es una URL directa, tal cual": la mayoría de gente
que sube una imagen a GitHub copia el enlace normal de la página que la muestra
(`github.com/.../blob/...`), no la URL raw que hace falta para que una `<img>` la cargue. En vez
de exigir que sepan hacer esa conversión a mano, `fetch_remote_news()` llama a
`normalize_image_url()`, que detecta ese patrón y lo reescribe a `raw.githubusercontent.com`
automáticamente antes de guardarlo en cache. Cualquier URL que no sea una página de GitHub se
deja intacta, ya que se asume que ya es un enlace directo.

## 5. Qué depende de GitHub ahora mismo (y qué no)

- **Las imágenes**: no dependen de GitHub en absoluto, ya son agnósticas del hosting (ver punto
  4) — la URL puede apuntar a cualquier sitio ya mismo, sin tocar código.
- **El listado y contenido de `news/`**: esto SÍ está hardcodeado a la API de GitHub, en la
  constante `NEWS_DIR_URL` en [news.rs](src-tauri/src/news.rs), más la URL que se construye por
  archivo dentro de `fetch_remote_news()`. Mover *esto* fuera de GitHub sí exige tocar código (ver
  punto 7).
- **La detección de idioma**: no depende de GitHub para nada — es una simple comparación de texto
  sobre el nombre del archivo (`detect_lang()`), da igual de dónde venga el listado.

## 6. Qué tendríamos que pensar si algún día nos vamos de GitHub

- **¿El sitio nuevo permite "listar una carpeta"?** Esto es lo más particular de este sistema
  frente al manifest o las excepciones de mods (que son archivos sueltos con URL fija). Un bucket
  S3/R2 lo permite (hay una llamada para "listar objetos con este prefijo"), un servidor propio
  necesitaría un endpoint que devuelva ese listado a mano. Si el sitio nuevo no ofrece nada
  parecido, la alternativa más simple es mantener un pequeño archivo índice (por ejemplo
  `news/index.json` con solo la lista de nombres de archivo) que sí se lee con una URL fija, y
  usarlo para saber qué archivos pedir después — perdemos el "solo sube el archivo y ya está" y
  hay que acordarse de añadir su nombre al índice también, pero funciona en cualquier sitio.
- **Caché del sitio nuevo**: exactamente la misma consideración que ya tuvimos con el manifest —
  antes de asumir que un cambio se ve al instante, comprobar con `curl -I` cuánto cachea el sitio
  nuevo (`Cache-Control`/`Age`) tanto para el listado como para cada archivo individual.
- **Las imágenes no cambian en nada** aunque migremos el JSON de las noticias a otro sitio — siguen
  funcionando igual, vivan donde vivan.

## 7. Qué tendríamos que tocar en el código para cambiar de sitio

Suponiendo que el sitio nuevo sí permite listar una carpeta (el caso simple, cubre S3/R2 y
similares), en [news.rs](src-tauri/src/news.rs):

- Cambiar la constante `NEWS_DIR_URL` y la construcción de la URL por archivo dentro de
  `fetch_remote_news()`.
- Adaptar el parseo de `GithubDirEntry` a la forma en que ese sitio devuelva su propio listado
  (los campos `name`/`path`/`type` son específicos de la respuesta de GitHub).
- Quitar la cabecera `Accept: application/vnd.github.raw+json` al pedir el contenido de cada
  archivo — es específica de la API de GitHub, un servidor de archivos normal no la necesita.

Si el sitio nuevo NO permite listar una carpeta, además habría que:

- Mantener un archivo índice (`news/index.json` o similar) con la lista de nombres de archivo, y
  cambiar `fetch_remote_news()` para leer primero ese índice en vez de pedir un listado de
  directorio a la API.

En ningún caso hace falta tocar nada del frontend (`main.tsx`) ni de cómo se muestran las
imágenes — esa parte es completamente independiente de dónde vivan los datos.
