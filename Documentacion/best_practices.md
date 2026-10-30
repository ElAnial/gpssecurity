# Best Practices de seguridad – GPSMapApp

**Paquete:** `com.example.myapplication` · **Plataforma:** Android (Java) · **Base:** hallazgos de `vulnerabilities.md` (MobSF v4.6.3)

Este documento detalla cada práctica, cómo mejora la seguridad y cómo aplicarla en el proyecto. La columna **Estado** y las casillas de verificación sirven para registrar qué ya quedó aplicado en el código. Márcalas solo cuando el cambio esté hecho y verificado con un nuevo escaneo de MobSF.

## Resumen

| # | Práctica | Hallazgo que corrige | Estado |
|---|---|---|---|
| 1 | Compilación release firmada con keystore propio | V-01, V-02 | [ ] Aplicada |
| 2 | Desactivar o restringir la copia de seguridad | V-03 | [ ] Aplicada |
| 3 | Proteger y restringir la clave de Google Maps | V-05 | [ ] Aplicada |
| 4 | Comunicación segura con HTTPS (`network_security_config`) | O-04 | [ ] Aplicada |
| 5 | Cifrado de datos sensibles con Android Keystore | Recomendación general | [ ] Aplicada |
| 6 | Validación y sanitización de entradas | Recomendación general | [ ] Aplicada |
| 7 | Mínimo privilegio en permisos | O-02 | [ ] Aplicada |
| 8 | Componentes exportados y receiver de AndroidX | V-04 | [ ] Aplicada |
| 9 | Consultas parametrizadas (si se usa SQLite/Room) | Recomendación general | [ ] Aplicada / [ ] No aplica |
| 10 | Ofuscación y reducción de código con R8 | Recomendación general | [ ] Aplicada |
| 11 | Dependencias y `targetSdk` actualizados | O-01 | [ ] Aplicada |
| 12 | No registrar datos sensibles en logs | Recomendación general | [ ] Aplicada |

---

## 1. Compilación release firmada con keystore propio

**Problema:** el APK analizado está firmado con `CN=Android Debug` y tiene `android:debuggable="true"` (V-01 y V-02).

**Cómo mejora la seguridad:** un keystore propio y privado permite que solo el desarrollador firme versiones legítimas, y sin `debuggable` no se puede acoplar un depurador ni usar `run-as` para leer datos internos.

**Implementación:**

1. Crear el keystore (una sola vez) y guardarlo **fuera del repositorio**:

```bash
keytool -genkeypair -v -keystore release.jks -alias gpsmapapp \
  -keyalg RSA -keysize 2048 -validity 10000
```

2. Crear `keystore.properties` en la raíz (no se sube a GitHub):

```properties
storeFile=../release.jks
storePassword=********
keyAlias=gpsmapapp
keyPassword=********
```

3. En `app/build.gradle`:

```groovy
def keystoreProps = new Properties()
def keystoreFile = rootProject.file("keystore.properties")
if (keystoreFile.exists()) {
    keystoreFile.withInputStream { keystoreProps.load(it) }
}

android {
    signingConfigs {
        release {
            storeFile file(keystoreProps['storeFile'] ?: "release.jks")
            storePassword keystoreProps['storePassword']
            keyAlias keystoreProps['keyAlias']
            keyPassword keystoreProps['keyPassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            debuggable false
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

4. Compilar con `./gradlew assembleRelease` y volver a analizar ese APK en MobSF.

**Verificación:** *Certificate Analysis* ya no muestra "signed with debug certificate" y el manifiesto no declara `debuggable="true"`.

---

## 2. Desactivar o restringir la copia de seguridad

**Problema:** `android:allowBackup="true"` (V-03).

**Cómo mejora la seguridad:** impide que los datos privados de la app se extraigan con `adb backup` o servicios de respaldo, reduciendo la exposición si el dispositivo cae en manos ajenas.

**Implementación en `AndroidManifest.xml`:**

```xml
<application
    android:allowBackup="false"
    android:fullBackupContent="false"
    ... >
```

Si se necesita respaldo, mantener `allowBackup="true"` y excluir lo sensible en `res/xml/backup_rules.xml`:

```xml
<full-backup-content>
    <exclude domain="sharedpref" path="secure_prefs.xml"/>
    <exclude domain="database" path="markers.db"/>
</full-backup-content>
```

---

## 3. Proteger y restringir la clave de Google Maps

**Problema:** la clave `google_maps_key` aparece en el APK y MobSF la lista como posible secreto (V-05).

**Cómo mejora la seguridad:** una clave restringida a tu app y a la API de Maps no sirve a terceros aunque la extraigan, y mantenerla fuera de Git evita publicarla por error.

**Implementación:**

1. **Restringir** en Google Cloud Console → *APIs y servicios* → *Credenciales* → tu clave:
   - *Restricciones de aplicaciones:* Aplicaciones Android → agregar `com.example.myapplication` y la huella SHA-1 del keystore (debug y release).
   - *Restricciones de API:* solo *Maps SDK for Android*.
2. **Sacarla del código versionado.** En `local.properties` (ya ignorado por Git en proyectos Android):

```properties
MAPS_API_KEY=tu_clave_aqui
```

3. En `app/build.gradle`:

```groovy
def localProps = new Properties()
def localFile = rootProject.file("local.properties")
if (localFile.exists()) {
    localFile.withInputStream { localProps.load(it) }
}

android {
    defaultConfig {
        manifestPlaceholders = [MAPS_API_KEY: localProps.getProperty("MAPS_API_KEY", "")]
    }
}
```

4. En el manifiesto, reemplazar `@string/google_maps_key`:

```xml
<meta-data
    android:name="com.google.android.geo.API_KEY"
    android:value="${MAPS_API_KEY}" />
```

5. Si la clave ya se subió a GitHub (por ejemplo en `google_maps_api.xml`), **rotarla** y borrar el archivo del historial o crear una clave nueva.

> No publiques en el repositorio los informes originales de MobSF (`.pdf` o `.html`): contienen la clave completa.

---

## 4. Comunicación segura con HTTPS

**Estado actual:** durante el análisis dinámico, el tráfico propio de la app y del SDK de Maps fue HTTPS. Con `targetSdk 30`, Android bloquea HTTP en claro por defecto. Falta dejarlo declarado de forma explícita.

**Cómo mejora la seguridad:** HTTPS cifra el tráfico y valida la identidad del servidor, lo que evita ataques de intermediario (MITM). Declarar la política evita que un cambio futuro habilite HTTP sin que nadie lo note.

**Implementación:** crear `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

Y referenciarlo en `<application>`:

```xml
android:networkSecurityConfig="@xml/network_security_config"
android:usesCleartextTraffic="false"
```

Si en el futuro la app consume una API propia, usar siempre URL `https://` y, para mayor protección frente a MITM, agregar *certificate pinning* con `<pin-set>` (con pin de respaldo y fecha de expiración).

**Verificación:** *Network Security* en MobSF muestra la configuración y, en el análisis dinámico, ejecutar el *TLS/SSL Security Tester* (en el primer informe quedó "Not Tested").

---

## 5. Cifrado de datos sensibles (Android Keystore + AES-GCM)

**Aplicación:** si la app guarda localmente información sensible (por ejemplo, marcadores o ubicaciones del usuario, tokens o preferencias privadas), debe guardarla cifrada. La clave se genera dentro del Android Keystore y nunca sale de él.

**Cómo mejora la seguridad:** aunque alguien acceda a los archivos de la app (copia de seguridad, dispositivo con root), no puede leer los datos sin la clave, que está protegida por el hardware o el sistema.

**Implementación (`SecureStore.java`):**

```java
import android.security.keystore.KeyGenParameterSpec;
import android.security.keystore.KeyProperties;
import android.util.Base64;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.security.GeneralSecurityException;
import java.security.KeyStore;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;

public final class SecureStore {
    private static final String ANDROID_KEYSTORE = "AndroidKeyStore";
    private static final String KEY_ALIAS = "gpsmapapp_aes_key";
    private static final String TRANSFORMATION = "AES/GCM/NoPadding";

    private SecureStore() {}

    private static SecretKey getOrCreateKey() throws GeneralSecurityException, IOException {
        KeyStore ks = KeyStore.getInstance(ANDROID_KEYSTORE);
        ks.load(null);
        if (ks.containsAlias(KEY_ALIAS)) {
            return ((KeyStore.SecretKeyEntry) ks.getEntry(KEY_ALIAS, null)).getSecretKey();
        }
        KeyGenerator kg = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, ANDROID_KEYSTORE);
        kg.init(new KeyGenParameterSpec.Builder(
                KEY_ALIAS, KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(256)
                .build());
        return kg.generateKey();
    }

    /** Devuelve IV + texto cifrado codificados en Base64. */
    public static String encrypt(String plainText) throws GeneralSecurityException, IOException {
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.ENCRYPT_MODE, getOrCreateKey());
        byte[] iv = cipher.getIV();
        byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

        byte[] out = new byte[1 + iv.length + cipherText.length];
        out[0] = (byte) iv.length;
        System.arraycopy(iv, 0, out, 1, iv.length);
        System.arraycopy(cipherText, 0, out, 1 + iv.length, cipherText.length);
        return Base64.encodeToString(out, Base64.NO_WRAP);
    }

    public static String decrypt(String encoded) throws GeneralSecurityException, IOException {
        byte[] data = Base64.decode(encoded, Base64.NO_WRAP);
        int ivLen = data[0] & 0xFF;
        byte[] iv = new byte[ivLen];
        System.arraycopy(data, 1, iv, 0, ivLen);

        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.DECRYPT_MODE, getOrCreateKey(), new GCMParameterSpec(128, iv));
        byte[] plain = cipher.doFinal(data, 1 + ivLen, data.length - 1 - ivLen);
        return new String(plain, StandardCharsets.UTF_8);
    }
}
```

**Uso (ejemplo con `SharedPreferences`):**

```java
SharedPreferences prefs = getSharedPreferences("secure_prefs", MODE_PRIVATE);
prefs.edit().putString("markers", SecureStore.encrypt(markersJson)).apply();

String json = SecureStore.decrypt(prefs.getString("markers", ""));
```

Notas: usar GCM (autenticado) y un IV nuevo en cada cifrado, que es lo que hace `cipher.getIV()`. No guardar contraseñas ni claves en el código. Capturar y manejar `GeneralSecurityException` sin mostrar detalles al usuario.

---

## 6. Validación y sanitización de entradas

**Aplicación:** toda entrada del usuario (título o descripción de un marcador, coordenadas, texto de búsqueda) debe validarse antes de usarse, guardarse o mostrarse.

**Cómo mejora la seguridad:** evita datos malformados, desbordes de longitud, caracteres de control y cargas maliciosas que podrían llegar a una consulta SQL, a un archivo o a otra parte de la app.

**Implementación (`InputValidator.java`):**

```java
import java.util.regex.Pattern;

public final class InputValidator {
    private static final int MAX_TITLE_LENGTH = 40;
    private static final Pattern TITLE_PATTERN =
            Pattern.compile("^[\\p{L}\\p{N} .,_#\\-]{1," + MAX_TITLE_LENGTH + "}$");

    private InputValidator() {}

    /** Coordenadas dentro de rango y numéricas. */
    public static boolean isValidLatLng(double lat, double lng) {
        return !Double.isNaN(lat) && !Double.isNaN(lng)
                && lat >= -90.0 && lat <= 90.0
                && lng >= -180.0 && lng <= 180.0;
    }

    /** Devuelve el título limpio o null si no es válido. */
    public static String sanitizeTitle(String raw) {
        if (raw == null) return null;
        String cleaned = raw.trim().replaceAll("\\p{Cntrl}", "");
        return TITLE_PATTERN.matcher(cleaned).matches() ? cleaned : null;
    }
}
```

**Uso al agregar un marcador:**

```java
String title = InputValidator.sanitizeTitle(editTitle.getText().toString());
if (title == null || !InputValidator.isValidLatLng(latLng.latitude, latLng.longitude)) {
    Toast.makeText(this, "Datos no válidos", Toast.LENGTH_SHORT).show();
    return;
}
map.addMarker(new MarkerOptions().position(latLng).title(title));
```

Reglas: validar con lista permitida (qué sí se acepta), limitar la longitud, rechazar en vez de "arreglar" cuando sea posible y validar también al leer datos guardados. En los `EditText` agregar `android:inputType` y `android:maxLength`.

---

## 7. Mínimo privilegio en permisos

**Problema:** la app declara `CAMERA` además de ubicación e internet (O-02), y MobSF marca esos permisos como peligrosos.

**Cómo mejora la seguridad:** menos permisos significa menos superficie de ataque y menos daño si la app es comprometida. Pedir los permisos en tiempo de ejecución da control al usuario.

**Implementación:**

- Si la app **no** usa la cámara, eliminar `<uses-permission android:name="android.permission.CAMERA" />` y `<uses-feature android:name="android.hardware.camera" ... />`.
- Solicitar ubicación solo cuando se necesite:

```java
private final ActivityResultLauncher<String[]> locationPermissionLauncher =
        registerForActivityResult(new ActivityResultContracts.RequestMultiplePermissions(),
                result -> {
                    boolean granted = Boolean.TRUE.equals(result.get(Manifest.permission.ACCESS_FINE_LOCATION))
                            || Boolean.TRUE.equals(result.get(Manifest.permission.ACCESS_COARSE_LOCATION));
                    if (granted) enableMyLocation();
                });

private void requestLocation() {
    locationPermissionLauncher.launch(new String[]{
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION});
}
```

---

## 8. Componentes exportados y receiver de AndroidX

**Problema:** `ProfileInstallReceiver` está exportado y protegido por `android.permission.DUMP` (V-04).

**Cómo mejora la seguridad:** reducir los componentes accesibles desde otras apps limita las vías de ataque entre aplicaciones.

**Implementación:**

- La `MainActivity` debe seguir exportada (es el launcher). Los demás componentes propios deben declarar `android:exported="false"` salvo que necesiten ser públicos.
- Mantener `androidx.profileinstaller` actualizado. Si no se usan perfiles de línea base, se puede eliminar el receiver del manifiesto:

```xml
<receiver
    android:name="androidx.profileinstaller.ProfileInstallReceiver"
    tools:node="remove" />
```

(requiere `xmlns:tools="http://schemas.android.com/tools"` en `<manifest>`). Si se conserva, documentar que `DUMP` es un permiso de nivel `signature|privileged` y que el riesgo es bajo.

---

## 9. Consultas parametrizadas (si se usa SQLite o Room)

**Aplicación:** si la app guarda marcadores en una base de datos, nunca concatenar texto del usuario en una consulta.

**Cómo mejora la seguridad:** los parámetros se envían separados de la consulta, por lo que el texto del usuario no puede alterarla (previene inyección SQL).

```java
// Mal: vulnerable a inyección SQL
db.rawQuery("SELECT * FROM markers WHERE title = '" + title + "'", null);

// Bien: parametrizada
db.rawQuery("SELECT * FROM markers WHERE title = ?", new String[]{title});
```

Con Room, usar parámetros en `@Query("SELECT * FROM markers WHERE title = :title")`. Si la app no usa base de datos, marcar "No aplica" en el resumen.

---

## 10. Ofuscación y reducción de código con R8

**Cómo mejora la seguridad:** `minifyEnabled true` renombra y elimina código no usado, lo que dificulta la ingeniería inversa y reduce el tamaño del APK. No sustituye a las demás medidas.

Ya incluido en la configuración de la práctica 1 (`minifyEnabled true`, `shrinkResources true`). Después de activarlo, probar la app release completa (mapa, marcadores, ubicación) y agregar reglas en `proguard-rules.pro` si algo falla.

---

## 11. Dependencias y `targetSdk` actualizados

**Cómo mejora la seguridad:** cada versión nueva de Android y de las librerías corrige fallos y endurece el comportamiento por defecto. La app apunta a `targetSdk 30` (O-01).

**Implementación:**

- Subir `compileSdk` y `targetSdk` a una versión reciente y revisar los cambios de comportamiento (permisos de ubicación, componentes exportados, backup con `dataExtractionRules`).
- Actualizar `play-services-maps` y `play-services-location` a las últimas versiones estables.
- Revisar periódicamente el SBOM de MobSF (47 paquetes) o ejecutar un escáner de dependencias.

---

## 12. No registrar datos sensibles en logs

**Cómo mejora la seguridad:** los logs pueden leerse con `adb logcat` y en dispositivos con otras herramientas, así que las coordenadas, claves o tokens no deben aparecer ahí.

```java
// Mal
Log.d("GPS", "Posición: " + lat + "," + lng);

// Bien: sin datos personales; en release, sin logs
if (BuildConfig.DEBUG) {
    Log.d("GPS", "Marcador agregado");
}
```

En `proguard-rules.pro` se puede eliminar `Log.d/v` del release:

```
-assumenosideeffects class android.util.Log {
    public static int d(...);
    public static int v(...);
}
```

---

## Verificación final

Después de aplicar las prácticas:

1. Generar el APK **release** (`./gradlew assembleRelease`).
2. Volver a analizarlo en MobSF (estático y dinámico, ejecutando también el *TLS/SSL Security Tester*).
3. Comparar el resultado con `vulnerabilities.md` y actualizar el **Estado** de cada vulnerabilidad (pendiente, corregida o pendiente de verificación).
4. Actualizar las casillas del resumen de este documento y guardar el nuevo reporte en `vulnerability_report.pdf`.

## Archivos que no deben subirse a GitHub

Agregar a `.gitignore`:

```
local.properties
keystore.properties
*.jks
*.keystore
```
