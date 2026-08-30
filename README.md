# Semitonos DTF · Removedor de fondo + Semitono para prenda negra

Herramienta web 100 % en el navegador para preparar arte de impresión DTF / serigrafía. Combina dos funciones en un solo flujo:

1. **Removedor de fondo por color** — Click con un gotero sobre la imagen para elegir el color a eliminar; controles de tolerancia y suavizado de bordes.
2. **Semitono AM** opcional — Cuando se habilita el checkbox, convierte la imagen en una trama de puntos rotada lista para serigrafía o DTF blanco sobre prenda negra.

**Sin backend, sin instalación.** Funciona en GitHub Pages o abriendo `index.html` directamente.

Cuando se abre con `?embed=builder`, recibe el diseño seleccionado mediante `postMessage` y sustituye la descarga por **Guardar en Semitonos**, devolviendo el PNG final al Builder sin exponer credenciales.

## Características

- **Preparación previa**: detecta el tamaño en cm y el DPI embebido del archivo subido (lee chunk `pHYs` de PNG y segmentos JFIF/EXIF de JPEG), y permite redimensionar al tamaño físico exacto de impresión.
- **Upscale con IA (Real-ESRGAN)** opcional: corre 100 % en el navegador con ONNX Runtime Web + WebGPU (con fallback a WASM). Útil cuando la imagen llega chica para el tamaño físico de impresión y el bicúbico no alcanza.
- **Gotero estilo Photoshop**: cursor de cruz, click en cualquier parte de la imagen para muestrear el color, Esc para cancelar.
- **Removedor de fondo configurable**: tolerancia + suavizado para evitar halos duros.
- **Trama AM** (puntos redondos) con LPI, ángulo y punto mínimo en mm.
- **PNG neón y semitransparencias**: detecta automáticamente alfa gradual y lo convierte en puntos imprimibles conservando la geometría de trama original.
- **Flujo simple conservado** con Ligero, Balanceado y Fuerte; los niveles negro, blanco y gamma permanecen en el modo avanzado.
- **Visor con zoom y pan** estilo Photoshop: rueda para zoom centrado en el cursor, clic-arrastrar para mover, atajos `0`, `1`, `+`, `−`, `C`, doble clic para ajustar.
- **Tiempo real**: cualquier slider actualiza el resultado al instante.
- **PNG con DPI embebido**: el archivo descargado lleva el chunk `pHYs` correcto.

## Uso

### Abrir localmente

Doble clic en `index.html` y listo.

### Publicar en GitHub Pages

1. Sube el repositorio a GitHub.
2. Settings → Pages → Source: **Deploy from a branch** → Branch: `main` / root → Save.
3. En 1–2 minutos GitHub te entrega una URL pública estilo `https://usuario.github.io/repo/`.

## Flujo de trabajo

1. **Sube una imagen** (PNG, JPG o WEBP).
2. **Define el tamaño físico** en cm al que vas a imprimir (con candado de proporción) y el DPI de trabajo (300 por defecto).
3. **(Opcional) Upscale con IA**: si la imagen original es chica para el tamaño físico (p. ej. 800×800 px para imprimir a 30 cm @ 300 dpi), pulsa **Mejorar con IA** en el card de Preparación. Elige modelo (anime/line art o foto general) y escala (×4 o ×2). Ver sección [Upscale con IA](#upscale-con-ia) abajo para configurar los modelos.
4. **Ajusta tonos** si lo necesitas: brillo, contraste, nitidez, gamma, corte de negro/blanco.
5. **Remover fondo**: pulsa **Seleccionar color de fondo**, luego click en la imagen sobre el color a eliminar. Ajusta tolerancia y suavizado hasta dejar limpio el sujeto.
6. **(Opcional) Habilitar semitono**: marca el checkbox y configura LPI, ángulo, punto mínimo.
7. **Descarga el PNG transparente** con DPI embebido.

## Upscale con IA

El upscale corre en el navegador con [ONNX Runtime Web](https://onnxruntime.ai/docs/tutorials/web/) cargando un modelo Real-ESRGAN. **Todo viene empaquetado en el repo** — el usuario final no descarga modelos ni configura nada: solo sube la imagen y pulsa el botón.

### Backends

- **WebGPU** (preferido) — Chrome 113+, Edge 113+, Firefox 141+. ~1–4 s para 1024² en RTX 3060 / M1.
- **WASM SIMD** (fallback automático) — funciona en cualquier navegador moderno. 10–40 s para la misma imagen.

El worker elige WebGPU si está disponible y cae a WASM en caso contrario, sin intervención.

### Archivos empaquetados

| Carpeta     | Contenido                                                                | Tamaño   |
| ----------- | ------------------------------------------------------------------------ | -------- |
| `ort/`      | `ort.webgpu.min.js`, `ort.min.js`, dos binarios `*.wasm` JSEP            | ~40 MB   |
| `models/`   | `realesr-anime-x4.onnx` — Real-ESRGAN AnimeVideo v3 (×4)                  | ~2.4 MB  |

Total agregado al repo: ~42 MB. Para usuarios finales se descarga una sola vez y queda en caché del navegador.

El modelo es `RealESR-AnimeVideo-v3_x4` (SRVGGNet compact, ~300 K parámetros, BSD-3 — autoría de Xintao Wang et al.). Entrenado para anime / line-art / logos. Ver [docs/anime_video_model.md](https://github.com/xinntao/Real-ESRGAN/blob/master/docs/anime_video_model.md).

> Si necesitas máxima calidad y no te importa esperar 10–20× más, puedes reemplazar el archivo por [`RealESRGAN_x4plus_anime_6B.onnx`](https://huggingface.co/deepghs/imgutils-models/resolve/main/real_esrgan/RealESRGAN_x4plus_anime_6B.onnx) (17.9 MB, RRDBNet 6B) — misma ruta, misma firma de entrada/salida.

### Reemplazar el modelo

Si querés probar otro modelo Real-ESRGAN ONNX (por ejemplo el general para fotos), simplemente reemplazá `models/realesr-anime-x4.onnx` por tu archivo — la ruta es fija y la UI no expone selectores. Los modelos deben ser variantes `×4` con input dinámico `[1, 3, H, W]` en float32 [0, 1].

### Pipeline del upscaler

- El procesamiento se hace por **tiles** de 128×128 px con padding de 8 px para evitar costuras visibles.
- El alpha original se preserva: el modelo solo procesa RGB y luego se mezcla con la transparencia upscaleada por separado.
- Después del upscale, los valores físicos en cm/in **no cambian**: el DPI interno se multiplica por el factor, así que el resize a tamaño de impresión hace ahora *down-sample* desde una versión densa y limpia, que es justo lo que la trama AM necesita.

### Servir la app

WebGPU **no** funciona desde `file://` en la mayoría de navegadores. Sirve la carpeta con cualquier servidor HTTP:

```powershell
# desde la raíz del repo
python -m http.server 8000
# luego abre http://localhost:8000
```

En GitHub Pages funciona out-of-the-box (HTTPS habilita WebGPU sin más).

### Banco de pruebas standalone

`upscale-test.html` es una página separada que usa solo el upscaler (sin pipeline de DTF). Útil para evaluar la calidad de un modelo sobre imágenes específicas antes de pasarlas al flujo completo.

## Pipeline interno

```
imagen original RGBA
  └─ (opcional) AI Upscale × 2/× 4 (Real-ESRGAN, tiles)
        └─ resize alta calidad al tamaño físico × DPI
              └─ brillo + contraste + nitidez sobre el RGB
              ├─ snapshot "workingCanvas" para el gotero
              ├─ si hay color de fondo seleccionado:
              │    distancia RGB al color elegido
              │    → bgAlpha (con tolerancia y feather)
              └─ si "habilitar semitono" está activo:
                   luminancia Rec.709
                   → niveles + gamma
                   → trama AM (puntos por celda rotada, radio ∝ √tono)
                   → halftoneAlpha
                          └─ alpha_final = min(alpha_original, bgAlpha, halftoneAlpha)
                                └─ PNG RGBA con chunk pHYs DPI
```

## Pila

HTML/CSS/JS puro, sin frameworks ni librerías externas. Todo se procesa con la API Canvas 2D, `TypedArray`s y matemáticas vectorizables.

El módulo de upscale carga **bajo demanda** (la primera vez que pulsas el botón) [ONNX Runtime Web](https://onnxruntime.ai/docs/tutorials/web/) desde `cdn.jsdelivr.net`. Si nunca usas la función, no se descarga nada.

## Notas

- El gotero lee el color de la imagen **después** de los ajustes tonales pero **antes** del removedor / semitono. Si cambias brillo o contraste tras seleccionar, el color guardado sigue refiriéndose al valor RGB de ese pixel; puedes re-seleccionar si quieres anclarlo al nuevo color visible.
- Tamaños grandes (32 cm @ 300 DPI ≈ 3780 px) pueden tardar entre 200 y 800 ms por actualización en tiempo real. Para velocidad máxima durante ajustes finos, baja temporalmente el DPI a 150 y súbelo a 300 antes de descargar.
- El backup del backend Python original (versión Flask) se conserva en `semitono_dtf_app.py.bak`.

## Licencia

MIT (o la que prefieras — ajústala antes de subir el repo).
