# Proposal

## Issue

**PENDIENTE:** el repositorio no contiene el número del GitHub Issue asociado ni acceso para consultarlo. Hasta que se indique, la trazabilidad es la HU [`docs/alcance/HU-01-frontend.md`](../../../docs/alcance/HU-01-frontend.md), que extiende la HU base [`docs/alcance/HU-01.md`](../../../docs/alcance/HU-01.md). Reemplazar este bloque por el enlace del Issue antes de aprobar.

## Why

El onboarding vigente (`hu-01-onboarding-mvp`) solo captura un celular de prueba, el OTP de demo y un documento `TEST-...`; no identifica a la persona como un usuario único. Una wallet de una entidad financiera supervisada por la SBS necesita recoger datos de identificación antes de operar, y la demo debe mostrar esa captura sin romper su objetivo de medir el tiempo del flujo (referencia de producto: de 8 a 3 minutos). Además, los pasos actuales no permiten revisar lo ingresado antes del envío ni corregir un dato sin perder los demás, lo que penaliza la experiencia en smartphone.

Definiciones confirmadas por el Product Owner:
- Los **datos DAC** son los datos críticos que identifican al cliente de forma única: **DNI, fecha de nacimiento y correo electrónico**.
- Los datos complementarios se definen según lo necesario para una fintech regulada por la SBS.
- La edad **no** es una restricción de acceso.
- El frontend **solo valida el formato** de DNI y correo. La verificación del correo (ingreso de correo y código, similar al SMS), la verificación del DNI y el bloqueo por registro existente se definen e implementan en **otra HU**.

## What Changes

- Añadir al onboarding la captura de datos DAC (DNI, fecha de nacimiento, correo) con validación de formato y mensajes accionables.
- Añadir la captura de datos complementarios propuestos para una fintech regulada por la SBS: nombres y apellidos, nacionalidad, domicilio, ocupación o actividad económica declarada y declaración de persona expuesta políticamente (PEP).
- Añadir la aceptación explícita de términos y del tratamiento de datos, nunca premarcada.
- Añadir una revisión previa al envío, con DNI y correo parcialmente enmascarados, que permita corregir un dato sin perder los demás.
- Conservar los datos válidos al navegar entre pasos y resumir los errores llevando el foco al primer campo inválido.
- Asegurar usabilidad en smartphone y recuperación ante errores de red.
- Garantizar que los datos de identificación no se persistan en el navegador ni aparezcan en logs o eventos.
- Mantener sin cambios el OTP de demo, los resultados KYC, el uso exclusivo de datos de prueba y la medición de duración.

## Capabilities

### New Capabilities
- `onboarding-identity-capture`: captura y validación de formato de datos DAC, datos complementarios y consentimiento, con protección de datos en el cliente y límites explícitos de la demo.
- `onboarding-guided-experience`: secuencia guiada ampliada, revisión previa al envío, conservación de datos, resumen de errores, usabilidad en smartphone y preservación del comportamiento de HU-01.

### Modified Capabilities
None. La capacidad base `wallet-onboarding` está definida por el cambio `hu-01-onboarding-mvp`, que aún no se archiva y no existe en `openspec/specs/`. Estas capacidades la extienden sin modificar sus requisitos; al archivar ambos cambios debe revisarse el solapamiento.

## Alcance

- Interfaz web responsive del onboarding de la demo (Angular), tal como la define HU-01-frontend.
- Captura y validación de formato de los campos listados en "What Changes".
- Revisión previa al envío, conservación de datos, resumen de errores y estados de carga y error de red.
- Observabilidad del cliente limitada a lo que ya define HU-01: sin datos de identificación en eventos ni logs.

## Fuera de alcance

- Verificación del correo por código y verificación del DNI contra cualquier fuente (otra HU).
- Detección de duplicados y bloqueo por registro existente por DNI, correo o celular (otra HU).
- Cualquier restricción de acceso por edad.
- KYC real, documentos o datos personales reales e integraciones productivas.
- Modificar `docs/openapi.json` o inventar campos del mock KYC.
- Definir el almacenamiento, el contrato de API o la arquitectura de los datos adicionales (Arquitectura y Backend).
- Redacción definitiva de textos legales y de consentimiento (Legal/Compliance).
- Pagos, QR, historial, reversos, disputas, conciliación, MAU y CAC.
- Diseño técnico e implementación: no forman parte de esta propuesta.

## Comportamiento a preservar

Tomado de `docs/alcance/HU-01.md` y de la sección "Comportamiento a preservar" de HU-01-frontend:

1. La secuencia de pasos y el paso actual son visibles; el avance se bloquea con datos faltantes o inválidos, con mensaje accionable y sin perder los datos válidos.
2. El OTP de demo `123456` lo valida el backend; no se envían SMS; solo celulares de prueba.
3. El frontend solo llama al backend propio, nunca al mock KYC.
4. Solo `ACTIVE` presenta la wallet como activa; `PENDING` se resuelve desde la misma vista de resultado, sin consola independiente; `REJECTED` y `KYC_UNAVAILABLE` conservan su comportamiento.
5. Un resultado de envío desconocido no ofrece reintento automático.
6. Datos de prueba exclusivamente: `documentType` `TEST` y documento con prefijo `TEST-` hacia el KYC.
7. Eventos y logs sin OTP, teléfono ni documento; la duración se mide desde el inicio hasta el envío correcto de todos los pasos, sin incluir revisión KYC ni activación.
8. Sin pagos, MAU ni CAC.

### Diferencias respecto al contrato vigente y por qué

- El diseño de `hu-01-onboarding-mvp` fija **tres etapas** y limitaba los datos a los de HU-01 y el contrato. Este cambio **levanta la restricción de no solicitar otros campos**: HU-01-frontend define nuevos datos de identificación, por lo que la secuencia incorpora los pasos necesarios. Antes de implementar deben actualizarse `spec.md`, `design.md` y `tasks.md` de ese cambio o reconciliarse al archivar.
- La baseline HU-01 pide definir los campos obligatorios de acuerdo con el alcance aprobado. Los datos DAC y la regla de edad están confirmados por el Product Owner; los **datos complementarios son una propuesta** sujeta a validación de Compliance.

## Supuestos y preguntas abiertas

Supuestos por confirmar:
- Los datos nuevos se capturan solo con valores ficticios.
- El DNI tiene 8 dígitos numéricos.
- La lista de datos complementarios es el mínimo razonable para una fintech regulada por la SBS; la confirma Compliance.
- Los datos KYC `TEST` actuales se mantienen y la captura de DNI los **complementa** (no los reemplaza).
- Lo diferido a otra HU incluye también la verificación del DNI y la comprobación de duplicados, no solo la verificación del correo.

Preguntas abiertas:
1. ¿Qué datos complementarios exige Compliance y cuáles son opcionales?
2. ¿Cómo se relaciona el DNI capturado con el `document` `TEST-` que exige el mock? (Arquitectura/Backend)
3. ¿Para qué se usa la edad si no restringe el acceso?
4. ¿Dónde se guardan los datos adicionales y por cuánto tiempo?
5. ¿Qué aviso de tratamiento de datos es exigible y quién redacta el texto?
6. ¿Qué estándar y nivel de accesibilidad se exige?
7. ¿Qué identificador tendrá la HU de registro y backend diferida?
8. ¿Cuál es el número del Issue de esta HU?

## Impact

- **Frontend:** nuevo contenido en el flujo de onboarding; afecta la secuencia y la medición de duración (más campos pueden alejar la mediana de los 3 minutos; debe medirse y reportarse).
- **Contratos protegidos:** `docs/openapi.json` (mock KYC) no cambia; el OTP de demo y los estados `ACTIVE`/`PENDING`/`REJECTED` no cambian; el esquema de eventos no puede incluir datos personales.
- **Contratos provisionales:** los endpoints propios del backend (`/api/onboarding/flows/...`, tarea 1.5 pendiente) deberán acordar cómo reciben los datos nuevos; requiere acuerdo con Backend y Arquitectura antes de implementar.
- **Cambio relacionado:** `hu-01-onboarding-mvp` (bloques 1, 3 y 4 y la tarea 2.5 siguen pendientes).
- **Riesgos:** los datos DAC son identificadores sensibles; sin la otra HU, una persona podría registrarse con un DNI o correo ajeno y la interfaz podría dar la impresión de identificar al usuario cuando no lo verifica (la demo debe indicarlo); pedir de más contradice la minimización y pedir de menos puede incumplir obligaciones de una entidad supervisada.
