# Advanced AA: source shader and algorithm audit

## Resultado nuevo

Se localizó el código fuente legible de un shader llamado **AdvancedAA** en la rama `gtu` de `frangarcj/vita-shader-collection`:

- [Fragment shader `advanced_aa_f.cg`](https://github.com/frangarcj/vita-shader-collection/blob/gtu/shaders/advanced_aa_f.cg)
- [Vertex shader `advanced_aa_v.cg`](https://github.com/frangarcj/vita-shader-collection/blob/gtu/shaders/advanced_aa_v.cg)

El menú de Adrenaline acredita a frangarcj por el filtrado gráfico y carga un par de programas GXM llamados `advanced_aa_v` / `advanced_aa_f`. El nombre y el par coinciden con los archivos de la colección. Esto hace que la colección sea una referencia muy sólida para el shader de Adrenaline, aunque no se ha hecho una comparación binaria del GXP compilado y por eso no afirmamos identidad byte por byte.

## Qué algoritmo es

No es correcto llamarlo FXAA. El fragment shader implementa un filtro antialiasing espacial adaptativo de 3x3:

1. Lee nueve muestras de textura: centro, cuatro vecinos cardinales y cuatro diagonales.
2. Calcula diferencias de color para estimar variación horizontal, vertical y diagonal.
3. Pondera las muestras en función de esas diferencias.
4. Calcula dos candidatos de color y mezcla los resultados según la diferencia respecto al píxel central.

El vertex shader precalcula las coordenadas de las ocho muestras vecinas, usando desplazamientos de medio texel. Los comentarios del propio shader indican usos como suavizar gráficos ampliados 2x a 4x, pantallas de alta resolución reducidas a 2x o gráficos de alta resolución llevados a tamaño original.

Por tanto, ya tenemos una descripción concreta del algoritmo de AdvancedAA: **antialiasing espacial de 9 muestras con ponderación adaptativa**, no un simple bilinear y no FXAA.

## Coste y compatibilidad esperables

El fragment shader realiza nueve lecturas de textura por píxel. A una superficie de salida de 960x544 son aproximadamente 4.70 millones de lecturas por fotograma, antes de contar el resto del trabajo gráfico. A 60 fps serían alrededor de 282 millones de lecturas de textura por segundo. Es una cuenta de muestras, no un benchmark de rendimiento en Vita.

La prueba real debe medir FPS, frame pacing y temperatura; no se debe asumir que el coste es gratuito. En particular, el shader solo es una etapa de composición de imagen. No aumenta la resolución interna del juego ni activa MSAA en la geometría PSP.

## Licencia

Los archivos fuente encontrados indican copyright de **guest (2006)** y licencia **GNU GPL versión 2 o posterior**. No copiar el código fuente ni los binarios al proyecto sin cumplir esa licencia, conservar los avisos y revisar las obligaciones aplicables. Si se opta por una implementación propia distinta, debe evitarse copiar la expresión protegida del shader.

## Qué falta para que sea una prueba real en Chovy-Sign

El shader resuelve el *qué* (un algoritmo concreto para suavizar bordes), pero no el *dónde/cuándo*:

- Todavía no se ha demostrado un callback seguro que entregue cada framebuffer PSP completo en una burbuja Chovy-Sign normal.
- No se ha demostrado que un plugin pequeño pueda asumir la presentación de pantalla y devolverla correctamente al salir/suspender.
- El renderer Adrenaline usa un hilo de dibujo propio, textura propia, GXM y sincronización con PSPemu. Copiar solo el shader no reproduce esa integración.
- La fuente de AdvancedAA no convierte por sí sola el shader en un plugin.

## Próximo criterio de avance

Antes de compilar un VPK, se debe construir una prueba técnica aislada que:
1. confirme dimensiones, formato, stride y estabilidad del framebuffer en FW 3.60;
2. capture un frame solo después de la sincronización correcta;
3. lo presente sin competir con el renderer de Sony;
4. permita desactivar/restaurar la presentación original sin reiniciar la consola;
5. compare el frame original y el filtrado, y mida rendimiento.

Hasta entonces, no hay nada que instalar ni una prueba segura para el usuario.
