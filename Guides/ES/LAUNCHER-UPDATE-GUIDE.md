# Cómo funciona la actualización automática del launcher

Esto es distinto a `launcher-manifest.json`/`mod-exceptions.json`/`news/`: aquellos actualizan
*contenido* (mods, noticias) sin tocar el launcher en sí. Esto actualiza el **propio launcher**
(el `.exe`) — y a diferencia de lo que planteamos al principio (un aviso con botón para descargar
a mano), esto es totalmente silencioso: el launcher se descarga la versión nueva, se instala solo
y se reinicia, sin que el jugador tenga que hacer nada ni buscar ningún archivo.

Usa el plugin oficial de Tauri (`tauri-plugin-updater`), que exige firmar criptográficamente cada
build para que el launcher pueda verificar que la actualización viene de verdad de nosotros y no
de alguien suplantándonos — por eso hay unas claves de por medio, explicadas abajo.

## Las claves de firma

Ya están generadas en la raíz del proyecto:

- **`crisis-launcher-update.key`** — la clave **privada**. Con esto se firma cada build. **Nunca
  la subas a ningún sitio público (ni a `dlauncher`, ni a ningún repositorio, ni la compartas por
  Discord).** Ya está en `.gitignore` para protegerla por si algún día se convierte este proyecto
  en un repositorio git. Si la pierdes, no podrás firmar más builds con la misma identidad — habría
  que generar una clave nueva y todo el mundo tendría que reinstalar a mano una última vez.
- **`crisis-launcher-update.key.pub`** — la clave **pública**. Esta sí es segura de compartir; ya
  está copiada dentro de `src-tauri/tauri.conf.json` (`plugins.updater.pubkey`), que es justo lo
  que necesita el launcher para poder verificar que una actualización es legítima.

## Cómo funciona

Al arrancar, el launcher pide un archivo `latest.json` (lo explico abajo) desde la URL configurada
en `plugins.updater.endpoints` de `tauri.conf.json`. Si ahí hay una versión más nueva que la suya,
se descarga el instalador, comprueba su firma contra la clave pública que lleva integrada, y si
todo cuadra, lo instala y reinicia el launcher solo — el jugador ve una barra de progreso mientras
pasa, nada más.

## Compilar firmando (automático)

`3_CREAR_EXE_WINDOWS.bat` ya se encarga de esto: si detecta `crisis-launcher-update.key` en la raíz
del proyecto, configura las variables de entorno necesarias (`TAURI_SIGNING_PRIVATE_KEY` con el
contenido del archivo, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` vacía porque la clave no tiene
contraseña) antes de compilar — no hace falta que hagas nada especial, solo que el archivo
`crisis-launcher-update.key` siga estando en la raíz del proyecto en el ordenador donde compiles.

Al terminar, junto al instalador de siempre aparecen dos archivos `.sig` nuevos:

```
target/release/bundle/nsis/Crisis Launcher_0.1.0_x64-setup.exe
target/release/bundle/nsis/Crisis Launcher_0.1.0_x64-setup.exe.sig
target/release/bundle/msi/Crisis Launcher_0.1.0_x64_en-US.msi
target/release/bundle/msi/Crisis Launcher_0.1.0_x64_en-US.msi.sig
```

El `.sig` es la firma de ESE archivo en concreto — si cambias el instalador (aunque sea recompilar
sin tocar código), hace falta un `.sig` nuevo, no vale reutilizar uno viejo.

## El archivo `latest.json`

Este es el archivo que hay que subir a GitHub (a diferencia de los `.sig`, este no se genera solo
— hay que escribirlo a mano cada vez, igual que editamos `launcher-manifest.json`):

```json
{
  "version": "0.2.0",
  "notes": "Arreglado el menú de cuenta, añadidas noticias.",
  "pub_date": "2026-07-08T02:45:00Z",
  "platforms": {
    "windows-x86_64": {
      "signature": "PEGA_AQUI_EL_CONTENIDO_DEL_ARCHIVO_.sig",
      "url": "https://github.com/delayis/dlauncher/releases/download/v0.2/Crisis.Launcher_0.2.0_x64-setup.exe"
    }
  }
}
```

- **`version`**: la versión nueva, debe coincidir con la de `Cargo.toml`/`tauri.conf.json` de ese
  build.
- **`notes`**: texto libre, es lo único que sí podríamos llegar a mostrar al jugador en el futuro
  (el plugin lo trae disponible, pero ahora mismo no lo mostramos en pantalla).
- **`pub_date`**: fecha en formato ISO 8601, no es crítica pero el plugin la pide.
- **`platforms.windows-x86_64.signature`**: el contenido completo del archivo `.sig` del `setup.exe`
  (ábrelo con el Bloc de notas y copia todo el texto, es una sola línea larga).
- **`platforms.windows-x86_64.url`**: la URL de descarga directa del `setup.exe` — usa el `setup.exe`
  (NSIS), no el `.msi`, mismo motivo que en las otras guías.

Sube `latest.json` a la raíz del repositorio `dlauncher`. Recuerda: `tauri.conf.json` apunta a
`https://raw.githubusercontent.com/delayis/dlauncher/main/latest.json`, así que tiene que estar
ahí exactamente con ese nombre.

## Al sacar una versión nueva

1. Sube la versión en `Cargo.toml` y `tauri.conf.json` (deben decir lo mismo).
2. Compila con `3_CREAR_EXE_WINDOWS.bat` (firma sola si la clave está presente, ver arriba).
3. Crea una Release **nueva** en GitHub (un tag nuevo, ej. `v0.2`) y sube ahí el `setup.exe` (y el
   `.msi` si quieres, aunque no lo use el updater). No sobrescribas una Release vieja.
4. Abre el `.sig` del `setup.exe` con el Bloc de notas, copia su contenido.
5. Edita (o crea) `latest.json` en la raíz del repo con la versión nueva, la URL del asset de la
   Release nueva, y el contenido del `.sig` que copiaste.

Hasta que no hagas el paso 5, nadie ve ninguna actualización — puedes compilar, firmar y subir la
release con tranquilidad antes de "activarla" para todos.

## Cómo probarlo

A diferencia del sistema anterior (donde bastaba con inventar una URL para probar el botón), aquí
la firma tiene que corresponder exactamente al archivo real en esa URL — el launcher rechaza
cualquier actualización cuya firma no cuadre. Para una prueba real de verdad:

1. Instala la versión actual (`0.1.0`) en tu ordenador desde el `setup.exe` ya firmado que
   generamos.
2. Sube la versión a `0.1.1` en `Cargo.toml`/`tauri.conf.json`, compila de nuevo, sube esa Release
   nueva y su `latest.json` correspondiente.
3. Abre el launcher **instalado** (no el que compilaste, el que quedó en Archivos de Programa) —
   debería detectar la `0.1.1`, descargarla, instalarla y reiniciarse solo.

## Qué depende de GitHub (y cómo migrar)

El campo `url` de `latest.json` es un enlace directo, agnóstico de dónde esté alojado, igual que en
mods/noticias — puede apuntar a una Release de GitHub o a cualquier otro sitio sin tocar código. Lo
único atado a GitHub es la URL de `plugins.updater.endpoints` en `tauri.conf.json` — para moverla,
solo hay que cambiar esa URL a donde sea que alojes `latest.json` en el futuro. Las claves de firma
(pública/privada) no dependen de GitHub para nada, son independientes de dónde alojes los archivos.
