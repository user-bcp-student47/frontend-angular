# HU-01-frontend - Mejoras de captura de datos en el onboarding de wallet

**Estado:** definición de producto trasladada al cambio OpenSpec [`openspec/changes/HU-01-frontend/`](../../openspec/changes/HU-01-frontend/proposal.md) (proposal y specs; sin `design.md` ni `tasks.md` todavía). Las reglas marcadas SUPUESTO POR CONFIRMAR o PENDIENTE siguen sin aprobación de Producto, Compliance o Legal. Este documento es la fuente de producto; ante diferencia, prevalece lo que Producto confirme y debe actualizarse ambos.

## Fuente

| Fuente | Uso |
|--------|-----|
| [docs/alcance/HU-01.md](HU-01.md) | Historia de onboarding vigente (veredicto, ajustes, criterios y evidencia). |
| [docs/alcance/alcance_onboarding.md](alcance_onboarding.md) | Alcance y fuera de alcance de la demo. |
| [docs/business/epic/peru-wallet-mvp.sp.md](../business/epic/peru-wallet-mvp.sp.md) | Contexto de épica, usuarios objetivo, supuestos y riesgos. |
| [docs/business/user-stories/user-onboarding-mvp-completion.sp.md](../business/user-stories/user-onboarding-mvp-completion.sp.md) | Historia de negocio y su supuesto: "los requisitos de KYC y documentación deben definirse con cumplimiento y producto antes de implementar". |
| `openspec/changes/hu-01-onboarding-mvp/` (proposal, design, spec, tasks) | Cambio aprobado y estado de implementación (tareas 2.1 a 2.4 completas; 2.5 y bloques 1, 3 y 4 pendientes). |
| `docs/openapi.json` | Contrato del mock KYC (`POST /kyc/verifications`). |
| Evidencia de implementación | `output/frontend-angular/` (20 pruebas de componente pasan con `npm test`; `npm run build` compila). Implementa solo la base de `hu-01-onboarding-mvp` (tres etapas); nada de HU-01-frontend está implementado. |

**Nota sobre fuentes (confirmada):** la HU-01 de onboarding es [docs/alcance/HU-01.md](HU-01.md). El `HU-01.md` de la raíz del repositorio trata de un token de autenticación y no se usa como fuente.

## Contexto actual

- El frontend Angular presenta tres etapas: celular + OTP, datos KYC `TEST` y resultado (`ACTIVE`, `PENDING`, `REJECTED`, y errores recuperable / desconocido).
- Los únicos datos de identificación que captura hoy son: celular de prueba, OTP de demo `123456`, tipo de documento fijo `TEST` y un documento `TEST-...`.
- El contrato del mock KYC acepta únicamente `documentType` (solo `TEST`), `document` (prefijo `TEST-`) y `scenario`. No define DNI, nombres, domicilio, edad, correo ni otros campos.
- El diseño aprobado de `hu-01-onboarding-mvp` limitaba los datos a los de HU-01 y el contrato; esa restricción **se levanta**: HU-01-frontend define los datos de identificación adicionales. Se mantiene que ninguna traza ni evento puede contener OTP, teléfono ni documento, ni datos reales.
- Los endpoints propios del backend que consume el frontend son **provisionales** (tarea 1.5 pendiente).
- El usuario objetivo de la demo son personas de 18 a 21 años en Perú, no bancarizadas, en smartphone con web responsive. Decisión confirmada: la edad **no es una restricción de acceso**; ese rango describe al público objetivo, no una regla de elegibilidad.

## Problema confirmado

El flujo actual no identifica a la persona más allá de un documento de prueba. El supuesto de la historia de negocio indica que el KYC y la documentación deben definirse antes de implementar, y la necesidad planteada es recoger los datos que permitan identificar al usuario final como una persona única.

**Definición confirmada por el Product Owner:** los datos DAC son los datos críticos del cliente que lo identifican de forma única: **DNI, fecha de nacimiento y correo electrónico**. Los datos complementarios deben definirse según lo necesario para una fintech regulada por la SBS.

**No está confirmado** el listado final de datos complementarios (se propone uno, pendiente de Compliance), su almacenamiento ni cómo se relaciona el DNI con el documento `TEST-` que acepta el mock (ver preguntas abiertas). Esta spec separa lo que ya se puede mejorar sin nuevos datos (Bloque A) de lo que depende de decisiones pendientes (Bloque B).

## Comportamiento a preservar

1. Secuencia visible con el paso actual y bloqueo del avance con datos faltantes o inválidos, con mensaje accionable y sin perder los datos válidos ya ingresados.
2. OTP de demo `123456` validado por el backend; sin SMS; solo celulares de prueba.
3. El frontend solo llama al backend propio; nunca al mock KYC.
4. Solo `ACTIVE` presenta la wallet como activa; `PENDING` se resuelve desde la misma vista de resultado, sin consola independiente.
5. `KYC_UNAVAILABLE` es recuperable con reintento; un resultado de envío desconocido no ofrece reintento automático (prevención de altas repetidas).
6. Uso exclusivo de datos de prueba: tipo de documento `TEST`, documento con prefijo `TEST-`.
7. Eventos y logs sin OTP, teléfono ni documento; la duración del flujo se mide desde el inicio hasta el envío correcto de todos los pasos, sin incluir revisión KYC ni activación.
8. Sin pagos, MAU ni CAC en esta demo.

## Alcance

### Bloque A - Mejoras de experiencia sin nuevos datos (propuesta)

- **A1.** Pantalla de revisión antes del envío final, con los datos ingresados y la opción de corregir cada uno sin reiniciar el flujo.
- **A2.** Conservación de los datos válidos al volver a un paso anterior dentro de la misma sesión de flujo.
- **A3.** Resumen de errores al intentar avanzar, con enlace al primer campo inválido y mensaje que indique cómo corregirlo.
- **A4.** Ayudas de captura para smartphone: teclado adecuado por campo, atributos de autocompletado donde corresponda y etiquetas siempre visibles.
- **A5.** Estados de carga y de error de red por paso, con acción segura de recuperación.

### Bloque B - Captura ampliada de datos de identificación (propuesta, condicionada a aprobación)

Campos candidatos. En la demo todos los valores deben ser ficticios y claramente identificables como datos de prueba.

#### B1. Datos DAC (definición confirmada por el Product Owner)

| Campo | Regla propuesta | Estado |
|-------|-----------------|--------|
| DNI | Obligatorio; formato de 8 dígitos numéricos. En la demo solo se admite un valor ficticio; no se aceptan DNI reales. Su verificación y la comprobación de duplicados quedan para otra HU | Formato SUPUESTO POR CONFIRMAR; relación con el `document` `TEST-` del mock PENDIENTE (Arquitectura/Backend) |
| Fecha de nacimiento | Obligatoria; fecha válida y no futura. Se usa para derivar la edad como dato informativo; la edad no bloquea el avance | Uso de la edad PENDIENTE (Compliance) |
| Correo electrónico | Obligatorio; el frontend solo valida que esté bien escrito (formato válido). La verificación por código y la comprobación de duplicados quedan para otra HU | Alcance diferido confirmado por el Product Owner |

#### B2. Datos complementarios propuestos para una fintech regulada por la SBS

La SBS regula el conocimiento del cliente y la prevención del lavado de activos; el contenido exacto exigible **debe confirmarlo Compliance** (esta spec no cita ni interpreta normas específicas). Lista propuesta de datos mínimos:

| Campo | Regla propuesta | Estado |
|-------|-----------------|--------|
| Nombres, apellido paterno y apellido materno | Obligatorios; solo caracteres válidos para nombres, longitud por definir | SUPUESTO POR CONFIRMAR |
| Nacionalidad | Obligatoria; selección de una lista | SUPUESTO POR CONFIRMAR |
| Domicilio | Departamento, provincia, distrito y dirección (calle y número) obligatorios; referencia opcional | SUPUESTO POR CONFIRMAR |
| Ocupación o actividad económica declarada | Obligatoria; selección de una lista | SUPUESTO POR CONFIRMAR |
| Declaración de condición de persona expuesta políticamente (PEP) | Obligatoria; sí/no | SUPUESTO POR CONFIRMAR |
| Aceptación de términos y consentimiento de tratamiento de datos | Obligatoria y explícita; el texto lo aprueba Legal | Texto PENDIENTE (Legal) |
| Celular verificado con OTP | Ya existe en el flujo actual | Preservar |

No se proponen otros datos (por ejemplo estado civil o datos de terceros) por el principio de minimización; cualquier adición requiere justificación de Compliance.

Cualquier campo aprobado debe incorporarse como un paso o sección identificable de la secuencia, y la secuencia, el diseño y las tareas del cambio OpenSpec deben actualizarse antes de implementar.

### Diferido a otra HU (registro y backend)

Decisión del Product Owner: el registro y el backend que faltan se definen e implementan en otra HU. Requisitos de producto ya confirmados, **fuera de esta HU**:

- El correo debe verificarse de forma similar a la verificación del celular con SMS: el usuario ingresa su correo, recibe un código y lo ingresa.
- El DNI debe verificarse.
- Si ya existe un registro con el mismo DNI, correo o celular, no se permite el registro.

En esta HU el frontend solo valida el formato de DNI y correo; no solicita códigos de correo, no consulta duplicados y no declara DNI o correo como "verificados". Los criterios de aceptación de esas capacidades pertenecen a la otra HU, que debe definir mecanismo de verificación del correo y del DNI, vigencia y límites de reenvío, momento de la comprobación de duplicados, mensaje seguro y acción siguiente para quien ya está registrado.

## Fuera de alcance

- KYC real, documentos o datos personales reales e integraciones productivas.
- Pagos, QR, historial, reversos, disputas y conciliación.
- Cálculo de MAU, CAC o caída en checkout.
- OTP real o envío de SMS.
- Modificar `docs/openapi.json` o inventar campos del mock.
- Decidir el almacenamiento, la arquitectura o la solución técnica de los datos adicionales (corresponde a Arquitectura y Backend).
- Textos legales y de consentimiento definitivos (corresponden a Legal/Compliance).

## Criterios de aceptación

### Bloque A

- **CA-A1.** Dado que el usuario completó todos los pasos, cuando llega a la revisión, entonces ve todos los datos ingresados (sin mostrar el OTP) y puede editar uno sin perder los demás.
- **CA-A2.** Dado un dato válido ya ingresado, cuando el usuario vuelve a un paso anterior y regresa, entonces el dato válido sigue presente.
- **CA-A3.** Dado un envío con varios errores, cuando intenta avanzar, entonces el avance se bloquea, se listan los errores con su corrección y el foco pasa al primer campo inválido.
- **CA-A4.** Dada una pantalla de 360 px de ancho, cuando se recorre el flujo, entonces no hay desplazamiento horizontal, los controles táctiles miden al menos 44 px y cada campo tiene etiqueta asociada. El estándar de accesibilidad exigible es **PENDIENTE de confirmar**.
- **CA-A5.** Dado un fallo de red en cualquier paso, cuando ocurre, entonces el usuario ve un mensaje con acción de reintento y no se presenta ningún resultado KYC.
- **CA-A6.** Los cambios del Bloque A no alteran las pruebas existentes ni los criterios de la sección "Comportamiento a preservar".

### Bloque B (aplican solo a campos aprobados)

- **CA-B1.** Dado un campo obligatorio aprobado vacío o inválido, cuando se intenta avanzar, entonces se bloquea el avance con un mensaje que indica el formato esperado.
- **CA-B1a.** Dado un DNI que no cumple el formato aprobado (por ejemplo, menos de 8 dígitos o caracteres no numéricos), cuando se intenta avanzar, entonces se bloquea el avance con un mensaje que indica el formato esperado.
- **CA-B1b.** Dado un correo con formato inválido, cuando se intenta avanzar, entonces se bloquea el avance con un mensaje que indica un ejemplo de formato válido.
- **CA-B1c.** Dado que la declaración PEP o la aceptación de términos y consentimiento no fueron respondidas, cuando se intenta avanzar, entonces se bloquea el avance; la aceptación nunca viene marcada por defecto.
- **CA-B2.** Dada una fecha de nacimiento, cuando se calcula la edad, entonces el resultado es correcto en las fechas límite (cumpleaños el mismo día, un día antes y un día después). Una fecha inexistente o futura bloquea el avance con un mensaje de corrección; ninguna edad, ya sea menor de 18 o mayor de 21, bloquea el avance ni cambia el resultado KYC.
- **CA-B3.** Dados datos de identificación ingresados, cuando se registran eventos o logs del flujo, entonces no contienen DNI, correo, nombres, domicilio, fecha de nacimiento, documento, OTP ni teléfono.
- **CA-B4.** Dados datos de identificación, cuando el usuario cierra o recarga el navegador, entonces el frontend no los conserva en almacenamiento persistente del navegador.
- **CA-B5.** Dado un dato que no sea de prueba (DNI real o documento sin prefijo `TEST-` en el campo enviado al mock, o tipo distinto de `TEST`), cuando se intenta enviar, entonces no se envía y se solicita un dato de prueba. La regla exacta depende de la respuesta a la pregunta abierta 7.
- **CA-B8.** Dado un DNI o correo con formato válido, cuando el usuario avanza, entonces el frontend no consulta duplicados ni presenta el dato como verificado; la unicidad y la verificación se cubren en otra HU.
- **CA-B10.** Dado un correo mal escrito (por ejemplo, sin `@` o sin dominio), cuando se intenta avanzar, entonces se bloquea el avance y se muestra cómo escribirlo bien con un ejemplo; un correo bien escrito no bloquea el avance y no se solicita ningún código.
- **CA-B11.** Dado un DNI mal escrito (distinto de 8 dígitos o con caracteres no numéricos), cuando se intenta avanzar, entonces se bloquea el avance e indica cómo corregirlo; un DNI bien escrito no bloquea el avance y no se consulta ningún servicio de verificación.
- **CA-B12.** Dados DNI y correo ingresados, cuando se muestran en la revisión final, entonces la interfaz no los etiqueta como "verificados"; la activación de la wallet sigue dependiendo solo de `ACTIVE` devuelto por el backend.
- **CA-B9.** Dado que el usuario llega a la revisión final, cuando se muestran DNI y correo, entonces se presentan parcialmente enmascarados (por ejemplo, solo los últimos caracteres) y el usuario puede editarlos.
- **CA-B6.** Dada la secuencia con los nuevos campos, cuando se reporta la duración, entonces se sigue midiendo desde el inicio hasta el envío correcto de todos los pasos y se compara con la referencia de 3 minutos sin afirmar impacto de lanzamiento. El efecto de los campos nuevos sobre la mediana debe medirse y reportarse.
- **CA-B7.** Dado un aviso de tratamiento de datos, cuando se muestre, entonces el texto proviene de Legal/Compliance; sin aprobación, el elemento queda marcado PENDIENTE y no se inventa contenido.

## Contratos afectados/protegidos

| Contrato | Situación |
|----------|-----------|
| `docs/openapi.json` (mock KYC) | **Protegido.** Solo acepta `documentType`, `document` y `scenario`; los campos nuevos no pueden enviarse al mock. |
| Endpoints propios del backend (`/api/onboarding/flows/...`) | **Provisionales** (tarea 1.5). El Bloque B agregaría datos a la petición; requiere acuerdo con Backend/Arquitectura. La verificación de DNI y correo y la respuesta de registro duplicado quedan para la otra HU. |
| OTP fijo `123456` y estados `ACTIVE`/`PENDING`/`REJECTED` | Protegidos. |
| Esquema de eventos y métrica de duración (bloque 3) | Protegidos; no pueden incluir datos personales. |
| `spec.md`, `design.md` y `tasks.md` de `hu-01-onboarding-mvp` | La restricción de no solicitar otros campos se levanta. La secuencia de tres etapas confirmadas cambia al incorporar los pasos de identificación; debe actualizarse el cambio (spec, design y tasks) antes de implementar. |

## Riesgos, supuestos y preguntas abiertas

### Riesgos

- Los datos DAC (DNI, fecha de nacimiento, correo) son identificadores únicos y de alta sensibilidad: su exposición en pantalla, logs, eventos o almacenamiento del navegador es un riesgo de privacidad y de suplantación.
- Hasta que exista la otra HU, DNI y correo se capturan solo con formato válido: la persona podría registrarse con un DNI o correo ajeno o ya registrado, y el flujo puede dar la impresión de que identifica al usuario cuando no lo verifica. Debe quedar visible que es una demo.
- Si la otra HU agrega verificación por código y comprobación de duplicados, añadirá pasos y esperas que pueden alejar la mediana de los 3 minutos.
- Un mensaje de duplicado demasiado específico permitiría confirmar si una persona está registrada (enumeración de usuarios); debe considerarse en la otra HU.
- Si el DNI de la demo debe convertirse al `document` `TEST-` del mock, la correspondencia podría confundirse con una validación real de identidad (RENIEC u otra); esta HU no verifica identidad.
- Los datos complementarios se proponen sin una lista normativa confirmada; pedir de más contradice la minimización y pedir de menos puede incumplir obligaciones de una entidad supervisada.
- Más campos pueden aumentar el tiempo de onboarding y alejar la mediana de la referencia de 3 minutos.
- Capturar datos personales aumenta la exposición de privacidad aun con datos de prueba; un uso con datos reales requiere aprobación de Legal/Compliance.
- Si los campos nuevos se capturan y no se usan en el KYC simulado, el flujo podría dar la impresión de una verificación de identidad que no existe.
- Divergencia entre esta propuesta y el cambio OpenSpec vigente si no se reconcilian antes de implementar.

### Supuestos (por confirmar)

- **SUPUESTO POR CONFIRMAR:** los campos nuevos se capturan solo con datos ficticios.
- **SUPUESTO POR CONFIRMAR:** el DNI tiene 8 dígitos numéricos.
- **SUPUESTO POR CONFIRMAR:** la lista de datos complementarios (B2) es el mínimo razonable para una fintech regulada por la SBS; la confirma Compliance.
- **SUPUESTO POR CONFIRMAR:** la edad se deriva de la fecha de nacimiento, es informativa y no se pide como dato libre.
- **SUPUESTO POR CONFIRMAR:** el Bloque A puede aprobarse y ejecutarse de forma independiente del Bloque B.
- **SUPUESTO POR CONFIRMAR:** la captura de DNI complementa el paso actual de documento `TEST-` y no lo reemplaza (pregunta abierta 7).

### Preguntas abiertas

1. ¿Cuáles son los datos complementarios exigibles a una fintech regulada por la SBS para este MVP (lista de B2) y cuáles son opcionales? Responsable: Compliance.
2. ¿La otra HU de registro y backend (verificación de correo y DNI, duplicados) ya existe o debe crearse, y qué identificador tendrá? ¿Cómo se verificará el DNI en la demo, dado que el mock no ofrece ese endpoint y no se usan datos reales? Responsable: Producto/Arquitectura.
3. ¿Para qué se usa la edad si no restringe el acceso (identificación, analítica u otro fin)? Su uso determina si es necesario capturarla. Responsable: Producto/Compliance.
4. ¿Dónde se guardan los datos adicionales y por cuánto tiempo, dado que el mock no los acepta? Responsable: Arquitectura/Backend/Datos.
5. ¿Hay aviso o consentimiento de tratamiento de datos exigible y qué normativa aplica? Responsable: Legal/Compliance.
6. ¿Qué estándar y nivel de accesibilidad se exige? Responsable: Producto/UX.
7. ¿Cómo se relaciona el DNI capturado con el `document` `TEST-` y el `documentType` `TEST` que exige el mock? ¿El DNI ficticio se envía tal cual al backend y este arma el `document` `TEST-...`, o se mantiene un campo de documento de prueba aparte? Responsable: Arquitectura/Backend/Producto.
8. Confirmación: ¿el alcance diferido a otra HU aplica también a la verificación del DNI y a la comprobación de duplicados, además de la verificación del correo? Esta spec lo asume así. Responsable: Producto.

## Trazabilidad con OpenSpec

| Contenido de esta HU | Spec del cambio `HU-01-frontend` |
|----------------------|----------------------------------|
| Bloque A (revisión, conservación de datos, errores, smartphone) y Comportamiento a preservar | `onboarding-guided-experience` |
| Bloque B (datos DAC, datos complementarios, consentimiento), protección de datos y límites de la demo | `onboarding-identity-capture` |
| Diferido a otra HU (verificación y duplicados) | Fuera de alcance del cambio |

La base del onboarding sigue definida por `hu-01-onboarding-mvp` (capacidad `wallet-onboarding`); las specs de `HU-01-frontend` la extienden y los escenarios de preservación repiten su comportamiento sin cambiarlo.

## Revisión humana requerida

Siguiente paso: Producto, Compliance y Legal responden las preguntas abiertas (incluido el número del Issue); después se crean `design.md` y `tasks.md` del cambio `HU-01-frontend` y se decide si el Bloque A se implementa de forma independiente.
