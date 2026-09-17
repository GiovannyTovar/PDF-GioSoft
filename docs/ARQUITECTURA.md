# PDF GioSoft — cómo está hecha

Guía para quien tenga que tocar este código dentro de seis meses, incluido su
autor. Explica **qué** se usa, **por qué** se eligió y **dónde** está cada cosa.
Lo que aquí se cuenta no siempre se encuentra buscando en internet: buena parte
salió de decompilar librerías, de leer cabeceras ELF y de probar en un teléfono
real.

- Manual para quien usa la app: [`web/ayuda.html`](../web/ayuda.html)
- Resumen corto para asistentes de código: [`CLAUDE.md`](../CLAUDE.md)

---

## 1. En dos minutos

| | |
|---|---|
| Paquete | `com.giosoft.pdf` (**inmutable**: cambiarlo sería otra app en Play) |
| Lenguaje | Kotlin, interfaz en Jetpack Compose |
| Módulos | Uno solo: `:app` |
| Motor de PDF | `androidx.pdf` (Apache 2.0, sin librerías nativas) |
| Base de datos | Room (`lectorpdf.db`), versión de esquema 6 |
| Inyección | A mano, en `LectorPdfApp` (`AppContainer`). Sin Hilt |
| minSdk / targetSdk / compileSdk | 28 / 36 / 37 |
| Java | 17 |
| Idioma del código | Español: comentarios, nombres de recursos y mensajes de commit |

```bash
./gradlew installDebug     # instalar en el celular conectado
./gradlew assembleRelease  # APK de release (R8)
./gradlew bundleRelease    # .aab para Play (necesita keystore.properties)
./gradlew lint test        # análisis y pruebas JVM
```

La variante `debug` instala con el sufijo `.debug` en el identificador, para que
convivan en el mismo teléfono la app de Play y la de desarrollo. **Ese sufijo no
llega a Play**: el `.aab` de release se publica como `com.giosoft.pdf`.

Logs útiles en ejecución:

```bash
adb logcat -s ViewerViewModel:I SafDocuments:I MediaStoreDocuments:I DocumentScanner:E PdfPrinter:E
```

---

## 2. El motor de PDF: por qué no MuPDF

Esta es la decisión que más condiciona el proyecto, y la que más cuesta
encontrar documentada. Casi todo lo que se lee por ahí recomienda MuPDF o
PDFium, y ninguno de los dos servía aquí.

### MuPDF — lo que usaba la versión 4.x

Funcionaba, pero tenía tres problemas que no se podían resolver:

1. **Licencia AGPL v3.** Es contagiosa: publicar una app que enlaza MuPDF
   obliga a liberar la app entera bajo AGPL, o a comprar una licencia comercial
   a Artifex. El README de este proyecto declara MIT; las dos cosas no pueden
   ser ciertas a la vez.
2. **Es una caja negra.** Se distribuye como AAR precompilado y su
   `DocumentActivity` no permite cambiar ni el tipo de scroll ni el diálogo de
   contraseña. Toda la interfaz venía impuesta.
3. **Tamaño.** Las librerías nativas de las cuatro arquitecturas hacían un APK
   de 27 MB.

### PDFium (`com.github.mhiew:pdfium-android`) — descartado por Play

El requisito de Google Play de **páginas de memoria de 16 KB** (obligatorio para
apps nuevas desde noviembre de 2025) exige que los `.so` estén alineados a
16 KB. Se comprobó leyendo las cabeceras de programa de los `.so` de
`arm64-v8a` de ese AAR: están alineados a 4 KB. Play rechaza el bundle.

Merece la pena saber comprobarlo antes de adoptar cualquier librería nativa:

```bash
unzip -o libreria.aar -d /tmp/aar
readelf -lW /tmp/aar/jni/arm64-v8a/*.so | grep LOAD   # la alineación debe ser 0x4000
```

### `PdfRenderer` de la plataforma — insuficiente por sí solo

Existe desde API 21 y no tiene licencia ni peso, pero **solo pinta páginas en un
bitmap**. No trae zoom, ni scroll continuo, ni selección de texto, ni búsqueda,
ni enlaces. Habría que escribir el visor entero. Sí se usa, directamente, para
un caso donde basta con un bitmap: las miniaturas (`data/Thumbnails.kt`).

### `androidx.pdf` — lo que se usa

Es el visor oficial de Google para PDF, en beta desde 2025.

- **Apache 2.0**, compatible con el MIT del proyecto.
- **No empaqueta ninguna librería nativa**: delega en el renderizador del
  sistema. El APK bajó de 27 MB a 4,2 MB y el requisito de 16 KB dejó de ser un
  problema, porque no hay `.so` que alinear.
- Trae visor completo en Compose: zoom, scroll, selección, búsqueda, enlaces.
- Parsea los documentos **en un proceso aparte** (ver §4).

Lo que hay que aceptar a cambio:

- **Está en beta**: la API puede cambiar antes de 1.0. Todas las llamadas van
  marcadas con `@OptIn(ExperimentalPdfApi::class)`. Al subir de versión, revisar
  `ViewerScreen` y `ViewerViewModel` antes que nada.
- **No sabe cifrar**, solo leer documentos cifrados. Para poner contraseña se
  usa PDFBox (§7).

### Resumen

| | Licencia | Nativas | 16 KB | Visor | Contraseña |
|---|---|---|---|---|---|
| MuPDF | AGPL v3 ❌ | sí (27 MB) | sí | completo, no personalizable | sí |
| PDFium (mhiew) | Apache 2.0 | sí, **4 KB** ❌ | **no** | hay que escribirlo | sí |
| `PdfRenderer` | plataforma | no | sí | **no hay** ❌ | API 35+ |
| **`androidx.pdf`** | **Apache 2.0** | **no** | **sí** | **completo** | según versión (§5) |

---

## 3. Mapa del código

```
com.giosoft.pdf/
├── LectorPdfApp.kt          Application + AppContainer (las dependencias, a mano)
├── MainActivity.kt          Única Activity. Recibe los intents VIEW y SEND
├── data/
│   ├── SafDocuments.kt      SAF: permisos persistentes, nombre, tamaño, renombrar, borrar
│   ├── MediaStoreDocuments.kt  Renombrado alternativo por MediaStore (§6)
│   ├── PublicDocuments.kt   Escribe en Documentos/Mis PDF y calcula el SHA-256
│   ├── DocumentRepository.kt   Único punto de acceso al historial
│   ├── CategoryRepository.kt   Categorías y su paleta
│   ├── SettingsRepository.kt   Preferencias (DataStore): tema, orden, miniaturas
│   ├── PdfEncryption.kt     Pone y quita la contraseña del archivo (PDFBox)
│   ├── Thumbnails.kt        Miniatura de la primera página (PdfRenderer del sistema)
│   └── db/                  Room: entidades, DAO, migraciones
├── security/DeviceLock.kt   Huella, rostro o PIN, con la API de la plataforma
├── scan/DocumentScanner.kt  ML Kit: devuelve el PDF ya armado
├── print/PdfPrinter.kt      Vuelca el archivo al servicio de impresión
└── ui/
    ├── AppNavigation.kt     Navigation Compose: lista ⇄ visor
    ├── library/             Lista, búsqueda, categorías, renombrar, favoritos, menús
    ├── viewer/              Visor, contraseña, última página leída
    ├── settings/ about/ scan/ components/ theme/
```

Reglas que conviene respetar:

- **La interfaz nunca toca SAF ni MediaStore directamente.** Todo pasa por
  `DocumentRepository`; los objetos `SafDocuments`, `MediaStoreDocuments` y
  `PublicDocuments` son sus herramientas.
- **Los `object` de `data/` no guardan estado**: reciben `Context` y devuelven
  `Result` o tipos sellados. Eso los hace fáciles de leer y de probar.
- **Cada texto visible va en `strings.xml`**, en español. No hay cadenas
  literales en los componibles.

---

## 4. Cómo se abre un documento

1. `MainActivity` recibe el intent (`VIEW`, `SEND`) o el usuario elige un
   archivo con el selector del sistema.
2. `SafDocuments.takePersistablePermission()` intenta quedarse el permiso
   duradero. Devuelve `false` si el proveedor no lo admite (típico de WhatsApp
   y del correo: su `FileProvider` no lo permite, y no es un fallo del código).
3. `DocumentRepository.registerOpened(uri, persistable)` registra la ficha en el
   historial. **Puede devolver una URI distinta de la recibida** (§6); los
   llamantes tienen que navegar a `entity.uri`, nunca a la original.
4. `ViewerViewModel` llama a `PdfLoader.openDocument(uri, password)` y recibe un
   `PdfDocument` **ya abierto**, que entrega al componible `PdfViewer`.
5. Al cerrar, se guarda la última página leída.

El cargador es `SandboxedPdfLoader`: el parseo ocurre en **otro proceso**. Un
PDF malformado puede tumbar ese proceso sin afectar a la app. A cambio, los
errores llegan como excepciones a través de IPC, y por eso `proguard-rules.pro`
conserva `PdfPasswordException`: si R8 la fusionara con otra `SecurityException`,
el visor daría el mensaje equivocado (§5).

---

## 5. PDF con contraseña: un límite real del sistema

Como `androidx.pdf` delega en el renderizador del sistema, **si el dispositivo
puede o no abrir un PDF cifrado no depende de la app**. Decompilando
`PdfDocumentRendererFactoryImpl` se ve qué implementación elige:

| Dispositivo | Implementación | ¿Contraseña? |
|---|---|---|
| Android 15 (API 35) o superior | `PdfDocumentRendererAdapter(pfd, password)` | Sí |
| Android 12–14 con **SDK Extension ≥ 13** | `PdfDocumentRendererPreVAdapter(pfd, password)` | Sí |
| El resto | `PdfRendererCompatAdapter(pfd)` — el constructor ni recibe contraseña | **No** |

`ViewerViewModel.supportsPasswordProtectedPdf()` consulta la extensión y, cuando
falta, muestra un mensaje honesto («este celular no puede abrir PDF protegidos»)
en vez de un error genérico.

El diálogo de contraseña es de la app, no de la librería: como somos nosotros
quienes llamamos a `openDocument()`, capturamos `PdfPasswordException` y
controlamos todo el flujo (reintentos, mensajes, y marcar `hasPassword` en el
historial para pintar el candado en la lista).

---

## 6. La identidad de un documento es su URI

En la versión 4.x cada PDF se copiaba a `getExternalFilesDir()` y se guardaba la
ruta de la copia. De ahí salían dos fallos: renombrar cambiaba solo la copia, y
dos PDF distintos con el mismo nombre se pisaban, de modo que el segundo no
abría nunca.

Ahora **la clave primaria de `DocumentEntity` es la URI de SAF**. El historial
apunta al archivo real del usuario.

### Las dos únicas copias

1. **Escaneos**: el escáner produce un archivo nuevo, que se guarda en
   `Documentos/Mis PDF`.
2. **`DocumentRepository.preserveTemporary()`**: los documentos que llegan con
   URI temporal (WhatsApp, correo) se copian a `Documentos/Mis PDF`
   automáticamente. Si no se hiciera, la entrada del historial dejaría de abrir
   en cuanto caducase el permiso.

Antes de copiar, `preserveTemporary()` compara el **SHA-256 del contenido**
(`PublicDocuments.contentHash`) con lo ya conservado. Si el mismo PDF se abre
diez veces desde WhatsApp, se reutiliza la copia en lugar de acumular
`factura(1).pdf`, `factura(2).pdf`… El hash identifica el documento con
independencia del nombre y de la URI.

Si la copia falla (sin espacio, por ejemplo), la entrada se registra con
`persistable = false`; entonces la tarjeta muestra un aviso y ofrece «Guardar en
Mis PDF» en su menú para reintentarlo a mano.

### Renombrar: dos caminos, y el segundo casi nadie lo cuenta

El renombrado normal es `DocumentsContract.renameDocument()`, pero **solo
funciona si el proveedor declara `FLAG_SUPPORTS_RENAME`**, y hay un caso muy
común en el que no lo declara:

- Si el usuario elige el PDF navegando por «Almacenamiento interno», la URI
  viene de `com.android.externalstorage.documents`, que **sí** admite renombrar.
- Si lo elige desde «Recientes» o desde la lista por tipo de archivo, viene de
  `com.android.providers.media.documents` (`MediaDocumentsProvider`), que **no
  implementa renombrar**. Medido en un Tecno con Android 12, sus flags son
  `0x10004`: está `FLAG_SUPPORTS_DELETE` (`0x4`) pero no `FLAG_SUPPORTS_RENAME`
  (`0x40`). De ahí que borrar funcionara y renombrar no.

Es decir: que un documento se pudiera renombrar dependía de por dónde hubiera
entrado en la app, cosa que el usuario no tiene por qué saber.

La salida está en que esos documentos **son archivos indexados por MediaStore**
y su identificador viaja dentro de la propia URI
(`document:1000105447`, `msf:1000108700`). `MediaStoreDocuments` lo traduce con
`MediaStore.getMediaUri()` —con parseo manual del identificador como reserva— y
renombra con un `update` sobre `MediaStore.MediaColumns.DISPLAY_NAME`.

Como el archivo es del usuario y no de la app, Android puede exigir su
consentimiento. El flujo lo contempla entero:

```
DocumentRepository.rename()
  ├─ SafDocuments.canRename() == true  → DocumentsContract.renameDocument()
  └─ si no                             → MediaStoreDocuments.rename()
        ├─ update directo funciona          → RenameResult.Renombrado
        ├─ SecurityException                → createWriteRequest() →
        │     RenameResult.PermisoRequerido(intentSender)
        │       → LibraryScreen lanza el diálogo del sistema
        │       → LibraryViewModel.resumeRename(true) reintenta
        └─ otro error (nombre repetido…)    → RenameResult.Fallo
```

`renameSupport()` devuelve `DIRECTO`, `CON_PERMISO` o `NO_DISPONIBLE`, y el
diálogo avisa **antes** de que Android pregunte, para que esa petición no
aparezca de la nada.

Detalle que ahorra un bug: MediaStore conserva el identificador del archivo, así
que **la URI no cambia** al renombrar por esta vía. `DocumentsContract`, en
cambio, puede devolver una URI nueva; como la URI es la clave primaria,
`applyName()` reinserta la fila con la clave nueva y borra la antigua. Romper
esto deja el historial apuntando a URIs muertas.

### Quitar de la lista ≠ eliminar del celular

Dos operaciones deliberadamente separadas:

- `removeFromHistory()` borra la fila y suelta el permiso. **No toca el
  archivo.** Ofrece deshacer.
- `deleteFromDevice()` borra el archivo de verdad. Es la **única** operación de
  la app que destruye algo del usuario: va en rojo, tras un divisor, y pide
  confirmación nombrando el archivo.

`deleteFromDevice()` solo borra la fila **si el archivo se borró de verdad**:
perder la entrada de un archivo que sigue existiendo sería peor que no hacer
nada. `canDelete()` comprueba antes si el proveedor lo permite, para poder
explicarlo en lugar de fallar.

---

## 7. Las dos protecciones, que no son lo mismo

Es la confusión más fácil de esta app, y por eso la interfaz la explica antes de
activarla (`ui/library/LockInfoDialog.kt`).

| | Proteger con huella | Poner contraseña al archivo |
|---|---|---|
| Dónde vive | En esta app y en este celular (`isLocked` en Room) | **Dentro del archivo PDF** |
| Qué implementa | `security/DeviceLock` → `BiometricPrompt` de la plataforma | `data/PdfEncryption` → PDFBox, AES 128 |
| Si compartes el archivo | Se abre sin pedir nada | Piden la contraseña en cualquier app |
| Si desinstalas | Se pierde la protección | Sigue ahí |
| Efecto secundario | No se genera miniatura | El visor pedirá la contraseña (§5) |

El diálogo de activación (`LockInfoDialog`) muestra primero lo que la protección
sí hace, y deja sus dos límites —no cifra el archivo y no viaja con él— detrás
de «Ver más», además de en *Acerca de → Privacidad*. Es deliberado: quien
protege sus documentos puede leerlos cuando quiera, pero a quien coja un celular
ajeno no se le regala en pantalla hasta dónde llega la protección que acaba de
encontrarse.

`DeviceLock` usa la API de la plataforma en vez de `androidx.biometric` porque
la versión estable de esa librería es de 2021 y obliga a que la Activity sea una
`FragmentActivity`. **La app nunca ve la huella ni el PIN**: solo recibe un sí o
un no del sistema.

Se pide desbloquear tanto para **poner** como para **quitar** la protección: si
solo se pidiera al quitarla, cualquiera con el teléfono desbloqueado podría
protegerte documentos, o retirarte la protección, sin demostrar que eres tú.

PDFBox está aquí únicamente porque `androidx.pdf` sabe leer documentos cifrados
pero no crearlos. Trabaja en disco a partir de 8 MB para no agotar la memoria.

---

## 8. Base de datos

`lectorpdf.db`, esquema en la versión **6**, con migraciones escritas a mano de
la 1 a la 6 (`AppDatabase.kt`). Nunca se usa `fallbackToDestructiveMigration`:
perder el historial del usuario no es una opción aceptable.

`DocumentEntity`: `uri` (clave primaria), `name`, `lastOpened`, `lastPage`,
`pageCount`, `sizeBytes`, `isFavorite`, `persistable`, `location`,
`contentHash`, `categoryId`, `isLocked`, `hasPassword`.

**Al añadir un campo** hay que hacer tres cosas, y olvidar la tercera ya costó
un bug real (las categorías se borraban al reabrir un documento):

1. Añadirlo a `DocumentEntity` con valor por defecto.
2. Subir la versión y escribir la `Migration` correspondiente.
3. Comprobar que `DocumentRepository.register()` lo conserva. Ese método parte
   de la fila existente (`base.copy(...)`) y solo pisa lo que cambia al abrir:
   nombre, fecha, tamaño, ubicación, huella y `persistable`. Si alguna vez
   vuelve a construirse un `DocumentEntity(...)` campo a campo, cualquier dato
   nuevo se perderá en silencio en cada apertura.

Las reglas de R8 conservan `data/db/**` porque los nombres de los campos son los
nombres de las columnas.

---

## 9. Interfaz

- Una sola Activity (`MainActivity`), navegación con Navigation Compose entre la
  biblioteca y el visor.
- `LibraryViewModel` expone un `StateFlow<LibraryUiState>` y un `Channel` de
  `UiMessage` para los avisos. Las peticiones de permiso del sistema
  (renombrado por MediaStore, escáner) van por su propio canal de `IntentSender`
  que la pantalla lanza con `StartIntentSenderForResult`.
- Las acciones de cada documento viajan agrupadas en `DocumentActions`, para que
  `DocumentRow` no reciba doce lambdas sueltas.
- Todos los diálogos pasan por `ui/components/AppDialog`, que fija icono,
  redondeo y jerarquía de botones. Antes cada uno se construía por separado y
  acababan ligeramente distintos.
- El menú de cada documento está agrupado de lo cotidiano a lo irreversible:
  uso diario · protección · quitar de la lista · eliminar del celular. **El
  divisor y el color rojo de la última no son decoración**: separan la única
  acción destructiva.

---

## 9 bis. Idiomas

La app habla **español (por defecto), inglés, francés y portugués**:

```
res/values/strings.xml      es  — el original; aquí se escribe primero
res/values-en/strings.xml   en
res/values-fr/strings.xml   fr  (cubre fr-CA y demás variantes)
res/values-pt/strings.xml   pt  (cubre pt-BR)
```

`androidResources.localeFilters` en `app/build.gradle.kts` limita el empaquetado
a esos cuatro, para no arrastrar los ochenta idiomas de las librerías.

Reglas al tocar textos:

1. **Toda cadena nueva va en los cuatro archivos.** `./gradlew lint` avisa de
   las que falten (`MissingTranslation`), pero es más rápido comprobarlo así:

   ```bash
   grep -c "<string" app/src/main/res/values*/strings.xml
   ```

2. Los marcadores de formato (`%1$s`, `%1$d`) deben aparecer en todas las
   traducciones y con el mismo índice.
3. Usar comillas tipográficas (« », “ ”, ’) en vez de `"` y `'`, que en un
   `strings.xml` hay que escapar y es fuente de errores tontos.
4. El nombre de la app, «PDF GioSoft», no se traduce.

Para comprobar cómo quedó una cadena en cada idioma sin cambiar el idioma del
teléfono:

```bash
aapt2 dump resources app/build/outputs/apk/debug/app-debug.apk | grep -A 4 lock_scope_title
```

---

## 10. Escáner, impresión y el permiso de Internet

`DocumentScanner` usa ML Kit Document Scanner, que corre dentro de los servicios
de Google Play: entrega la interfaz de captura completa y devuelve el PDF ya
armado, así que la app **no necesita permiso de cámara** ni procesa imágenes.

A cambio, ML Kit arrastra `com.google.android.datatransport:transport-backend-cct`,
que **añade `android.permission.INTERNET` al manifiesto final** aunque el
manifiesto del proyecto no lo declare. Conviene comprobarlo antes de afirmar en
público que la app no se conecta a nada:

```bash
./gradlew :app:processReleaseManifestForPackage
grep -o 'uses-permission[^>]*' app/build/intermediates/merged_manifest/release/*/AndroidManifest.xml
grep -B2 -A2 INTERNET app/build/outputs/logs/manifest-merger-release-report.txt
```

La política de privacidad publicada lo dice explícitamente. Si algún día se
quita el escáner, ese permiso desaparece con él.

`PdfPrinter` no renderiza nada: el PDF ya está paginado, así que vuelca el
archivo tal cual al descriptor del servicio de impresión y el documento llega
exactamente como está.

---

## 11. Publicar

- **`versionCode` se edita a mano** en `app/build.gradle.kts`. Play rechaza uno
  repetido.
- La firma se lee de `keystore.properties`, que **no está en git**. Sin ese
  archivo el release compila **sin firmar** y el build no falla, a propósito,
  para que el proyecto se pueda clonar y compilar sin las llaves. El paso a paso
  está en [`PUBLICACION.md`](../PUBLICACION.md).
- La página legal que exige Play está en [`web/`](../web) y se publica aparte
  (Cloudflare Pages). Su URL vive en Play Console, no en el APK: **cambiar de
  dominio no obliga a publicar una versión nueva**.

---

## 11 bis. R8: por qué hay que probar SIEMPRE el paquete de release

`installDebug` **no** pasa por R8. El paquete que va a Play, sí. Son dos binarios
distintos, y hay fallos que solo existen en el segundo.

### Qué hace R8

Al compilar en release, R8 recorta el código: elimina clases y métodos que cree
que nadie usa, y acorta los nombres que quedan. Así el APK pasa de ~90 MB (debug,
sin recortar) a ~11 MB.

El problema es cómo decide qué «nadie usa»: siguiendo las llamadas del código. Lo
que se invoca **por reflexión** —es decir, buscando una clase por su nombre en
tiempo de ejecución— R8 no lo ve, y se lo lleva por delante.

### Lo que pasó en este proyecto (septiembre de 2026)

Síntoma: en el APK de release firmado, tocar «Escanear» cerraba la app al
instante. En debug, perfecto.

La traza no decía nada útil, porque los nombres ya estaban ofuscados:

```
FATAL EXCEPTION: main
java.lang.NullPointerException
    at java.util.Objects.requireNonNull(Objects.java:222)
    at a15.<init>(SourceFile:54)
    at hn4.f(SourceFile:75)
```

La pista estaba unos segundos antes, al arrancar la app:

```
W ComponentDiscovery: Invalid component registrar.
W ComponentDiscovery: Could not instantiate com.google.mlkit.common.internal.CommonComponentRegistrar
    Caused by: java.lang.NoSuchMethodException: ...CommonComponentRegistrar.<init> []
    at com.google.mlkit.common.internal.MlKitInitProvider.onCreate
```

Traducido: ML Kit arranca con un `ContentProvider` que busca sus «registrars» por
nombre y los instancia con `Class.newInstance()`. R8 había eliminado el
constructor vacío de esas clases, porque en el código nadie lo llama de forma
visible. ML Kit se quedaba sin registrar sus componentes, y al pedir el escáner
algo llegaba `null`.

El arreglo son cuatro reglas en `app/proguard-rules.pro`, que conservan el
constructor de cualquier `ComponentRegistrar` y las clases internas de ML Kit.

### Cómo probar el release sin tener la llave de firma

Un APK de release sin firmar no se instala. Para probarlo sin esperar a tener el
`.jks` definitivo, se firma con la llave de depuración, que existe en cualquier
máquina con Android Studio:

```bash
./gradlew assembleRelease

BT="$LOCALAPPDATA/Android/Sdk/build-tools/36.0.0"
"$BT/zipalign.exe" -f -p 4 app/build/outputs/apk/release/app-release-unsigned.apk /tmp/release.apk
"$BT/apksigner.bat" sign --ks "$USERPROFILE/.android/debug.keystore"   --ks-pass pass:android --ks-key-alias androiddebugkey --key-pass pass:android /tmp/release.apk
adb install -r /tmp/release.apk
```

Ese APK se instala como `com.giosoft.pdf` (sin el sufijo `.debug`), así que
**desinstálalo antes de instalar la versión real de Play**: al estar firmado con
otra llave, Android rechazaría la actualización.

### Qué probar en release, siempre

Lo que R8 puede romper es justo lo que depende de librerías externas o reflexión:

| Función | Qué se ejercita |
|---|---|
| Abrir un PDF | `androidx.pdf`, proceso aislado, Room |
| Escanear | ML Kit (el que falló) |
| Imprimir | servicio de impresión del sistema |
| Poner contraseña y reabrir | PDFBox (resuelve filtros y algoritmos por nombre) |
| Renombrar | SAF y MediaStore |
| Eliminar del celular | SAF |

Las seis se comprobaron sobre el release firmado antes de dar por buena esta
versión.

---

## 12. Trampas conocidas

- **AGP 9 lleva Kotlin integrado.** Aplicar `org.jetbrains.kotlin.android` da
  error. En `libs.versions.toml` solo están el plugin de Compose y KSP.
- **`registerOpened()` puede devolver otra URI.** Navegar siempre a
  `entity.uri`.
- **`androidx.pdf` está en beta.** Cada subida de versión obliga a revisar el
  visor.
- **No quitar el `@OptIn(ExperimentalPdfApi::class)`** sin comprobar qué API
  dejó de ser experimental.
- **Los logs de nivel `Log.d` no se ven en algunos dispositivos** (varios
  fabricantes los filtran). Para diagnóstico en campo, `Log.i`.
- **La copia de seguridad de Android está activada** (`allowBackup="true"` con
  reglas vacías): la base de datos del historial entra en la copia cifrada del
  usuario. Es intencional, y la política de privacidad lo declara; si se quiere
  excluir, hay que escribirlo en `backup_rules.xml` y
  `data_extraction_rules.xml`.
- **Nunca des por buena una versión que solo probaste con `installDebug`.** R8
  rompió el escáner en release y en debug no se veía: la historia completa, con
  el diagnóstico y cómo probar el release sin llave de firma, está en la §11 bis.
- **La URL del sitio web que abre «Acerca de» no es la del sitio.**
  `about_website_url` apunta a una dirección propia que redirige
  (`web/_redirects`), porque lo que se empaqueta en un APK ya no se puede
  cambiar sin publicar otra versión. Para mudar el sitio se edita la
  redirección, nunca la cadena.
- **La versión anterior (Java + MuPDF, 4.1.0)** está preservada en el tag
  `v4.1.0-java` y la rama `legado-java-v4.1.0`. No borrarlos.
