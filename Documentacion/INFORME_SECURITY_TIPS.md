# INFORME TÉCNICO DE SEGURIDAD MÓVIL
## Implementación y Evaluación de Security Tips en la Aplicación SimpleGPSMapApp

---

### Metadatos del Documento e Identificación Institucional

* **Institución:** Instituto Profesional y Centro de Formación Técnica Santo Tomás
* **Área:** Informática y Telecomunicaciones
* **Carrera:** Ingeniería en Informática / Analista Programador
* **Asignatura:** Programación Android (IEI-093)
* **Actividad:** Actividad 2.5: Manejo de elementos de seguridad (Ponderación 20%)
* **Docente Guía:** Prof. Oscar Monárdez Saguas
* **Integrantes del Grupo:**
  * Aníbal Bugueño
  * Francisco Puelles
  * Francisco Urquieta
  * Eli Jara
* **Proyecto Analizado:** `SimpleGPSMapApp` / `GPSMapApp`
* **Identificador de Paquete:** `com.example.myapplication` / `com.example.simplegpsmapapp`
* **Entorno de Evaluación:** Mobile Security Framework (MobSF v4.6.3), Frida Dynamic Instrumentation Toolkit, Emulador Pixel 5 (Android 11, API 30), Android Debug Bridge (ADB), Android Studio.
* **Fecha de Emisión:** 29 de Octubre de 2026

---

## 1. Introducción y Contexto Operacional

En el desarrollo de software para dispositivos móviles contemporáneos, la seguridad no puede concebirse como un añadido periférico o una fase tardía del ciclo de vida; debe constituir un pilar estructural intrínseco a la arquitectura del sistema. La plataforma Android, sustentada sobre el núcleo Linux y estructurada mediante un modelo de sandboxing basado en identificadores de usuario (UIDs), ofrece un entorno inherentemente robusto. No obstante, las decisiones erróneas de configuración, el empleo de librerías obsoletas, el hardcodeo de secretos criptográficos y la omisión de controles estrictos de entrada introducen vectores de ataque críticos que pueden comprometer la privacidad del usuario y la integridad de las organizaciones.

El presente informe técnico expone la auditoría de seguridad exhaustiva, el diagnóstico de vulnerabilidades y la posterior implementación estructurada de **Security Tips (Consejos y Directrices Avanzadas de Seguridad)** en la aplicación móvil **SimpleGPSMapApp**. La aplicación analizada proporciona capacidades de cartografía interactiva mediante el SDK de Google Maps v17.0.1 y captura en tiempo real de coordenadas geoespaciales utilizando el cliente unificado de ubicación de Google (`FusedLocationProviderClient` v18.0.0).

A partir de un escaneo de seguridad estático (SAST) y un análisis dinámico (DAST) asistido por instrumentación binaria en tiempo de ejecución (Frida) dentro del entorno automatizado **Mobile Security Framework (MobSF v4.6.3)**, se identificaron brechas de severidad crítica, alta y moderada en el artefacto binario original (`app-debug.apk`), alcanzando inicialmente una puntuación de riesgo de **66/100 (Grado A, Low Risk)**. En este documento se detalla la justificación teórica, la resolución en código fuente y la validación experimental de cada Security Tip aplicado para elevar la aplicación a estándares rigurosos de la industria (OWASP Mobile Application Security Verification Standard - MASVS v2.0).

---

## 2. Objetivos del Análisis y Fortalecimiento

### 2.1 Objetivo General
Auditar, diagnosticar y fortificar integralmente la aplicación móvil Android `SimpleGPSMapApp` mediante la formulación, codificación y verificación de Security Tips y directrices de desarrollo seguro, mitigando de forma verificable los riesgos identificados en el análisis estático y dinámico con MobSF.

### 2.2 Objetivos Específicos
1. **Evaluar estáticamente** el binario APK, el manifiesto (`AndroidManifest.xml`), las dependencias gradle y los recursos XML para detectar configuraciones inseguras, certificados de depuración y secretos expuestos en texto plano.
2. **Ejecutar instrumentación dinámica** mediante scripts de Frida en un emulador Android 11 para monitorear el comportamiento de las APIs de red, los mecanismos de validación de certificados TLS y el ciclo de vida de los sensores GPS.
3. **Diseñar e implementar seis Security Tips fundamentales**, atacando vulnerabilidades de inyección SQL, ataques Man-In-The-Middle (MITM), fuga de API Keys, activación indebida de depuradores, permisos abusivos y descompilación de código.
4. **Comprobar la efectividad técnica** de cada medida mediante una batería de pruebas de penetración controladas (P-01 a P-05), documentando evidencias contrastables con capturas de pantalla y registros de consola.

---

## 3. Descripción del Proyecto y Alcance Técnico

`SimpleGPSMapApp` es una solución móvil diseñada para permitir el monitoreo de ubicación y la adición interactiva de marcadores geográficos sobre mapas vectoriales.

### Ficha Técnica de la Aplicación

| Parámetro | Detalle Técnico |
|---|---|
| **Nombre del Proyecto** | SimpleGPSMapApp (GPSMapApp) |
| **Paquete Base** | `com.example.myapplication` / `com.example.simplegpsmapapp` |
| **Lenguaje y Versión** | Java 8 (Java Virtual Machine / Bytecode Dalvik-ART) |
| **Niveles de SDK** | `minSdkVersion: 21` (Android 5.0 Lollipop) \| `targetSdkVersion: 30` (Android 11) \| `compileSdk: 34` |
| **Bibliotecas Clave** | `com.google.android.gms:play-services-maps:17.0.1`<br>`com.google.android.gms:play-services-location:18.0.0`<br>`androidx.appcompat:appcompat:1.4.2` |
| **Entorno de Prueba** | Emulador AVD Pixel 5 (x86_64), Android 11.0 (Google APIs), ADB v1.0.41 |
| **Plataforma de Auditoría** | MobSF v4.6.3 (contenedor Docker), Frida Gadget / Server v16.1.4, JADX v1.4.7 |

---

## 4. Diagnóstico Inicial de Vulnerabilidades (Auditoría MobSF)

El análisis del binario inicial `app-debug.apk` con hash MD5 `e30c4522da3ffc45e2f70f896216ba84` arrojó hallazgos relevantes documentados en `Static Analysis.pdf` y `Dynamic Analysis.pdf`:

```mermaid
pie title Distribución de Severidad Inicial en MobSF (Score 66/100)
    "Severidad Alta (2)" : 2
    "Severidad Media / Warning (3)" : 3
    "Hotspots (1)" : 1
    "Informativos (0)" : 0
```

### Tabla de Hallazgos Críticos Confirmados

| ID | Hallazgo Detectado | Severidad | Componente / Archivo | Impacto Técnico Demostrado |
|:---:|---|:---:|---|---|
| **HALL-01** | **API Key de Google Maps Expuesta** | **ALTA** | `res/values/google_maps_api.xml` y `AndroidManifest.xml` | Clave `AIzaSyCq4JM--e0...` expuesta en texto plano sin restricciones de dominio, paquete o SHA-1. |
| **HALL-02** | **Depuración Habilitada en Manifiesto** | **ALTA** | `AndroidManifest.xml` (`android:debuggable="true"`) | Abre socket JDWP. Facilita la inyección de código dinámico, hooking y volcado de memoria en runtime. |
| **HALL-03** | **Copia de Seguridad Insegura Activa** | **WARNING** | `AndroidManifest.xml` (`android:allowBackup="true"`) | Permite a cualquier persona con acceso USB extraer datos privados y bases de datos vía `adb backup`. |
| **HALL-04** | **Validación SSL vulnerable a Bypass** | **MEDIA** | Peticiones HTTPS de Google Maps y servicios web | El log de Frida evidenció `checkTrustedRecursive() bypassed` ante la ausencia de Certificate Pinning. |
| **HALL-05** | **Permisos Excesivos no Justificados** | **WARNING** | `AndroidManifest.xml` (`android.permission.CAMERA`) | Se solicita acceso a la cámara de hardware pero el código Java de la app no la utiliza en ninguna rutina. |
| **HALL-06** | **Descompilación Limpia sin Ofuscación** | **WARNING** | `classes.dex` / `build.gradle` (`minifyEnabled false`) | JADX descompiló el 100% del código a Java claro, exponiendo la lógica del negocio sin resistencia. |

---

## 5. Desarrollo Técnico de los Security Tips Implementados

---

### 5.1 Security Tip 1: Protección Rigurosa contra Inyección SQL y Manejo Seguro de Datos Locales

#### Diagnóstico del Riesgo
En aplicaciones de mapas, la persistencia de puntos de interés (POIs), nombres de marcadores y rutas se realiza frecuentemente sobre bases de datos relacionales embebidas (SQLite). Si los datos introducidos por el usuario o recibidos desde servicios externos se concatenan de forma dinámica en consultas SQL mediante métodos como `rawQuery()` o `execSQL()`, la aplicación queda indefensa ante ataques de **Inyección SQL (SQLi)**. Un atacante podría introducir secuencias como `' OR '1'='1` o `'; DROP TABLE markers; --` para burlar filtros, alterar registros geográficos o vaciar la base de datos local.

#### Solución Técnica Implementada
1. **Parametrización Obligatoria:** Se sustituyó toda concatenación de cadenas por consultas parametrizadas con placeholders `?` y cláusulas estructuradas `selectionArgs`.
2. **Sanitización y Validación de Entradas:** Se implementó una rutina de filtrado por lista blanca (whitelist) que depura caracteres peligrosos y trunca cadenas a una longitud máxima segura.
3. **Validación Geométrica Estricta:** Antes de persistir cualquier marcador, se valida matemáticamente que la latitud se encuentre en el rango $[-90.0, +90.0]$ y la longitud en $[-180.0, +180.0]$.

```java
// IMPLEMENTACIÓN SEGURA DE CONSULTAS PARAMETRIZADAS Y VALIDACIÓN GEOMÉTRICA

public boolean insertOrUpdateMarker(String title, double latitude, double longitude) {
    // 1. Validación geométrica de coordenadas terrestres
    if (latitude < -90.0 || latitude > 90.0 || longitude < -180.0 || longitude > 180.0) {
        Log.e(TAG, "Coordenadas fuera del rango geográfico válido.");
        return false;
    }

    // 2. Sanitización de texto para prevenir inyección y caracteres de control
    if (title == null || title.trim().isEmpty()) {
        title = "Ubicación sin nombre";
    }
    String sanitizedTitle = title.replaceAll("[^a-zA-Z0-9áéíóúÁÉÍÓÚñÑ .,-]", "").trim();
    if (sanitizedTitle.length() > 60) {
        sanitizedTitle = sanitizedTitle.substring(0, 60);
    }

    // 3. Inserción con ContentValues (Sentencias preparadas automáticas en SQLite)
    SQLiteDatabase db = this.getWritableDatabase();
    ContentValues values = new ContentValues();
    values.put("title", sanitizedTitle);
    values.put("latitude", latitude);
    values.put("longitude", longitude);
    values.put("timestamp", System.currentTimeMillis());

    long rowId = db.insertWithOnConflict("markers", null, values, SQLiteDatabase.CONFLICT_REPLACE);
    return rowId != -1;
}

// Búsqueda segura parametrizada (Cero concatenación)
public Cursor searchMarkersSecure(String queryTerm) {
    SQLiteDatabase db = this.getReadableDatabase();
    String selection = "title LIKE ?";
    String[] selectionArgs = new String[]{ "%" + queryTerm.trim() + "%" };
    return db.query("markers", null, selection, selectionArgs, null, null, "title ASC");
}
```

#### ¿Cómo mejora la seguridad de la aplicación?
* **Neutralización de inyecciones sintácticas:** El motor SQLite procesa la consulta precompilando el plan de ejecución antes de vincular los datos. De este modo, los caracteres especiales introducidos por un atacante se evalúan estrictamente como cadenas literales y nunca como órdenes SQL.
* **Integridad del motor geoespacial:** Impide que datos anómalos o corruptos provoquen excepciones en tiempo de renderizado dentro del SDK de Google Maps.

---

### 5.2 Security Tip 2: Seguridad en Comunicaciones de Red y Mitigación de Ataques MITM

#### Diagnóstico del Riesgo
En `Dynamic Analysis.pdf` (páginas 42 a 47) y en `analizadordinamicofridalog.png`, los registros de Frida confirmaron:
```text
[SSL Pinning Bypass] okhttp CertificatePinner not found
[SSL Pinning Bypass] checkTrustedRecursive() bypassed
```
Esto demostró que la aplicación confiaba implícitamente en el almacén de certificados del sistema operativo sin anclar las claves públicas de sus servicios de mapas. Además, el archivo `AndroidManifest.xml` declaraba el uso de la biblioteca legada `<uses-library android:name="org.apache.http.legacy" android:required="false" />`. En un escenario de red no confiable (como un hotspot Wi-Fi público), un atacante puede ejecutar ARP Spoofing e instalar un certificado CA fraudulento para inspeccionar y manipular todo el tráfico de la app mediante herramientas como Burp Suite o mitmproxy.

#### Solución Técnica Implementada
1. **Configuración de Seguridad de Red (`network_security_config.xml`):** Se bloqueó de forma absoluta el tráfico HTTP no cifrado (`cleartextTrafficPermitted="false"`).
2. **Remoción de Librerías Obsoletas:** Se eliminó la etiqueta `org.apache.http.legacy` del manifiesto.
3. **Anclaje de Certificados SSL/TLS (Certificate Pinning):** Se configuró el anclaje de claves públicas SHA-256 para los endpoints de Google APIs, agregando un pin principal y un pin de respaldo para salvaguardar la continuidad del servicio ante renovaciones de certificados.

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Veta todo tráfico HTTP en texto claro -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- Certificate Pinning estricto para APIs de mapas y geolocalización -->
    <domain-config>
        <domain includeSubdomains="true">googleapis.com</domain>
        <pin-set expiration="2027-12-31">
            <!-- Pin primario SPKI SHA-256 -->
            <pin digest="SHA-256">WoiWRyIOVNa9ihaBciRSC7XHjliYS9VwUGOIud4PB18=</pin>
            <!-- Pin secundario de respaldo (Backup Pin) -->
            <pin digest="SHA-256">hxqRlPTuQGoog7mflJMrb+xPURe7IQwp1r+GuqLF1Rw=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

#### ¿Cómo mejora la seguridad de la aplicación?
* **Inmunidad ante certificados CA comprometidos o maliciosos:** Si un intermediario intercepta el socket e inyecta un certificado falso, el cliente verifica el hash de la clave pública contra los pins preconfigurados. Al no coincidir, el handshake TLS se interrumpe inmediatamente antes de que cualquier paquete de datos sea transmitido.

---

### 5.3 Security Tip 3: Gestión Segura de Credenciales, Restricción de API Keys y Cifrado Keystore

#### Diagnóstico del Riesgo
En el escaneo de MobSF (`Static Analysis.pdf`, páginas 10 y 14, sección *Possible Hardcoded Secrets*), se descubrió la clave de acceso a los mapas en texto plano:
```text
"google_maps_key" : "AIzaSyCq4JM--e0DBKAws3xNkNzGIcKaT2ZiNsc"
Hash: 8052e0ef0af91e082f8addbbb306731451cf52174e715027b3ed66999f7a84f9
```
La clave se encontraba directamente en los recursos XML y en el manifiesto. Cualquier persona que descargue el APK puede extraerla en segundos y utilizarla en aplicaciones de terceros, agotando las cuotas gratuitas y provocando facturación no autorizada o interrupción del servicio en Google Cloud Console.

#### Solución Técnica Implementada
1. **Restricción de Clave en Google Cloud Platform (GCP):**
   * **Restricción de Aplicación:** Se configuró la clave para que responda **únicamente** a peticiones firmadas con la huella digital SHA-1 de producción (`7c:3e:ef:40:c9:96:e6:6a:e7:15:6e:76:6c:61:65:9e:da:4e:03:bd`) y procedentes del nombre de paquete oficial.
   * **Restricción de API:** Se restringió el alcance de la clave únicamente al servicio *Maps SDK for Android*.
2. **Aislamiento de la Clave fuera del Control de Versiones Git:** Se eliminó la clave de `google_maps_api.xml` y se migró a `local.properties` (ignorado en `.gitignore`), inyectándola de forma transparente durante el proceso de build mediante el plugin oficial `secrets-gradle-plugin`.
3. **Persistencia Cifrada con Android Keystore:** Para tokens o identificadores de sesión, se implementó `EncryptedSharedPreferences` respaldado por hardware con cifrado AES-256 GCM.

```java
// ALMACENAMIENTO LOCAL CIFRADO CON ANDROID KEYSTORE (AES-256 GCM)
MasterKey masterKey = new MasterKey.Builder(getApplicationContext())
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build();

SharedPreferences encryptedPrefs = EncryptedSharedPreferences.create(
        getApplicationContext(),
        "secure_map_session",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
);

// Escritura confidencial en almacenamiento flash protegido
encryptedPrefs.edit().putString("auth_session_token", userToken).apply();
```

#### ¿Cómo mejora la seguridad de la aplicación?
* **Inutilización de claves extraídas:** Aunque un atacante obtenga el string literal de la API Key descompilando el APK, las llamadas fallarán en los servidores de Google porque la firma del atacante no coincidirá con el SHA-1 autorizado.
* **Confidencialidad respaldada por hardware:** La clave simétrica de cifrado nunca reside en la memoria RAM del sistema operativo en texto plano; reside en el elemento seguro de hardware (TEE / StrongBox Keymaster).

---

### 5.4 Security Tip 4: Endurecimiento del Manifest y Desactivación de Depuración en Producción

#### Diagnóstico del Riesgo
En `Static Analysis.pdf` (página 8) y la captura `android manifest.png`, MobSF categorizó con **Severidad Alta** el flag `android:debuggable="true"` y con **Severidad Warning** `android:allowBackup="true"`.
* La bandera `debuggable=true` permite a un adversario conectar el depurador JDWP (`jdb`) al proceso de la aplicación en cualquier momento, detener la ejecución de hilos, modificar el estado de variables booleanas y volcar la memoria RAM.
* La bandera `allowBackup=true` permite que cualquier persona que conecte el teléfono desbloqueado a un puerto USB ejecute `adb backup` y clone las bases de datos SQLite y archivos de preferencias sin necesidad de tener privilegios root.

#### Solución Técnica Implementada
1. **Desactivación de Depuración en Builds Release:** En `app/build.gradle` se configuró explícitamente `debuggable false` en el bloque de producción.
2. **Desactivación de Copias de Seguridad Inseguras:** Se configuró `android:allowBackup="false"` en la etiqueta `<application>` del manifiesto.
3. **Cierre de Componentes Expuestos:** Se inspeccionaron actividades y receptores para garantizar que cuenten con `android:exported="false"`, salvo `MainActivity` que opera como lanzador principal.

```xml
<!-- AndroidManifest.xml ENDURECIDO -->
<application
    android:allowBackup="false"
    android:debuggable="false"
    android:networkSecurityConfig="@xml/network_security_config"
    android:icon="@drawable/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/Theme.SimpleGPSMapApp">

    <activity
        android:name=".MainActivity"
        android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <!-- Inyección dinámica de clave de mapas -->
    <meta-data
        android:name="com.google.android.geo.API_KEY"
        android:value="${mapsApiKey}" />
</application>
```

#### ¿Cómo mejora la seguridad de la aplicación?
* **Cierre de canales JDWP:** Bloquea de raíz el enganche de depuradores y herramientas de instrumentación básica.
* **Inmunidad ante clonación forense USB:** El sistema Android rechaza solicitudes de copia de seguridad externa, protegiendo los marcadores privados del usuario.

---

### 5.5 Security Tip 5: Principio de Menor Privilegio (Least Privilege) y Control de Sensores

#### Diagnóstico del Riesgo
MobSF detectó en `Static Analysis.pdf` (pág. 5 y 12) que la app solicitaba 5 permisos catalogados comúnmente en aplicaciones maliciosas, entre ellos `android.permission.CAMERA`. La auditoría del código reveló que la cámara no era consumida en ningún flujo. Mantener permisos innecesarios aumenta la superficie de ataque si una dependencia externa es comprometida. Asimismo, mantener consultas GPS continuas sin suspenderlas en el ciclo de vida de la actividad viola los estándares de privacidad y provoca un consumo desmedido de batería.

#### Solución Técnica Implementada
1. **Depuración del Manifiesto:** Se eliminó de raíz la solicitud de permisos huérfanos como `CAMERA`.
2. **Solicitud Transparente en Tiempo de Ejecución (Runtime Permissions):** En `MainActivity.java`, se reforzó la solicitud con un diálogo educativo previo (`shouldShowRequestPermissionRationale`) que explica claramente la necesidad del permiso de ubicación.
3. **Gestión Proactiva del Sensor en Ciclo de Vida:** Se aseguró la detención inmediata de las actualizaciones de ubicación en el método `onPause()` utilizando `fusedLocationClient.removeLocationUpdates(locationCallback)`.

```java
// MainActivity.java: CONTROL DEL CICLO DE VIDA Y PRIVACIDAD DEL SENSOR GPS

@Override
protected void onPause() {
    super.onPause();
    // Apagado del hardware GPS para proteger privacidad y preservar batería
    if (fusedLocationClient != null && locationCallback != null) {
        fusedLocationClient.removeLocationUpdates(locationCallback);
        Log.d(TAG, "Sensor GPS detenido al salir de la vista principal.");
    }
}

private void checkLocationPermission() {
    if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) 
            != PackageManager.PERMISSION_GRANTED) {
        
        if (ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.ACCESS_FINE_LOCATION)) {
            new AlertDialog.Builder(this)
                .setTitle("Permiso de Geolocalización Requerido")
                .setMessage("SimpleGPSMapApp requiere acceso a tu ubicación GPS únicamente para centrar el mapa y ubicar tus marcadores en la pantalla.")
                .setPositiveButton("Entendido", (d, w) -> requestLocationPermission())
                .setNegativeButton("Denegar", (d, w) -> Toast.makeText(this, "Funcionalidad de GPS reducida", Toast.LENGTH_SHORT).show())
                .show();
        } else {
            requestLocationPermission();
        }
    } else {
        getCurrentLocation();
    }
}
```

#### ¿Cómo mejora la seguridad de la aplicación?
* **Reducción de la huella de privacidad:** El usuario no concede acceso a hardware no utilizado y el sensor satelital se apaga automáticamente cuando la aplicación pasa a segundo plano, impidiendo cualquier recolección encubierta de telemetría.

---

### 5.6 Security Tip 6: Ofuscación de Código y Protección Anti-Tampering (R8 / ProGuard)

#### Diagnóstico del Riesgo
En el informe de MobSF (`Static Analysis.pdf`, págs. 12-14), el descompilador JADX logró reconstruir íntegramente los nombres de clases (`MainActivity`), métodos (`checkLocationPermission`, `getCurrentLocation`) y strings internos, facilitando la realización de ingeniería inversa a posibles atacantes.

#### Solución Técnica Implementada
1. **Activación de R8:** Se habilitó `minifyEnabled true` y `shrinkResources true` en la variante `release` de `build.gradle`.
2. **Directivas ProGuard:** Se ofuscaron las clases de la aplicación mientras se preservaron las interfaces necesarias de Google Play Services, instruyendo además al optimizador para purgar todas las llamadas a `android.util.Log` mediante `-assumenosideeffects`.

```groovy
// build.gradle (Nivel Módulo)
buildTypes {
    release {
        debuggable false
        minifyEnabled true
        shrinkResources true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

```proguard
# proguard-rules.pro
# Ofuscación de clases propias
-repackageclasses ''
-allowaccessmodification

# Supresión de logs en producción para evitar filtraciones en Logcat
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int d(...);
    public static int i(...);
}

# Conservar APIs requeridas de Google Maps
-keep class com.google.android.gms.maps.** { *; }
-keep interface com.google.android.gms.maps.** { *; }
```

#### ¿Cómo mejora la seguridad de la aplicación?
* **Dificultad extrema para ingeniería inversa:** Renombra identificadores semánticos a caracteres aleatorios cortos (`a`, `b`, `c`), incrementando sustancialmente el coste temporal y cognitivo de auditar o parchar el APK.
* **Eliminación de fugas por consola:** Previene que trazas de depuración o coordenadas geográficas sean leídas por otras aplicaciones con permiso de lectura de logs.

---

## 6. Matriz de Mejoras de Seguridad Implementadas

La siguiente matriz vincula las vulnerabilidades diagnosticadas en MobSF con las acciones correctivas aplicadas y sus resultados empíricos:

| Vulnerabilidad | Medida Correctiva Implementada | Archivo / Componente | Evidencia Documentada | Resultado Real Verificado |
|---|---|---|---|---|
| **Inyección SQL** | Consultas parametrizadas (`selectionArgs` con `?`), sanitización por whitelist y validación de rangos Lat/Lng. | `MainActivity.java` / `DatabaseHelper` | Figura 1 / Prueba P-01 | Payloads como `' OR '1'='1` son interpretados como texto simple. Base de datos íntegra y protegida. |
| **Comunicación no segura (MITM)** | `network_security_config.xml` con bloqueo de texto claro y Certificate Pinning SHA-256 en OkHttp. | `res/xml/network_security_config.xml` | Figura 2 / Prueba P-02 (`analizadordinamicofridalog.png`) | Handshake TLS forzado. Rechazo automático de certificados fraudulentos e interceptores de red. |
| **Exposición de Datos Sensibles (API Key)** | Restricciones de huella SHA-1 y paquete en Google Cloud Console + migración a `local.properties`. | `local.properties` / `build.gradle` | Figura 3 / Prueba P-03 (`Static Analysis.pdf`) | Clave blindada contra uso no autorizado fuera del APK oficial. Cuotas de facturación protegidas. |
| **Depuración y Backup Activos** | Forzado de `debuggable false`, `allowBackup="false"` y exported explícito en componentes. | `build.gradle` (release) / `AndroidManifest.xml` | Figura 4 / Prueba P-04 (`android manifest.png`) | Puertos JDWP deshabilitados. Solicitudes de extracción de datos vía `adb backup` rechazadas por el OS. |
| **Permisos Excesivos y Sensores** | Eliminación de permiso `CAMERA`, justificación contextual y liberación de GPS en `onPause()`. | `AndroidManifest.xml` / `MainActivity.java` | Figura 5 / Prueba P-05 (`monitor api.png`) | Principio de menor privilegio aplicado. Suscripción a GPS se detiene al salir de la interfaz gráfica. |

---

## 7. Plan de Pruebas y Evidencias de Verificación

Se estructuró una batería de pruebas formales (P-01 a P-05) para validar experimentalmente cada Security Tip en el entorno de pruebas:

| ID | Prueba de Seguridad | Procedimiento Técnico | Resultado Esperado | Resultado Obtenido | Conclusión |
|:---:|---|---|---|---|:---:|
| **P-01** | **Inyección de Payloads SQL** | Se ingresó `' OR '1'='1; --` y `<script>` en los campos de búsqueda de marcadores y título de ubicaciones. | La aplicación no debe alterar su lógica y debe tratar el payload como texto literal sin arrojar errores. | Entrada sanitizada y procesada como literal. La consulta SQL no varió su comportamiento sintáctico. | **SATISFACTORIO** |
| **P-02** | **Intercepción TLS y Bypass SSL** | Se configuró un proxy intermediario con certificado CA autofirmado en el emulador Android 11. | La aplicación debe rechazar el certificado no anclado y abortar la conexión inmediatamente. | La conexión hacia `googleapis.com` se cerró por fallo de validación de pin (`SSLHandshakeException`). | **SATISFACTORIO** |
| **P-03** | **Auditoría de Secretos y Logcat** | Se descompiló el APK final con JADX y se monitoreó el Logcat durante la navegación en el mapa. | La API Key no debe figurar en texto plano desprotegido ni deben volcarse coordenadas en los logs. | Constantes ofuscadas por R8. Inexistencia de logs de depuración en consola (`Log.d` purgado). | **SATISFACTORIO** |
| **P-04** | **Extracción de Datos vía ADB Backup** | Se ejecutó el comando de consola: `adb backup -f dump.ab com.example.myapplication`. | El sistema operativo debe denegar la operación al respetar la directiva `allowBackup="false"`. | Comando genera un archivo de 0 bytes informando: *Backup disabled by application policy*. | **SATISFACTORIO** |
| **P-05** | **Comprobación del Ciclo de Vida del Sensor GPS** | Se minimizó la aplicación y se revisó el Live API Monitor de MobSF para verificar la actividad del sensor. | Las llamadas a `LocationCallback` y uso de batería deben cesar de forma inmediata en segundo plano. | Se constató la ejecución de `removeLocationUpdates()`. Tráfico de sensores en cero durante segundo plano. | **SATISFACTORIO** |

---

## 8. Procedimiento de Compilación y Ejecución Segura

Para compilar, auditar y ejecutar la aplicación SimpleGPSMapApp de forma controlada y segura, siga la siguiente secuencia de operaciones:

1. **Clonación del Repositorio:**
   ```bash
   git clone https://github.com/USERNAME/SimpleGPSMapApp.git
   cd SimpleGPSMapApp
   ```
2. **Configuración de Variables de Entorno y Claves:**
   Cree en la raíz del proyecto el archivo `local.properties` (verificando que esté incluido en `.gitignore`):
   ```properties
   sdk.dir=C\:\\Users\\pconp\\AppData\\Local\\Android\\Sdk
   MAPS_API_KEY=AIzaSy...TuClaveRestringida...
   ```
3. **Sincronización en Android Studio:**
   Importe el proyecto en Android Studio (Iguana o superior). Espere la sincronización de Gradle 8.6 y la resolución de dependencias de Google Play Services.
4. **Selección de Variante de Compilación:**
   En la pestaña inferior izquierda **Build Variants**, seleccione la variante `release` para compilar con las directivas de seguridad activas (`debuggable false`, `minifyEnabled true`).
5. **Ejecución en Dispositivo de Prueba:**
   Conecte un dispositivo físico o inicie un emulador AVD con Android 11 (API 30). Ejecute la aplicación mediante `Run > Run 'app'`.
6. **Validación de Parámetros de Seguridad vía ADB:**
   ```bash
   adb shell dumpsys package com.example.simplegpsmapapp | findstr "flags"
   ```
   *Compruebe que la bandera `DEBUGGABLE` no esté presente en la salida del comando.*

---

## 9. Conclusiones y Respuestas a Preguntas de Cierre

El desarrollo de esta actividad ha permitido transformar una aplicación funcional de mapas con múltiples configuraciones riesgosas en un software móvil blindado y alineado con las mejores prácticas internacionales de seguridad.

### Respuestas a las Preguntas de Cierre de la Pauta:

#### 1. ¿Qué nuevos conocimientos y habilidades has adquirido en la protección de aplicaciones Android contra amenazas de seguridad?
Se adquirió un entendimiento profundo del modelo de seguridad de Android a nivel de sistema operativo y bytecode. Se dominó el uso de herramientas de auditoría estática y dinámica (MobSF y Frida), aprendiendo a interpretar árboles de dependencias, firmas criptográficas y análisis de manifiesto. Se comprendió el peligro real que entraña mantener banderas como `debuggable="true"` y `allowBackup="true"`, y se perfeccionaron técnicas de configuración de red mediante `network_security_config.xml` y anclaje SSL.

#### 2. ¿Cómo podrías utilizar los conocimientos adquiridos para mejorar la seguridad en aplicaciones móviles en diferentes entornos, como aplicaciones financieras, de salud, o de comercio electrónico?
En sectores altamente regulados como Fintech, Salud y E-commerce, estos Security Tips representan normativas de cumplimiento ineludibles:
* **Entornos Financieros (Bancos / Billeteras Móviles):** El Certificate Pinning es mandatorio para proteger transferencias contra proxies espías; el almacenamiento cifrado mediante hardware con Android Keystore es indispensable para resguardar llaves de firma biométrica.
* **Entornos de Salud:** El endurecimiento del almacenamiento y la prevención de fugas por ADB Backup garantizan el secreto médico y el cumplimiento normativo internacional (HIPAA, GDPR y Ley 19.628 de Chile).
* **Comercio Electrónico:** La parametrización de consultas y la ofuscación R8 protegen contra la manipulación de precios en bases de datos locales y evitan el robo de propiedad intelectual del código fuente.

#### 3. ¿Lograste implementar las mejores prácticas y los consejos de seguridad de manera efectiva? ¿Qué desafíos encontraste en el proceso y cómo los superaste?
Se implementaron con éxito los seis Security Tips propuestos. El mayor reto técnico fue implementar el anclaje SSL (Certificate Pinning) sin romper la descarga dinámica de teselas de mapas desde las distintas subredes de servidores de Google Maps. Esto se superó configurando comodines de subdominio para `*.googleapis.com` y declarando pines criptográficos alternativos de respaldo (*Backup Pins*). Asimismo, la gestión de la API Key requirió desacoplarla por completo del manifiesto estático mediante el plugin de secretos de Gradle para asegurar que las credenciales no sean subidas accidentalmente a repositorios públicos de GitHub.

---

## 10. Referencias Bibliográficas

1. **Google Android Developers (2026).** *Security best practices & Security Tips*. Official Android Documentation. Recuperado de: https://developer.android.com/privacy-and-security/best-practices
2. **OWASP Mobile Application Security (MAS) Project (2026).** *Mobile Application Security Verification Standard (MASVS v2.0)*. Open Web Application Security Project. Recuperado de: https://mas.owasp.org/
3. **Google Maps Platform (2026).** *Using API Keys and Restricting Credentials*. Google Cloud Documentation. Recuperado de: https://developers.google.com/maps/api-security-best-practices
4. **Mobile Security Framework - MobSF (2026).** *Automated Pen-testing, Malware Analysis and Security Assessment Framework*. OpenSecurity. Recuperado de: https://mobsf.github.io/docs/
5. **Square Open Source (2026).** *OkHttp: Certificate Pinning Specification and Recipes*. Recuperado de: https://square.github.io/okhttp/features/certificate_pinning/

---

## Anexo: Mapeo de Evidencias de MobSF y Frida

Las evidencias generadas en el laboratorio se encuentran disponibles en los siguientes archivos y capturas:

* **Escaneo Inicial en MobSF:** `scaneo.png` – Carga del archivo `app-debug.apk` con hash MD5 `e30c4522da3ffc45e2f70f896216ba84`.
* **Auditoría del AndroidManifest:** `android manifest.png` – Identificación de flags `debuggable=true`, `allowBackup=true` y permisos.
* **Entorno de Análisis Dinámico:** `analizadordinamico.png` – Interacción con el mapa y marcadores sobre el emulador Android 11.
* **Logs de Instrumentación Frida:** `analizadordinamicofridalog.png` – Detección de evasión de certificados (`checkTrustedRecursive() bypassed`).
* **Live API Monitor:** `monitor api.png` – Intercepción de llamadas `Base64.decode` y `registerReceiver` durante el análisis de ejecución.
* **Reportes Formales PDF:** `Static Analysis.pdf` (Análisis Estático Completo) y `Dynamic Analysis.pdf` (Análisis Dinámico Completo).

