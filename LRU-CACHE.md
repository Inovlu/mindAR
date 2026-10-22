# Caché LRU de modelos GLB para MindAR

## Objetivo

Evitar que A-Frame descargue e inicialice los 21 GLB al iniciar la escena. La aplicación carga el modelo de un target solo cuando MindAR lo detecta y mantiene como máximo tres entidades de modelo en la caché de la aplicación.

Este límite no incluye la cámara, MindAR ni los datos de los 21 targets que contiene `targets/targets.mind`. Tampoco controla la caché HTTP interna del navegador ni garantiza que iOS recupere inmediatamente toda la memoria liberada.

## Componentes

- `index.html` conserva las 21 entidades `mindar-image-target`, pero ya no declara los modelos en `<a-assets>`.
- Un mapa de rutas JavaScript mantiene la asociación entre cada `targetIndex` y su archivo GLB. El orden del mapa debe corresponder al orden con el que se compiló `targets/targets.mind`.
- `#model-cache` es el contenedor de retención de los modelos que no están siendo seguidos; cada modelo retenido se oculta individualmente.
- El controlador escucha `targetFound` y `targetLost` en cada entidad de target.

## Política LRU

1. En `targetFound`, si el GLB ya está en caché, se mueve al final del `Map`: pasa a ser el más recientemente usado (MRU). Si está listo, se vuelve a colocar bajo la entidad MindAR correspondiente.
2. Si el GLB no está en caché, se agrega a la cola de solicitudes. El controlador carga solicitudes mientras haya espacio.
3. La caché cuenta modelos listos y cargas en curso; ambos ocupan un slot y el total nunca debe superar tres.
4. Si está llena, el controlador expulsa el elemento menos recientemente usado que esté inactivo y ya haya terminado de cargar (LRU), libera sus recursos de Three.js y luego carga la solicitud pendiente.
5. Si los tres slots están activos o todavía cargando, la solicitud queda en espera. Cuando llega `targetLost` o termina una carga, vuelve a intentar procesar la cola.
6. `targetLost` oculta el modelo y lo mueve al contenedor de caché. No lo elimina por sí solo. Si su solicitud todavía estaba en cola, la cancela.
7. Si un target se pierde durante su carga, la carga termina en el contenedor oculto. No se muestra hasta un nuevo `targetFound`.
8. Los errores de carga eliminan la entrada fallida, registran el error en la consola y liberan espacio para la siguiente solicitud.

Los modelos activos nunca se expulsan. Esto es importante si varios targets aparecen en cámara simultáneamente: una solicitud adicional espera hasta que haya un slot seguro.

## Liberación de recursos

Al expulsar un modelo se recorren sus objetos y se llaman `dispose()` para texturas, materiales y geometrías. También se retira el componente `gltf-model`, la entidad y, cuando `THREE.Cache` está habilitada, la entrada de URL correspondiente.

El navegador puede conservar bytes descargados en su caché HTTP, y el recolector de basura de JavaScript/iOS no tiene obligación de devolver memoria inmediatamente. La garantía del código es un máximo de tres modelos mantenidos por esta caché de escena, no un límite absoluto de RAM del proceso.

## Pruebas manuales recomendadas

1. Abrir la consola remota de Safari/Web Inspector y cargar la página desde un servidor HTTPS.
2. Escanear targets 0, 1 y 2. Confirmar que la consola muestre tres cargas y que el contador nunca supere `3/3`.
3. Perder un target y encontrarlo de nuevo antes de usar un cuarto. Debe aparecer el mensaje `cache hit, moved to MRU`, sin una nueva carga.
4. Con los targets 0, 1 y 2 en seguimiento, presentar el target 3. Debe quedar esperando y no debe empezar una cuarta carga.
5. Perder uno de los tres targets. Debe expulsarse un modelo inactivo según orden LRU y comenzar la carga del target 3.
6. Confirmar visualmente que los GLB se mantienen bajo el target correcto, y que un modelo no sigue visible después de `targetLost`.
7. Repetir varias rondas en Safari/iOS; observar memoria, cierres, errores de carga y la consola `[GLB LRU]`.
8. Repetir en escritorio como prueba funcional complementaria. El comportamiento de memoria debe confirmarse en el dispositivo iOS objetivo.

## Diagnóstico en consola

El controlador emite mensajes `[GLB LRU]` para cargas, hits, pérdidas, expulsiones y errores. Los mensajes muestran el tamaño de la caché y permiten comprobar que no exceda `3/3`.

## Archivos y rutas

Las rutas GLB permanecen relativas al HTML (`assets/...`) y el archivo de targets sigue configurado como `./targets/targets.mind`, por lo que la estructura puede servirse desde una ruta raíz o subruta estática compatible.
