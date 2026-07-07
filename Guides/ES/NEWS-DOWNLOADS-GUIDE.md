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
  { "name": "2026-07-10-mazmorra-norte.json", "path": "news/2026-07-10-mazmorra-norte.json", "type": "file" },
  { "name": "2026-06-28-bienvenida.json", "path": "news/2026-06-28-bienvenida.json", "type": "file" }
]
```

Con esa lista, pedimos el contenido de cada archivo `.json` uno por uno (igual que pedimos
`launcher-manifest.json`), los juntamos todos en una sola lista, y los ordenamos por el campo
`date` de más reciente a más antigua — automático, sin que nadie tenga que colocarlos a mano en
ningún orden concreto.

Si un archivo en concreto está mal formado, simplemente se descarta esa noticia y seguimos con
las demás — un error de sintaxis en una no debe tirar abajo el panel entero.

Código relevante: `fetch_remote_news()` en [news.rs](src-tauri/src/news.rs).

## 3. Cómo funciona lo de la imagen (la parte que no hacemos nosotros)

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

## 4. Qué depende de GitHub ahora mismo (y qué no)

- **Las imágenes**: no dependen de GitHub en absoluto, ya son agnósticas del hosting (ver punto
  3) — la URL puede apuntar a cualquier sitio ya mismo, sin tocar código.
- **El listado y contenido de `news/`**: esto SÍ está hardcodeado a la API de GitHub, en la
  constante `NEWS_DIR_URL` en [news.rs](src-tauri/src/news.rs), más la URL que se construye por
  archivo dentro de `fetch_remote_news()`. Mover *esto* fuera de GitHub sí exige tocar código (ver
  punto 6).

## 5. Qué tendríamos que pensar si algún día nos vamos de GitHub

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

## 6. Qué tendríamos que tocar en el código para cambiar de sitio

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
