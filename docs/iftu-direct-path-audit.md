# Auditoría de la ruta IFTU directa de AdrBubbleBooter

## Resumen ejecutivo

Se encontró una modificación pública de AdrBubbleBooter v1.3 que intercepta la importación kernel `ksceIftuSetInputFrameBuffer` de `SceDisplay`. La modificación activa un modo llamado *direct sharp scale* para PSP a escala exacta 2x.

**No es antialiasing. No es FXAA. No aplica el shader Advanced AA de Adrenaline.** En esa ruta cambia el argumento de muestreo IFTU de `bilinear = 1` a `bilinear = 0` (point/nearest) cuando el framebuffer mide 480x272. La intención es obtener una duplicación de píxeles 2x sin pasar por la cola de dibujo Vita2D.

Fuentes:
- [AdrenalineBubbleManager, README](https://github.com/ONElua/AdrenalineBubbleManager/blob/5d71fd67b567309e8f55ab1a63667b9f76be42e3/README.md)
- [Parche direct-IFTU](https://github.com/ONElua/AdrenalineBubbleManager/blob/5d71fd67b567309e8f55ab1a63667b9f76be42e3/patches/adrbubblebooter-v1.3-direct-iftu.patch)
- [AdrBubbleBooter-oss](https://github.com/shoui520/AdrBubbleBooter-oss)

## Qué hace exactamente el parche

El parche añade un hook kernel sobre la importación de `ksceIftuSetInputFrameBuffer` de `SceDisplay` (NID de librería/importación indicados en el propio parche: `0xCAFCFE50` / `0x7CE0C4DA`).

La función interceptora cambia `bilinear` a cero solo si se cumplen todas estas condiciones:
- El modo de escalado directo está activado.
- La llamada entrante pide `bilinear == 1`.
- El estado y el framebuffer son válidos.
- El framebuffer es exactamente 480x272.

El módulo de usuario solicita ese modo solo para PSP en Vita portátil, no PS1, no framebuffer nativo, filtro *Sharp Bilinear Simple*, suavizado desactivado, f.lux desactivado y escala 2.0x2.0. Si el hook no está disponible, conserva el renderer Vita2D de respaldo. El parche también introduce una syscall de usuario para activar/desactivar el modo.

## Por qué no resuelve el objetivo

La operación se limita a seleccionar el muestreo del IFTU. No hay un paso de lectura de vecinos, detección de bordes, mezcla adaptativa ni shader de postprocesado. El resultado es un escalado de punto/nearest a 2x, no antialiasing. Es justamente una ruta de píxel duro y no debe presentarse como una mejora para reducir dientes de sierra.

El código depende del stack AdrBubbleBooter/Adrenaline, con un par de módulos kernel/usuario ABI-matcheados. No es un plugin independiente listo para instalar sobre una burbuja Chovy-Sign normal. El README del proyecto también exige tener Adrenaline instalado y abierto al menos una vez.

## Lo que sí aporta a la investigación

- Confirma otro punto de intervención en la ruta de presentación PSP: la llamada IFTU desde `SceDisplay`.
- Demuestra que es posible interceptar el parámetro de muestreo y omitir un renderer Vita2D en una configuración muy concreta.
- No demuestra que IFTU admita shaders, ni que el framebuffer PSP pueda ser modificado por un shader desde ese hook.
- No elimina la necesidad de resolver sincronización y propiedad del framebuffer para un postprocesado real.

## Decisión para este proyecto

**No portar ni recomendar este parche como solución.** No cumple el objetivo visual, añade dependencias de Adrenaline/AdrBubbleBooter y requiere sustituir módulos relacionados. No debe instalarse en la consola del usuario.

La investigación de antialiasing debe continuar por una de estas rutas:
1. Encontrar un callback/post-proceso oficial de PSPemu que entregue un frame completo a una etapa de shader; hasta ahora no se ha encontrado uno público.
2. Reproducir de forma independiente y segura el control de presentación que Adrenaline implementa, con un hook validado específicamente para el PSPemu de FW 3.60.
3. Solo después de estabilizar captura, sincronización y presentación, probar un shader FXAA limpio compilado para GXM.

## Estado de seguridad

- No se creó ni instaló ningún módulo.
- No se modificó ninguna ISO, burbuja, licencia o partida.
- No se recomienda instalar AdrenalineBubbleManager, AdrBubbleBooter ni los módulos del parche para este objetivo.
- La ruta directa IFTU queda descartada como solución de antialiasing.
