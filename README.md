# Mesa de ayuda universitaria · Actividad 7

**Datos asíncronos, estados de carga, accesibilidad y pruebas**

| | |
|---|---|
| **Asignatura** | Aplicaciones Móviles |
| **Docente** | Gabriel Eduardo Morejón López |
| **Grupo** | 8 · «Mesa de ayuda» |
| **Integrantes** | Froilan Steven Gonzalez Game · Colon Gustavo Leon Giler · Victor Jesus Mena Choez |
| **Fecha** | 14 de noviembre de 2026 |
| **Documento asociado** | `Actividad7_Grupo8_Mesa_de_ayuda.docx` (informe técnico de máximo 8 páginas) |

---

## Índice

1. [Qué es este proyecto](#1-qué-es-este-proyecto)
2. [Relación con el informe en Word](#2-relación-con-el-informe-en-word)
3. [Estructura de carpetas](#3-estructura-de-carpetas)
4. [Requisitos](#4-requisitos)
5. [Cómo ejecutar la Parte A (Dart puro)](#5-cómo-ejecutar-la-parte-a-dart-puro)
6. [Cómo ejecutar la Parte B (Flutter)](#6-cómo-ejecutar-la-parte-b-flutter)
7. [Arquitectura](#7-arquitectura)
8. [Decisiones de diseño y su justificación](#8-decisiones-de-diseño-y-su-justificación)
9. [Referencia de la lógica de datos](#9-referencia-de-la-lógica-de-datos)
10. [Referencia de la interfaz](#10-referencia-de-la-interfaz)
11. [Pruebas](#11-pruebas)
12. [Accesibilidad](#12-accesibilidad)
13. [Estado de verificación](#13-estado-de-verificación)
14. [Solución de problemas](#14-solución-de-problemas)
15. [Qué falta para una versión real](#15-qué-falta-para-una-versión-real)
16. [Fuentes](#16-fuentes)

---

## 1. Qué es este proyecto

Es la continuación de la propuesta de la **Actividad 2**: una aplicación de mesa de ayuda universitaria en la que los estudiantes registran solicitudes de soporte y consultan el estado de su atención.

En la Actividad 2 se investigó cómo llevar una app Flutter a Google Play. En esta actividad la pregunta es otra: **cómo se obtienen, se guardan y se muestran los datos, incluso cuando la red falla o responde mal.**

La premisa del trabajo es **justificar las estructuras y métodos de almacenamiento apropiados para el proyecto**, más que escribir la aplicación en sí. Por eso el código es evidencia de que las decisiones se pueden implementar y probar. No es una aplicación terminada: no tiene servidor, base de datos real ni pantalla de «Nueva solicitud».

### Qué demuestra el código

| Requisito de la actividad | Dónde se cumple |
|---|---|
| Repositorio abstracto e implementación simulada con latencia y fallos configurables | `SolicitudRepository`, `SolicitudRepositorySimulado`, `SimulacionConfig` |
| Estados de pantalla (cargando, datos, vacío, fallo) con jerarquía `sealed` y `switch` exhaustivo | `EstadoPantalla<T>` y su uso en `SolicitudesPage` |
| Manejo de errores de red, tiempos de espera y respuestas mal formadas sin que la app se caiga, conservando el último dato válido | `SolicitudesController.cargar()` |
| `FutureBuilder` y `StreamBuilder` correctos (Future y Stream creados fuera de `build`) | `SolicitudesPage` y `DetallePage` (`initState`) |
| Auditoría de accesibilidad con criterios verificables y correcciones | Sección 12 y `flutter_app/test/pantalla_test.dart` |
| Pruebas unitarias de la lógica en Dart puro | `dart_logica/test/solicitudes_test.dart` |
| Pruebas de widgets | `flutter_app/test/pantalla_test.dart` |

---

## 2. Relación con el informe en Word

El código corresponde a los listados del informe:

| Listado del informe | Archivo |
|---|---|
| Listado 1: modelo, fallos, interfaz y repositorio simulado | `dart_logica/lib/solicitudes.dart` |
| Listado 2: estados y controlador | `dart_logica/lib/solicitudes.dart` |
| Listado 3: pantallas con `StreamBuilder` y `FutureBuilder` | `flutter_app/lib/pantalla.dart` |
| Listado 4: pruebas unitarias | `dart_logica/test/solicitudes_test.dart` |
| Tabla 4: auditoría de accesibilidad | `flutter_app/lib/pantalla.dart` (correcciones) y `flutter_app/test/pantalla_test.dart` (verificación) |
| Tabla 5: pruebas de widgets | `flutter_app/test/pantalla_test.dart` |

**Regla de oro:** si cambian el código, actualicen el listado correspondiente del Word para que ambos coincidan.

### Código que está en el proyecto pero no en el informe

El informe muestra los listados como fragmentos. Estas piezas existen en el código y el Word no las incluye:

- `RepoScope` (un `InheritedWidget`): entrega el repositorio a las rutas hijas para que `ListaSolicitudes(datos)` pueda abrir el detalle sin cambiar su firma.
- `main.dart`: arranque de la app con el repositorio simulado.
- Las vistas auxiliares `VistaCarga`, `VistaVacia`, `VistaFallo`, `ListaSolicitudes`, `DetalleSolicitud` y `EstadoEtiqueta`.

---

## 3. Estructura de carpetas

```
proyecto_actividad7/
├── README.md
├── dart_logica/                     ← PARTE A: Dart puro (sin Flutter)
│   ├── pubspec.yaml                 (paquete: mesa_logica; usa package:test)
│   ├── lib/
│   │   └── solicitudes.dart         (Listados 1 y 2)
│   └── test/
│       └── solicitudes_test.dart    (Listado 4)
└── flutter_app/                     ← PARTE B: pantalla y pruebas de widgets
    ├── pubspec.yaml                 (paquete: mesa_ayuda)
    ├── lib/
    │   ├── main.dart                (arranque de la app)
    │   ├── pantalla.dart            (Listado 3 + vistas)
    │   └── solicitudes.dart         (COPIA de dart_logica/lib/solicitudes.dart)
    └── test/
        └── pantalla_test.dart       (pruebas de widgets y accesibilidad)
```

> **Importante:** `flutter_app/lib/solicitudes.dart` es una **copia idéntica** del archivo de `dart_logica`. Si modifican uno, copien el cambio al otro.

> **Importante:** `flutter_app/` **no es un proyecto Flutter completo**. Le faltan las carpetas `android/`, `ios/`, `web/`, etc. Para ejecutarlo hay que crear un proyecto con `flutter create` y copiar encima `lib/`, `test/` y `pubspec.yaml` (ver [sección 6](#6-cómo-ejecutar-la-parte-b-flutter)).

### Por qué hay dos carpetas

El informe afirma que la lógica está escrita en **Dart puro** y que las pruebas unitarias usan **solo `package:test`**. Separar `dart_logica` hace esa afirmación comprobable: se puede analizar y probar con `dart analyze` y `dart test`, sin Flutter. Solo la pantalla (`flutter_app`) depende de los widgets.

---

## 4. Requisitos

| Herramienta | Para qué | Comprobación |
|---|---|---|
| **Dart SDK 3.3 o superior** | Parte A | `dart --version` |
| **Flutter SDK** (incluye Dart) | Parte B | `flutter --version` y `flutter doctor` |
| **VS Code** (recomendado) con las extensiones **Dart** y **Flutter** | Editar y ejecutar | Panel de extensiones |

Se necesita Dart 3.3+ porque el código usa características de Dart 3: clases `sealed`, `final class`, `abstract interface class`, expresiones `switch` con patrones y registros (*records*).

### Instalación en Windows

- **Solo Dart (suficiente para la Parte A):** en PowerShell, `winget install Google.DartSDK`. Después, **cierren y vuelvan a abrir VS Code** y comprueben con `dart --version`.
- **Flutter (necesario para la Parte B):** descarguen el SDK desde <https://docs.flutter.dev/get-started/install/windows>, descomprímanlo en una ruta sin espacios (por ejemplo `C:\flutter`), agreguen `C:\flutter\bin` al `Path` de Windows y reabran VS Code. Con Flutter instalado, el comando `dart` también funciona.

Para estas pruebas **no hace falta** Android Studio ni un emulador. Si `flutter doctor` marca problemas con Android, se pueden ignorar.

---

## 5. Cómo ejecutar la Parte A (Dart puro)

Abran una terminal **dentro de `dart_logica`** y ejecuten, en este orden:

```powershell
cd dart_logica
dart pub get
dart analyze
dart test
```

| Comando | Qué hace | Resultado esperado |
|---|---|---|
| `dart pub get` | Descarga el paquete `test` | Termina sin errores |
| `dart analyze` | Revisa el código en busca de errores | `No issues found!` |
| `dart test` | Corre las 6 pruebas unitarias | `All tests passed!` |

> **Atención al directorio.** Si ejecutan `dart analyze` en la carpeta superior (`proyecto_actividad7`), también analizará `flutter_app`, que todavía no es un proyecto Flutter, y aparecerán cientos de falsos errores. Ejecútenlo siempre dentro de `dart_logica`.

---

## 6. Cómo ejecutar la Parte B (Flutter)

### 6.1 Crear el proyecto y copiar los archivos

En una carpeta aparte (por ejemplo `Downloads`, **fuera** de `proyecto_actividad7`):

```powershell
flutter create mesa_ayuda
```

Copien estos tres elementos de `proyecto_actividad7\flutter_app\` a la carpeta nueva `mesa_ayuda\`, aceptando reemplazar lo existente:

- la carpeta `lib`
- la carpeta `test`
- el archivo `pubspec.yaml`

**Borren `mesa_ayuda\test\widget_test.dart`** si quedó. Es la prueba de ejemplo del contador que genera `flutter create` y no corresponde a esta app (causa errores como `uri_does_not_exist` y `creation_with_non_type`).

### 6.2 Analizar y probar

```powershell
cd mesa_ayuda
flutter pub get
flutter analyze
flutter test
```

| Comando | Resultado esperado |
|---|---|
| `flutter pub get` | Descarga las dependencias sin errores |
| `flutter analyze` | Sin errores ni advertencias. Pueden aparecer avisos `info` por `deprecated_member_use` (ver [sección 14](#14-solución-de-problemas)) |
| `flutter test` | `All tests passed!` (7 pruebas: 4 de widgets y 3 de accesibilidad) |

La primera ejecución de `flutter test` puede tardar uno o dos minutos.

### 6.3 Ver la app en funcionamiento (opcional)

```powershell
flutter run -d chrome
```

Elijan Chrome si no tienen emulador. La app arranca con el repositorio simulado configurado con **40 % de probabilidad de fallo de red**, así que algunas veces verán la lista y otras el mensaje de error con el botón «Reintentar».

Para ver cada estado de forma controlada, editen la configuración en `lib/main.dart`:

```dart
// Ver siempre la lista
const SimulacionConfig(probabilidadFallo: 0.0)

// Ver siempre el error de red
const SimulacionConfig(probabilidadFallo: 1.0, fallo: TipoFallo.red)

// Ver siempre el tiempo agotado
const SimulacionConfig(probabilidadFallo: 1.0, fallo: TipoFallo.tiempo)

// Ver siempre «datos incorrectos» (JSON mal formado)
const SimulacionConfig(probabilidadFallo: 1.0, fallo: TipoFallo.malformada)

// Ver el estado vacío
const SimulacionConfig(vacio: true)

// Ver la carga durante más tiempo
const SimulacionConfig(latencia: Duration(seconds: 4))
```

Para probar el **último dato válido** en la app real: con `probabilidadFallo: 0.4`, pulsen «Reintentar» varias veces. Cuando una carga correcta es seguida por un fallo, la lista anterior permanece visible debajo del aviso.

---

## 7. Arquitectura

```
┌──────────────────────────────────────────────────────────┐
│  Interfaz (Flutter)                                      │
│  SolicitudesPage ─ StreamBuilder ─ switch exhaustivo     │
│  DetallePage ─ FutureBuilder + StreamBuilder<String>     │
└───────────────▲──────────────────────────────────────────┘
                │ Stream<EstadoLista>  (estados, no errores)
┌───────────────┴──────────────────────────────────────────┐
│  SolicitudesController (Dart puro)                       │
│  captura fallos · aplica timeout · conserva último dato  │
└───────────────▲──────────────────────────────────────────┘
                │ Future<List<Solicitud>> / lanza FalloDatos
┌───────────────┴──────────────────────────────────────────┐
│  SolicitudRepository  (clase abstracta / interfaz)       │
└───────▲──────────────────────────────▲───────────────────┘
        │ HOY                          │ FUTURO
┌───────┴──────────────┐   ┌───────────┴──────────────────┐
│ SolicitudRepository  │   │ Repositorio real             │
│ Simulado             │   │ API remota + SQLite local    │
│ (latencia y fallos   │   │ (offline-first con respaldo) │
│  configurables)      │   │                              │
└──────────────────────┘   └──────────────────────────────┘
```

Como la interfaz es la misma, **ni el controlador ni la pantalla cambian** al pasar del repositorio simulado al real.

### Flujo de una carga

1. La página llama a `controller.cargar()` desde `initState`.
2. El controlador emite `Cargando(previo: últimoDatoVálido)`.
3. Llama a `repositorio.listar()` con `.timeout(limite)`.
4. Según el resultado emite `ConDatos`, `Vacio` o `Fallo` (con el último dato válido).
5. El `StreamBuilder` recibe el estado y el `switch` exhaustivo decide qué dibujar.

---

## 8. Decisiones de diseño y su justificación

### 8.1 Almacenamiento para el proyecto completo

Una mesa de ayuda maneja datos de naturaleza distinta. Guardarlos todos en el mismo lugar sería un error de diseño. Se clasificaron según quién los modifica, cuánto duran y qué tan sensibles son:

| Dato | Almacenamiento elegido | Por qué |
|---|---|---|
| Solicitudes y respuestas del personal | **Servidor remoto (API)** como fuente de verdad | El personal responde desde otro dispositivo; solo un servidor central garantiza que todos vean el mismo estado. |
| Copia local de las solicitudes del estudiante | **SQLite** (`sqflite`) | Datos relacionales (asunto, estado, fecha) que se filtran por estado; permite mostrar el último dato sin conexión. |
| Preferencias de la interfaz (filtro, orden) | **Clave-valor** (`shared_preferences`) | Pocos valores simples; una base de datos sería excesiva. |
| Sesión del estudiante | **Almacenamiento cifrado del sistema** (Keystore en Android) | Dato sensible: no debe quedar en texto plano ni en la base local. Debe declararse en *Seguridad de los datos* de Google Play (Actividad 2). |
| Adjuntos (capturas de pantalla) | **Archivos en disco**; en SQLite solo la ruta | Evita inflar la base de datos. |

> En este proyecto **solo está implementado el repositorio simulado**. SQLite, `shared_preferences`, el almacenamiento cifrado y los archivos son decisiones **justificadas en el informe**, no código existente.

### 8.2 Patrón repositorio con clase abstracta

La guía de arquitectura de Flutter recomienda usar el patrón repositorio y declarar los repositorios como clases abstractas. Así se aísla la lógica de la aplicación de los mecanismos de almacenamiento y se permiten implementaciones distintas según el entorno (desarrollo, pruebas, producción).

### 8.3 Estados con jerarquía `sealed`

Al declarar `EstadoPantalla<T>` como `sealed`, el compilador conoce todos los subtipos y comprueba que un `switch` los cubra todos. Si se agrega un estado nuevo, cada `switch` que no lo trate **deja de compilar**.

Por eso los `switch` sobre jerarquías `sealed` del proyecto **no usan `default` ni comodines**: un comodín desactivaría justamente esa comprobación. Solo se usa un comodín al decidir sobre un `String` (el nombre del estado de una solicitud), que no es una jerarquía cerrada.

«Vacío» **no es un fallo**: indica que la consulta funcionó y que no hay registros. Distinguirlos evita decirle al estudiante que algo salió mal cuando solo no ha creado solicitudes.

### 8.4 Errores como estados, no como excepciones

El controlador captura todo en un único punto y lo convierte en un estado `Fallo`. La pantalla **nunca recibe una excepción**, y el `StreamBuilder` nunca queda en `hasError`.

Tres decisiones sostienen esto:

1. `_ultimoValido` solo se actualiza cuando la respuesta se analizó correctamente. Un fallo no puede contaminar lo que ya se mostraba.
2. El estado `Cargando` lleva ese dato previo, de modo que al reintentar la lista no desaparece.
3. `Future.timeout` solo deja de esperar la respuesta; **no cancela** la operación subyacente. Una respuesta tardía se ignora y no modifica el estado.

### 8.5 `FutureBuilder` y `StreamBuilder` correctos

La documentación de `FutureBuilder` advierte que, si el `Future` se crea junto con el widget, cada reconstrucción del padre reinicia la tarea asíncrona, y que debe suponerse que `build` puede ejecutarse en cada fotograma. Lo mismo vale para `Stream`.

Por eso el `Future` y el `Stream` **se crean en `initState`**, se guardan en campos y `build` solo los lee. Si hay que repetir una carga, el nuevo `Future` se asigna dentro de `setState` desde el manejador del botón, nunca dentro de `build`.

### 8.6 Controlador con `StreamController` de suscripción única

Los eventos emitidos antes de que el `StreamBuilder` se suscriba quedan en búfer y no se pierden, aunque la latencia sea cero. Esto es lo que permite que `cargar()` se llame en `initState` sin riesgo de perder el estado `Cargando`.

---

## 9. Referencia de la lógica de datos

Archivo: `dart_logica/lib/solicitudes.dart`.

### Modelo

| Elemento | Descripción |
|---|---|
| `Solicitud` | Datos de una solicitud: `id`, `asunto`, `estado` (`abierta`, `en_proceso` o `resuelta`). |
| `Solicitud.fromJson(Object?)` | Valida que el JSON sea un objeto y que cada campo sea `String`. Si no, lanza `FormatException`. Así ningún dato corrupto llega a la interfaz. |

### Fallos (jerarquía cerrada `FalloDatos`)

| Clase | Significado |
|---|---|
| `SinConexion` | No hay red |
| `TiempoAgotado` | El servidor tardó más que el límite |
| `RespuestaInvalida` | El servidor respondió con datos mal formados |
| `FalloInesperado` | Cualquier otro error |

### Repositorio

```dart
abstract interface class SolicitudRepository {
  Future<List<Solicitud>> listar();
  Future<Solicitud> obtener(String id);
  Stream<String> observarEstado(String id);
}
```

### Simulación configurable

`SolicitudRepositorySimulado({SimulacionConfig config, Random? azar})`

| Parámetro de `SimulacionConfig` | Por defecto | Efecto |
|---|---|---|
| `latencia` | 800 ms | Espera antes de cada respuesta |
| `probabilidadFallo` | 0.0 | Probabilidad (0 a 1) de que cada llamada falle |
| `fallo` | `TipoFallo.red` | Tipo de fallo: `red`, `tiempo` o `malformada` |
| `vacio` | `false` | Si es `true`, `listar()` devuelve una lista vacía |

El generador aleatorio se puede inyectar (`azar`). Con probabilidades de `0` o `1` el comportamiento es **totalmente predecible**, lo que hace deterministas las pruebas.

Datos de ejemplo del repositorio simulado:

| id | asunto | estado |
|---|---|---|
| 1 | Sin acceso al Wi-Fi | abierta |
| 2 | Error al matricularme | en_proceso |

`observarEstado(id)` emite en orden `abierta`, `en_proceso` y `resuelta`, esperando la latencia entre cada valor. Puede emitir un error dentro del flujo.

### Estados

| Estado | Contiene | Cuándo |
|---|---|---|
| `Cargando<T>` | `previo` (último dato válido, opcional) | Al abrir o reintentar |
| `ConDatos<T>` | `datos` | Hay al menos una solicitud |
| `Vacio<T>` | nada | Respuesta válida sin solicitudes |
| `Fallo<T>` | `causa` (`FalloDatos`), `ultimoDato` (opcional) | Cualquier fallo |

`typedef EstadoLista = EstadoPantalla<List<Solicitud>>;`

### Controlador

`SolicitudesController(SolicitudRepository repo, {Duration limite = 5 s})`

| Miembro | Descripción |
|---|---|
| `estados` | `Stream<EstadoLista>` con los estados emitidos |
| `cargar()` | Emite `Cargando`, consulta con `timeout` y emite el resultado. **Nunca lanza excepciones.** |
| `dispose()` | Cierra el flujo. Tras esto, `_emitir` ignora nuevos eventos. |

Traducción de errores en `cargar()`:

| Lo que ocurre | Estado emitido |
|---|---|
| `TimeoutException` | `Fallo(TiempoAgotado)` |
| `FalloDatos` (cualquiera) | `Fallo(esa causa)` |
| Cualquier otro error | `Fallo(FalloInesperado)` |

---

## 10. Referencia de la interfaz

Archivo: `flutter_app/lib/pantalla.dart`.

| Widget | Función |
|---|---|
| `SolicitudesPage(repo, onNueva)` | Pantalla «Mis solicitudes». Crea el controlador y el `Stream` en `initState`; dibuja con `StreamBuilder` y `switch` exhaustivo. |
| `DetallePage(repo, id)` | Detalle de una solicitud. `FutureBuilder` para la consulta y `StreamBuilder<String>` interno para el estado de atención. |
| `VistaCarga({previo})` | Indicador de progreso anunciado a lectores de pantalla; si hay lista previa, se mantiene visible debajo. |
| `VistaVacia(onNueva)` | Mensaje explicativo y botón «Nueva solicitud». |
| `VistaFallo(causa, previo, onReintentar)` | Mensaje según la causa, botón «Reintentar» y la lista anterior si existe. |
| `ListaSolicitudes(datos)` | Tarjetas con asunto y estado. Al tocarlas abre el detalle. |
| `DetalleSolicitud(solicitud, estado:)` | Muestra el asunto y el estado de atención en vivo. |
| `EstadoEtiqueta(estado)` | Estado con **icono y texto** (no solo color). |
| `RepoScope` | `InheritedWidget` que entrega el repositorio a las rutas hijas. Debe colocarse sobre el `Navigator` (en `MaterialApp.builder`). |

Mensajes por causa (en `VistaFallo`):

| Causa | Mensaje al estudiante |
|---|---|
| `SinConexion` | «No hay conexión. Revisa tu red e inténtalo de nuevo.» |
| `TiempoAgotado` | «El servidor tardó demasiado en responder.» |
| `RespuestaInvalida` | «Recibimos datos incorrectos del servidor.» |
| `FalloInesperado` | «Ocurrió un error inesperado. Inténtalo de nuevo.» |

---

## 11. Pruebas

### 11.1 Pruebas unitarias (Dart puro) · 6 pruebas

Archivo: `dart_logica/test/solicitudes_test.dart`. Usan solo `package:test`. El repositorio simulado se configura con latencia cero y probabilidades de 0 o 1. Para el caso «éxito y luego fallo» se escribió a mano un repositorio guionado (`_RepoGuionado`), alternativa que la documentación de Flutter acepta frente a generar mocks con Mockito.

| Prueba | Qué comprueba |
|---|---|
| `fromJson` rechaza valores mal formados | Lanza `FormatException` con un número y con un objeto de campos nulos |
| Devuelve datos cuando no hay fallos | La lista tiene 2 solicitudes |
| Respuesta malformada se traduce a `RespuestaInvalida` | El repositorio convierte `FormatException` en `RespuestaInvalida` |
| Éxito: `Cargando` y luego `ConDatos` | Orden de los estados emitidos |
| Un fallo conserva el último dato válido | Tras éxito y luego `SinConexion`, el `Fallo` trae la lista anterior |
| Respuesta más lenta que el límite produce `TiempoAgotado` | `limite` de 20 ms con latencia de 200 ms |

### 11.2 Pruebas de widgets y accesibilidad (Flutter) · 7 pruebas

Archivo: `flutter_app/test/pantalla_test.dart`. Se inyecta siempre el repositorio simulado o uno de prueba, así que no hace falta red.

| Prueba | Qué comprueba |
|---|---|
| Muestra la carga | Un `CircularProgressIndicator` y la etiqueta «Cargando solicitudes» |
| Muestra el estado vacío | Mensaje explicativo y botón «Nueva solicitud»; sin botón «Reintentar» |
| El fallo conserva visible la lista anterior | Aviso de fallo y la lista previa a la vez; el botón «Reintentar» responde |
| No reinicia la carga al reconstruir el padre | Con `setState` en el padre, `listar()` se llama **una sola vez** (el `Future` no se crea en `build`) |
| Accesibilidad · lista con datos | Las cuatro guías, con texto al 200 % y sin desbordes |
| Accesibilidad · estado de fallo con lista previa | Igual |
| Accesibilidad · estado vacío | Igual |

> **Nota sobre la Tabla 5 del informe.** La fila «Fallo con último dato» dice «se pulsa Reintentar tras una primera carga correcta». Pero ese botón solo existe cuando ya hay un fallo. Por eso la prueba de widgets muestra directamente el estado `VistaFallo` con lista previa. El escenario secuencial «éxito y luego fallo» lo cubre la prueba unitaria `un fallo conserva el último dato válido`. Se recomienda ajustar el texto de esa fila del Word.

---

## 12. Accesibilidad

Las correcciones de la Tabla 4 están aplicadas en el código y se verifican con las guías automáticas de `flutter_test` (`meetsGuideline`):

| Criterio | Guía automática | Cómo se cumple en el código |
|---|---|---|
| Área táctil mínima (48×48 dp en Android, 44×44 en iOS) | `androidTapTargetGuideline`, `iOSTapTargetGuideline` | Botones con `minimumSize: Size(48, 48)` y texto visible |
| Contraste de texto de al menos 4.5:1 | `textContrastGuideline` | Texto secundario en gris `#616161` (6.19:1 sobre blanco) |
| Todo control tiene etiqueta | `labeledTapTargetGuideline` | Botones con texto; el botón flotante de `main.dart` con `tooltip: 'Nueva solicitud'` |
| Los cambios de estado se anuncian | Revisión con lector de pantalla | `Semantics(liveRegion: true)` en la carga («Cargando solicitudes») y en el mensaje de fallo |
| No depender solo del color | Revisión visual en escala de grises | `EstadoEtiqueta` combina icono y texto («Abierta», «En proceso», «Resuelta») |
| Texto ampliable al 200 % | `textScaler` en 2.0 sin desbordes | Sin alturas fijas; las tarjetas crecen con el contenido |

Las guías automáticas **no sustituyen** el recorrido manual con TalkBack (lector de pantalla de Android).

### Importante para el informe

La columna **«Hallazgo en la primera versión»** de la Tabla 4 del Word contiene hallazgos **típicos de ejemplo**, no resultados medidos sobre una primera versión real de la pantalla. Hay dos opciones honestas:

1. Redactar que esos hallazgos corresponden a un **diseño inicial de referencia**.
2. Ejecutar un recorrido real (por ejemplo con TalkBack) y reemplazar la columna por los hallazgos reales.

---

## 13. Estado de verificación

Marquen cada casilla cuando lo comprueben y guarden una captura de la terminal.

| Comprobación | Estado |
|---|---|
| `dart analyze` en `dart_logica` sin problemas | ☑ Verificado por el grupo |
| `dart test` en `dart_logica` pasa | ☑ Verificado por el grupo |
| `flutter analyze` sin errores ni advertencias (solo avisos `info`) | ☑ Verificado por el grupo |
| `flutter test` en `mesa_ayuda` pasa | ☐ Pendiente de confirmar |
| `flutter run -d chrome` muestra la app | ☐ Opcional |
| Hallazgos de la Tabla 4 reemplazados o aclarados como diseño inicial | ☐ Pendiente |
| Texto de la fila «Fallo con último dato» (Tabla 5) ajustado | ☐ Pendiente |
| Listados del Word coinciden con los archivos | ☐ Pendiente |
| Informe de 8 páginas o menos en Word | ☐ Pendiente |
| Enlaces de las referencias abiertos y comprobados | ☐ Pendiente |
| Nombres, fecha y docente del encabezado revisados | ☐ Pendiente |

---

## 14. Solución de problemas

| Síntoma | Causa | Solución |
|---|---|---|
| `dart: El término 'dart' no se reconoce...` | Dart no está instalado o no está en el `Path` | Instalar Dart (`winget install Google.DartSDK`) o Flutter, y **reabrir VS Code** |
| `dart analyze` muestra 300 o más problemas | Se ejecutó en `proyecto_actividad7` y analizó `flutter_app` | Entrar con `cd dart_logica` y ejecutarlo ahí. Alternativa: abrir solo la carpeta `dart_logica` en VS Code |
| Punto rojo y «9+» en `solicitudes_test.dart` antes de ejecutar nada | Falta `dart pub get` | Ejecutar `dart pub get` dentro de `dart_logica` |
| `uri_does_not_exist`, `creation_with_non_type` y `undefined_identifier` en `test\widget_test.dart` | Es la prueba de ejemplo que genera `flutter create` | Borrar `mesa_ayuda\test\widget_test.dart` |
| Avisos `info` con `deprecated_member_use` sobre `package:flutter/material.dart` | En la versión de Flutter usada, ese import figura como obsoleto a favor de `package:material_ui/material_ui.dart` | No son errores. El código funciona. Pueden dejarse como están para la entrega |
| `flutter test` falla con «A Timer is still pending» | Una prueba terminó con un temporizador activo | Avisar con el mensaje completo; hay que avanzar el reloj simulado antes de terminar la prueba |
| `Stream has already been listened to` | Se suscribieron dos veces a un `Stream` de suscripción única | Crear el `Stream` una vez en `initState` y no recrear el widget que lo consume |

### Si alguna prueba o guía de accesibilidad falla

- **Guía de accesibilidad** (`Expected: Tappable...`, `meets guideline`): es un **hallazgo real**. Anoten qué guía fue y en qué estado, úsenlo en la Tabla 4 y corrijan el widget.
- **Prueba de widgets o unitaria**: copien el bloque de error completo para poder corregirlo.

---

## 15. Qué falta para una versión real

Fuera del alcance de esta actividad, y mencionado como trabajo siguiente en el informe:

- Implementar el **repositorio real**: API remota más SQLite, con enfoque *offline-first* con respaldo (intentar la red y, si falla, devolver lo guardado).
- Almacenar la sesión en almacenamiento cifrado y declarar el tratamiento de datos en la sección *Seguridad de los datos* de Google Play.
- Construir la pantalla «Nueva solicitud» y el flujo de adjuntos (archivos en disco, ruta en SQLite).
- Ejecutar un recorrido completo con TalkBack y ampliar las pruebas de widgets.

---

## 16. Fuentes

Documentación oficial citada en el informe:

1. Flutter. *Architecture recommendations and resources*. <https://docs.flutter.dev/app-architecture/recommendations>
2. Flutter. *Guide to app architecture*. <https://docs.flutter.dev/app-architecture/guide>
3. Flutter. *Offline-first support*. <https://docs.flutter.dev/app-architecture/design-patterns/offline-first>
4. Flutter. *Persistence cookbook* y *Persist data with SQLite*. <https://docs.flutter.dev/cookbook/persistence> y <https://docs.flutter.dev/cookbook/persistence/sqlite>
5. Dart team. *Sealed types: feature specification*. <https://github.com/dart-lang/language/blob/main/accepted/future-releases/sealed-types/feature-specification.md>
6. Flutter. *FutureBuilder class*. <https://api.flutter.dev/flutter/widgets/FutureBuilder-class.html>
7. Flutter. *An introduction to widget testing*. <https://docs.flutter.dev/cookbook/testing/widget/introduction>
8. Flutter. *Mock dependencies using Mockito*. <https://docs.flutter.dev/cookbook/testing/unit/mocking>
9. Flutter. *Accessibility technologies*. <https://docs.flutter.dev/ui/accessibility/assistive-technologies>
10. Flutter. Guías de accesibilidad de `flutter_test` (contraste de texto y área táctil de 48 y 44 píxeles). Ver las referencias completas en el informe.
