App Security Improvement Program — GPSMapApp

**Aplicación:** GPSMapApp (`com.example.myapplication`)  
**Plataforma:** Android (Java / Google Maps SDK)  
**Evaluación Base:** Auditoría estática y dinámica en MobSF v4.6.3 (Score inicial: 66/100)

---

## 1. Desarrollo del Programa de Mejora de Seguridad

Este programa establece la estructura formal para evaluar, corregir y mantener la seguridad del software a lo largo del ciclo de vida del desarrollo de la aplicación GPSMapApp.

### A. Evaluación e Identificación de Brechas
Basado en la auditoría inicial de MobSF, se definen tres áreas prioritarias de intervención:
1. **Gestión de Credenciales:** Migración y restricción de la clave `google_maps_key` fuera del control de versiones.
2. **Hardening del Entorno:** Eliminación del modo depuración (`debuggable="false"`), desactivación de respaldos por ADB (`allowBackup="false"`) y firma digital con Keystore de producción.
3. **Comunicaciones y Red:** Declaración explícita de directivas TLS/HTTPS y restricción de tráfico en claro.

### B. Proceso de Revisión Periódica de Seguridad (Ciclo Continuo)
Para garantizar que la seguridad no sea un evento único, se establece el siguiente flujo de revisión continua:
* **Fase de Desarrollo (Pre-Commit):** Análisis de código fuente para evitar hardcoding de credenciales o secretos en recursos XML o archivos Java.
* **Fase de Compilación (CI/CD / Pre-Release):** Generación obligatoria del APK variante *Release* con ofuscación mediante R8/ProGuard activa.
* **Auditoría Estática y Dinámica (Post-Build):** Ejecución periódica de escaneos automatizados en **MobSF** previo a cada despliegue a producción.
* **Revisión Trimestral de Dependencias:** Auditoría de las 47 librerías/paquetes declarados en el SBOM de la aplicación para mitigar vulnerabilidades conocidas en terceros.

### C. Métricas Clave de Seguridad (KPIs)
Para medir la efectividad del programa y el estado de la aplicación, se utilizarán los siguientes indicadores:
* **Security Score de MobSF:** Meta de mantener una puntuación **$\ge 85/100$** en compilaciones Release (superando la nota base de 66/100).
* **Conteo de Vulnerabilidades de Severidad Alta/Media:** Mantenimiento de **0 vulnerabilidades de severidad Alta** y **0 de severidad Media** en producción.
* **Índice de Exposición de Secretos:** 0 credenciales, tokens o API Keys presentes en el repositorio Git o en el binario sin restringir.
* **Tiempo Medio de Remediación (MTTR):** Resolver cualquier hallazgo de severidad Alta en menos de 48 horas tras ser detectado por MobSF.

---

## 2. Documentación del Plan de Acción y Futuras Mejoras

### A. Plan de Acción Inmediato (Mejoras Aplicadas)

| ID | Hallazgo Detectado | Acción de Mejora Programada | Estado |
|---|---|---|---|
| **IMP-01** | Firma Debug (`CN=Android Debug`) | Configuración de Keystore privada para compilaciones Release | En Proceso |
| **IMP-02** | `android:debuggable="true"` | Forzar flag a `false` en la variante Release con Gradle | En Proceso |
| **IMP-03** | `android:allowBackup="true"` | Configurar `android:allowBackup="false"` en `AndroidManifest.xml` | En Proceso |
| **IMP-04** | API Key en recursos XML | Mover clave a `local.properties` y restringir por SHA-1 en Google Cloud | En Proceso |
| **IMP-05** | Ausencia de política de red | Crear `network_security_config.xml` para forzar TLS/HTTPS | En Proceso |

### B. Hoja de Ruta (Roadmap) para Futuras Mejoras de Seguridad

Para garantizar la evolución de la seguridad en futuras versiones de GPSMapApp, se definen las siguientes fases de implementación:

#### Fase 1: Hardening de Almacenamiento Local (Próxima Versión)
* Implementación de la API **Android KeyStore** para cifrar datos locales (puntos de interés o datos del usuario) utilizando algoritmos simétricos AES-256-GCM.
* Implementación de la librería `EncryptedSharedPreferences` de Jetpack Security para la gestión segura de preferencias.

#### Fase 2: Protección Avanzada en Capa de Red
* Implementación de **SSL Certificate Pinning** (vía OkHttp o Network Security Config) para proteger la comunicación con APIs de Google o servidores backend contra ataques Man-In-The-Middle (MITM) en redes Wi-Fi públicas.

#### Fase 3: Integración de Pruebas Automatizadas en Pipeline
* Integración de herramientas SAST/DAST en el flujo de trabajo de GitHub (ej. GitHub Actions con MobSF o SonarQube) para bloquear automáticamente cualquier Pull Request que introduzca código vulnerable o credenciales expuestas.