# Cómo editar `launcher-manifest.json`

Este es el archivo de configuración base del launcher: versión de Minecraft, loader, y la lista
de mods que se instalan siempre en todo el mundo. Se descarga automáticamente cada vez que
alguien abre el launcher o le da a "Play" — editarlo y subirlo a GitHub basta, **no hace falta
recompilar ni redistribuir el launcher** para que un cambio aquí llegue a todos los jugadores.

Igual que con `mod-exceptions.json`: si el JSON queda mal formado, el launcher lo ignora y sigue
con la última versión buena que tuviera guardada — no rompe el juego a nadie, pero tampoco se
aplicará tu cambio hasta que lo arregles.

## Estructura

```json
{
  "launcherName": "Crisis Launcher",
  "gameDirName": "CrisisLauncher",
  "minecraftVersion": "1.20.1",
  "loader": "forge",
  "loaderVersion": "47.4.20",
  "portableMcVersion": "5.0.4",
  "microsoftClientId": "86f4bf3c-db1f-4695-ab11-4026b9af8cf5",
  "mods": [
    {
      "name": "crisisclient-1.0.0.jar",
      "url": "https://github.com/delayis/dlauncher/releases/download/v.10/crisisclient-1.0.0.jar"
    }
  ]
}
```

- **`launcherName`**: nombre que se muestra en el mensaje de estado tras verificar los mods. Solo
  cosmético.
- **`gameDirName`**: nombre de la carpeta donde vive todo (partida, mods, configs) dentro de la
  carpeta de datos de Windows. **No la cambies a la ligera**: si la cambias, el launcher empieza a
  usar una carpeta nueva vacía en vez de la que ya tenía el jugador — perdería su progreso/config
  local (no se borra nada, pero deja de verlo).
- **`minecraftVersion`** / **`loader`** / **`loaderVersion`**: qué versión de Minecraft y de Forge
  (u otro loader) se instala. `loader` acepta `"forge"`, `"fabric"`, `"quilt"` o `"neoforge"`.
- **`portableMcVersion`**: versión recomendada de PortableMC, solo informativa (aparece en el
  mensaje de estado).
- **`microsoftClientId`**: el Client ID de la app de Azure para el login "Sign in with
  Microsoft". No lo toques salvo que sepas lo que haces — ver el hilo sobre el error "Invalid app
  registration" si hace falta cambiarlo.
- **`mods`**: la lista base, instalada siempre para todo el mundo. Cada entrada tiene:
  - `name`: nombre del archivo `.jar` tal cual debe quedar en la carpeta de mods.
  - `url`: de dónde se descarga. El launcher compara el ETag (lo da GitHub automáticamente) para
    saber si hay que descargar de nuevo — no hace falta llevar ningún número de versión a mano,
    basta con que el contenido del archivo cambie.

## ⚠️ El campo `"server"` no hace nada

Puede que veas (o te encuentres tentado a añadir) un bloque así:

```json
"server": { "host": "localhost", "port": 25565 }
```

**El launcher lo ignora por completo.** El servidor real al que conecta el botón "Play" está
configurado dentro del propio mod `crisis-client` (`ClientConfig.java`, campos `serverHost` /
`serverPort`), no aquí. Cambiar esto en el manifest no tiene ningún efecto — si necesitas cambiar
a qué servidor se conecta el juego, hay que tocar el mod, no este archivo.

## Cómo añadir, actualizar o quitar un mod

- **Añadir**: agrega una entrada nueva a `mods` con su `name` y `url`. Se descargará solo la
  próxima vez que alguien abra el launcher.
- **Actualizar** (mismo mod, contenido nuevo): sube el `.jar` nuevo a la misma URL (mismo nombre
  de asset en la misma release), sin tocar el manifest. El launcher detecta que el contenido
  cambió (por el ETag) y lo redescarga solo.
- **Quitar**: borra la entrada de `mods`. La próxima vez que un jugador abra el launcher, ese
  `.jar` desaparece automáticamente de su carpeta de mods (salvo que esté protegido por
  `mod-exceptions.json` — ver esa guía).

## Cuánto tarda en verse un cambio

El launcher pide este archivo a la API de GitHub, que cachea unos ~60 segundos como mucho (en la
práctica suele ser casi al instante). No hace falta esperar 5 minutos ni reiniciar nada más de una
vez - dale a "Play" y ya está.

## Errores típicos a evitar

- ❌ Olvidar la coma entre dos mods de la lista.
- ❌ Dejar una coma sobrante después del último elemento.
- ❌ Cambiar `gameDirName` pensando que es cosmético (ver aviso arriba).
- ❌ Poner una `url` que no sea HTTPS directa al archivo (debe descargar el `.jar` tal cual, no
  una página HTML de GitHub).
- ❌ Editar el bloque `"server"` esperando que cambie a qué servidor conecta el juego (no hace
  nada, ver aviso arriba).

## Antes de hacer commit: valida el JSON

Pega el contenido en https://jsonlint.com/ y comprueba que no marca ningún error antes de guardar
el cambio en GitHub.
