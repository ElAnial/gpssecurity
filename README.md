# GPSMapApp (SimpleGPSMapApp) - Análisis y Plan de Seguridad Móvil

# Descripción
Este proyecto es una aplicación Android (desarrollada en Java e integrada con el SDK de Google Maps) que ha sido sometida a una auditoría de seguridad. El repositorio documenta el análisis estático y dinámico realizado con **MobSF v4.6.3** y **Frida**, identificando vulnerabilidades comunes y estableciendo un plan estructurado de mitigación alineado con el estándar **OWASP MASVS v2.0**.

*Nota: Las soluciones técnicas documentadas en la carpeta de documentación constituyen el plan de acción y hoja de ruta (propuestas), no estando implementadas aún en el código fuente de esta compilación.*

## Vulnerabilidades Identificadas
Tras la auditoría, se identificaron los siguientes riesgos principales:
*   **Inyección SQL:** Riesgo en la persistencia local de marcadores mediante SQLite.
*   **Comunicación no segura (MITM):** Ausencia de *Certificate Pinning* y configuración explícita de TLS.
*   **Exposición de datos sensibles:**
    *   API Key de Google Maps hardcodeada (`google_maps_key`).
    *   Copia de seguridad por ADB habilitada (`android:allowBackup="true"`).
*   **Configuraciones de compilación inseguras:** APK firmado con certificado de debug (`CN=Android Debug`) y depuración habilitada (`debuggable="true"`).
*   **Permisos excesivos:** Solicitud del permiso `CAMERA` sin ser utilizado.
*   **Falta de ofuscación:** Código fácilmente descompilable al tener `minifyEnabled false`.

## Mejoras Propuestas (Hoja de Ruta)
Las siguientes medidas están documentadas detalladamente en la carpeta de documentación para ser aplicadas:
*   **Cifrado de datos sensibles:** Implementación futura de `Android Keystore` con AES-256-GCM para proteger tokens y datos locales.
*   **Comunicación segura (HTTPS):** Creación de `network_security_config.xml` bloqueando tráfico en texto claro e implementando *SSL Certificate Pinning*.
*   **Validación y sanitización de entradas:** Uso de consultas parametrizadas y validación geométrica de coordenadas.
*   **Hardening:** Configuración de `debuggable="false"`, `allowBackup="false"` y uso de un Keystore de producción.
*   **Ofuscación de Código:** Activación de R8/ProGuard en compilaciones *release*.

## Documentación
Toda la auditoría, informes técnicos y guías de implementación se encuentran organizados en la carpeta `Documentacion`:

**Informes y Guías Técnicas (.md):**
*   [Vulnerabilidades](Documentacion/vulnerabilities.md) - Registro de hallazgos de MobSF.
*   [Best Practices](Documentacion/best_practices.md) - Guía de código seguro y configuraciones.
*   [Security Tips](Documentacion/INFORME_SECURITY_TIPS.md) - Justificación de los 6 Security Tips y pruebas planificadas.
*   [Security Improvement Program](Documentacion/security_improvement_program.md) - KPIs, ciclo continuo y hoja de ruta.

**Informe Académico y Reportes (.pdf / .docx):**
*   [Informe Técnico de Seguridad](Documentacion/Informe/Informe_Seguridad_Aplicacion_Android(Alumno).docx) *(Nota: verifica el nombre exacto del archivo docx)*
*   [Reporte de Análisis Estático](Documentacion/Static%20Analysis.pdf)
*   [Reporte de Análisis Dinámico](Documentacion/Dynamic%20Analysis.pdf)
*   [Reporte Estático MobSF](Documentacion/reporte%20statico%20mobSF.pdf)
*   [Reporte Consolidado](Documentacion/vulnerability_report.pdf)

## Cómo Ejecutar la Aplicación de Forma Segura
Para importar y probar el proyecto en tu entorno local:
1. **Clonar el repositorio:**
   ```bash
   git clone <URL_DEL_REPOSITORIO>