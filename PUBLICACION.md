# Cómo publicar PDF GioSoft en Google Play

Guía paso a paso, explicando cada cosa. No hace falta saber nada previo: si sigues
el orden, funciona.

**Estado a 14 de septiembre de 2026:** la llave de firma todavía no existe. Ese es
el paso 2.

---

## 1. Primero, lo que hay que entender (5 minutos de lectura)

### ¿Qué es «firmar» una app?

Cuando subes una app a Google Play, Google necesita una forma de saber que las
actualizaciones futuras las manda **la misma persona** que subió la primera
versión. Si no, cualquiera podría publicar una actualización falsa de tu app.

Para eso está la **firma digital**. Funciona como la firma de tu cédula: es tuya,
nadie más la tiene, y sirve para demostrar que algo lo hiciste tú.

Esa firma vive dentro de un archivo. Ese archivo es la **llave**.

### ¿Qué es el archivo `.jks`?

Es la llave. Un archivo pequeño que tu computador genera una sola vez y que
queda protegido con una contraseña que tú eliges.

```
pdfgiosoft-release.jks   ←  este archivo firma todas tus actualizaciones
```

> ⚠️ **Lo más importante de toda esta guía:**
> si pierdes ese archivo **o su contraseña**, no puedes volver a actualizar la app
> en Play. Nunca. Ni escribiendo a Google. Tendrías que publicar una app nueva,
> desde cero, y quienes tuvieran la vieja no recibirían nunca más una
> actualización.
>
> Por eso el paso 2 insiste tanto en guardar copia.

### ¿Qué es un gestor de contraseñas y por qué te lo pido?

Un **gestor de contraseñas** es una aplicación que guarda contraseñas por ti,
cifradas, y te las muestra cuando las necesitas. Tú solo recuerdas una contraseña
(la del gestor) y él recuerda todas las demás.

Seguramente ya usas uno sin saberlo: **el de Google**, ese que te pregunta
«¿guardar contraseña?» en Chrome; lo ves en
[passwords.google.com](https://passwords.google.com). Otros conocidos:
**Bitwarden** (gratis), **1Password**, **KeePass**.

Te lo pido porque la contraseña de la llave **no se puede recuperar**. No hay un
«olvidé mi contraseña» como en Facebook: no existe nadie al otro lado que pueda
resetearla. Si la apuntas en un papel y lo pierdes, o confías en acordarte dentro
de dos años, te quedas sin poder actualizar tu app.

Lo mismo con el archivo `.jks`: guarda una copia en dos sitios distintos (por
ejemplo, una USB y tu Google Drive privado). Nunca dentro de la carpeta del
proyecto, porque esa se sube a GitHub y la llave quedaría a la vista de
cualquiera.

### ¿Qué es un `.aab`?

Es el paquete que se sube a Play. Antes se subían `.apk`; hoy Google pide `.aab`
(*Android App Bundle*). La diferencia práctica: el `.aab` lleva la app para todos
los celulares posibles, y Play arma para cada usuario solo el pedacito que su
celular necesita, así descarga menos.

Tu proyecto ya está configurado para generarlo. Es un comando.

---

## 2. Crear la llave (una sola vez en la vida de la app)

### Dónde guardarlas cuando tienes varias apps

Cada app lleva **su propia llave**. Podrían compartir una, pero no conviene: si
algún día vendes una app, o una llave se te escapa, no querrás que eso arrastre
a las demás. Son archivos de 3 KB; no hay ninguna razón para ahorrar.

Todas juntas en una carpeta madre, una subcarpeta por app:

```
C:\Users\GIOVANNY\Documents\Trabajo\Giosoft\llaves\
├── pdfgiosoft\
│   ├── pdfgiosoft-release.jks
│   └── datos.txt          ← qué app es, alias y fecha (SIN la contraseña)
├── maskoti\
│   ├── maskoti-release.jks
│   └── datos.txt
└── copitas\
    └── ...
```

Por qué una carpeta madre y no la llave dentro de cada proyecto:

- **Se respaldan todas de una vez.** Copias `llaves\` a una USB y ya está.
- **No se comparte sin querer.** Si algún día mandas la carpeta de un proyecto o
  la subes a algún lado, la llave no va dentro.

**La carpeta no puede estar dentro de un proyecto de git.** `Documents\Trabajo\
Giosoft` no lo es (lo comprobé), así que sirve. Lo que no vale es meterlas en
`AndroidStudioProjects\...`, porque eso sí se sube a GitHub.

> ⚠️ Esa carpeta **no se sincroniza con OneDrive** en tu equipo: Documents está
> en el disco local. O sea que ahí no hay copia automática de nada. La copia la
> tienes que hacer tú.

### Crear la llave

Abre **PowerShell** (búscalo en el menú de inicio) y pega:

```powershell
mkdir C:\Users\GIOVANNY\Documents\Trabajo\Giosoft\llaves\pdfgiosoft
cd C:\Users\GIOVANNY\Documents\Trabajo\Giosoft\llaves\pdfgiosoft

keytool -genkeypair -v -keystore pdfgiosoft-release.jks -storetype PKCS12 -keyalg RSA -keysize 4096 -validity 10000 -alias pdfgiosoft
```

Para la siguiente app, cambias las tres veces que aparece `pdfgiosoft` por el
nombre de esa app y ya está.

**Si dice que `keytool` no existe**, es porque esa herramienta viene con Java y
Windows no sabe dónde está. Usa esta versión, que apunta al Java de Android
Studio:

```powershell
& "C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe" -genkeypair -v -keystore pdfgiosoft-release.jks -storetype PKCS12 -keyalg RSA -keysize 4096 -validity 10000 -alias pdfgiosoft
```

### Qué te va a preguntar

**Primero, una contraseña.** Invéntate una larga. **Antes de escribirla, guárdala
en tu gestor de contraseñas** con un nombre que diga de qué app es: «Llave de
firma — PDF GioSoft». Con varias apps esto deja de ser un detalle; dentro de un
año no vas a saber qué contraseña era de cuál llave.

No la escribas primero y la guardes después: al teclearla no se ve en pantalla y
es fácil equivocarse sin enterarse.

**Después, unos datos tuyos**, uno por línea. Puedes responder así:

| Pregunta en inglés | Qué poner |
|---|---|
| What is your first and last name? | Giovanny Tovar |
| What is the name of your organizational unit? | GioSoft |
| What is the name of your organization? | GioSoft |
| What is the name of your City or Locality? | Bogota |
| What is the name of your State or Province? | Cundinamarca |
| What is the two-letter country code? | CO |

Al final te pregunta `Is CN=Giovanny Tovar, OU=GioSoft... correct?` → escribe
**`yes`** y Enter.

Esos datos quedan dentro del certificado para siempre, pero **no se muestran a
los usuarios** en Play. Evita tildes y la ñ.

### Dejar constancia de qué es cada llave

Al lado de cada `.jks`, un `datos.txt` con lo que vas a necesitar recordar.
**Nunca la contraseña**: esa vive en el gestor.

```powershell
notepad datos.txt
```

```
App:           PDF GioSoft
applicationId: com.giosoft.pdf
Alias:         pdfgiosoft
Creada:        septiembre de 2026
Contraseña:    en el gestor, como "Llave de firma — PDF GioSoft"
```

Y si algún día abres una carpeta y no sabes qué llave es, esto te lo dice (pide
la contraseña del almacén):

```powershell
keytool -list -v -keystore pdfgiosoft-release.jks
```

Te muestra el alias, cuándo se creó, hasta cuándo vale y las huellas del
certificado.

### Y ahora, la copia de seguridad

**Antes de seguir**, copia la carpeta `llaves` a otro sitio: una USB, un disco
externo, tu Drive personal. Es el momento, no «luego». Si el disco de este
computador muere mañana, con él se van todas tus apps.

## 3. Decirle al proyecto dónde está la llave

El proyecto no sabe dónde guardaste la llave ni cuál es su contraseña. Se lo dices
en un archivo llamado `keystore.properties`. Ese archivo **no se sube a GitHub**
(está en la lista de exclusiones), así que tu contraseña no se hace pública.

```powershell
cd C:\Users\GIOVANNY\AndroidStudioProjects\PDFGioSoft
copy keystore.properties.template keystore.properties
notepad keystore.properties
```

Se abre el Bloc de notas. Déjalo así, con tu contraseña de verdad:

```properties
storeFile=C:/Users/GIOVANNY/Documents/Trabajo/Giosoft/llaves/pdfgiosoft/pdfgiosoft-release.jks
storePassword=AQUI_TU_CONTRASEÑA
keyAlias=pdfgiosoft
keyPassword=AQUI_TU_CONTRASEÑA
```

Dos detalles que dan guerra:

- Las barras van **así `/`**, no así `\`. Aunque Windows use las otras.
- `storePassword` y `keyPassword` son la misma contraseña si, cuando `keytool` te
  preguntó por la contraseña de la llave, solo pulsaste Enter para reutilizar la
  del almacén (que es lo normal).

Guarda y cierra.

---

## 4. Generar el paquete para Play

```powershell
cd C:\Users\GIOVANNY\AndroidStudioProjects\PDFGioSoft
.\gradlew.bat bundleRelease
```

Tarda unos minutos. Cuando termine con `BUILD SUCCESSFUL`, tu paquete está en:

```
app\build\outputs\bundle\release\app-release.aab
```

**Comprueba que quedó firmado.** Esto importa: si algo falla con la llave, el
proyecto genera el paquete igual, pero **sin firma**, y no falla el build (es a
propósito, para que cualquiera pueda compilar el proyecto sin tener tus llaves).
Un paquete sin firma, Play lo rechaza.

```powershell
.\gradlew.bat assembleRelease
& "$env:LOCALAPPDATA\Android\Sdk\build-tools\36.0.0\apksigner.bat" verify --print-certs app\build\outputs\apk\release\app-release.apk
```

Tiene que salir tu nombre en `Signer #1 certificate DN:`. Si sale
`Android Debug` o un error de «no firmado», revisa el paso 3.

---

## 5. Probar el paquete de release ANTES de subirlo

Este paso parece opcional y no lo es. Lo que instalas normalmente en el celular
(`installDebug`) **no es lo mismo** que lo que va a Play.

La versión que va a Play pasa por **R8**, un programa que recorta y comprime el
código: borra lo que cree que nadie usa y acorta los nombres internos para que la
app pese menos. El problema es que a veces se equivoca y borra algo que sí hacía
falta, y entonces la app falla **solo en la versión de Play**.

No es teoría: en este proyecto pasó. R8 dejaba sin un pedazo al escáner de
documentos, y tocar «Escanear» cerraba la app de golpe. En la versión de
desarrollo funcionaba perfecto. Está arreglado (ver `app/proguard-rules.pro`),
pero la lección queda: **prueba siempre la versión real.**

Cómo probarla, de la forma más parecida a lo que recibirá la gente: sube el
`.aab` a un canal de **prueba interna** en Play Console e instálalo desde el
enlace que te da Google. Ahí abre la app y comprueba a mano:

- [ ] Abrir un PDF
- [ ] Escanear un papel
- [ ] Imprimir
- [ ] Ponerle contraseña a un PDF, cerrar y volver a abrirlo
- [ ] Renombrar un documento
- [ ] Eliminar del celular

---

## 6. Play App Signing: actívalo, es tu seguro

Cuando crees la app en Play Console, Google te ofrece **Play App Signing**. Di que
sí.

Qué significa: tu llave pasa a ser la *llave de carga* (con ella firmas lo que
subes), y Google guarda una copia de la llave de distribución real. Si algún día
pierdes tu `.jks`, puedes pedirle a Google cambiar la llave de carga y seguir
actualizando tu app.

Es la única red de seguridad que existe para lo que se advierte en el paso 1.
Sin esto, perder el archivo significa perder la app.

---

## 7. Subir la página web (privacidad, términos y ayuda)

Google **exige** una dirección web pública con tu política de privacidad, y no
acepta un PDF ni un documento de Google: tiene que ser una página normal, abierta,
sin pedir cuenta para verla.

Ya está escrita, en la carpeta [`web/`](web) del proyecto:

```
web/index.html        portada con enlaces
web/ayuda.html        cómo usar la app
web/privacidad.html   ← esta es la URL que le das a Play
web/terminos.html     condiciones de uso
web/estilos.css       los colores y la tipografía (no se toca)
```

### Subirla a Cloudflare, paso a paso

Como ya tienes `cremanti.com` comprado ahí, lo más fácil es **Cloudflare Pages**,
que es gratis:

1. Entra en [dash.cloudflare.com](https://dash.cloudflare.com) con tu cuenta.
2. En el menú de la izquierda: **Compute (Workers & Pages)** → botón
   **Create** → pestaña **Pages** → **Upload assets**.
3. Ponle nombre al proyecto: `pdfgiosoft`. Clic en **Create project**.
4. Arrastra ahí la carpeta `web` entera desde el explorador de Windows
   (`C:\Users\GIOVANNY\AndroidStudioProjects\PDFGioSoft\web`). Suelta y espera a
   que suban los cinco archivos.
5. Clic en **Deploy site**. En menos de un minuto te da una dirección tipo
   `https://pdfgiosoft.pages.dev`.
6. Ábrela en el navegador y comprueba que se ve bien, sobre todo
   `https://pdfgiosoft.pages.dev/privacidad.html`.

**Esa dirección ya sirve para Play.** Puedes parar aquí si quieres.

### Ponerle tu dominio (opcional)

Si prefieres que se vea `legal.cremanti.com` en vez de `pdfgiosoft.pages.dev`:

1. Dentro del proyecto recién creado, pestaña **Custom domains** → **Set up a
   custom domain**.
2. Escribe `legal.cremanti.com` y confirma. Como el dominio ya está en tu misma
   cuenta de Cloudflare, él solo crea lo que hace falta; no tienes que tocar DNS.
3. En un par de minutos funcionan **las dos** direcciones.

### El enlace «Sitio web» que sale en la app

Dentro de «Acerca de» hay una fila que abre tu sitio web. La dirección que viaja
dentro de la app es:

```
https://pdfgiosoft.pages.dev/web
```

**Esa no es tu web: es un reenvío.** Funciona como cuando te mudas de casa y
dejas orden en el correo de reenviar las cartas a la dirección nueva: quien
escribe sigue usando la dirección vieja, y las cartas llegan igual.

La regla está en el archivo [`web/_redirects`](web/_redirects):

```
/web    /index.html    302
```

Hoy lleva a la página de ayuda. Cuando tengas el sitio de GioSoft, cambias esa
línea por la dirección nueva, vuelves a subir la carpeta a Cloudflare, y listo:

```
/web    https://giosoft.com    302
```

**La app no se toca y no hay que publicar nada en Play.** Si en su lugar se
hubiera puesto la dirección final dentro de la app, cada mudanza obligaría a
subir una versión nueva y a esperar a que cada usuario la instalara.

Dos avisos:

- **Antes de publicar en Play**, comprueba que el proyecto de Cloudflare Pages se
  llama exactamente `pdfgiosoft` (eso es lo que produce `pdfgiosoft.pages.dev`).
  Si Cloudflare te da otro nombre porque ese esté ocupado, cambia la línea
  `about_website_url` en `app/src/main/res/values/strings.xml` **antes** de subir
  la app. Después de publicada, esa dirección ya no se puede cambiar sin una
  versión nueva.
- Que sea **302** y no 301 es importante: los navegadores recuerdan las 301
  durante meses, así que una 301 mal puesta seguiría llevando a la dirección
  vieja aunque la cambiaras.

### Cuando tengas el dominio de GioSoft

No hay que republicar la app, y esto es importante que quede claro: **la dirección
de la política vive en Play Console, no dentro de la app**. Cambiarla es editar un
campo en la web de Google.

Cuando llegue el momento:

1. Añade el dominio nuevo al **mismo** proyecto de Pages (igual que el paso
   anterior).
2. En Play Console, cambia la URL de la política de privacidad por la nueva.
3. Deja `cremanti.com` apuntando al mismo sitio, o crea una redirección, mientras
   sigan circulando enlaces viejos. Y mantén ese dominio renovado.

### Para actualizar la página más adelante

Cambias los archivos de `web/` en tu computador, vuelves al proyecto en Cloudflare
Pages → **Create deployment** → arrastras la carpeta otra vez. La dirección no
cambia.

---

## 8. Rellenar la ficha en Play Console

Además del paquete, Google pide:

- [ ] **Capturas de pantalla**: mínimo 2 de teléfono. Se sacan con el celular
      (botón de encendido + volumen abajo) abriendo la app. Consejos en
      [`play/README.md`](play/README.md).
- [x] **Icono** de 512×512 y **gráfico destacado** de 1024×500: ya están hechos,
      en la carpeta [`play/`](play). Se regeneran con
      `py -3 play/generar-graficos.py` si cambia el icono de la app.
- [x] **Categoría**: Productividad.
- [x] **Etiquetas** (hasta 5): PDF, escáner de documentos, gestor de
      documentos, impresión, oficina.
- [x] **Descripción corta** (80 caracteres) y **descripción larga**: redactadas,
      ver conversación / historial de commits de esta guía.
- [ ] **Clasificación de contenido**: un cuestionario. Responde que no hay
      violencia, ni sexo, ni apuestas, ni compras. Sale «Apto para todos».
- [ ] **Política de privacidad**: la URL del paso 7.
- [x] **Seguridad de los datos**: ver abajo, tiene truco. (Hecho, ver detalle.)
- [ ] **Datos fiscales y país** del desarrollador.
- [ ] `versionCode` subido en `app/build.gradle.kts` si ya subiste una versión
      antes (Play rechaza dos paquetes con el mismo número).

### El formulario de «Seguridad de los datos»

Es una declaración jurada sobre qué datos recoge tu app. Miente aquí (aunque sea
sin querer) y te pueden retirar la app.

La definición exacta de Google (verificada en su [página de ayuda]
(https://support.google.com/googleplay/android-developer/answer/10787469)):

> «Recoger» significa transmitir datos a otro lugar que está fuera del
> dispositivo del usuario. No es necesario declarar los datos a los que acceda
> tu app si solo se tratan de forma **local** y no se envían fuera del
> dispositivo. Esto incluye los datos que **cualquier SDK o librería** que uses
> transmita fuera del dispositivo, aunque no viajen a tu propio servidor.

Aplicado a PDF GioSoft:

- **¿La app recoge o comparte datos de usuario?** → **Sí.** (Ver el motivo
  abajo: no es por tus documentos, es por una librería de Google.)
- **Fotos y vídeos** (por el escáner) → **No se marca.** El escaneo lo hace ML
  Kit dentro del celular; la app solo recibe el PDF ya armado. Ninguna foto sale
  del dispositivo.
- **Archivos y documentos** (por leer PDF) → **No se marca.** El usuario elige
  el PDF con el selector del sistema y se procesa enteramente en local (incluso
  en un proceso aislado, ver `docs/ARQUITECTURA.md`). No se sube a ningún
  servidor.
- **App info y rendimiento → Diagnósticos** → **Sí se marca.** El escáner
  arrastra `com.google.android.datatransport:transport-backend-cct`, la
  librería de telemetría de Google que ya vimos que añade el permiso de
  Internet al manifiesto final. Por la definición de arriba, lo que esa
  librería envíe cuenta como "recogido" aunque no sea tu servidor el que lo
  reciba y no sean tus documentos lo que viaja. Está declarado en la política
  de privacidad publicada, sección 8.

  Dentro de esa categoría hay tres casillas; marca **solo «Diagnósticos»**:
  «Registros de fallos» es específico de reportes de errores (número de
  fallos, rastreos de pila) y `transport-backend-cct` no es un reportero de
  fallos, sino una tubería genérica de eventos de uso/rendimiento — la propia
  definición de Google para «Diagnósticos» incluye textualmente
  «diagnósticos técnicos», que es justo esto. «Otros datos de rendimiento» no
  hace falta: es un cajón de sastre para lo que no encaje en las otras dos.

  Para ese dato, Play también pregunta si se **recoge**, se **comparte**, o
  ambas cosas. Marca **las dos**: los datos salen del celular hacia Google
  (cuenta como "recogidos" aunque nunca lleguen a ti), y Google no actúa aquí
  como simple "proveedor de servicios" — no trata esos datos solo por tus
  instrucciones y para tu negocio, sino para su propio producto (mejorar
  ML Kit), así que también cuenta como "compartidos" con un tercero.

  El resto de preguntas de esa misma pantalla, para «Diagnósticos»:

  - **¿Se tratan de forma temporal?** → **No.** `transport-backend-cct` guarda
    eventos y los manda por lotes más tarde; no es "solo en memoria durante
    una solicitud en tiempo real".
  - **¿Es necesaria o el usuario puede elegir?** → **Necesaria.** La app no
    ofrece ningún interruptor para desactivarla; se activa sola al escanear.
  - **¿Por qué se recogen / se comparten?** → **Solo «Análisis»** en ambas. Es
    justo la definición de Google: "monitorizar el estado de la app,
    diagnosticar errores, mejorar el rendimiento". No es "Funcionalidad de la
    aplicación" (ninguna función de la app depende de ese dato) ni ninguna de
    las demás (no hay notificaciones, publicidad, cuentas, ni antifraude).
- **¿Comparte datos con terceros?** → **Sí**, en el sentido estricto de Google:
  la telemetría de ML Kit va a Google. No hay ningún otro tercero.

**¿Los datos se cifran en tránsito?** → **Sí.** No hace falta suponerlo: ni la
app ni ninguna librería (comprobado en el manifiesto de release fusionado)
declaran `usesCleartextTraffic` ni un `networkSecurityConfig` propio. Con
`targetSdk 36`, Android **bloquea por defecto cualquier tráfico HTTP sin
cifrar** (desde API 28); cualquier conexión de la app o de sus librerías tiene
que ser HTTPS/TLS o el sistema la rechaza antes de que salga del celular.

Puedes comprobarlo tú mismo:

```powershell
.\gradlew.bat :app:processReleaseManifestForPackage
Select-String "uses-permission" app\build\intermediates\merged_manifest\release\*\AndroidManifest.xml
```

---

### ID de publicidad

En algún punto del formulario, Play pregunta si la app usa un ID de
publicidad (`com.google.android.gms.permission.AD_ID`). Respuesta: **No**.
Verificado en el manifiesto de release fusionado — solo aparecen
`USE_BIOMETRIC`, `DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`,
`ACCESS_NETWORK_STATE` e `INTERNET`. Ninguna dependencia del proyecto es un
SDK de publicidad; el permiso `AD_ID` no aparece en ningún sitio:

```powershell
.\gradlew.bat :app:processReleaseManifestForPackage
Select-String "AD_ID" appuild\intermediates\merged_manifest
elease\*\AndroidManifest.xml
```

Si algún día añades un SDK que sí lo traiga, este comando vuelve a decírtelo.

---

## 9. Detalles técnicos (para consultar, no para memorizar)

- `compileSdk` **37** · `targetSdk` **36** · `minSdk` **28** (Android 9 en
  adelante).
- AGP **9.4.0**, Gradle **9.7.1**, Java **17**. Kotlin va integrado en AGP:
  aplicar `org.jetbrains.kotlin.android` da error.
- El release usa **R8** (`isMinifyEnabled`) y `shrinkResources`.
- Motor de PDF: `androidx.pdf`, licencia Apache 2.0, **sin librerías nativas
  propias** (ver [`docs/ARQUITECTURA.md`](docs/ARQUITECTURA.md) §2).
- El paquete incluye dos `.so` que vienen de AndroidX
  (`libandroidx.graphics.path.so` y `libdatastore_shared_counter.so`), **ambas
  alineadas a 16 KB**, que es lo que exige Play; verificado leyendo las cabeceras
  del ELF (`p_align = 0x4000`).
- La app de desarrollo se instala como `com.giosoft.pdf.debug`, para que convivan
  en el mismo celular con la de Play. **Ese sufijo `.debug` no llega a Play**: el
  `.aab` de release se publica como `com.giosoft.pdf`.
- Idiomas incluidos: español, inglés, francés y portugués.
- El diálogo «Acerca de» ya **no** incluye el número de cuenta para donaciones que
  tenía la v4.x. No lo vuelvas a poner sin revisar la política de pagos de Play:
  pedir donaciones fuera de su sistema de facturación está restringido, y la
  excepción suele ser para fundaciones registradas, no para personas.
- Si Play sube el `targetSdk` mínimo exigido (suele anunciarlo cada agosto),
  comprueba el nivel vigente en Play Console antes de publicar.

---

## 10. Resumen en cinco líneas

1. Genera la llave en `Documents\Trabajo\Giosoft\llaves\<app>\` y **guárdala en
   dos sitios, con la contraseña en un gestor**.
2. `copy keystore.properties.template keystore.properties` y rellénalo.
3. `.\gradlew.bat bundleRelease` y comprueba que quedó firmado.
4. Sube la carpeta `web/` a Cloudflare Pages y quédate con la URL de
   `privacidad.html`.
5. Sube el `.aab` a prueba interna, pruébalo de verdad en el celular, y publica.
