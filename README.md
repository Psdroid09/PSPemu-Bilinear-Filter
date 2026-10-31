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

**Investigación inicial. No existe todavía un plugin, parche ni VPK funcional.**

### Hallazgo confirmado en Adrenaline

El código de Adrenaline incluye opciones de filtrado gráfico: Original, Bilinear, Sharp bilinear, Advanced AA, LCD3x y Sharp bilinear without scanlines. En `user/menu.c`, su hilo de dibujo:

1. Obtiene el framebuffer PSP desde `SCE_PSPEMU_FRAMEBUFFER`.
2. Copia los datos a una textura PSP propia.
3. Configura filtros GXM LINEAR o POINT según la opción Smooth Graphics.
4. Dibuja la textura con un shader seleccionado en su propio renderer Vita2D.

Esto confirma que el suavizado de Adrenaline forma parte de su ruta de composición/dibujo. **No demuestra que esos shaders o ajustes puedan aplicarse directamente al renderer normal de PSPemu ni a las burbujas Chovy-Sign.**

Fuente: [TheOfficialFloW/Adrenaline](https://github.com/TheOfficialFloW/Adrenaline), especialmente `user/menu.c` y `user/includes/`.

### Restricción importante

No se copiará ni instalará el plugin de Adrenaline como si fuera compatible con PSPemu. Primero hay que demostrar un punto de integración válido en el proceso/emulador que ejecuta las burbujas independientes.

## Plan de investigación

- [ ] Identificar exactamente cómo Chovy-Sign configura y lanza las burbujas PSP.
- [ ] Determinar si la burbuja usa el renderer normal de PSPemu o componentes/ruta de Adrenaline.
- [ ] Localizar interfaces documentadas o puntos de integración seguros para acceder a la imagen final.
- [ ] Estudiar los shaders bilineales de Adrenaline y sus dependencias/licencia.
- [ ] Evaluar coste de GPU, latencia, compatibilidad y conflictos con plugins habituales.
- [ ] Diseñar una prueba reversible sin tocar ISO, licencia ni datos de guardado.
- [ ] Compilar y probar únicamente cuando exista una ruta técnica demostrable.

## Criterios para aceptar una solución

- No requiere iniciar manualmente Adrenaline.
- No modifica ISO, licencia ni partidas guardadas.
- No añade escritura/logs persistentes innecesarios.
- No provoca regresiones de rendimiento, crashes ni problemas de suspensión.
- Tiene un método claro de desinstalación y recuperación.

## Licencia

Este repositorio de investigación no incorpora código de Adrenaline. Antes de reutilizar cualquier código o shader externo se revisarán su licencia, avisos de copyright y dependencias.