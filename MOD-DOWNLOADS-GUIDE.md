# Cómo funcionan las descargas y actualizaciones de mods

Esta guía explica el mecanismo interno que decide si un mod se descarga, se actualiza, o se deja
tal cual — y qué pasaría si algún día dejamos de alojar mods/config en GitHub.

## 1. Cómo se detecta un cambio: el ETag

Cada mod en `launcher-manifest.json` tiene una `url`. Cada vez que el launcher comprueba el
modpack (al pulsar Play), hace una petición a esa `url` y lee la cabecera HTTP **ETag** de la
respuesta — una huella del contenido del archivo, calculada por el servidor (GitHub, en nuestro
caso). Esa huella se guarda en `mod_etags.json`, en la carpeta de datos del launcher, indexada
por el `name` del mod.

En la siguiente comprobación, el launcher compara el ETag que acaba de recibir contra el que
tenía guardado:

- **Coinciden** → el contenido no cambió de verdad, no se descarga nada (aunque hayas subido el
  archivo de nuevo, borrado y resubido, etc. — lo que importa es el contenido, no la acción).
- **No coinciden** (o no había ETag guardado todavía) → se descarga. Si el archivo **ya existía**
  en el disco del jugador, se cuenta como **actualización**; si **no existía**, como **descarga
  nueva**. Esta distinción es la que alimenta la barra de progreso ("Updating 1/3" vs
  "Downloading 1/7").

Código relevante: `check_modpack()` en [modpack.rs](src-tauri/src/modpack.rs).

## 2. El caso "subí el mismo archivo y no detectó nada"

Esto es intencionado, no un fallo. El ETag es una huella del **contenido en sí** (básicamente un
hash/checksum de los bytes), no del "evento de subida". Si borras un asset en GitHub y subes un
archivo **byte a byte idéntico** al que había antes, GitHub calcula la misma huella que ya tenía
— porque, para el archivo, no cambió nada. El launcher hace bien en no re-descargar algo que no
cambió: sería tráfico y tiempo desperdiciados.

Por eso, para que un cambio se detecte de verdad, hace falta que los **bytes del archivo**
cambien, no solo que "lo subas de nuevo". En la práctica esto pasa solo con:
- Un cambio real de código, por pequeño que sea.
- Recompilar sin cambiar nada: cada build de Gradle mete timestamps internos distintos en el
  `.jar`, así que los bytes cambian igualmente aunque el código sea el mismo (por eso las pruebas
  de este launcher usaban "solo recompila, no hace falta cambiar código" como forma rápida de
  generar un ETag distinto).

## 3. Qué depende de GitHub ahora mismo (y qué no)

Hay dos cosas completamente distintas que hoy viven en GitHub, y conviene no confundirlas:

- **Los mods en sí** (`crisisclient-1.0.0.jar`, etc.): su ubicación es solo lo que pongas en el
  campo `url` de cada entrada en `launcher-manifest.json`. El código no sabe ni le importa que
  sea GitHub — **ya es agnóstico del hosting**. Podrías cambiar esa URL hoy mismo a cualquier
  otro sitio (tu propio servidor, un bucket S3, Cloudflare R2, Backblaze...) sin tocar ni una
  línea de Rust, siempre que ese sitio devuelva una cabecera `ETag` de verdad.
- **`launcher-manifest.json` y `mod-exceptions.json` en sí** (la config base y las excepciones):
  estos SÍ están hardcodeados a la API de GitHub, en dos constantes (`MANIFEST_URL` y
  `MOD_EXCEPTIONS_URL` en [modpack.rs](src-tauri/src/modpack.rs)). Mover *estos dos archivos*
  fuera de GitHub sí requiere tocar código (ver sección 5).

## 4. Consideraciones si algún día os vais de GitHub

Si en el futuro decidís mover mods y/o la config a otro sitio, esto es lo que cambiaría de fondo:

- **¿El nuevo host devuelve ETag?** La mayoría de almacenamiento de objetos (S3, R2, Backblaze,
  un nginx sirviendo archivos estáticos) sí lo hace de forma automática, calculado sobre el
  contenido — el mecanismo actual seguiría funcionando igual. Si el host NO da ETag fiable
  (algunos servidores custom no lo hacen, o lo basan en fecha de modificación en vez de
  contenido), habría que cambiar la estrategia de detección (ver más abajo).
- **Caché del propio host**: ya nos pasó con GitHub — `raw.githubusercontent.com` cacheaba 5
  minutos y su CDN ignoraba nuestros intentos de saltárselo, así que cambiamos a la API de
  GitHub (~60s de caché). Cualquier host nuevo hay que revisarlo de la misma forma: pedir la URL
  con `curl -I` y mirar la cabecera `Cache-Control`/`Age` antes de asumir que los cambios se ven
  al instante.
- **Autenticación**: ahora mismo todo es público, sin credenciales. Si el nuevo host exige alguna
  clave/token para leer los archivos, ese token tendría que ir embebido en el `.exe` del
  launcher — lo que lo hace tan "público" como el Client ID de Microsoft del que hablamos (ver
  esa conversación): cualquiera puede extraerlo del binario. No sirve para alojar contenido
  realmente privado sin un backend propio con autenticación por jugador, que es un proyecto
  bastante más grande que esto.
- **Sin ETag fiable, alternativas para detectar cambios**:
  - Cabecera `Last-Modified` en vez de `ETag` — funciona parecido, pero es menos preciso (una
    fecha, no un hash; dos subidas distintas el mismo segundo no se distinguirían).
  - Mantener un hash o número de versión a mano en el manifest (ej. añadir un campo `sha256` por
    mod) y comparar eso en vez del ETag — pierde la comodidad de "no hay que llevar nada a mano",
    pero funciona con cualquier host, sin importar qué cabeceras dé.

## 5. Qué habría que tocar en el código para cambiar de sitio

Suponiendo que os vais a un host que **sí** da ETag fiable (el caso simple, cubre S3/R2/Backblaze
y la mayoría de servidores de archivos estáticos):

1. **Para los mods en sí**: nada en el código — solo cambiar el campo `url` de cada mod en
   `launcher-manifest.json`.
2. **Para mover `launcher-manifest.json`/`mod-exceptions.json` fuera de GitHub**, en
   [modpack.rs](src-tauri/src/modpack.rs):
   - Cambiar las constantes `MANIFEST_URL` y `MOD_EXCEPTIONS_URL` a las nuevas URLs.
   - Quitar (o adaptar) la cabecera `Accept: application/vnd.github.raw+json` en
     `fetch_remote_manifest()`/`fetch_remote_mod_exceptions()` — es específica de la API de
     GitHub (pide el contenido "crudo" en vez de JSON envuelto en base64); un servidor de
     archivos normal no la entiende ni la necesita, así que sencillamente se quita esa línea
     `.header(...)`.
   - Revisar de nuevo el tema de caché del nuevo host (punto anterior) y ajustar la frecuencia
     con la que avisáis a los jugadores de que esperen tras un cambio, si hiciera falta.

Si en cambio el nuevo host **no** da ETag fiable, adicionalmente habría que:
   - Añadir un campo de hash/versión manual a `ModEntry` en modpack.rs (ej. `pub sha256:
     Option<String>`), y en `check_modpack()` usar ese campo en vez de (o además de) el ETag para
     decidir `up_to_date` — descargando y calculando el hash real del contenido si hace falta
     comparar contra ese valor.
   - Esto sí añade mantenimiento manual (mantener el hash actualizado en el manifest cada vez que
     cambie un mod), a cambio de funcionar en cualquier host sin depender de sus cabeceras.
