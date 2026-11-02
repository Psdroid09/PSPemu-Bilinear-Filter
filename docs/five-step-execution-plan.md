# Ejecución de los cinco pasos — PSPemu / Chovy-Sign

Fecha: 2026-11-02

## Alcance que no cambia

- PS Vita 1000 (3G), firmware 3.60 + Ensō.
- Solo juegos PSP ejecutados como burbujas LiveArea de Chovy-Sign.
- Primera burbuja de prueba: Ben 10: Protector of Earth USA, ID PSP `ULUS10307`.
- Sin Adrenaline, sin modificar ISO/EBOOT/licencia/guardados y sin tocar juegos Vita nativos.
- Sin registros persistentes, procesos en segundo plano ni cambios globales bajo `*ALL`.
- El bilinear nativo ya fue probado y rechazado por el usuario: no debe activarse como solución.

## Resultado de la ejecución

### 1. Identificar la burbuja y el proceso PSPemu

**Parcialmente resuelto.** Se conoce el ID del juego PSP (`ULUS10307`) y la ruta de contenido indicada por el usuario (`ux0:pspemu/PSP/GAME/ULUS10307`). Eso NO es necesariamente el TITLEID Vita de la burbuja. El TITLEID Vita exacto debe leerse de la burbuja instalada (por ejemplo, de su metadato SFO / estructura app) o confirmarse en el dispositivo. No inventar ni usar `ULUS10307` como TITLEID taiHEN.

taiHEN tiene una etapa de carga de plugins por TITLEID después de iniciar los módulos predeterminados. Esto hace plausible una prueba limitada a la burbuja, pero todavía no demuestra en hardware que el módulo vea `ScePspemu` en esta burbuja.

### 2. Preparar una prueba de carga aislada

**Diseño definido; prueba en Vita pendiente.** El primer módulo debe ser un plugin de usuario mínimo, asociado únicamente al TITLEID Vita confirmado de Ben 10. Su única tarea será consultar la presencia de `ScePspemu` y salir sin modificar el estado gráfico si no lo encuentra.

No se debe instalar un módulo que no haya sido compilado y revisado para VitaSDK/FW 3.60. No se añade nada a `*ALL` ni a `*KERNEL` para esta prueba. La carga real y el TITLEID aún requieren verificación en el dispositivo.

### 3. Encontrar una ruta de framebuffer / presentación segura

**Bloqueo técnico identificado; no resuelto.** Adrenaline demuestra una ruta propia de captura, DMA, textura GXM y sincronización de PSPemu, pero esa implementación no demuestra que se pueda sustituir de forma segura el renderer de Sony en una burbuja Chovy-Sign normal.

La llamada IFTU `ksceIftuSetInputFrameBuffer` solo documenta los modos NEAREST y BILINEAR. No expone un shader programable. Forzar bilinear repetiría el resultado visual que el usuario rechazó. No se debe interceptar ni reemplazar el framebuffer hasta demostrar dimensiones, formato, stride, plano, sincronización y restauración.

### 4. Aplicar antialiasing solo a Ben 10

**No implementado.** El candidato visual sigue siendo AdvancedAA espacial adaptativo de nueve muestras, pero el shader por sí solo no es un plugin. Falta una ruta válida para capturar y presentar el frame de PSPemu sin competir con el renderer original.

No compilar un supuesto filtro funcional ni entregar un VPK hasta resolver el paso 3. No reutilizar binarios de Adrenaline.

### 5. Validar rendimiento, estabilidad y aislamiento

**Pruebas físicas pendientes.** Cuando exista una ruta de presentación segura, comparar la misma escena y cámara en la burbuja Ben 10 con filtro apagado/encendido; revisar bordes, texto/HUD, movimiento, FPS/frame pacing, suspensión, regreso a LiveArea y relanzamiento. Verificar también que un juego Vita nativo permanezca idéntico.

Criterio de rechazo inmediato: pantalla negra, tearing nuevo, cuelgue, crash, fallo al suspender/reanudar, caída perceptible de rendimiento o cualquier cambio en juegos nativos.

## Por qué no hay todavía un VPK instalable

La investigación ha identificado la carga por TITLEID como vía plausible y el shader candidato, pero no ha demostrado el punto seguro de captura/presentación en PSPemu normal. Un plugin que solo cambia IFTU a bilinear no satisface el objetivo. Publicar un VPK ahora sería presentar una hipótesis como si fuera una implementación y podría poner en riesgo la sesión de la Vita.

## Dato necesario para continuar en el dispositivo

Obtener el TITLEID Vita real de la burbuja Ben 10 instalada. No confundirlo con el ID PSP `ULUS10307`. Después se podrá preparar una prueba de carga por TITLEID, separada de cualquier hook gráfico.
