---
title: Con la IA, ya nunca me preocupo por dejar la puerta abierta
pubDate: '2026-04-29'
author: jin
draft: false
categories:
- Tecnología
tags:
- IA
- Home Assistant
- Hogar inteligente
- Herramientas
---

Estás fuera de casa y de repente te asalta la duda: ¿me habré dejado la puerta abierta?

Seguro que mucha gente ha sentido esa ansiedad. Yo, desde luego, sí. Cada vez que salgo de casa, no paro de rebobinar el recuerdo: ¿cerré bien la puerta? A veces ya había salido del barrio y volvía corriendo a comprobarlo.

Para solucionarlo, me construí una página «caja de herramientas» que reúne toda la información de mi casa —estado de la puerta, temperatura, datos del router— en un solo sitio, accesible desde cualquier lugar y en cualquier momento.

La historia empieza con un servicio de alojamiento de imágenes.

![948.avif](https://user0102.cn.imgto.link/public/20260429/948.avif)

### Empezando con el alojamiento de imágenes

Escribo un blog y a veces necesito insertar imágenes con Markdown. Usar un servicio de imágenes de otro siempre me daba desconfianza. Usar mi propio cliente de Alibaba Cloud OSS me parecía demasiado engorroso. Nunca encontraba una solución web que se sintiera realmente sencilla.

Ahora que tengo Hermes Agent, quería sacarle el máximo partido. Le pedí que me construyera un servicio de alojamiento de imágenes para poder subir fotos desde el móvil.

La implementación fue sencilla:

Paso uno: le expliqué mi requisito — usar Alibaba Cloud OSS para crear un servicio privado de imágenes con autenticación por contraseña para evitar abusos. La IA recomendó usar Alibaba Cloud Function Compute como backend, y el frontend podía alojarse en cualquier sitio.

Yo solo hice una cosa: proporcionar las credenciales de la API de Alibaba Cloud OSS.

El resto lo hizo la IA.

Casualmente, Xiaomi estaba ofreciendo miles de millones de tokens gratis a desarrolladores y conseguí algunos. También era una oportunidad para probar el modelo MiMo-v2.5-pro. A los pocos minutos de dar la instrucción, ya había generado el código del frontend y del backend, y me pidió que subiera el backend al servidor. Tras un par de rondas de depuración, la página de subida de imágenes a OSS estaba lista.

Una vez que la página estuvo en marcha, sentí que era demasiado simple. ¿Podría hacerla más completa?

Así que me puse a construir una «Caja de herramientas».

### La Caja de herramientas

Combinando la información que consulto a diario, primero construí visualizaciones de bolsa y criptomonedas usando las APIs de Tencent Finance y OKX.

Estas herramientas eran muy sencillas porque todos los datos vienen de APIs externas. Luego quise probar algo más avanzado.

Lo que quería ver:

1. Si la puerta de casa está cerrada (esto requiere instalar Home Assistant con Docker y el plugin miot de Xiaomi)
2. El estado del router
3. El estado del servidor

Mi red doméstica funciona en un mini PC con Linux, y también tengo un entorno Linux en internet público. Pero el servidor público no puede acceder a mi red local. La IA sugirió varios enfoques, y decidí recolectar datos desde la red local y enviarlos al servidor público.

### Implementación técnica

El núcleo de todo el sistema es la recolección y el envío de datos.

**Estado del hogar**

Home Assistant conecta varios dispositivos inteligentes: cerraduras, sensores de temperatura y humedad, purificadores de aire, etc.

Escribí un script de recolección local que obtiene datos de la API de Home Assistant cada minuto y los sube al servidor público.

El estado de la puerta usa un sensor de puerta/ventana que detecta si está abierta o cerrada mediante un imán. Los sensores de temperatura y humedad están colocados en el exterior, el interior y el armario del servidor, así que puedo monitorizar el ambiente del hogar en todo momento.

**Estado del router**

El router ejecuta OpenWrt. A través de la API web de LuCI, puedo obtener la temperatura de la CPU, el tiempo de actividad, el número de usuarios conectados y otros datos. Escribí un script en shell que recolecta información cada 5 minutos y la sube mediante SCP.

**Estado del servidor**

El estado del servidor es aún más sencillo: basta con leer los archivos del sistema para obtener la temperatura de la CPU, el uso de memoria, el espacio en disco y otras métricas.

### El resultado

Ahora, cada vez que abro la página de la caja de herramientas, puedo ver:

- **Estado del hogar**: si la puerta está cerrada, temperatura interior y exterior, temperatura del armario del servidor
- **Tendencias del mercado**: precios de Xiaomi, Tesla y Oro
- **Cripto**: precios en tiempo real de BTC y ETH
- **Estado del router**: temperatura de la CPU, número de usuarios conectados
- **Estado del Homelab**: uso de CPU, memoria y disco del servidor
- **Reloj mundial**: hora en Pekín, Londres y Nueva York
- **Poesía diaria**: un poema clásico chino al azar

Toda la página también es compatible con PWA, así que se puede añadir a la pantalla de inicio del móvil y usarla como una aplicación nativa.

### Sacándole el máximo partido

El valor de esta caja de herramientas está en reunir en un solo lugar información dispersa en diferentes plataformas.

Antes, para comprobar la cerradura tenía que abrir la app de Mi Home. Para ver el estado del router, abría el navegador y escribía la IP. Para consultar el precio de las acciones, abría una aplicación de trading.

Ahora, una sola página lo cubre todo.

Y como lo he construido yo mismo, puedo añadir nuevas funciones en cualquier momento. Por ejemplo, podría agregar pronósticos del tiempo, seguimiento de paquetes o recordatorios del calendario más adelante.

Pero, sinceramente, ahora mismo no se me ocurren más funciones que necesite. Con esto ya tengo suficiente por ahora.
