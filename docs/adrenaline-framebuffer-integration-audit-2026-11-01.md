# Auditoría de integración de framebuffer: Adrenaline frente a Chovy-Sign

Fecha: 2026-11-01

## Resultado de la inspección de código

Se inspeccionó directamente `user/menu.c` de TheOfficialFloW/Adrenaline, commit
`7d382b7837d9d211d830017ba7aee982fa49a8c6`.

Fuente:
- https://github.com/TheOfficialFloW/Adrenaline/blob/7d382b7837d9d211d830017ba7aee982fa49a8c6/user/menu.c

Hallazgos verificables:

1. Adrenaline crea una textura PSP de `PSP_SCREEN_LINE x PSP_SCREEN_HEIGHT` y obtiene su puntero de datos.
2. Crea shaders GXM para Original, Sharp Bilinear, Advanced AA y LCD3x.
3. En su bucle de dibujo copia `SCE_PSPEMU_FRAMEBUFFER` a la textura mediante `sceDmacMemcpy`, usando `SCE_PSPEMU_FRAMEBUFFER_SIZE`.
4. Selecciona Advanced AA como un programa de vértices/fragmentos GXM y dibuja la textura PSP a escala 2x.
5. Para sincronizar el resultado PSP utiliza `sceCompatLCDCSync()`; no trata la memoria del framebuffer como una imagen que se pueda leer arbitrariamente en cualquier momento.
6. Su ruta de presentación también controla cuándo dibujar, espera VBlank en ciertas ramas y evita dibujar mientras hay diálogos del sistema.
7. El código implementa un hilo de dibujo dedicado y un controlador de ajustes propio. No es únicamente un shader que pueda añadirse a `*ALL`.

## Consecuencia para una burbuja Chovy-Sign normal

La existencia de `ScePspemu` y de la dirección de framebuffer usada por Adrenaline no basta para crear un postprocesador seguro. La burbuja normal no recibe automáticamente el hilo de dibujo, el controlador de ajustes ni la coordinación de pantalla de Adrenaline.

Un plugin aislado tendría que demostrar, en firmware 3.60:

- que el proceso de la burbuja permite acceder legalmente al framebuffer y a los módulos requeridos;
- que el formato, stride, tamaño y dirección son los esperados en esa ejecución;
- que el fotograma ya terminó de escribirse antes de copiarlo;
- que el acceso DMA y las cachés están sincronizados;
- que se puede presentar el resultado sin competir con la salida original de PSPemu;
- que al volver a LiveArea, suspender o cerrar el juego se devuelve el control de pantalla correctamente.

No se deben copiar offsets internos de Adrenaline a ciegas ni asumir que sus direcciones y estructuras son estables en todos los firmwares o contextos.

## Decisión técnica

La vía más prometedora para el objetivo visual sigue siendo un postproceso GXM de la imagen PSP, porque el shader Advanced AA de Adrenaline demuestra que el filtrado puede mejorar la imagen ya renderizada. Sin embargo, replicar la ruta de presentación completa de Adrenaline dentro de una burbuja Chovy-Sign sería una integración de alto riesgo, no un plugin liviano de usuario.

No se recomienda intentar MSAA GXM sobre la geometría PSP: la API GXM de Vita no demuestra que el Graphics Engine de PSPemu use esos render targets. Tampoco se recomienda el parche IFTU de escalado directo, porque cambia el muestreo a point/nearest y no realiza antialiasing.

## Próximo hito seguro

Antes de escribir el plugin:

1. Auditar el código del cargador de Adrenaline y sus hooks sobre `ScePspemu` para identificar qué sincronización y llamadas internas son imprescindibles.
2. Confirmar qué parte de esa integración depende de direcciones internas específicas de firmware.
3. Comparar el proceso y módulos cargados por una burbuja Chovy-Sign con el proceso `PSPEMUCFW`.
4. Solo si existe una ruta de presentación que no requiera tomar el control global de pantalla, crear una prueba de captura de un solo título, inicialmente sin modificar el framebuffer ni instalar un hook kernel global.

## Estado

- Inspección del renderer y del bucle de dibujo de Adrenaline: realizada.
- Integración de framebuffer en una burbuja Chovy-Sign: no demostrada.
- Plugin `.suprx`, VPK o binario de prueba: no existe todavía.
- Instalación o cambios en la Vita del usuario: ninguno.

No publicar ni instalar un binario hasta completar el hito de sincronización y recuperación de la pantalla.
