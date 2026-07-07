# Cómo funcionan las descargas y actualizaciones de mods

Esta guía explica cómo decidimos si un mod se descarga, se actualiza, o se deja tal cual — y qué
cambiaríamos si algún día dejamos de alojar mods/config en GitHub.

## 1. La idea, en plata

Cada vez que alguien abre el launcher (o le da a Play), comprobamos si el mod que tenemos
guardado en el ordenador del jugador sigue siendo exactamente igual al que hay ahora mismo en
internet. Si es igual, no hacemos nada — no tiene sentido volver a descargar algo que no cambió.
Si es distinto, lo descargamos de nuevo y sustituimos el archivo viejo.

La pregunta es: ¿cómo sabemos si "es igual" sin tener que descargarlo entero cada vez solo para
comparar? Ahí es donde entra un pequeño truco técnico, explicado abajo con más detalle para quien
vaya a tocar el código — pero la idea de fondo es esta, nada más.

## 2. Cómo lo hacemos por dentro: el ETag

Cada mod en `launcher-manifest.json` tiene una `url`. Cada vez que comprobamos el modpack,
pedimos esa `url` y leemos una cabecera especial que viene en la respuesta, llamada **ETag** —
una especie de huella dactilar del contenido del archivo, que calcula el propio servidor (GitHub,
en nuestro caso) sin que nosotros tengamos que descargar nada para obtenerla. Guardamos esa huella
en `mod_etags.json`, en la carpeta de datos del launcher, una por mod.

La siguiente vez que comprobamos, comparamos la huella que acabamos de recibir contra la que
teníamos guardada:

- **Coinciden** → el contenido no cambió de verdad, no descargamos nada (da igual si subiste el
  archivo de nuevo, lo borraste y resubiste, etc. — lo que miramos es el contenido, no la acción).
- **No coinciden** (o no teníamos huella guardada todavía) → descargamos. Si el archivo **ya
  existía** en el disco del jugador, lo contamos como **actualización**; si **no existía**, como
  **descarga nueva**. Esta distinción es la que alimenta la barra de progreso ("Updating 1/3" vs
  "Downloading 1/7").

Código relevante: `check_modpack()` en [modpack.rs](src-tauri/src/modpack.rs).

## 3. El caso "subimos el mismo archivo y no detectó nada"

Esto es lo esperado, no un fallo. La huella (ETag) depende del **contenido en sí** (básicamente
un hash/checksum de los bytes), no de "la acción de subirlo". Si borramos un asset en GitHub y
subimos un archivo **byte a byte idéntico** al que había antes, GitHub calcula la misma huella
que ya tenía — porque, para el archivo, no cambió nada. Hacemos bien en no re-descargar algo que
no cambió: sería tráfico y tiempo desperdiciados.

Por eso, para que detectemos un cambio de verdad, hacen falta bytes distintos, no solo "subirlo
de nuevo". En la práctica esto pasa con:
- Un cambio real de código, por pequeño que sea.
- Recompilar sin cambiar nada: cada build de Gradle mete timestamps internos distintos en el
  `.jar`, así que los bytes cambian igual aunque el código sea el mismo (por eso al probar esto
  nos bastaba con recompilar, sin tocar código, para generar una huella distinta).

## 4. Qué depende de GitHub ahora mismo (y qué no)

Hay dos cosas completamente distintas que hoy viven en GitHub, y conviene no confundirlas:

- **Los mods en sí** (`crisisclient-1.0.0.jar`, etc.): su ubicación es solo lo que pongamos en el
  campo `url` de cada entrada en `launcher-manifest.json`. El código no sabe ni le importa que
  sea GitHub — **ya es agnóstico del hosting**. Podríamos cambiar esa URL hoy mismo a cualquier
  otro sitio (nuestro propio servidor, un bucket S3, Cloudflare R2, Backblaze...) sin tocar ni
  una línea de Rust, siempre que ese sitio devuelva una cabecera `ETag` de verdad.
- **`launcher-manifest.json` y `mod-exceptions.json` en sí** (la config base y las excepciones):
  estos SÍ están hardcodeados a la API de GitHub, en dos constantes (`MANIFEST_URL` y
  `MOD_EXCEPTIONS_URL` en [modpack.rs](src-tauri/src/modpack.rs)). Mover *estos dos archivos*
  fuera de GitHub sí nos exigiría tocar código (ver sección 6).

## 5. Qué tendríamos que pensar si algún día nos vamos de GitHub

Si en el futuro decidimos mover mods y/o la config a otro sitio, esto es lo que cambiaría de
fondo:

- **¿El nuevo sitio devuelve ETag?** La mayoría de almacenamiento de objetos (S3, R2, Backblaze,
  un nginx sirviendo archivos estáticos) lo hace de forma automática, calculado sobre el
  contenido — nuestro mecanismo actual seguiría funcionando igual. Si el sitio NO da ETag fiable
  (algunos servidores custom no lo hacen, o lo basan en fecha de modificación en vez de
  contenido), tendríamos que cambiar de estrategia (ver más abajo).
- **Caché del propio sitio**: ya nos pasó con GitHub — `raw.githubusercontent.com` cacheaba 5
  minutos y su CDN ignoraba nuestros intentos de saltárnoslo, así que cambiamos a la API de
  GitHub (~60s de caché). Con cualquier sitio nuevo tendríamos que revisarlo igual: pedir la URL
  con `curl -I` y mirar la cabecera `Cache-Control`/`Age` antes de asumir que los cambios se ven
  al instante.
- **Autenticación**: ahora mismo todo es público, sin credenciales. Si el sitio nuevo exige
  alguna clave/token para leer los archivos, tendríamos que meter ese token dentro del `.exe`
  del launcher — lo que lo haría tan "público" como el Client ID de Microsoft del que hablamos
  antes: cualquiera podría extraerlo del binario. No nos serviría para alojar contenido realmente
  privado sin montar un backend propio con autenticación por jugador, que es un proyecto bastante
  más grande que esto.
- **Sin ETag fiable, alternativas para detectar cambios**:
  - Cabecera `Last-Modified` en vez de `ETag` — funciona parecido, pero es menos preciso (una
    fecha, no un hash; dos subidas distintas el mismo segundo no se distinguirían).
  - Mantener un hash o número de versión a mano en el manifest (ej. añadir un campo `sha256` por
    mod) y comparar eso en vez del ETag — perderíamos la comodidad de "no hay que llevar nada a
    mano", pero funcionaría con cualquier sitio, sin importar qué cabeceras dé.

## 6. Qué tendríamos que tocar en el código para cambiar de sitio

Suponiendo que nos vamos a un sitio que **sí** da ETag fiable (el caso simple, cubre
S3/R2/Backblaze y la mayoría de servidores de archivos estáticos):

1. **Para los mods en sí**: nada en el código — solo cambiar el campo `url` de cada mod en
   `launcher-manifest.json`.
2. **Para mover `launcher-manifest.json`/`mod-exceptions.json` fuera de GitHub**, en
   [modpack.rs](src-tauri/src/modpack.rs):
   - Cambiar las constantes `MANIFEST_URL` y `MOD_EXCEPTIONS_URL` a las nuevas URLs.
   - Quitar (o adaptar) la cabecera `Accept: application/vnd.github.raw+json` en
     `fetch_remote_manifest()`/`fetch_remote_mod_exceptions()` — es específica de la API de
     GitHub (pide el contenido "crudo" en vez de JSON envuelto en base64); un servidor de
     archivos normal no la entiende ni la necesita, así que sencillamente quitaríamos esa línea
     `.header(...)`.
   - Revisar de nuevo el tema de caché del nuevo sitio (punto anterior) y ajustar, si hiciera
     falta, cuánto tiempo le decimos a los jugadores que esperen tras un cambio.

Si en cambio el sitio nuevo **no** da ETag fiable, además tendríamos que:
   - Añadir un campo de hash/versión manual a `ModEntry` en modpack.rs (ej. `pub sha256:
     Option<String>`), y en `check_modpack()` usar ese campo en vez de (o además de) el ETag para
     decidir `up_to_date` — descargando y calculando el hash real del contenido si hiciera falta
     comparar contra ese valor.
   - Esto sí añadiría mantenimiento manual (mantener el hash actualizado en el manifest cada vez
     que cambie un mod), a cambio de funcionar en cualquier sitio sin depender de sus cabeceras.
