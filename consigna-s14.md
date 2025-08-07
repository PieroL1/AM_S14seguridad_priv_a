# Evaluación Técnica: Análisis y Mejora de Seguridad en Aplicación Android

## Introducción
Esta evaluación técnica se basa en una aplicación Android que implementa un sistema de demostración de permisos y protección de datos. La aplicación utiliza tecnologías modernas como Kotlin, Android Security Crypto, SQLCipher y patrones de arquitectura MVVM.

## Parte 1: Análisis de Seguridad Básico (0-7 puntos)

### 1.1 Identificación de Vulnerabilidades (2 puntos)
Analiza el archivo `DataProtectionManager.kt` y responde:
- ¿Qué método de encriptación se utiliza para proteger datos sensibles?

    - **La aplicación utiliza la clase MasterKey de la librería Android Security Crypto, configurada con el esquema AES256_GCM.**
    - **Los datos sensibles son almacenados en EncryptedSharedPreferences, lo que garantiza que tanto claves como valores estén cifrados en disco.**

- Identifica al menos 2 posibles vulnerabilidades en la implementación actual del logging

    1. **Los registros de accesos se almacenan en accessLogPrefs, que no está cifrado. Esto representa un riesgo de exposición de metadatos de seguridad, como fechas y patrones de uso.**
    2. **La información registrada incluye fecha y hora exacta de cada acceso. Un atacante podría correlacionar dicha información para inferir comportamientos del usuario, incluso sin acceso a los datos cifrados.**

- ¿Qué sucede si falla la inicialización del sistema de encriptación?

    La función initialize() está envuelta en un bloque try/catch. Si la inicialización de la encriptación falla, los objetos encryptedPrefs y accessLogPrefs no se instancian correctamente. Esto provocará una excepción del tipo UninitializedPropertyAccessException en posteriores llamadas, dejando la aplicación en un estado inconsistente. No existe un mecanismo de recuperación ni alerta clara al usuario.

### 1.2 Permisos y Manifiesto (2 puntos)
Examina `AndroidManifest.xml` y `MainActivity.kt`:
**- Lista todos los permisos peligrosos declarados en el manifiesto**
La aplicación solicita los siguientes permisos considerados peligrosos por Android:
    - CAMERA
    - READ_EXTERNAL_STORAGE
    - READ_MEDIA_IMAGES
    - RECORD_AUDIO
    - READ_CONTACTS
    - CALL_PHONE
    - SEND_SMS
    - ACCESS_COARSE_LOCATION

- ¿Qué patrón se utiliza para solicitar permisos en runtime?

    En MainActivity.kt se emplea el patrón basado en ActivityResultContracts.RequestPermission junto con ContextCompat.checkSelfPermission.
    Este es el enfoque moderno recomendado por Jetpack, que permite una gestión centralizada de permisos y callbacks seguros.

- Identifica qué configuración de seguridad previene backups automáticos

    El archivo AndroidManifest.xml establece la propiedad:
    android:allowBackup="false"
    Esta configuración impide que los datos de la aplicación se incluyan en los backups automáticos de Android (Google Drive u otros), reduciendo el riesgo de fuga de información sensible.

### 1.3 Gestión de Archivos (3 puntos)
Revisa `CameraActivity.kt` y `file_paths.xml`:
- ¿Cómo se implementa la compartición segura de archivos de imágenes?

    La aplicación utiliza un FileProvider para compartir archivos de imagen generando URIs seguras del tipo content://. Esto evita exponer rutas absolutas del sistema de archivos.

- ¿Qué autoridad se utiliza para el FileProvider?

    La autoridad está configurada en el manifiesto siguiendo la convención:
    com.example.seguridad_priv_a.fileprovider
    Esta autoridad vincula el FileProvider con los paths definidos en res/xml/file_paths.xml.

- Explica por qué no se debe usar `file://` URIs directamente
    
    - Desde Android 7.0 (API 24), el uso de file:// URIs genera una excepción (FileUriExposedException).
    - Las URIs basadas en rutas absolutas pueden filtrar información sensible sobre la estructura interna del sistema de archivos.
    - A diferencia de file://, las URIs content:// permiten aplicar permisos temporales y específicos por archivo, lo que incrementa la seguridad al compartir datos entre aplicaciones.

## Parte 2: Implementación y Mejoras Intermedias (8-14 puntos)

### 2.1 Fortalecimiento de la Encriptación (3 puntos)
Modifica `DataProtectionManager.kt` para implementar:

 ##**SOLUCION**
La aplicación protege los datos sensibles mediante cifrado robusto AES-256-GCM, gestionado por Android Security Crypto y una clave maestra segura.  
Las siguientes mejoras refuerzan la protección:

- **Rotación de clave maestra:**  
  Permite renovar la clave periódicamente para minimizar riesgos ante una eventual filtración. La rotación puede activarse manualmente desde la interfaz y puede programarse para ser automática cada 30 días (con un scheduler/WorkManager).
- **Verificación de integridad (HMAC):**  
  Cada dato almacenado se acompaña de un HMAC, garantizando que cualquier intento de modificación fraudulenta sea detectado de inmediato.
- **Derivación de clave con salt único:**  
  (En esta implementación, la derivación de clave se plantea como mejora futura, aprovechando Android Keystore y un salt generado al inicializar el usuario.)

**Funcionamiento:**  
Cuando el usuario almacena datos sensibles, estos se cifran y se almacena su HMAC. En cada acceso, el sistema verifica que el valor no haya sido alterado.  
La clave maestra de cifrado puede rotarse para mejorar la seguridad y todos los eventos críticos quedan registrados en logs.

**Capturas:**

- **Antes y después de la rotación de clave maestra:**
  ![Rotación de clave maestra](images/captura_rotacion.png)

- **Lectura y guardado de datos protegidos con HMAC:**
  ![Protección HMAC](images/captura_hmac_ok.png)
  ![Error de integridad HMAC](images/captura_hmac_error.png)

---

### 2.2 Sistema de Auditoría Avanzado (3 puntos)

 ##**SOLUCION**

El sistema de auditoría registra automáticamente todas las acciones relevantes, tales como accesos a datos, cambios de configuración y rotación de claves.  
Cada registro incluye: fecha/hora, tipo de acción, recurso afectado y resultado.

- **Consulta de logs:**  
  El usuario puede visualizar el historial completo de operaciones, permitiendo así auditoría transparente y trazabilidad total.

**Funcionamiento:**  
Cada vez que se accede, modifica o elimina un dato protegido, se crea una entrada de log.  
La información puede ser consultada desde la sección "Protección de Datos".

**Captura:**
![Logs de acceso](images/captura_logs.png)

---

### 2.3 Biometría y Autenticación (3 puntos)

 ##**SOLUCION**

Para reforzar la confidencialidad de los datos y los registros, se implementa autenticación biométrica:

- **BiometricPrompt API:**  
  Antes de mostrar información sensible o logs de auditoría, se requiere autenticación por huella digital, rostro, o PIN de respaldo.
- **Fallback a PIN/Pattern:**  
  Si la biometría no está disponible, el sistema permite ingresar un PIN o patrón configurado previamente por el usuario.
- **Timeout de sesión:**  
  Si el usuario permanece inactivo durante más de 5 minutos, se requiere volver a autenticarse antes de acceder a datos protegidos.

**Funcionamiento:**  
Al intentar ver logs o datos sensibles, se muestra un prompt biométrico.  
Si no se autentica, el acceso es denegado hasta completar correctamente la autenticación.  
Un temporizador de sesión asegura que tras 5 minutos de inactividad, se vuelva a solicitar autenticación.

**Captura:**
(al poner la opcion de biometria no deja tomar captura, por lo que solo se adjunta la validacion de la biometria)
![Autenticación biométrica](images/captura_biometria.png)

---

## Parte 3: Arquitectura de Seguridad Avanzada (15-20 puntos)

### 3.1 Implementación de Zero-Trust Architecture (3 puntos)
Diseña e implementa un sistema que:
- Valide cada operación sensible independientemente
- Implemente principio de menor privilegio por contexto
- Mantenga sesiones de seguridad con tokens temporales
- Incluya attestation de integridad de la aplicación

### 3.2 Protección Contra Ingeniería Inversa (3 puntos)
Implementa medidas anti-tampering:
- Detección de debugging activo y emuladores
- Obfuscación de strings sensibles y constantes criptográficas
- Verificación de firma digital de la aplicación en runtime
- Implementación de certificate pinning para comunicaciones futuras

### 3.3 Framework de Anonimización Avanzado (2 puntos)
Mejora el método `anonymizeData()` actual implementando:
- Algoritmos de k-anonimity y l-diversity
- Differential privacy para datos numéricos
- Técnicas de data masking específicas por tipo de dato
- Sistema de políticas de retención configurables

```kotlin
class AdvancedAnonymizer {
    fun anonymizeWithKAnonymity(data: List<PersonalData>, k: Int): List<AnonymizedData>
    fun applyDifferentialPrivacy(data: NumericData, epsilon: Double): NumericData
    fun maskByDataType(data: Any, maskingPolicy: MaskingPolicy): Any
}
```

### 3.4 Análisis Forense y Compliance (2 puntos)
Desarrolla un sistema de análisis forense que:
- Mantenga chain of custody para evidencias digitales
- Implemente logs tamper-evident usando blockchain local
- Genere reportes de compliance GDPR/CCPA automáticos
- Incluya herramientas de investigación de incidentes

## Criterios de Evaluación

### Puntuación Base (0-7 puntos):
- Correcta identificación de vulnerabilidades y patrones de seguridad
- Comprensión de conceptos básicos de Android Security
- Documentación clara de hallazgos

### Puntuación Intermedia (8-14 puntos):
- Implementación funcional de mejoras de seguridad
- Código limpio siguiendo principios SOLID
- Manejo adecuado de excepciones y edge cases
- Pruebas unitarias para componentes críticos

### Puntuación Avanzada (15-20 puntos):
- Arquitectura robusta y escalable
- Implementación de patrones de seguridad industry-standard
- Consideración de amenazas emergentes y mitigaciones
- Documentación técnica completa con diagramas de arquitectura
- Análisis de rendimiento y optimización de operaciones criptográficas

## Entregables Requeridos

1. **Código fuente** de todas las implementaciones solicitadas
2. **Informe técnico** detallando vulnerabilidades encontradas y soluciones aplicadas
3. **Diagramas de arquitectura** para componentes de seguridad nuevos
4. **Suite de pruebas** automatizadas para validar medidas de seguridad
5. **Manual de deployment** con consideraciones de seguridad para producción

## Tiempo Estimado
- Parte 1: 2-3 horas
- Parte 2: 4-6 horas  
- Parte 3: 8-12 horas

## Recursos Permitidos
- Documentación oficial de Android
- OWASP Mobile Security Guidelines
- Libraries de seguridad open source
- Stack Overflow y comunidades técnicas

---

**Nota**: Esta evaluación requiere conocimientos sólidos en seguridad móvil, criptografía aplicada y arquitecturas Android modernas. Se valorará especialmente la capacidad de aplicar principios de security-by-design y el pensamiento crítico en la identificación de vectores de ataque.
