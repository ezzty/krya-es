---
title: Cómo desplegar un blog estático gratis con Astro
pubDate: '2026-04-20'
author: jin
draft: false
categories:
- Tecnología
tags:
- Blog
- Astro
- Typecho
- Migración
---

![as-3.avif](https://user0102.cn.imgto.link/public/20260422/as-3.avif)

Con los blogs dinámicos, siempre me preocupaba olvidar renovar el servidor o encontrarme con problemas durante las actualizaciones del programa. Tras descubrir los blogs estáticos, me di cuenta de que era la solución ideal. Después de comparar Hugo, Hexo y Astro, finalmente elegí Astro.

### Ventajas de los blogs estáticos

1. **Alojamiento gratuito** — Aparte del dominio, las grandes empresas tecnológicas ofrecen alojamiento gratuito para desarrolladores individuales.
2. **Cero mantenimiento** — Los artículos se almacenan en tu propio ordenador. Mientras la plataforma de alojamiento no cierre, tu blog puede funcionar para siempre.
3. **Pre-renderizado** — Los blogs estáticos generan archivos HTML/CSS/JS puros antes del despliegue, sin backend.
4. **Compatible con CDN** — Al ser todo archivos estáticos, distribuirlos en nodos globales de borde permite una carga de milisegundos para usuarios de todo el mundo.

### Por qué elegir Astro

1. **Estático por defecto** — Genera archivos HTML puros, sin necesidad de base de datos ni servidor.
2. **Rendimiento excelente** — Cero JavaScript por defecto, carga extremadamente rápida.
3. **Alta flexibilidad** — Soporta Markdown, MDX y frameworks mixtos.
4. **Ecosistema activo** — Comunidad vibrante, documentación completa, rica biblioteca de plugins.

### El tema Sintu de una columna para Astro

Este tema de plantilla de una columna para Astro fue diseñado y construido por mí mismo. Recientemente, Hermes lo migró de Typecho a Astro. Durante la migración, hice bastantes mejoras de funcionalidades, y ahora el código es un tercio más conciso que antes. Aprovecho para comentar cómo la popularización de la IA ha simplificado tareas que antes eran complejas: el progreso del desarrollo ha sido casi divino, la calidad del código es casi perfecta, y me acerca infinitamente al nivel de un programador genio 😂.

#### Aspectos destacados

1. **Rendimiento perfecto** — Carga en milisegundos, Google Lighthouse puntúa 100 tanto en móvil como en escritorio.
2. **Modo oscuro** — Añadí un interruptor de modo oscuro junto al logotipo.
3. **Zoom al pasar el ratón sobre miniaturas** — Además de ampliar, al hacer clic en la imagen se navega a la página del artículo.
4. **Sin imágenes de vista previa en móvil** — Controla la carga de recursos según el tamaño de pantalla, ahorrando ancho de banda.
5. **Procesamiento de imágenes con Aliyun OSS** — Las imágenes se procesan mediante OSS, con parámetros añadidos tras el renderizado HTML, ahorrando ancho de banda.
6. **Optimización de rendimiento** — CSS empaquetado mediante Vite, carga diferida de imágenes.
7. **Diseño responsive** — Adaptación perfecta para escritorio y móvil, usando un sistema de cuadrícula unificado.
8. **Caja de comentarios** — Integrada usando la solución oficial Twikoo.

### Flujo de publicación

1. Escribe tu artículo en Markdown localmente.
2. Envía un mensaje al agente Hermes para subirlo al repositorio de GitHub (o súbelo tú mismo mediante GitHub).
3. Tencent EdgeOne, Vercel, Netlify o Cloudflare Pages detectan la actualización y la compilan automáticamente.
4. Se despliega automáticamente en los nodos globales de CDN.

### Código abierto

Este tema es de código abierto. Si te gusta, también puedes invitar al autor a un café:

[https://github.com/ezzty/krya-en](https://github.com/ezzty/krya-en)
