# PSPemu Bilinear Filter

Investigación y desarrollo de una posible mejora de suavizado para juegos PSP ejecutados como burbujas independientes de LiveArea mediante Chovy-Sign en PS Vita.

## Objetivo

Investigar una técnica de **antialiasing** que reduzca los dientes de sierra en los bordes de los juegos PSP ejecutados como burbujas Chovy-Sign en el emulador oficial **PSPemu**, sin requerir Adrenaline y sin modificar ISOs, licencias ni partidas guardadas. El objetivo no es desenfocar toda la imagen.

## Entorno objetivo

- PS Vita 1000 (hardware 3G)
- Firmware 3.60 con Ensō
- Juegos PSP instalados como burbujas independientes con Chovy-Sign
- Prioridades: estabilidad, seguridad, bajo impacto en rendimiento y desinstalación sencilla

## Estado

**Investigación técnica. No existe todavía un plugin, parche ni VPK propio funcional.** El usuario confirmó que el ajuste Bilinear de Sony aparece en sus burbujas Chovy-Sign, pero probó el resultado y lo consideró horrible. Por tanto, no se debe forzar bilinear: el objetivo cambió a reducir dientes de sierra (antialiasing), no a suavizar globalmente la imagen.

### Hallazgo confirmado en Adrenaline

El código de Adrenaline incluye opciones de filtrado gráfico: Original, Bilinear, Sharp bilinear, Advanced AA, LCD3x y Sharp bilinear without scanlines. En `user/menu.c`, su hilo de dibujo:

1. Obtiene el framebuffer PSP desde `SCE_PSPEMU_FRAMEBUFFER`.
2. Copia los datos a una textura PSP propia.
3. Configura filtros GXM LINEAR o POINT según la opción Smooth Graphics.
4. Dibuja la textura con un shader seleccionado en su propio renderer Vita2D.

Esto confirma que el suavizado de Adrenaline forma parte de su ruta de composición/dibujo. **No demuestra que esos shaders o ajustes puedan aplicarse directamente al renderer normal de PSPemu ni a las burbujas Chovy-Sign.**

Fuente: [TheOfficialFloW/Adrenaline](https://github.com/TheOfficialFloW/Adrenaline), especialmente `user/menu.c` y `user/includes/`.


### Estrategia de prueba aislada

Se confirmó que el cargador de Adrenaline solo inserta su renderer en el título `PSPEMUCFW`; no se activa automáticamente en burbujas Chovy-Sign. La vía menos invasiva para una prueba futura sería cargar un módulo de usuario solo bajo el TITLEID de una burbuja de prueba, nunca con `*ALL` ni con un plugin kernel global.

Detalles: [`docs/chovy-titleid-loading-strategy.md`](docs/chovy-titleid-loading-strategy.md).

### Fuente del shader Advanced AA localizada

Se encontró el código fuente coincidente `advanced_aa_v.cg` / `advanced_aa_f.cg` en la colección de shaders de frangarcj. Es un filtro espacial adaptativo de nueve muestras, no FXAA. El algoritmo ya está identificado, pero aún falta una ruta segura para capturar y presentar el framebuffer PSP de una burbuja Chovy-Sign.

Detalles y licencia: [`docs/advanced-aa-shader-source.md`](docs/advanced-aa-shader-source.md).

### Auditoría adicional: ruta IFTU directa de AdrBubbleBooter

Se revisó un parche público que intercepta `ksceIftuSetInputFrameBuffer` para activar un modo de escalado directo 2x. El parche cambia el muestreo de bilinear a point/nearest bajo condiciones muy concretas; **no ejecuta FXAA ni ningún antialiasing**. Además, depende de módulos de AdrBubbleBooter/Adrenaline y no es un plugin independiente para burbujas Chovy-Sign. Se descarta como solución para este proyecto.

Detalles: [`docs/iftu-direct-path-audit.md`](docs/iftu-direct-path-audit.md).

### Prueba del filtro nativo: resultado rechazado

El usuario confirmó que **Bilinear Filtering** sí aparece en el menú rápido de sus burbujas Chovy-Sign, pero el resultado visual le pareció horrible. Mantenerlo desactivado. El bilinear de salida mezcla muestras de toda la imagen y no equivale a antialiasing geométrico.

Detalles: [`docs/native-quick-menu.md`](docs/native-quick-menu.md).

### Límite del IFTU

La documentación de VitaSDK confirma los modos IFTU `NEAREST = 0` y `BILINEAR = 1`. Sin embargo, cambiar ese argumento solo cambia el muestreo de salida; no activa MSAA ni FXAA. Como el usuario rechazó el resultado bilinear, un hook que simplemente fuerce ese modo no es una solución válida.

Detalles: [`docs/vitasdk-api-verification.md`](docs/vitasdk-api-verification.md) y [`docs/plugin-design.md`](docs/plugin-design.md).

### Hallazgo de arquitectura: MSAA de GXM no es un interruptor para PSPemu

La auditoría del código de Sharpscale encontró que ya intercepta `ksceIftuSetInputFrameBuffer` y usa `ksceSblACMgrIsPspEmu(SCE_KERNEL_PROCESS_ID_SELF)` desde ese hook. Sin embargo, su opción Bilinear solo conserva el valor entrante; no fuerza un 0 a 1. Por tanto, Sharpscale no garantiza el efecto buscado si PSPemu envía nearest. Un plugin propio podría forzar bilinear solo para llamadas PSPemu validadas, pero aún no está implementado.

No se ha probado todavía en una burbuja Chovy-Sign real. Su ajuste es global y el paquete disponible es antiguo (v1.50, 2020); el código fuente desactiva el file logging por defecto, pero habría que verificar el binario exacto. Detalles y secuencia de prueba: [`docs/sharpscale-findings.md`](docs/sharpscale-findings.md).

### Restricción importante

No se copiará ni instalará el plugin de Adrenaline como si fuera compatible con PSPemu. Primero hay que demostrar un punto de integración válido en el proceso/emulador que ejecuta las burbujas independientes.

### Hallazgo confirmado en Chovy-Sign

El README de Chovy-Sign describe la herramienta como un conversor/firmador de imágenes PSP y PS1 para que puedan ejecutarse en PS Vita usando contenido oficial/licencias. No documenta un ajuste de filtrado gráfico ni un renderer propio. Por tanto, Chovy-Sign parece encargarse de preparar el contenido de la burbuja, no de ofrecer un punto directo para cambiar cómo PSPemu escala la imagen.

Fuente: [LiEnby/chovy-sign](https://github.com/LiEnby/chovy-sign).

## Plan de investigación

- [x] Identificar el papel de Chovy-Sign: prepara/firma el contenido; no implementa el renderer PSP.
- [ ] Probar el ajuste Bilinear oficial del menú rápido en una burbuja Chovy-Sign.
- [x] Localizar un punto de integración documentado: `ksceIftuSetInputFrameBuffer`.
- [x] Auditar las rutas de filtrado de Adrenaline y Sharpscale, sin copiar código externo.
- [ ] Verificar la compatibilidad exacta de Sharpscale con el firmware 3.60 y el conjunto de plugins instalado.
- [x] Diseñar una prueba reversible que no toca ISO, licencia ni datos de guardado.
- [x] Confirmar que Sony documenta un ajuste Bilinear nativo para juegos PSP.
- [x] Verificar contra VitaSDK las firmas IFTU y los NID de detección PSPemu para FW 3.60.
- [ ] Medir el resultado visual, rendimiento y estabilidad en la PS Vita real.
- [ ] Crear un plugin propio únicamente si la opción nativa no está disponible o no cubre el caso.

## Criterios para aceptar una solución

- No requiere iniciar manualmente Adrenaline.
- No modifica ISO, licencia ni partidas guardadas.
- No añade escritura/logs persistentes innecesarios.
- No provoca regresiones de rendimiento, crashes ni problemas de suspensión.
- Tiene un método claro de desinstalación y recuperación.

## Licencia

Este repositorio de investigación no incorpora código de Adrenaline. Antes de reutilizar cualquier código o shader externo se revisarán su licencia, avisos de copyright y dependencias.
