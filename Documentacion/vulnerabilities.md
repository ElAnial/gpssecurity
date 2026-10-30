# Informe de Vulnerabilidades

## 1. Información general

- **Proyecto:** GPSMapApp (SimpleGPSMapApp)
- **Plataforma:** Android (Java), integración con Google Maps
- **Paquete analizado:** `com.example.myapplication`
- **Archivo analizado:** `app-debug.apk` (15.0 MB, versión 1.0, código 1)
- **MD5:** `e30c4522da3ffc45e2f70f896216ba84`
- **SDK:** minSdk 30 / targetSdk 30
- **Herramienta:** MobSF (Mobile Security Framework) v4.6.3, análisis estático y dinámico (Frida)
- **Entorno de pruebas:** emulador Pixel 5 (`sdk_gphone_x86`), Android 11 (API 30), conectado por ADB a MobSF en Docker
- **Fecha de las pruebas:** 28 y 29 de octubre de 2026

## 2. Objetivo

Identificar vulnerabilidades y configuraciones inseguras de la aplicación Android mediante análisis estático y dinámico con MobSF, evaluar su riesgo y proponer correcciones.

## 3. Resumen

| Severidad | Cantidad |
|---|---|
| Crítica | 0 |
| Alta | 2 |
| Media | 3 |
| Baja | 0 |
| Informativa | 5 observaciones (sección 7) |
| **Total de hallazgos confirmados** | **5** |

MobSF asignó una puntuación de **66/100 (LOW RISK, grado A)** al escaneo del 29/10/2026. Su resumen reporta 2 hallazgos altos y 3 medios. Un primer escaneo del 28/10/2026 (MobSF v4.6.2) mostró 45/100 con la misma lista de hallazgos; se usa el segundo como resultado oficial. Esa puntuación no significa que la app sea segura: es un indicador automático y varios análisis quedaron incompletos (ver sección 8).

> **Importante:** el APK analizado es una compilación **debug**. Los hallazgos V-01 y V-02 son esperables en ese tipo de compilación; su riesgo real depende de qué APK se publique. Se recomienda repetir el análisis con el APK **release**.

## 4. Análisis estático

MobSF analizó el APK sin ejecutarlo: permisos, manifiesto, certificado, componentes, secretos y código.

### 4.1 Resultados por categoría

| Vulnerabilidad o hallazgo | Severidad (MobSF) | Evidencia | Medida correctiva |
|---|---|---|---|
| V-01 Aplicación firmada con certificado de depuración (`CN=Android Debug`) | Alta | Certificate Analysis; `descarga__1_.pdf`, sección *Certificate Analysis* | Firmar la versión de producción con un keystore propio |
| V-02 Depuración habilitada (`android:debuggable="true"`) | Alta | Manifest Analysis, hallazgo 1; captura `android_manifest.png`, línea 17 | Compilar en modo release (`debuggable false`) |
| V-03 Copia de seguridad permitida (`android:allowBackup="true"`) | Media (warning) | Manifest Analysis, hallazgo 2; `android_manifest.png`, línea 17 | `allowBackup="false"` o reglas de exclusión |
| V-04 Receiver exportado `androidx.profileinstaller.ProfileInstallReceiver` protegido por `android.permission.DUMP` | Media (warning) | Manifest Analysis, hallazgo 3; `android_manifest.png`, líneas 33-46 | Verificar el nivel de protección del permiso; ver análisis en sección 5 |
| V-05 Clave de API de Google Maps incluida en el APK (`google_maps_key`) | Media (listada en *Possible Hardcoded Secrets*) | Sección *Hardcoded Secrets* de `descarga__1_.pdf` | Restringir la clave por paquete y huella SHA-1; sacarla del repositorio |

### 4.2 Categorías revisadas sin hallazgos

Estos resultados describen únicamente lo que MobSF reportó; no demuestran que no existan problemas.

- **Network Security:** sin entradas (la app no define `networkSecurityConfig`).
- **Code Analysis (SAST):** sin hallazgos, pero ver limitación 8.1: no todos los archivos DEX se descompilaron.
- **Browsable Activities:** ninguna.
- **Shared Library Binary Analysis:** no hay librerías nativas.
- **Firebase Database Analysis:** sin datos.
- **Trackers:** 0 de 432 detectados.
- **Componentes exportados:** 0/2 activities (la `MainActivity` figura como `exported="true"` por ser el launcher), 0/0 services, 1/1 receivers, 0/1 providers.

### 4.3 Permisos solicitados

| Permiso | Nivel | Observación |
|---|---|---|
| `INTERNET` | normal | Necesario para cargar el mapa |
| `ACCESS_FINE_LOCATION` | peligroso | Necesario para mostrar la ubicación del usuario |
| `ACCESS_COARSE_LOCATION` | peligroso | Necesario junto con el anterior |
| `CAMERA` | peligroso | **Verificar** si la app realmente usa la cámara; si no, eliminarlo |
| `ACCESS_NETWORK_STATE` | normal | Uso habitual |
| `DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | desconocido | Permiso propio generado por AndroidX (`signature`) |

## 5. Registro detallado de vulnerabilidades confirmadas

### Vulnerabilidad 1: Aplicación firmada con certificado de depuración

- **Severidad:** Alta (MobSF)
- **Origen:** Análisis estático
- **Descripción:** El APK está firmado con el certificado `CN=Android Debug, O=Android, C=US`. MobSF indica que una aplicación de producción no debe distribuirse con ese certificado.
- **Impacto potencial:** El keystore de depuración lo generan todas las instalaciones de Android Studio, así que cualquiera podría firmar una versión alterada que parezca legítima. Además, Google Play no acepta APK firmados con él.
- **Pasos para reproducir:** Cargar `app-debug.apk` en MobSF → *Static Analyzer* → sección *Certificate Analysis*.
- **Evidencia:** Informe estático de MobSF, *Certificate Analysis* y *Signer Certificate*.
- **Recomendación:** Generar un keystore propio, guardarlo fuera del repositorio y configurarlo como `signingConfig` de la variante release. Ver `best_practices.md`.
- **Estado:** Pendiente

### Vulnerabilidad 2: Depuración habilitada (`android:debuggable="true"`)

- **Severidad:** Alta (MobSF)
- **Origen:** Análisis estático
- **Descripción:** El manifiesto declara `android:debuggable="true"` en la etiqueta `<application>`.
- **Impacto potencial:** Permite acoplar un depurador, volcar trazas y acceder a clases auxiliares de depuración, lo que facilita la ingeniería inversa y la extracción de datos internos.
- **Pasos para reproducir:** MobSF → *Manifest Analysis* → hallazgo 1; o revisar la línea 17 del manifiesto en la captura `android_manifest.png`.
- **Evidencia:** `android_manifest.png` y *Manifest Analysis* del informe estático.
- **Recomendación:** No fijar el atributo a mano: las variantes release lo dejan en `false` por defecto. Publicar solo APK/AAB release.
- **Estado:** Pendiente de verificación (debe comprobarse en el APK release)

### Vulnerabilidad 3: Copia de seguridad permitida (`android:allowBackup="true"`)

- **Severidad:** Media (MobSF: warning)
- **Origen:** Análisis estático
- **Descripción:** La aplicación permite respaldar sus datos mediante ADB. El manifiesto además referencia `@xml/backup_rules`.
- **Impacto potencial:** Quien tenga el dispositivo con la depuración USB activada podría copiar los datos de la app fuera del equipo. El riesgo aumenta si la app llega a guardar datos sensibles localmente.
- **Pasos para reproducir:** MobSF → *Manifest Analysis* → hallazgo 2.
- **Evidencia:** `android_manifest.png`, línea 17.
- **Recomendación:** Establecer `android:allowBackup="false"`, o mantenerlo y definir en `backup_rules` la exclusión de archivos sensibles.
- **Estado:** Pendiente

### Vulnerabilidad 4: Receiver exportado `ProfileInstallReceiver`

- **Severidad:** Media según MobSF (warning). Evaluación propia: **baja**, por lo que se explica abajo.
- **Origen:** Análisis estático
- **Descripción:** `androidx.profileinstaller.ProfileInstallReceiver` está declarado con `android:exported="true"` y protegido por `android.permission.DUMP`. MobSF advierte que ese permiso no está definido en la app y que su nivel de protección debe comprobarse.
- **Impacto potencial:** Si el permiso fuera de nivel `normal` o `dangerous`, otra app podría obtenerlo e interactuar con el receiver. En la documentación de Android `DUMP` es un permiso de nivel `signature|privileged`, por lo que en condiciones normales solo el sistema o el shell pueden enviar esas acciones. Este componente lo agrega la librería AndroidX para instalar perfiles de rendimiento.
- **Pasos para reproducir:** MobSF → *Manifest Analysis* → hallazgo 3; revisar las líneas 33-46 del manifiesto.
- **Evidencia:** `android_manifest.png`.
- **Recomendación:** Mantener la dependencia `androidx.profileinstaller` actualizada. Si no se usan perfiles de línea base, se puede excluir con `tools:node="remove"`. Documentar el riesgo aceptado.
- **Estado:** Pendiente de verificación

### Vulnerabilidad 5: Clave de API de Google Maps incluida en el APK

- **Severidad:** Media (evaluación propia; MobSF la lista sin severidad en *Possible Hardcoded Secrets*)
- **Origen:** Análisis estático
- **Descripción:** MobSF encontró `"google_maps_key"` con un valor que empieza por `AIzaSyCq4J…` y termina en `…ZiNsc` (enmascarado aquí a propósito). El manifiesto la referencia como `@string/google_maps_key` dentro del `meta-data` `com.google.android.geo.API_KEY`.
- **Impacto potencial:** Cualquiera puede extraerla del APK. Si la clave no tiene restricciones, otra persona podría usarla y generar consumo de cuota o costos en la cuenta de Google Cloud. Que la clave figure en el APK es normal en el SDK de Maps; el problema real es que no esté restringida o que se suba en claro a un repositorio público.
- **Pasos para reproducir:** MobSF → *Static Analyzer* → *Possible Hardcoded Secrets*.
- **Evidencia:** Sección *Hardcoded Secrets* del informe estático.
- **Recomendación:** En Google Cloud Console, restringir la clave a *Aplicaciones Android* (paquete `com.example.myapplication` más huella SHA-1) y a la API *Maps SDK for Android*. No dejarla en el repositorio: inyectarla desde `local.properties` o con *Secrets Gradle Plugin*. Si ya se subió a GitHub o el informe MobSF original se publicó, rotarla.
- **Estado:** Pendiente

## 6. Análisis dinámico

MobSF se conectó al emulador Android 11 por ADB, instaló una versión del APK con Frida Gadget y el certificado raíz de MobSF, y arrancó la instrumentación.

### 6.1 Observaciones registradas

- MobSF informó *Environment is ready for Dynamic Analysis*, con *Screen streaming* y *Logcat streaming* iniciados (`analizadordinamico.png`).
- Se cargaron los scripts de Frida `api_monitor`, `ssl_pinning_bypass`, `root_bypass`, `debugger_check_bypass` y `dump_clipboard` (`analizadordinamicofridalog.png`).
- El script de SSL pinning no encontró bibliotecas de pinning comunes (OkHttp `CertificatePinner`, TrustKit, Cronet, etc.) y registró dos veces `checkTrustedRecursive() bypassed`.
- La app se ejecutó en el emulador, mostró el mapa y permitió agregar marcadores (por ejemplo, "Punto #3").
- MobSF mostró un aviso: la sesión usa un APK modificado (Frida Gadget, CA raíz de MobSF, *network-security pins* XML eliminados).

Estas observaciones describen el comportamiento de las herramientas. Por sí solas no demuestran una vulnerabilidad explotable. En particular, que el script haya "bypassed" una comprobación no prueba que la app tenga pinning ni que carezca de él.

### 6.2 Resultados del monitoreo de API

- **Solicitudes observadas:** Las URL capturadas son casi todas HTTPS hacia servicios de Google: `mt0.google.com` (iconos y teselas del mapa), `csi.gstatic.com`, `clients4.google.com/glm/mmap/api`, `dl.google.com` y `redirector.gvt1.com` (voces TTS), `www.google.com/generate_204`. Dos URL usan HTTP: `http://connectivitycheck.gstatic.com/generate_204` (comprobación de conectividad de Android) y `http://goo.gl/8rd3yj` (texto de enlace de documentación en las librerías de Google). No se confirmó que la app haya enviado datos por esas dos URL.
- **Datos sensibles observados:** No se observaron credenciales ni datos personales en el tráfico ni en los registros revisados. Se vio actividad de las librerías de Google (consultas a `ContentResolver` por `maps_client_id`, lectura del país de la SIM `us`, hashes `MessageDigest` del nombre del paquete y archivos `DATA_disk_creation_time_*` creados con `openFileOutput` modo privado).
- **Problemas de seguridad identificados:** Ninguno confirmado en esta etapa. Las llamadas `Base64.decode`, `registerReceiver` (`BATTERY_CHANGED`) y las 9 662 consultas a `ContentResolver` provienen de Google Play Services y del SDK de Maps. El `DexClassLoader` que carga un `.dex` desde `cache/` es el agente de Frida, no código de la app.
- **Portapapeles:** el módulo *Clipboard Dump* no registró contenido.
- **Evidencia:** `monitor_api.png`, `analizadordinamicofridalog.png`, `Dynamic_Analysis.pdf` (secciones *Binder*, *File IO*, *URLs*, *Clipboard Dump*).

### 6.3 TLS/SSL Security Tester

Los cuatro tests de MobSF figuran como **Not Tested**: *Cleartext Traffic*, *TLS Misconfiguration*, *TLS Pinning/CT Bypass* y *TLS Pinning/CT*. **No se puede afirmar** que el tráfico esté libre de problemas TLS.

## 7. Observaciones informativas

| ID | Observación | Fuente | Comentario |
|---|---|---|---|
| O-01 | `targetSdkVersion` 30 | Manifiesto | Es una versión antigua del SDK; Google Play exige apuntar a versiones recientes para publicar o actualizar. Conviene subirlo y revisar el cambio de comportamiento |
| O-02 | Permiso `CAMERA` solicitado | Manifiesto | Aplicar mínimo privilegio si no se usa |
| O-03 | APKiD detecta "Anti-VM Code" en `classes.dex` y "Packer: DexGuard" en el APK | APKiD | Probablemente provenga de librerías de terceros; no se confirmó que la app use ofuscación propia |
| O-04 | Cleartext por defecto | Manifiesto | Con `targetSdk` 30 el tráfico HTTP en claro está bloqueado por defecto; no hay `networkSecurityConfig` explícito |
| O-05 | Dependencias | SBOM | 47 paquetes versionados, sin vulnerabilidades marcadas por MobSF; no se ejecutó un escáner de dependencias dedicado |

## 8. Limitaciones de las pruebas

1. **Descompilación incompleta:** el log de MobSF indica que JADX falló en `classes.dex` y `classes5.dex` ("Some DEX files failed to decompile"). El resultado vacío de *Code Analysis* puede subestimar hallazgos en el código propio (por ejemplo, almacenamiento de datos o entradas sin validar).
2. **APK debug modificado** para el análisis dinámico, con pinning XML eliminado, lo que puede cambiar el comportamiento de red.
3. **TLS Security Tester sin ejecutar** (4 tests "Not Tested").
4. **Sin análisis del código fuente** en este informe: no se revisó manualmente el uso de SQLite, `SharedPreferences`, archivos ni entradas del usuario.
5. **No se usó OWASP ZAP ni interceptación de tráfico con proxy.**
6. **Interacción limitada:** la cobertura dinámica depende de las acciones realizadas sobre la app (mapa, marcadores).
7. **Emulador x86 sin hardware real**: el comportamiento del GPS y de la cámara puede variar en un dispositivo físico.
8. **Diferencia de puntaje entre escaneos** (45/100 con v4.6.2 vs. 66/100 con v4.6.3) sin explicación en los informes.

## 9. Recomendaciones generales de seguridad

1. Entregar siempre una compilación **release** firmada con un keystore propio, con `debuggable` en `false`.
2. Desactivar o restringir la copia de seguridad (`allowBackup="false"`).
3. Restringir la clave de Google Maps y no publicarla en el repositorio.
4. Usar HTTPS para todo el tráfico propio y declarar un `network_security_config.xml` con `cleartextTrafficPermitted="false"`.
5. Si la app guarda marcadores o ubicaciones, cifrarlos con Android Keystore (AES-GCM).
6. Validar y sanear toda entrada del usuario (títulos de marcadores, coordenadas).
7. Usar consultas parametrizadas si se emplea SQLite o Room.
8. Pedir solo los permisos necesarios (revisar `CAMERA`) y solicitar los de ubicación en tiempo de ejecución.
9. Mantener actualizadas las dependencias y subir `targetSdk`.
10. Repetir MobSF (estático y dinámico, incluyendo el TLS Tester) con el APK corregido.

Ver los detalles de implementación en [`best_practices.md`](best_practices.md).

## 10. Conclusión

El análisis estático y dinámico con MobSF encontró **5 hallazgos**: 2 altos (certificado y modo debug, propios de una compilación debug) y 3 medios (copia de seguridad, receiver exportado de AndroidX y clave de Maps incluida en el APK). No se observaron datos sensibles ni conexiones inseguras confirmadas durante la ejecución, pero varias pruebas quedaron incompletas (descompilación parcial, TLS Tester sin ejecutar), por lo que **no se puede concluir que la aplicación esté libre de vulnerabilidades**.

Todos los hallazgos están en estado **pendiente**. Tras aplicar las correcciones, se debe generar el APK release, repetir los análisis y actualizar el estado de cada vulnerabilidad.
