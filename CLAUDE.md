# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

"PDF GioSoft" (`com.giosoft.pdf`): visor de PDF para Android sin publicidad, con historial, escáner de documentos e impresión. **Kotlin + Jetpack Compose**, un solo módulo `:app`.

El código, los comentarios y los mensajes de commit están en **español**. Mantener ese idioma.

Los textos de la interfaz se escriben primero en `values/strings.xml` (español) y **toda cadena nueva hay que añadirla también** a `values-en`, `values-fr` y `values-pt`.

Guía técnica completa (por qué no MuPDF, cómo funciona cada pieza, recetas): [`docs/ARQUITECTURA.md`](docs/ARQUITECTURA.md).

**Nombre visible: «PDF GioSoft»** (string `app_name`). El `applicationId` sigue siendo `com.giosoft.pdf` y **no se puede cambiar nunca** una vez publicada en Play: sería otra app distinta y los usuarios perderían las actualizaciones. No intentar "corregirlo" para que coincida con el nombre.

**Mensajes de commit: máximo 100 caracteres**, una sola línea.

La versión anterior (Java + MuPDF, v4.1.0) está preservada en el tag `v4.1.0-java` y la rama `legado-java-v4.1.0`. No borrarlos.

## Comandos

```bash
./gradlew assembleDebug          # APK debug (applicationId acaba en .debug)
./gradlew installDebug           # Instalar en dispositivo conectado
./gradlew assembleRelease        # APK release con R8
./gradlew bundleRelease          # .aab firmado para Play (requiere keystore.properties)
./gradlew lint                   # Android Lint
./gradlew test                   # Tests unitarios JVM
```

Logs en ejecución: los tags son `ViewerViewModel`, `SafDocuments`, `DocumentScanner`, `PdfPrinter`.

```bash
adb logcat -s ViewerViewModel:D SafDocuments:D DocumentScanner:E PdfPrinter:E
```

## Toolchain

| Pieza | Versión | Nota |
|---|---|---|
| AGP | 9.4.0 | **Kotlin va integrado**: aplicar `org.jetbrains.kotlin.android` da error |
| Gradle | 9.7.1 | |
| compileSdk | 37 | Lo exigen las AndroidX recientes; AGP lo descarga solo |
| targetSdk | 36 | Lo que pide Play; verificar el mínimo vigente antes de publicar |
| minSdk | 28 | Android 9. Piso real de `androidx.pdf` (su AAR declara minSdk 28) |
| Java | 17 | |

Build en KTS (`.gradle.kts`) con version catalog en `gradle/libs.versions.toml`.

## Decisiones de arquitectura que no son obvias

### El motor de PDF es `androidx.pdf`, y la elección importa

Se descartaron dos alternativas por motivos concretos:

- **MuPDF** (lo que usaba la v4.x): licencia **AGPL v3**, incompatible con el MIT que declara el README. Además es un AAR precompilado: su `DocumentActivity` es una caja negra y no permite cambiar ni el scroll ni el diálogo de contraseña.
- **`com.github.mhiew:pdfium-android`**: sus `.so` de `arm64-v8a` están alineadas a 4 KB y **fallan el requisito de 16 KB** de Play (verificado leyendo las cabeceras ELF).

`androidx.pdf` es Apache 2.0, **no empaqueta ninguna librería nativa** (por eso el APK bajó de 27 MB a 4,2 MB) y usa el renderizador del sistema.

### La app no copia los PDF que el usuario ya tiene guardados

La v4.x copiaba cada PDF a `getExternalFilesDir()` y guardaba la ruta de la copia. De ahí venían el bug de renombrado y un bug silencioso: dos PDFs distintos con el mismo nombre hacían que el segundo nunca se abriera.

Ahora **la clave de identidad de un documento es su URI de SAF** (`DocumentEntity.uri`, clave primaria en Room). `SafDocuments.rename()` llama a `DocumentsContract.renameDocument()` sobre el archivo real.

Hay exactamente **dos** sitios donde se copia, ambos deliberados y visibles para el usuario:

1. **Escaneos**: el escáner produce un archivo nuevo que aún no tiene sitio.
2. **`DocumentRepository.preserveTemporary()`**: los documentos que llegan con URI temporal (WhatsApp, correo) se copian a `Documentos/Mis PDF` **automáticamente**, porque si no la entrada del historial dejaría de abrir al caducar el permiso.

Antes de copiar, `preserveTemporary()` compara el **SHA-256 del contenido** (`PublicDocuments.contentHash`) contra lo ya conservado. Si el usuario reabre el mismo PDF desde WhatsApp diez veces, se reutiliza la copia existente en lugar de acumular `factura(1).pdf`, `factura(2).pdf`… El hash identifica el documento con independencia del nombre y de la URI.

**`registerOpened()` puede devolver una URI distinta de la que recibió.** Los llamantes (`MainActivity`, `LibraryScreen`) deben navegar a `entity.uri`, nunca a la URI original: si no, el visor abriría la temporal mientras el historial apunta a la copia.

### Dos operaciones distintas, deliberadamente separadas

- **`removeFromHistory()`** quita la fila de la lista y suelta el permiso. **No toca el archivo.** Ofrece deshacer.
- **`deleteFromDevice()`** borra el archivo de verdad. Es la **única** operación de la app que destruye algo del usuario.

La separación es intencional: en la v4.x el diálogo prometía "esto no borra el documento del almacenamiento" mientras el código llamaba a `File.delete()`. En el menú van separadas por un divisor y la destructiva va en rojo, con confirmación que nombra el archivo.

`deleteFromDevice()` **solo borra el historial si el archivo se borró de verdad**: perder la entrada de un archivo que sigue existiendo sería peor que no hacer nada. `canDelete()` comprueba antes si el proveedor lo permite, para explicarlo en vez de dejar fallar la acción.

### Renombrar puede cambiar la URI

Algunos proveedores emiten una URI nueva tras renombrar. Como la URI es la clave primaria, `DocumentRepository.rename()` reinserta la fila con la clave nueva y borra la antigua. Romper esto deja el historial apuntando a URIs muertas.

### PDF con contraseña: hay un límite real del sistema

`androidx.pdf` elige renderizador según la versión de Android (decompilando `PdfDocumentRendererFactoryImpl`):

- API 35+ → `PdfDocumentRendererAdapter(pfd, password)` ✅
- Android 12 + **SDK Extension ≥ 13** → `PdfDocumentRendererPreVAdapter(pfd, password)` ✅
- Por debajo → `PdfRendererCompatAdapter(pfd)`, **cuyo constructor ni recibe contraseña** ❌

`ViewerViewModel.supportsPasswordProtectedPdf()` comprueba la extensión para dar un mensaje honesto en lugar de un error genérico.

### El diálogo de contraseña es nuestro porque cargamos el documento nosotros

El componible `PdfViewer` recibe un `PdfDocument` **ya abierto**. La app llama a `PdfLoader.openDocument(uri, password)` y captura `PdfPasswordException`, así que el flujo de contraseña es enteramente de la app. Por eso `proguard-rules.pro` conserva `PdfPasswordException`: el visor distingue por tipo de excepción, y fusionarla con otra `SecurityException` daría el mensaje equivocado.

### URIs no persistibles

Un PDF que llega compartido desde WhatsApp o Gmail trae una URI de un FileProvider ajeno, que **no se puede persistir**: no es una limitación del código, es cómo esas apps exponen sus archivos. Por eso `preserveTemporary()` los conserva automáticamente (ver arriba).

Si la copia falla (sin espacio, por ejemplo), se registra la URI temporal con `persistable = false`; entonces `DocumentRow` muestra un aviso y ofrece "Guardar en Mis PDF" en su menú, para que el usuario pueda reintentarlo a mano.

### Proceso aislado

`SandboxedPdfLoader` parsea los PDF en un proceso separado: un documento malformado no puede afectar al proceso de la app.

## Estructura

```
com.giosoft.pdf/
├── LectorPdfApp.kt          Application + AppContainer (DI a mano, sin Hilt)
├── MainActivity.kt          Única Activity; recibe los intents VIEW/SEND
├── data/
│   ├── SafDocuments.kt      SAF: permisos, nombre, tamaño, renombrar
│   ├── DocumentRepository.kt
│   └── db/                  Room
├── scan/DocumentScanner.kt  ML Kit: devuelve el PDF ya armado
├── print/PdfPrinter.kt      Vuelca el archivo tal cual a la impresora
└── ui/
    ├── AppNavigation.kt
    ├── library/             Lista, renombrar, favoritos, buscar
    ├── viewer/              Visor, contraseña, última página
    ├── about/
    └── theme/
```

## Avisos

- **`versionCode` se edita a mano** en `app/build.gradle.kts`; Play rechaza repetidos.
- La firma se lee de `keystore.properties` (ignorado por git). Sin ese archivo el release compila **sin firmar** y el build no falla, a propósito.
- Las reglas de R8 conservan `data/db/**` porque los nombres de campo son las columnas de Room.
- `androidx.pdf` está en **beta**: su API puede cambiar antes de 1.0. Las llamadas van marcadas con `@OptIn(ExperimentalPdfApi::class)`.
- El diálogo "Acerca de" **ya no incluye el número de cuenta para donaciones** que tenía la v4.x; ver `PUBLICACION.md`.
