# Cómo editar `mod-exceptions.json`

Este archivo controla qué mods "extra" (fuera de la lista base de `launcher-manifest.json`)
puede tener un jugador sin que el launcher se los borre. **El launcher nunca descarga nada de
aquí** — el jugador se pone el `.jar` él mismo, y esto solo evita que desaparezca.

Si rompes el JSON (falta una coma, una llave sin cerrar...), no pasa nada grave: el launcher
valida el archivo antes de usarlo. Si está mal formado, simplemente lo ignora y sigue con la
última versión buena que tuviera guardada — no rompe el juego a nadie. Pero tampoco se aplicará
tu cambio hasta que lo arregles, así que conviene revisarlo antes de dar el commit por bueno.

## Estructura

```json
{
  "global": [
    "xaeros-minimap.jar"
  ],
  "perPlayer": {
    "0369b5d781cd4bc7b46f0216927b4e3f": {
      "lastKnownUsername": "SamsaraMode",
      "mods": ["debug-tool.jar"]
    }
  }
}
```

- **`global`**: lista de nombres de archivo permitidos para **todos** los jugadores. Si no quieres
  ninguno, deja `[]` (una lista vacía) — nunca borres la clave entera.
- **`perPlayer`**: un jugador por entrada. La clave es su **UUID** (sin guiones), no su nombre.
  - `lastKnownUsername`: solo para que tú sepas quién es quién al mirar el archivo. No hace nada,
    el launcher no lo lee para emparejar - eso es siempre por el UUID.
  - `mods`: lista de archivos permitidos solo para ese jugador, además de los de `global`.

Un jugador que no tenga la excepción "global" y "perPlayer" a la vez, sino que las dos se suman:
si un mod está en `global` Y en su `perPlayer`, no pasa nada, simplemente está permitido igual.

## Cómo conseguir el UUID de un jugador

En el panel **Account** del launcher, el jugador tiene un botón **"Copiar mi UUID"**. Con eso lo
copia directo al portapapeles y te lo puede pasar (por Discord, chat, donde sea) para pedirte que
le añadas una excepción de mod, global o per-player.

## Ejemplo completo (varios mods globales + varios jugadores)

```json
{
  "global": [
    "xaeros-minimap.jar",
    "xaeros-worldmap.jar",
    "appleskin.jar"
  ],
  "perPlayer": {
    "0369b5d781cd4bc7b46f0216927b4e3f": {
      "lastKnownUsername": "SamsaraMode",
      "mods": [
        "debug-tool.jar",
        "jei-1.20.1.jar",
        "worldedit-cui.jar"
      ]
    },
    "e1a2c3d4b5f60718293a4b5c6d7e8f90": {
      "lastKnownUsername": "OtroJugador",
      "mods": [
        "shulkerboxtooltip.jar",
        "inventoryhud.jar",
        "toolstats.jar"
      ]
    },
    "9f8e7d6c5b4a39281706f5e4d3c2b1a0": {
      "lastKnownUsername": "TercerJugador",
      "mods": [
        "mousetweaks.jar",
        "controlling.jar",
        "clienttweaks.jar",
        "betterf3.jar"
      ]
    }
  }
}
```

Aquí, `SamsaraMode` y `OtroJugador` tienen 3 mods propios + los 3 globales (6 en total), y
`TercerJugador` tiene 4 propios + los 3 globales (7 en total). Los UUIDs de `OtroJugador` y
`TercerJugador` son de ejemplo — sustitúyelos por UUIDs reales.

## Errores típicos a evitar

- ❌ Poner el nombre de usuario como clave de `perPlayer` en vez del UUID (`"SamsaraMode": {...}`
  no funciona - tiene que ser el UUID).
- ❌ Olvidar la coma entre dos entradas del mismo objeto/lista.
- ❌ Dejar una coma sobrante después del último elemento (ej. `["a.jar", "b.jar",]` — la coma final
  no es válida en JSON).
- ❌ Olvidar comillas dobles alrededor de los nombres (`[xaeros.jar]` en vez de `["xaeros.jar"]`).
- ❌ Escribir el nombre del mod sin `.jar` al final, o distinto a como se llama el archivo real.
- ❌ Editar este archivo pensando que también hace falta tocar `launcher-manifest.json` — son
  independientes, no hace falta duplicar nada ahí.

## Antes de hacer commit: valida el JSON

Pega el contenido en https://jsonlint.com/ (o cualquier validador de JSON) y comprueba que no
marca ningún error, antes de guardar el cambio en GitHub. Toma menos de un minuto y evita que tu
edición quede "ignorada en silencio" por estar mal formada.
