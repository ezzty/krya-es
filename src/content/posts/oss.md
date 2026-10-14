---
title: "Montar mi propio alojamiento de imágenes de bajo coste con Function Compute de Aliyun + OSS"
description: "Todo blog estático necesita un alojamiento de imágenes de terceros; pero ¿cuál de ellos no desaparece? Difícil de saber. Además, si usas el de otro: las reglas antirrobo no las controlas tú, si el servicio cierra las imágenes acaban en 404, la moderación de contenido te bloquea por error y la cuota gratuita tiene trampas. Después de tropezar con muchos problemas, decidí montarme el mío con un coste de uso bajísimo. El objetivo principal es ahorrar dinero, ¡sin límites!"
pubDate: 2026-10-14
heroImage: ""
categories:
  - Tecnología
tags:
  - Alojamiento de imágenes autoalojado en OSS
  - Alojamiento de imágenes con Function Compute de Aliyun
  - Alojamiento de imágenes con cloud function FC
author: jin
---
![image](https://i.190808.xyz/m/26j13/0f38e9.png)

> Código fuente: https://github.com/ezzty/oss-upload

**Tres cosas antes de empezar:**

1. Todos los `AccessKey`, contraseñas, dominios y nombres de bucket de este artículo son **marcadores de posición**; reemplázalos por los tuyos antes de copiar nada.
2. El código es un único archivo Node.js (unas 380 líneas), las dependencias se inyectan mediante una **capa (Layer)** de FC, el código de negocio no tiene ningún valor hardcodeado y toda la configuración va por variables de entorno.
3. Al terminar este artículo tendrás el flujo completo de `login → obtener token → subir → devolver la sintaxis de imagen en Markdown`, más una lista de verificación ejecutable antes de ponerlo en producción.

---

### 1. Por qué montar tu propio alojamiento de imágenes

Todo blog estático necesita un alojamiento de imágenes de terceros; pero ¿cuál de ellos no desaparece? Difícil de saber. Además, si usas el de otro: las reglas antirrobo no las controlas tú, si el servicio cierra las imágenes acaban en 404, la moderación de contenido te bloquea por error y la cuota gratuita tiene trampas.

Después de tropezar con muchos problemas, decidí montarme el mío con un coste de uso bajísimo. El objetivo principal es ahorrar dinero, ¡sin límites!

- **Los datos están en tu propia cuenta**, puedes exportarlos y migrarlos cuando quieras
- **La facturación es por uso**, el consumo de un alojamiento personal casi siempre queda en el borde de la cuota gratuita
- **La política de seguridad la pones tú**: quién puede subir, cuánto pesa, en qué directorio se guarda y quién puede verlo, todo bajo tu control

La contrapartida es que te toca encargarte tú de la autenticación, la protección contra fuerza bruta y la validación de tipos de archivo — este artículo ya lo ha hecho por ti.

---

### 2. Arquitectura y flujo de peticiones

```typescript
Navegador upload.html
   │  ① POST /login   {user, pass}
   │  ← 200 {token} (válido 24 h)
   │  ② POST /upload  Authorization: Bearer *** + binario de la imagen
   ▼
Function Compute de Aliyun (Node.js 18, disparador HTTP)
   │  Autenticación / limitación de tasa / validación de magic bytes / validación de tamaño
   │  ③ ossClient.put(key, stream)
   ▼
Bucket de OSS (lectura y escritura privadas, lectura pública prohibida)
   │  ④ Recuperación de origen del CDN (red interna, gratis)
   ▼
https://img.example.com/m/26j13/ab12cd.jpg   ← dominio público de acceso
```

**Tres decisiones de diseño:**

- **Bucket privado**: no se concede a nadie permiso de lectura por enlace directo; las imágenes solo se pueden ver a través de la URL del dominio de CDN vinculado. Si lo dejas en lectura pública, tu bucket se convierte en un recurso común: cualquiera puede listarlo y consumir tu tráfico sin pagar.
- **Hay que iniciar sesión para subir**: si no, tu dominio de API se usará como alojamiento de imágenes gratuito y tanto el coste como el almacenamiento se te irán de las manos.
- **El código de negocio nunca toca las claves**: el AK/SK solo existen en las variables de entorno; en el código y en el repositorio nunca aparecen.

---

### 3. Trabajo previo

#### 3.1 Activar los productos

Activa **Function Compute FC** y **OSS (Object Storage Service)** (ambos de pago por uso, sin prepaid).

#### 3.2 Crear el bucket

- **Nombre**: debe ser único a nivel global, por ejemplo `my-img`
- **Región**: elige la más cercana a ti/tus usuarios, por ejemplo `China East 1 (Hangzhou)` → `oss-cn-hangzhou`
- **Permiso de lectura/escritura**: **privado** (importante)
- **Tipo de almacenamiento**: almacenamiento estándar (las imágenes de acceso frecuente deben usar esto; no elijas Infrequent Access ni Archive — esos tienen una duración mínima de almacenamiento y si los borras antes de tiempo te cobran de más)

#### 3.3 Crear un usuario RAM con permisos limitados a un solo bucket (**muy recomendable**)

Muchos tutoriales te dicen directamente que uses la AccessKey de la cuenta principal — **no lo hagas**. Que se filtre el AK principal equivale a que se pierda toda tu cuenta de Aliyun.

Consola RAM → Políticas de permiso → Crear política de permiso → pestaña **JSON**:

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:GetObject",
        "oss:PutObject",
        "oss:DeleteObject",
        "oss:ListObjects",
        "oss:HeadObject"
      ],
      "Resource": [
        "acs:oss:*:*:my-img",
        "acs:oss:*:*:my-img/*"
      ]
    }
  ]
}
```

Nombre de la política: `ImageHostSingleBucket`. Después:

1. RAM → Usuarios → Crear usuario, marcando **"Acceso con AccessKey permanente"**
2. Vincular **solo** la política de arriba a este usuario
3. Generar el AK/SK y **guardarlo de inmediato en tu gestor de contraseñas local**

**Algunas decisiones deliberadas:**

- **No** damos `oss:ListBuckets` — si se filtra, ni siquiera se podrán ver qué buckets tienes
- **No** damos `oss:CopyObject` — el alojamiento de imágenes no necesita permisos de copia; damos lo mínimo posible
- **No** vincules `AliyunOSSFullAccess` — eso permitiría al AK de tu alojamiento leer todos tus buckets de datos

#### 3.4 Disciplina en el almacenamiento de credenciales

- ❌ No las escribas en el código, no las subas a Git
- ✅ Solo van en las variables de entorno de FC
- El `.gitignore` de la raíz del repositorio debe incluir al menos:

```
.env
.env.local
*.key
*.pem
*.p12
credentials.json
```

---

### 4. Preparar la capa de dependencias (Layer)

El código de negocio solo depende de `ali-oss`. Al convertirlo en una capa, el paquete de código se queda en pocos KB.

#### Opción A: descargar el adjunto del Release directamente (recomendado)

El proyecto ya publica un Release, y `layer.zip` y el paquete de código se pueden descargar tal cual:

**https://github.com/ezzty/oss-upload/releases/latest**

| Adjunto | Tamaño | Contenido |
| :--- | :--- | :--- |
| `layer.zip` | 3,25 MB | `ali-oss@6.23.0` y sus dependencias, 2338 archivos en total |
| `oss-upload-code.zip` | unos 6 KB | `index.js` + `package.json`, paquete de despliegue de la función |

**Ventajas**: no necesitas instalar Node.js ni empaquetar un zip, y evitas el problema de que tu `npm install` local obtenga una versión más nueva del SDK y genere una "diferencia con lo que el autor validó". La capa contiene **solo código fuente de paquetes npm, sin credenciales**.

#### Opción B: construirlo tú mismo

Si quieres cambiar la versión de una dependencia o no confías en el paquete ya hecho, empaquétalo tú:

```bash
mkdir -p layer-build/nodejs
cd layer-build
npm init -y
npm install ali-oss
# El directorio debe tener esta estructura:
#   nodejs/
#     node_modules/
#       ali-oss/
#       ...sus dependencias transitivas
cd nodejs && zip -r ../layer.zip node_modules && cd ..
```

#### Subida

**Consola de FC → Gestión de capas → Crear capa:**

- Nombre: `oss-upload-deps`
- Runtime: **Node.js 18**
- Subir `layer.zip`

> La capa solo hay que crearla una vez. Para actualizar la versión de `ali-oss`, empaqueta y publica una versión nueva y apunta la función a esa versión; no hace falta tocar el código de negocio.

---

### 5. Desplegar la función

### 5.1 Obtener el código de negocio

**Recomendado: usar directamente el adjunto del Release**: https://github.com/ezzty/oss-upload/releases/latest y descargar `oss-upload-code.zip` (unos 6 KB); al descomprimirlo ya tienes el paquete de despliegue estándar.

Si empaquetas el tuyo, en la raíz del zip **solo pueden haber estos dos archivos** (no lo anides en una carpeta, o el Handler no encontrará `index.handler`):

```
code.zip
├── index.js        ← código fuente del negocio
└── package.json    ← declara las dependencias (solo referencia; en runtime se cargan desde la capa)
```

```bash
zip code.zip index.js package.json
```

#### 5.2 Crear la función

- Runtime: **Node.js 18**
- Handler: `index.handler`
- Especificación: **0,05 vCPU / 128 MB de memoria / 512 MB de disco** (la mínima que funciona en la práctica)
- **Configuración de capas**: vincular la `oss-upload-deps` que acabas de crear

> Con el tema de la memoria hay que ir con cuidado: el código lee el body completo en memoria (`bodyBuf`) y al decodificar base64 genera además una copia. **Con 128 MB las imágenes normales van sobradas**, pero si subes el límite de subida a 20 MB y sueles enviar imágenes grandes cercanas al límite, habrá que subir la memoria en consecuencia.

#### 5.3 Variables de entorno (8 obligatorias, sin una sola no arranca)

Al principio el código las comprueba una a una; si falta alguna hace `process.exit(1)` directamente, la función devuelve 500 y en los logs aparece `❌ FATAL: Missing required environment variable: X`.

**Las 8 obligatorias** (en el código, `requiredEnvVars` son exactamente estas 8):

- **`AUTH_USER`** `admin` — usuario de inicio de sesión
- **`AUTH_PASS`** `<configura una contraseña fuerte>` — contraseña de inicio de sesión; usa una cadena aleatoria larga
- **`JWT_SECRET`** `<cadena aleatoria de más de 32 caracteres>` — clave de firma del token
- **`OSS_ACCESS_KEY_ID`** `LTAIxxxxxxxxxxxxxxxx` — AK del usuario RAM
- **`OSS_ACCESS_KEY_SECRET`** `<tu AccessKey Secret>` — SK del usuario RAM
- **`OSS_BUCKET`** `my-img` — nombre del bucket
- **`OSS_REGION`** `oss-cn-hangzhou` — ID de la región del bucket
- **`CDN_DOMAIN`** `https://img.example.com` — **debe llevar el prefijo de protocolo**, si no, el arranque falla

**1 opcional:**

- **`ALLOWED_ORIGINS`** `example.com,myblog.net` — orígenes permitidos para CORS, separados por comas. Sin definirlo también arranca; **solo interviene cuando el CORS a nivel de código está activado** (por defecto el CORS lo gestiona la plataforma devolviendo el `Origin`, así que `isValidOrigin` ni se llega a llamar). Si activas el CORS a nivel de código sí que hace falta definirlo, porque si no el valor por defecto `example.com` bloquearía todas las subidas desde el navegador.

![image](https://i.190808.xyz/m/26j13/f09581.png)

Para generar valores aleatorios:

```bash
openssl rand -base64 32   # JWT_SECRET
openssl rand -base64 24   # AUTH_PASS
```

> `CDN_DOMAIN` debe empezar por `http://` o `https://` — el código tiene una validación específica para eso; si olvidas el prefijo de protocolo, el arranque se aborta.

#### 5.4 Vincular el dominio personalizado

Consola de FC → Gestión de dominios → Añadir dominio → sigue las instrucciones para configurar el CNAME y el certificado. Así obtienes una dirección de API como `https://api.example.com`.

---

### 6. Página frontend

`upload.html` es un frontend de un solo archivo; basta con soltarlo en el directorio de tu propio sitio.

**Hay que cambiar una cosa**: busca `_FC_PARTS`:

```javascript
// Cámbialo por tu dominio personalizado de Function Compute
const _FC_PARTS = ['https://', 'api.example.com'];
```

**El límite de tamaño debe coincidir con el del backend** (por defecto ambos son 20 MB):

```javascript
if (file.size > 20 * 1024 * 1024) { /* bloquear */ }
```

Hay un límite en el frontend y otro en el backend, **y manda el más restrictivo**. Si solo cambias el frontend, los archivos grandes recibirán un 413 en el servidor; si solo cambias el backend, el usuario quedará bloqueado en el navegador y ni siquiera podrá enviar nada.

---

### 7. ¿Quieres ir a fondo? El detalle está en el README del código fuente

Ese es el flujo completo para tener en marcha tu propio alojamiento de imágenes. **El porqué del código, el detalle de cada capa de protección y la lista de verificación completa** están recopilados en el README del repositorio; consúltalo cuando lo necesites:

**https://github.com/ezzty/oss-upload** → las secciones "🧩 Implementación clave (a nivel de código)" y "🧪 Verificación de puesta en producción" del `README.md`

Ahí hay cosas que aquí no se desarrollan:

- **Estructura del token y cuatro puntos de seguridad** (firma HMAC, comparación en tiempo constante, los límites de la protección contra replay)
- **Tabla de referencia de magic bytes** (bytes de cabecera de JPEG / PNG / GIF / WebP / BMP)
- **Triple validación de tamaño**, límite de 4 KB para el body de login, y por qué la protección anti fuerza bruta por IP debe usar un `Map` (pruebas con `X-Forwarded-For: __proto__` y contaminación de prototipos)
- **Compatibilidad con el formato de eventos de FC y decodificación del body** (con dos trampas grandes: decodificación perezosa y binarios sin codificar que no se deben decodificar como UTF-8)
- **Cómo envía el frontend** (iOS Safari pierde el body de `xhr.send(File)`; hay que usar ArrayBuffer)
- **Disparador HTTP y las tres formas de CORS** (incluida una aclaración: en la consola de FC no existe ninguna casilla de "permitir peticiones cruzadas")
- **Procesamiento de imágenes en OSS** (de 1,89 MB a 81 KB medido en pruebas, un ahorro del 96%)
- **Lista de verificación completa de puesta en producción** (comandos de sondeo + script de verificación en OSS + checklist de seguridad)
- **Tabla rápida de problemas frecuentes** (síntoma → causa → solución; también los casos difíciles como cuerpos perdidos en iOS o falsos positivos de magic bytes)

> En el cuerpo del blog solo quería explicar bien el "porqué de montarlo tú, cómo ponerlo en marcha y qué tener en cuenta"; si de verdad vas a desplegarlo, el README es la documentación técnica completa.

---

### 8. Coste

Pago por uso, en tres bloques:

| Concepto | Forma de facturación | Referencia de magnitud |
| --- | --- | --- |
| **Cálculo de FC** | CU de cálculo (vCPU + memoria × duración) + número de invocaciones | Hay cuota gratuita; un alojamiento personal suele moverse cerca de ese límite |
| **Almacenamiento en OSS** | yuanes/GB/mes | Almacenamiento estándar unos **0,09~0,12 yuanes/GB/mes** |
| **Peticiones a OSS** | Por petición | Muy bajo, se puede ignorar |
| **Tráfico** | Yuanes/GB | Ver abajo |

**El tráfico es la partida grande de la factura, y también la más fácil de optimizar:**

- **Subida a OSS: gratis**
- **Red interna de OSS (FC y OSS en la misma región): gratis**
- **Salida por la red pública de OSS: de pago** (del orden de 0,5 yuanes/GB, varía según la región y el tramo horario)
- **Si pasa por CDN: la recuperación de origen del CDN va por la red interna y ese tráfico es gratis**; solo se paga el tráfico del CDN

**Conclusión: el alojamiento de imágenes debe ir siempre detrás de un CDN**, que además de acelerar te evita la tarifa de salida pública de OSS. Ojo también a que FC y OSS **deben estar en la misma región**: si están en regiones distintas el tráfico sale por la red pública, que es más lenta y puede generarte costes adicionales.

Según mis propias mediciones, si subes unas pocas decenas de imágenes al mes, el coste ronda el 1 yuan al mes.

---

### 9. Mantenimiento y actualizaciones

**Cambiar el código de negocio (por ejemplo, el formato de los enlaces de imagen):**

1. Modifica `index.js` en local
2. `zip code.zip index.js package.json`
3. Consola → Despliegue de código → Subir → Desplegar
4. **Ejecuta las sondas de verificación del README para confirmar que la nueva versión está activa**

**Cambiar el frontend:**

1. Modifica `upload.html`
2. Súbelo al sitio; se recomienda `Cache-Control: no-cache`, o los usuarios podrían recibir la versión antigua

**Actualizar dependencias:**

1. Vuelve a ejecutar `npm install ali-oss` y empaqueta `layer.zip`
2. Publica una nueva versión de la capa en la consola
3. Cambia la función a la nueva versión de la capa

**Rotar credenciales (hazlo periódicamente):**

1. Genera un AK/SK nuevo en RAM
2. Actualiza las variables de entorno de FC → despliega
3. Cuando confirmes que el servicio funciona, **deshabilita el AK antiguo en la consola de RAM**

---

### 10. Referencias

- Código fuente: https://github.com/ezzty/oss-upload
- [Documentación de Function Compute de Aliyun](https://help.aliyun.com/zh/functioncompute/)
- [Precios del producto OSS de Aliyun](https://www.aliyun.com/price/detail/oss)
- [Explicación de la facturación de Function Compute](https://help.aliyun.com/zh/functioncompute/billing-overview-of-fc)
