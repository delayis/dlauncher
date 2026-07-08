# Cómo avisar de una actualización del launcher

Esto es distinto a `launcher-manifest.json`/`mod-exceptions.json`/`news/`: aquellos actualizan
*contenido* (mods, noticias) sin tocar el launcher en sí. Esto avisa a los jugadores cuando el
**propio launcher** (el `.exe`) tiene una versión nueva disponible — pero no lo instala solo, solo
muestra un aviso con un botón para descargarlo. Decidimos esto a propósito en vez de una
actualización silenciosa automática: así no hace falta firmar cada build con una clave criptográfica
ni mantener esa infraestructura, a cambio de que el jugador tiene que darle clic a "Descargar" y
ejecutar el instalador él mismo (igual que hasta ahora).

## Cómo funciona

Al arrancar, el launcher pide `launcher-version.json` desde GitHub (mismo patrón que el manifest:
API de contenidos de GitHub, ~60s de caché) y compara la versión de ahí contra la suya propia
(la que se compiló dentro del `.exe`, no puede desincronizarse por accidente). Si la de GitHub es
más nueva, aparece un aviso arriba del todo del launcher con un botón "Descargar" que abre esa URL
en el navegador. Si no hay actualización, o si falla la comprobación (sin internet, archivo
todavía no creado...), simplemente no aparece nada — no bloquea ni retrasa el arranque.

Código relevante: `check_for_update()` en [update_check.rs](src-tauri/src/update_check.rs).

## Estructura de `launcher-version.json`

Este archivo va suelto en la raíz del repositorio, igual que `launcher-manifest.json`:

```json
{
  "version": "0.2.0",
  "url": "https://github.com/delayis/dlauncher/releases/download/v0.2.0/Crisis%20Launcher_0.2.0_x64-setup.exe",
  "notes": "Arreglado el menú de cuenta, añadidas noticias."
}
```

- **`version`**: la versión nueva, en formato `x.y.z` (debe coincidir con la que pongas en
  `src-tauri/Cargo.toml` y `src-tauri/tauri.conf.json` al compilar esa versión). Se compara
  número por número (`0.10.0` es más nueva que `0.9.0`), no como texto.
- **`url`**: enlace directo de descarga del instalador nuevo — normalmente el asset del `setup.exe`
  o `.msi` que subas a una Release de GitHub.
- **`notes`**: texto libre, no se muestra todavía en el launcher (solo lo guardamos por si en el
  futuro queremos mostrarlo), pero es útil dejarlo igualmente como registro de qué cambió.

## Al sacar una versión nueva

1. Sube la versión en `Cargo.toml` y `tauri.conf.json` (deben decir lo mismo).
2. Compila con `3_CREAR_EXE_WINDOWS.bat`.
3. Sube el instalador (`setup.exe` y/o `.msi`) como asset de una nueva Release en GitHub.
4. Edita `launcher-version.json` con la versión nueva y la URL de ese asset.

Hasta que no hagas el paso 4, nadie ve ningún aviso — puedes compilar y subir la release con
tranquilidad antes de "activar" el aviso para todos.

## Qué depende de GitHub (y cómo migrar)

Igual que con las noticias: el campo `url` de `launcher-version.json` es un enlace directo,
totalmente agnóstico de dónde esté alojado — puede apuntar a una Release de GitHub, a vuestro
propio servidor, o a cualquier sitio, sin tocar código. Lo único atado a GitHub es cómo se pide
el propio `launcher-version.json`, en la constante `LAUNCHER_VERSION_URL` de
[update_check.rs](src-tauri/src/update_check.rs) — para moverlo a otro sitio, cambia esa constante
y quita la cabecera `Accept: application/vnd.github.raw+json` (es específica de la API de GitHub),
exactamente igual que se explica para el manifest y las noticias en sus guías de migración
respectivas.
