# PSPemu Bilinear Filter

Investigación y desarrollo de una posible mejora de suavizado para juegos PSP ejecutados como burbujas independientes de LiveArea mediante Chovy-Sign en PS Vita.

## Objetivo

Evaluar si es posible aplicar filtrado bilineal (o un filtro de suavizado equivalente) a la imagen de los juegos PSP ejecutados por el emulador oficial **PSPemu**, sin requerir que el usuario abra Adrenaline y sin modificar ISOs, licencias ni partidas guardadas.

## Entorno objetivo

- PS Vita 1000 (hardware 3G)
- Firmware 3.60 con Ensō
- Juegos PSP instalados como burbujas independientes con Chovy-Sign
- Prioridades: estabilidad, seguridad, bajo impacto en rendimiento y desinstalación sencilla

## Estado

**Investigación técnica avanzada. No existe todavía un plugin, parche ni VPK propio funcional.** La ruta más prometedora identificada es el ajuste Bilinear de Sharpscale; falta validarlo en una burbuja Chovy-Sign real.

### Hallazgo confirmado en Adrenaline

El código de Adrenaline incluye opciones de filtrado gráfico: Original, Bilinear, Sharp bilinear, Advanced AA, LCD3x y Sharp bilinear without scanlines. En `user/menu.c`, su hilo de dibujo:

1. Obtiene el framebuffer PSP desde `SCE_PSPEMU_FRAMEBUFFER`.
2. Copia los datos a una textura PSP propia.
3. Configura filtros GXM LINEAR o POINT según la opción Smooth Graphics.
4. Dibuja la textura con un shader seleccionado en su propio renderer Vita2D.

Esto confirma que el suavizado de Adrenaline forma parte de su ruta de composición/dibujo. **No demuestra que esos shaders o ajustes puedan aplicarse directamente al renderer normal de PSPemu ni a las burbujas Chovy-Sign.**

Fuente: [TheOfficialFloW/Adrenaline](https://github.com/TheOfficialFloW/Adrenaline), especialmente `user/menu.c` y `user/includes/`.

### Hallazgo principal: Sharpscale ya tiene una ruta IFTU relevante

La auditoría del código de Sharpscale encontró que ya intercepta `ksceIftuSetInputFrameBuffer` y ofrece `Point` o `Bilinear`. El filtro se aplica en la ruta genérica de escalado de pantalla, no solamente dentro de Adrenaline. Por ello es el candidato más prometedor para suavizar la salida de PSPemu sin iniciar Adrenaline ni modificar burbujas.

No se ha probado todavía en una burbuja Chovy-Sign real. Su ajuste es global y el paquete disponible es antiguo (v1.50, 2020); el código fuente desactiva el file logging por defecto, pero habría que verificar el binario exacto. Detalles y secuencia de prueba: [`docs/sharpscale-findings.md`](docs/sharpscale-findings.md).

### Restricción importante

No se copiará ni instalará el plugin de Adrenaline como si fuera compatible con PSPemu. Primero hay que demostrar un punto de integración válido en el proceso/emulador que ejecuta las burbujas independientes.

### Hallazgo confirmado en Chovy-Sign

El README de Chovy-Sign describe la herramienta como un conversor/firmador de imágenes PSP y PS1 para que puedan ejecutarse en PS Vita usando contenido oficial/licencias. No documenta un ajuste de filtrado gráfico ni un renderer propio. Por tanto, Chovy-Sign parece encargarse de preparar el contenido de la burbuja, no de ofrecer un punto directo para cambiar cómo PSPemu escala la imagen.

Fuente: [LiEnby/chovy-sign](https://github.com/LiEnby/chovy-sign).

## Plan de investigación

- [x] Identificar el papel de Chovy-Sign: prepara/firma el contenido; no implementa el renderer PSP.
- [ ] Confirmar en consola que la burbuja Chovy-Sign usa la ruta SceDisplay/IFTU observada en Sharpscale.
- [x] Localizar un punto de integración documentado: `ksceIftuSetInputFrameBuffer`.
- [x] Auditar las rutas de filtrado de Adrenaline y Sharpscale, sin copiar código externo.
- [ ] Verificar la compatibilidad exacta de Sharpscale con el firmware 3.60 y el conjunto de plugins instalado.
- [x] Diseñar una prueba reversible que no toca ISO, licencia ni datos de guardado.
- [ ] Medir el resultado visual, rendimiento y estabilidad en la PS Vita real.
- [ ] Crear un plugin propio únicamente si Sharpscale no cubre el caso o sus logs/ajuste global no son aceptables.

## Criterios para aceptar una solución

- No requiere iniciar manualmente Adrenaline.
- No modifica ISO, licencia ni partidas guardadas.
- No añade escritura/logs persistentes innecesarios.
- No provoca regresiones de rendimiento, crashes ni problemas de suspensión.
- Tiene un método claro de desinstalación y recuperación.

## Licencia

Este repositorio de investigación no incorpora código de Adrenaline. Antes de reutilizar cualquier código o shader externo se revisarán su licencia, avisos de copyright y dependencias.