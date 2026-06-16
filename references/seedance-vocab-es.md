
# seedance-vocab-es

Use Spanish cinematic vocabulary when the user asks for Spanish prompts, bilingual delivery, or compact translation of camera, lighting, action, VFX, audio, and production constraints. Preserve reference tags exactly: `[Image1]`, `[Video1]`, `[Audio1]` must never be translated.

## Intent

Spanish carries rhythm even in technical direction. Serve users who think in Spanish with vocabulary that keeps its musicality while staying camera-precise - they should never feel that directing in their language is a downgrade.

## Usage Rule

Translate production meaning, not word-for-word English. Keep the prompt concrete and concise: subject, visible action, camera, light, sound, and constraint.

| Function | Spanish wording |
|---|---|
| Camera | `travelling de acercamiento`, `plano medio`, `primer plano`, `seguimiento lateral`, `cámara fija` |
| Lighting | `contraluz`, `luz suave de ventana`, `luz práctica cálida`, `sombra marcada`, `halo frío de luna` |
| Motion | `gira lentamente`, `cruza rápido el encuadre`, `avanza con estabilidad`, `las gotas se deslizan` |
| Audio | `sonido ambiente`, `diálogo claro`, `golpe metálico suave`, `sin música` |
| Constraints | `mantener el logotipo, la etiqueta y la forma sin cambios` |

## Compact Pattern

`[Image1] es la referencia; mantener identidad, color y forma sin cambios. Solo cambia [movimiento/luz/cámara]. Cámara: [un movimiento]. Sonido: [señal].`

## De-Slop Rule

When the prompt leans on `cinematográfico`, `épico`, `impresionante`, `mágico`, or `de alta calidad`, load the Slop Traps table in `references/vocab/es.md` and decompose each into the physical elements that produce it - movimiento de cámara, fuente de luz, material, sonido.

## Output Contract

Return Spanish prompt wording, optional English gloss when useful, and unchanged reference tags.
