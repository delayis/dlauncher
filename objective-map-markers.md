# Marcadores de objetivos en el mapa de la PDA — Documento de diseño

> Estado: **borrador para revisar con el equipo**. Nada de esto está implementado todavía;
> solo hay 4 templates de iconos (`art_templates/objectives/`).

## 1. Objetivo

En la pestaña **Map** de la PDA, cada quest **activa** mostrará marcadores en el minimapa
(BlazeMap) apuntando a **sus objetivos**. Los marcadores NO son de la quest en conjunto,
sino **uno por objetivo** (una quest puede tener varios objetivos).

Dos tipos de marcador:

- **Exacto** — un punto concreto.
- **Aproximado** — un **círculo de radio real** (estilo "área" de addons de WoW): el
  objetivo está en algún lugar dentro del círculo. El círculo se dibuja a `2 · radio · zoom`
  px, así que crece/encoge con el radio configurado y con el zoom del mapa.

Cada tipo tiene estado **Active** y **Complete** → **4 iconos custom** (con transparencia).

## 2. Decisiones ya tomadas

- Aproximado = **círculo de radio real escalado** (no un icono difuso de tamaño fijo).
- Marcadores **por objetivo**, no por quest.
- Solo se muestran los objetivos **desbloqueados** (no spoilear los que aún no tocan).
- Estado **Active / Complete** según el progreso de ESE objetivo (`progress >= required`).
- **4 iconos custom** (exacto/aprox × active/complete), pintados a 2x → 1x. Templates ya
  generados en `art_templates/objectives/` (48×48 exacto, 128×128 aprox).

## 3. El punto clave a decidir: ¿cómo se determina la posición de cada objetivo?

Este es el motivo principal de la consulta. **Los objetivos no tienen una posición de mundo
uniforme.** Cada tipo "sabe" (o no) dónde está de una forma distinta:

| Tipo objetivo | ¿Tiene posición natural? | De dónde saldría | Notas / dificultad |
|---|---|---|---|
| **LOCATION** | ✅ Sí | Centro de la `LocationZone` (x1..x2, z1..z2) | Ya está en los datos. Puede haber varias zonas (hasta 3) y modo aleatorio. |
| **COLLECT**  | ✅ Sí | `collectX/Y/Z` (el cofre configurado) | Ya está en los datos. Un único punto. |
| **TRADE**    | ⚠️ Derivable | Posición del NPC trader referenciado | El objetivo guarda `tradeNpcNames` + slot. Habría que resolver la **posición del NPC** (¿entidad cargada? ¿posición guardada del NPC?). |
| **DIALOG**   | ⚠️ Derivable | Posición del NPC del diálogo | Similar a TRADE: hay que resolver qué NPC y dónde está. |
| **INTERACT** | ⚠️ Parcial | El/los bloque(s) a interactuar | El objetivo guarda IDs de bloque, **no coordenadas**. Un bloque no "se autolocaliza" salvo que se fije un punto. |
| **KILL**     | ❌ No natural | ¿Zona de spawn del mob? ¿instancia más cercana? | Un mob puede estar en cualquier sitio. Derivar posición es difícil/dinámico. |
| **ITEM**     | ❌ No natural | (los ítems se consiguen en cualquier parte) | Normalmente no tiene sentido un punto fijo. |

### Enfoques posibles

**A) Posición manual (la fija el diseñador).**
El diseñador teclea X/Y/Z (o pulsa "usar mi posición") en el editor, para cualquier tipo.
- ✔️ Simple, uniforme, funciona para los 7 tipos.
- ✔️ Sin lógica de "detección" por tipo.
- ✖️ Trabajo manual; si el mundo cambia, la coord queda desactualizada.

**B) Posición derivada por tipo (automática donde se pueda).**
- LOCATION → centro de zona; COLLECT → sus coords; TRADE/DIALOG → posición del NPC;
  INTERACT/KILL/ITEM → manual (o sin marcador).
- ✔️ Menos trabajo para el diseñador en los tipos con posición.
- ✖️ Requiere **lógica de resolución por tipo** (sobre todo NPCs: saber su UUID y su
  posición actual — esto es lo que el equipo tiene que valorar).
- ✖️ Para NPCs, ¿la posición es la del NPC **vivo** (dinámica) o una guardada?

**C) Híbrido (recomendado como punto de partida).**
- El marcador tiene una posición **manual** por defecto (enfoque A), y **opcionalmente**
  un botón "derivar del objetivo" que la autocompleta cuando el tipo lo permite (LOCATION,
  COLLECT, y NPC si resolvemos su posición). Así arrancamos simple y añadimos derivación
  por tipo de forma incremental sin bloquear el resto.

> **Pregunta para el equipo:** ¿queremos posiciones **estáticas** (las fija el diseñador /
> se derivan una vez) o **dinámicas** (p. ej. el marcador de un TRADE/DIALOG sigue al NPC en
> tiempo real si se mueve)? Lo dinámico implica resolver la entidad NPC en cliente cada frame.

## 4. Estado Active / Complete (detección de completado)

Esto **ya lo resuelve el sistema de quests actual** — no hay que inventar detección nueva:

- El cliente tiene el progreso por objetivo en `PlayerData` (quests activas sincronizadas por
  `SyncPlayerQuestProgressPacket`).
- Por objetivo `i`: `QuestProgress.getProgress(i)` vs `QuestObjective.requiredProgress()`, y
  `QuestProgress.isUnlocked(i)`.
- El marcador usa el icono **Complete** cuando `progress >= required`, **Active** si no.
  Solo se dibuja si el objetivo está **desbloqueado**.

(Cada tipo de objetivo ya se "detecta" a su manera para avanzar el progreso: KILL cuenta
muertes, TRADE cuenta intercambios, LOCATION detecta entrar en zona, etc. El marcador solo
**lee** ese progreso ya existente; no cambia cómo se completan.)

## 5. Modelo de datos propuesto

Nuevo objeto de configuración, uno (o varios — ver preguntas) por `QuestObjective`:

```java
class ObjectiveMarker {
    boolean enabled;      // mostrar en el mapa
    int x, y, z;          // posición de mundo (Y solo por si se usa en futuro / beam)
    boolean approximate;  // false = exacto, true = círculo de radio
    int radius;           // bloques (solo si approximate)
    // (posible) String source;  // "manual" | "derived" para el enfoque híbrido
}
```

- Se añade a `QuestObjective` con su get/set.
- Serialización en **NBT** (`toNBT`/`fromNBT`) y **JSON** (`toJson`/`fromJson`), junto al
  resto del objetivo. Retrocompatible: si no existe la clave, `enabled = false`.
- **Sync a cliente:** las quests ya viajan enteras con `SyncQuestsPacket`, así que el
  marcador llega gratis (el cliente resuelve la quest con `GlobalQuestManager.getQuest`).

> **Pregunta para el equipo:** ¿**un** marcador por objetivo, o **varios**? Ej.: un KILL con
> 4 mobs o un LOCATION con 3 zonas → ¿un punto o uno por sub-slot? El modelo simple (uno por
> objetivo) es el punto de partida; varios implica una lista y más UI.

## 6. Editor (QuestEditorScreen)

La config del marcador es **transversal** (vale para cualquier tipo), así que va en la zona
**común** del objetivo, junto a *description* y *order step* (no en cada pantalla por tipo):

```
[ ] Mostrar en el mapa
X [____]  Y [____]  Z [____]   [ Usar mi posición ]   ( [ Derivar del objetivo ] * )
( ) Exacto   ( ) Aproximado    Radio [__] bloques
```

`* Derivar del objetivo` solo aparece/activo si el enfoque elegido es B o C y el tipo lo
soporta.

## 7. Render en `MapTab`

- Recorrer las quests **activas** del jugador; por cada objetivo **desbloqueado** con
  `marker.enabled`:
  - Proyectar mundo→pantalla con el estado del minimapa (la interfaz `PdaMinimap` ya expone
    `getCenterX/getCenterZ/getZoom`):
    `sx = mapX + mapW/2 + (marker.x - centerX) · zoom`, análogo para `sy` con `z`.
  - **Exacto:** blit del icono (tamaño fijo en pantalla) centrado en `(sx, sy)`.
  - **Aproximado:** blit del círculo escalado a `diámetro = 2 · radius · zoom` px, centrado.
  - Elegir textura `active/complete` según el progreso del objetivo.
  - **Recorte** a la caja del mapa (que un marcador fuera de la caja no se pinte fuera).
- Se dibuja como overlay encima del mapa (igual que ya se hace la máscara de bounds).
- Posible extra: al pasar el ratón por un marcador, tooltip con la descripción del objetivo.

## 8. Iconos / templates

Ya generados en `art_templates/objectives/` (con transparencia). Ver su `README.md`.

| Tipo | Template 2x | Ship 1x (`textures/gui/pda/`) |
|---|---|---|
| Exacto · Active   | `TEMPLATE_pda_obj_exact_active.png` (48×48)   | `pda_obj_exact_active.png` (24×24) |
| Exacto · Complete | `TEMPLATE_pda_obj_exact_complete.png` (48×48) | `pda_obj_exact_complete.png` (24×24) |
| Aprox · Active    | `TEMPLATE_pda_obj_approx_active.png` (128×128) | `pda_obj_approx_active.png` (64×64) |
| Aprox · Complete  | `TEMPLATE_pda_obj_approx_complete.png` (128×128) | `pda_obj_approx_complete.png` (64×64) |

## 9. Preguntas abiertas para el equipo

1. **Posición:** ¿enfoque A (manual), B (derivada por tipo) o C (híbrido)? → determina cuánta
   lógica de "detección de posición" por tipo hay que escribir.
2. **NPCs (TRADE/DIALOG):** ¿podemos resolver la posición de un NPC en cliente? ¿por UUID?
   ¿estática (guardada) o dinámica (entidad viva)?
3. **KILL / ITEM:** ¿tienen marcador (manual) o directamente no aplican?
4. **¿Uno o varios** marcadores por objetivo (sub-slots)?
5. **Estático vs dinámico:** ¿el marcador puede seguir a una entidad en tiempo real?
6. **¿Y (altura)** se usa para algo (beam/tooltip) o basta X/Z para el mapa 2D?

## 10. Plan de implementación por fases (una vez decidido lo anterior)

1. `ObjectiveMarker` + serialización NBT/JSON en `QuestObjective` (retrocompatible).
2. UI en `QuestEditorScreen` (zona común del objetivo) — enfoque A primero.
3. Render en `MapTab` (proyección + iconos + círculo + estados + recorte).
4. (Si B/C) Derivación de posición por tipo, empezando por los fáciles (LOCATION, COLLECT).
5. (Opcional) NPCs dinámicos, tooltips, múltiples marcadores.

Cada fase compila y es probable en juego de forma aislada.












Respecto a la doc, voy por partes.
1. En cuanto al estado de Complete, el icono no seguirá dibujando un radio si está completo (creo?). Solo si está Activo es mi idea, aunque no sé como se mostraría sin ser asi radio.
2. Respecto a "Solo se muestran los objetivos desbloqueados (no spoilear los que aún no tocan)." he de decir que los Objectives de una Quest, pueden estar en Follow Order o no. Es decir, hay quests que los objetivos van en orden, y hay otras quests que no. En el primer caso, se mostraría los objetivos de 1 en 1. En el segundo caso, todos los objetivos aparecerían en el mapa.
Es cierto que el Quest Editor tiene un grafico de nodos, donde podemos hacer que una Main Quest tenga cadena de misiones, y haya misiones opcionales. Tambien existen objetivos opcionales, así que necesitamos otro tipo de icono para los opcionales. Por cierto ,para los 3 tipos, necesitamos 1 estado mas, el hover.
3. Para este punto, creo que: 
Location, Collect están bien.
Trade y Dialog, podrían basarse en la HomePos del NPC.
Interact, es lo mas dudoso, que supongo que habrá que permitir el modo manual.
Kill, como este objetivo admite tanto nombres de NPC's, entidades (ejemplo: minecraft:zombie, modexample:ultra_boss), y Players, habrá que gestionar la idea.
Item, al igual que Interact, supongo que habrá que permitir el modo manual.

El marcador de un Trade/Dialog por ejemplo no siguen a un NPC en tiempo real, nos vamos a basar en su HomePos. Será estatico.

5. Respecto a esto, como no sé de tema de modelo de datos solo te puedo decir respecto a lo que pones "Pregunta para el equipo".
Un objetivo Item -> 1 solo marcador
Un objetivo Dialog -> 1 marcador por cada slot (3 en total)
Trade -> 1 marcador por slot (4 en total)
Un objetivo Kill -> 1 marcador por cada slot (3 en total)
Location -> 1 marcador por cada location (3 en total)
Interact -> Creo que tambien 1 marcador por cada location (4 en total)
Collect -> 1 solo marcador




