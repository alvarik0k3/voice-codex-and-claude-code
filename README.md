# Voice → Claude / Codex con Handy

Sistema de asistente de voz que usa **Handy** para transcribir el audio (local, rápido, sin internet) y luego envía el texto a **Claude** o **Codex** para obtener una respuesta en voz o texto.

---

## Cómo funciona

```
Super+Z (presionas)
  → handy-mode escribe "claude-speak" en /tmp/handy-ai-mode
  → dispara: handy --toggle-transcription   (Handy empieza a grabar)

(hablas)

Super+Z (presionas de nuevo)
  → handy --toggle-transcription   (Handy para la grabación)
  → parakeet transcribe en ~200ms
  → Handy llama a handy-ai-bridge con el texto
  → bridge lee el modo → manda a Claude CLI
  → Claude responde → piper TTS lo lee en voz alta
```

El `Ctrl+Space` sigue funcionando como dictado puro (sin IA).

---

## Atajos de teclado

| Atajo | Acción |
|-------|--------|
| `Ctrl+Space` | Dictado normal — transcribe y escribe donde esté el cursor |
| `Super+Z` | Pregunta a **Claude** → responde en **voz** |
| `Super+Shift+Z` | Pregunta a **Claude** → **escribe** la respuesta en la ventana activa |
| `Super+Q` | Pregunta a **Codex** → responde en **voz** |
| `Super+Shift+Q` | Pregunta a **Codex** → **escribe** la respuesta en la ventana activa |

---

## Archivos del sistema (no tocar)

Estos son los archivos que realmente están en uso. Los de esta carpeta son solo copias de referencia.

| Archivo | Ubicación real | Descripción |
|---------|---------------|-------------|
| `handy-ai-bridge` | `~/.local/bin/handy-ai-bridge` | Script Python principal — recibe la transcripción de Handy y la enruta |
| `handy-mode` | `~/.local/bin/handy-mode` | Script bash — escribe el modo y dispara Handy |
| Bindings Hyprland | `~/.config/hypr/bindings.lua` | Atajos de teclado |
| Config Handy | `~/.local/share/com.pais.handy/settings_store.json` | `paste_method: external_script` + ruta al bridge |

---

## Instalación desde cero

### 1. Requisitos

```bash
# Handy (AUR)
yay -S handy

# Claude Code CLI
# Seguir instrucciones en https://claude.ai/code

# Codex CLI
# Seguir instrucciones de OpenAI Codex CLI

# piper TTS
# Descargar binario desde https://github.com/rhasspy/piper/releases
# Colocar en ~/.local/bin/piper
# Modelo de voz: es_ES-davefx-medium.onnx → ~/.local/share/piper/voices/

# wtype (para escribir en ventanas Wayland)
sudo pacman -S wtype
```

### 2. Copiar los scripts

```bash
cp handy-ai-bridge ~/.local/bin/handy-ai-bridge
cp handy-mode ~/.local/bin/handy-mode
chmod +x ~/.local/bin/handy-ai-bridge ~/.local/bin/handy-mode
```

### 3. Configurar Handy

Detener Handy si está corriendo:
```bash
pkill handy
```

Editar `~/.local/share/com.pais.handy/settings_store.json` y cambiar:
```json
"paste_method": "external_script",
"external_script_path": "/home/TU_USUARIO/.local/bin/handy-ai-bridge",
```

Reiniciar Handy:
```bash
handy --start-hidden &
```

### 4. Configurar Hyprland

En `~/.config/hypr/bindings.lua` agregar o reemplazar las líneas de los asistentes de voz:

```lua
-- Dictado puro con Ctrl+Space
o.bind("CTRL + SPACE", "Handy: transcribir voz", "handy --toggle-transcription")

-- Asistentes de voz con Handy
o.bind("SUPER + Z",       "Asistente Claude (voz)",     "handy-mode claude-speak")
o.bind("SUPER + SHIFT + Z", "Asistente Claude (escribe)", "handy-mode claude-type")
o.bind("SUPER + Q",       "Asistente Codex (voz)",      "handy-mode codex-speak")
o.bind("SUPER + SHIFT + Q", "Asistente Codex (escribe)", "handy-mode codex-type")
```

Recargar Hyprland:
```bash
hyprctl reload
```

---

## Por qué Handy en vez de faster-whisper

| | faster-whisper (anterior) | Handy / parakeet (actual) |
|--|--|--|
| Velocidad | ~3-5s (recarga modelo cada vez) | ~200ms (modelo en RAM) |
| Español | Sí (multilingual) | Sí (multilingual) |
| Integración con scripts | Sí (stdout) | Vía `external_script_path` |
| Modelo | Whisper small 462MB | parakeet-tdt-0.6b-v3 ONNX 640MB |

---

## Cómo funciona handy-ai-bridge internamente

1. Handy llama al script con el texto transcrito (via stdin o argv)
2. El script comprueba si existe `/tmp/handy-ai-mode`
   - **No existe** → dictado normal, usa `wtype` para escribir el texto
   - **Existe** → lee el modo (`claude-speak`, `claude-type`, `codex-speak`, `codex-type`) y lo borra
3. Envía el texto a Claude o Codex CLI
4. Según el modo:
   - `speak` → sintetiza la respuesta con piper TTS y la reproduce con aplay
   - `type` → escribe la respuesta en la ventana activa con wtype

---

## Cambiar el modelo de Handy

Abrir la app de Handy, ir a Settings → Model, descargar el modelo deseado y marcarlo como predeterminado. No hay que tocar ningún script — el bridge recibe el texto ya transcrito independientemente del modelo.

---

## Solución de problemas

**El atajo no hace nada:**
```bash
pgrep handy   # verificar que Handy esté corriendo
handy --start-hidden &   # si no está corriendo, iniciarlo
```

**Error en la transcripción:**
```bash
tail -20 ~/.local/share/com.pais.handy/logs/handy.log
```

**El bridge no llama a Claude/Codex:**
```bash
# Probar el bridge manualmente
echo "hola qué es Python" | ~/.local/bin/handy-ai-bridge
# Verifica que aparezca la notificación y respuesta
```

**Modo incorrecto se quedó en /tmp:**
```bash
rm -f /tmp/handy-ai-mode
```
