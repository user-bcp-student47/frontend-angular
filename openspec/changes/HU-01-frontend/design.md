# Design

## Context

Ver `proposal.md` para motivación y alcance, y `specs/onboarding-identity-capture/spec.md` y `specs/onboarding-guided-experience/spec.md` para el contrato observable. Este diseño solo cubre el **frontend Angular** de la demo; el backend Quarkus aún no existe en el repositorio (tareas 1.x de `hu-01-onboarding-mvp` pendientes).

Estado observado del frontend (`output/frontend-angular/`):

- Angular 17.3 con componentes standalone, señales y formularios reactivos; pruebas con Karma/Jasmine en `ChromeHeadless` (`npm test`, 20 pruebas verdes); `npm run build` compila.
- `OnboardingComponent` orquesta todo el flujo: una señal `stage` (`otp` | `kyc` | `result`), una señal `busy`, `formError`, `result`, y dos formularios independientes (`otpForm`, `kycForm`). La plantilla única usa `@switch` por etapa.
- `OnboardingApiService` encapsula las llamadas al backend propio (provisionales, tarea 1.5): `POST /api/onboarding/flows`, `.../otp`, `.../kyc` (`documentType: TEST`, `document`) y `.../decision`. Los errores se traducen con `toApiErrorCode`.
- Los estilos viven en `src/styles.scss` porque el presupuesto `anyComponentStyle` del build avisa a 2 kB y falla a 4 kB.
- Hoy los valores se reinician si el componente se recrea; no hay almacenamiento en el navegador, y el stepper (3 pasos) y los textos "Paso N de 3" están cubiertos por las pruebas actuales.
- `dev-mock-backend.mjs` es un stub local solo para ver la interfaz; no es el backend.

Decisiones de producto ya tomadas (proposal): los datos DAC son DNI, fecha de nacimiento y correo; el frontend solo valida formato; verificación de DNI y correo y bloqueo por duplicados son otra HU; la edad no restringe; los datos complementarios son propuesta pendiente de Compliance.

## Goals / Non-Goals

**Goals:**
- Extender la secuencia con pasos de identificación y una revisión previa al envío, sin rediseñar la aplicación.
- Mantener las validaciones, mensajes y reglas en funciones puras y reutilizables, comprobables sin DOM.
- Conservar datos válidos al navegar, corregir desde la revisión y resumir errores con foco accesible.
- Mantener intacto el comportamiento de OTP, resultados KYC, revisión manual y reintentos.
- Mantener los datos de identificación solo en memoria del componente, sin logs ni almacenamiento del navegador.
- Entregar en fases pequeñas, reversibles individualmente.

**Non-Goals:**
- Verificar DNI o correo, consultar duplicados o solicitar códigos de correo (otra HU).
- Definir el almacenamiento, el esquema de datos o el backend; solo se propone un contrato frontend provisional.
- Catálogos oficiales (ubigeo, países, ocupaciones) ni textos legales definitivos.
- Restricciones por edad, nuevos estados de wallet, pagos, MAU o CAC.
- Introducir enrutamiento, gestión de estado global, librerías de UI o pruebas e2e.
- Implementar los eventos y la medición de duración (bloque 3 de `hu-01-onboarding-mvp`).

## Contratos protegidos

| Contrato | Protección en este diseño |
|----------|---------------------------|
| `docs/openapi.json` (mock KYC) | Sin cambios. El frontend nunca llama al mock; ningún campo nuevo viaja al KYC. |
| `POST .../flows/{id}/kyc` (`documentType: TEST`, `document: TEST-...`) | Cuerpo y respuestas sin cambios (`ACTIVE`, `PENDING`, `REJECTED`, `503 KYC_UNAVAILABLE`, `409 SUBMISSION_UNKNOWN`). |
| `POST .../otp` y OTP `123456` | Sin cambios; sigue validándolo el backend. |
| `POST .../flows` y `POST .../decision` | Sin cambios. |
| Estados de wallet y resolución de `PENDING` en la vista de resultado | Sin cambios. |
| Eventos y logs sin OTP, teléfono ni documento | Se extiende a todos los datos de identificación. El frontend no emite eventos propios. |
| Contrato provisional con el backend | Solo se **añade** un endpoint (ver D5); requiere acuerdo de Backend/Arquitectura antes de la fase 4. |

## Componentes afectados

| Componente | Cambio |
|------------|--------|
| `onboarding.component.ts/.html` | Sigue siendo el contenedor: etapa, `busy`, llamadas a la API y errores. Se generaliza `steps` y la navegación; las plantillas actuales de OTP, KYC y resultado no se reescriben. |
| `onboarding.models.ts` | Se amplía `OnboardingStage` y se añaden tipos del payload de identidad. |
| `onboarding-api.service.ts` | Un método nuevo `submitIdentity`; los existentes no cambian. |
| Nuevos: `onboarding.validators.ts`, `onboarding.messages.ts`, `onboarding.masking.ts`, `onboarding.age.ts`, `onboarding.catalogs.ts` | Funciones puras y constantes. |
| Nuevos: componentes de paso `identity-step`, `profile-step`, `review-step` y un resumen de errores | Presentacionales: reciben el `FormGroup` por `input` y emiten eventos. |
| `src/styles.scss` | Estilos de los pasos nuevos y del stepper compacto; vigilar el presupuesto de build. |
| `*.spec.ts` | Pruebas de caracterización, unitarias de funciones puras y de componentes. |
| `dev-mock-backend.mjs` | Soporte local del endpoint nuevo, solo para ver la interfaz; no registra cuerpos. |
| `README.md` del frontend | Documentar pasos, datos de prueba y el contrato provisional. |

No cambian: `app.config.ts`, `main.ts`, `angular.json`, `proxy.conf.json`, `docs/openapi.json`, ningún backend.

## Decisions

### D1. Secuencia: cada paso agrupa campos, el contenedor sigue orquestando

Secuencia propuesta (6 etapas visibles): **Celular** (existente) → **Identificación** (DNI, fecha de nacimiento, correo, nombres y apellidos) → **Perfil y domicilio** (nacionalidad, domicilio, ocupación, PEP) → **Documento de prueba** (existente, `TEST-...`) → **Revisión y confirmación** (incluye consentimiento) → **Resultado** (existente).

Se mantiene el paso de documento `TEST-` separado porque el supuesto vigente es que la captura de DNI lo **complementa** (pregunta abierta 2 del proposal); así el comportamiento caracterizado del KYC no se toca. Los pasos son datos (`steps: StepInfo[]`) y todos los textos de progreso derivan de `steps.length`.

Alternativas: (a) una sola pantalla larga: peor en smartphone y sin progreso; (b) fusionar más campos por paso para acortar la secuencia: reduce toques pero concentra errores; se reevaluará con la medición de duración (R1); (c) enrutamiento por pasos: añade una dependencia de arquitectura que el alcance no exige.

### D2. Estado del formulario: un formulario raíz con grupos anidados, solo en memoria

Un `FormGroup` raíz (`identity`, `profile`, `kycDocument`, `consent`) vive en el componente contenedor mientras dura el flujo. `otpForm` se mantiene separado, porque el OTP y el teléfono no entran en la revisión ni en el payload de identidad. Navegar entre pasos no destruye los valores (conservación de datos válidos). Un paso inválido marca como tocados solo sus controles.

No se usan `localStorage`, `sessionStorage`, `IndexedDB`, cookies ni servicios singleton que retengan los datos tras destruir el componente.

Alternativas: un formulario por paso con copia a un servicio (duplica estado y riesgo de fugas), y persistir borradores para recuperación (contradice la spec de protección de datos).

### D3. Reglas en funciones puras, mensajes en un único diccionario

Validadores (DNI de 8 dígitos, correo con formato, fecha existente y no futura, nombres con caracteres admitidos, campos de domicilio no vacíos) y la edad derivada (`calculateAge(birthDate, today)` con `today` inyectable) son funciones puras sin dependencias de Angular. Los mensajes accionables están en un diccionario único `código de error -> texto` que alimentan tanto los mensajes en línea como el resumen de errores, para evitar divergencias.

La edad se calcula solo para mostrarla como dato informativo en la revisión; no se envía ni condiciona ningún avance. Los mensajes no repiten el valor ingresado.

Alternativas: validadores incrustados en el componente (difíciles de probar en los límites de cumpleaños) y librerías de validación externas (dependencia nueva sin necesidad).

### D4. Controles y catálogos mínimos

- Fecha de nacimiento con `<input type="date">` y `max` igual a la fecha actual; un valor vacío o inválido se trata como "ingresa una fecha válida". Alternativas: tres selectores o texto con máscara (más fricción o más errores de formato).
- DNI, celular y correo con `inputmode` y `type` adecuados; sin `autocomplete` para DNI y con `autocomplete` estándar donde corresponda en nombre, correo y dirección.
- Departamento, provincia, distrito y dirección como texto con validación de no vacío y longitud; no se incorpora un catálogo de ubigeo.
- Nacionalidad y ocupación como `<select>` alimentados por constantes **provisionales de demo** en `onboarding.catalogs.ts`, claramente marcadas para que Compliance las reemplace; no se presentan como lista oficial.
- PEP con dos opciones (sí/no) y sin selección inicial. Consentimiento con casilla sin marcar y texto tomado de una constante marcada "provisional de demo", reemplazable por el texto aprobado por Legal.

### D5. Envío: identidad primero, luego el KYC existente

Al confirmar en la revisión se valida el formulario completo y se ejecutan, en orden: (1) `PUT /api/onboarding/flows/{id}/identity` con el payload de identidad; (2) el `POST .../kyc` actual, sin cambios. Payload provisional: `dni`, `birthDate` (ISO `yyyy-mm-dd`), `email`, `firstNames`, `paternalSurname`, `maternalSurname`, `nationality`, `address` (`department`, `province`, `district`, `line`, `reference` opcional), `occupation`, `isPep`, `consentAccepted`.

- `PUT` es idempotente por flujo: un reintento tras un fallo de red no crea un segundo registro de identidad.
- Si el paso (1) falla, no se llama al KYC, se conservan los datos y se ofrece reintento; si el KYC devuelve `503`, solo se reintenta el KYC y no se reenvía la identidad (indicador interno `identitySent`).
- El resultado desconocido (`409 SUBMISSION_UNKNOWN`) conserva el comportamiento actual: sin reintento automático.
- Este endpoint es **propuesta del frontend**; Backend y Arquitectura deben aceptarlo o sustituirlo antes de la fase 4.

Alternativas: (a) ampliar el cuerpo de `.../kyc`: cambia un contrato caracterizado y volvería a enviar datos personales en cada reintento del KYC; (b) `POST` en lugar de `PUT`: riesgo de duplicados por reintentos; (c) capturar sin enviar nada hasta que exista la otra HU: deja la demo sin evidencia de envío y sin forma de cerrar la medición.

### D6. Revisión con edición dirigida

La revisión muestra los datos agrupados con DNI y correo enmascarados (funciones puras en `onboarding.masking.ts`), sin OTP, sin teléfono y sin etiquetas "verificado". Cada grupo ofrece "Editar", que lleva al paso correspondiente con los valores presentes y, al guardar, regresa a la revisión (indicador `returnToReview`). Los datos enmascarados se calculan al renderizar; no se almacenan.

### D7. Errores, foco y accesibilidad

Al intentar avanzar con errores se marca el paso como tocado, se muestra un resumen que lista cada error con su corrección y se lleva el foco al primer control inválido, que ya expone `aria-invalid`. Se conservan los mensajes en línea con `role="alert"` ya existentes. El estándar de accesibilidad exigible es una pregunta abierta; este diseño solo asegura etiquetas asociadas, foco gestionado, estados no comunicados solo por color y objetivos táctiles de al menos 44 px, sin declarar conformidad.

### D8. Stepper y presentación en 360 px

Con 6 etapas, el stepper con etiquetas no cabe en 360 px: en pantallas angostas se muestra "Paso N de 6" con el nombre del paso actual y puntos compactos; en pantallas amplias se conservan las etiquetas. Un aviso persistente "Demo: usa solo datos de prueba" aparece en los pasos que piden datos de identificación. Los estilos van a `src/styles.scss`; si el presupuesto de estilos o de bundle se acerca al límite se dividirá, no se subirá el umbral.

## Estrategia incremental

Cada fase es un conjunto de commits independiente y deja `npm test` y `npm run build` en verde.

0. **Puertas y caracterización.** Confirmar número de Issue, lista de Compliance y contrato `PUT .../identity` con Backend. Añadir las pruebas de caracterización de la sección siguiente, sin cambiar comportamiento.
1. **Fundaciones puras.** Validadores, edad, enmascarado, mensajes y catálogos provisionales, con pruebas unitarias (incluidos los límites de cumpleaños). Sin cambios de interfaz.
2. **Infraestructura de pasos (Bloque A).** `steps` guiado por datos, estado conservado al navegar, resumen de errores y foco, stepper compacto. La secuencia sigue siendo de 3 etapas; las pruebas de caracterización deben seguir verdes.
3. **Pasos de identificación.** Pasos de Identificación y de Perfil y domicilio con aviso de demo; secuencia ampliada y actualización deliberada de las pruebas que asumen "de 3".
4. **Revisión, consentimiento y envío.** Paso de revisión, orquestación de `PUT .../identity` seguido del KYC, reintentos separados. Requiere el contrato acordado y soporte en el stub local.
5. **Cierre.** Verificar protección de datos (sin almacenamiento ni consola), revisión manual responsive en 360 px y escritorio, `npm test`, `npm run build`, actualización del README y reconciliación con las tareas 2.5 y 3.x de `hu-01-onboarding-mvp`.

## Pruebas de caracterización necesarias antes de modificar comportamiento crítico

Las 20 pruebas actuales ya fijan el avance de etapas, validaciones, los resultados `ACTIVE`/`PENDING`/`REJECTED`, `KYC_UNAVAILABLE` con reintento, `SUBMISSION_UNKNOWN` sin reintento, resolución de revisión manual, doble envío y los cuerpos de las solicitudes. Antes de la fase 2 se añaden, **sobre el código actual y sin modificarlo**:

1. El flujo se recupera cuando falla `POST .../flows` al iniciar y funciona en el siguiente envío (`ensureFlow`).
2. Un error de red o inesperado durante el KYC muestra el estado recuperable, no un resultado KYC.
3. El reintento tras `503` reutiliza el mismo documento y no vuelve a validar el OTP.
4. Los botones de revisión manual quedan deshabilitados mientras la decisión está en curso.
5. El componente no usa `localStorage`, `sessionStorage` ni `document.cookie` en todo el recorrido (espías sobre `Storage`).
6. No se escribe en `console` durante el recorrido.
7. Cada control del flujo tiene etiqueta asociada y el paso actual expone `aria-current="step"`.

Cambios deliberados de pruebas existentes (la secuencia ampliada es parte del cambio aprobado): las aserciones de "Paso N de 3" y del recuento de pasos pasan a derivar de `steps.length`, y los atajos de prueba `goToKyc` recorren los pasos nuevos. Cualquier otro cambio en una prueba existente es una señal de regresión y debe justificarse.

## Risks / Trade-offs

- **[R1] Duración.** Seis etapas y más campos alejan la mediana de 3 minutos → medirla en la fase 5, evaluar fusionar pasos (D1) y reportarla sin afirmar impacto de lanzamiento.
- **[R2] Contrato inexistente.** `PUT .../identity` es una propuesta sin backend → no iniciar la fase 4 sin acuerdo; si se sustituye, el cambio queda acotado a `OnboardingApiService` y al orquestador.
- **[R3] Eventos de pasos.** Los pasos nuevos se completan en el cliente y el backend solo conoce el envío de identidad; la métrica de duración del bloque 3 debe acordar si el evento de envío exitoso ocurre al aceptar la identidad → coordinar al definir las tareas 3.x.
- **[R4] Datos sensibles en memoria y DOM.** Los valores existen en el heap y en los inputs; el autocompletado del navegador está fuera del control de la aplicación → minimizar `autocomplete` en DNI, no persistir ni registrar, usar solo datos ficticios en pruebas y en el stub.
- **[R5] Catálogos y texto legal provisionales.** Pueden confundirse con listas oficiales → rotularlos como de demo y aislarlos en un solo archivo.
- **[R6] Stepper en 360 px y presupuesto de estilos.** Más contenido en pantalla angosta y en `styles.scss` → probar en 360 px y vigilar el presupuesto.
- **[R7] Comportamiento de `<input type="date">`.** Varía entre navegadores y teclados → validar en la función pura y comprobar en dispositivo real.
- **[R8] Sensación de verificación.** Capturar DNI y correo sin verificarlos puede parecer una identificación real → aviso de demo persistente y ausencia de etiquetas "verificado".
- **[R9] Datos del stub.** El stub local podría registrar cuerpos con datos personales → no registrar cuerpos y usar solo datos ficticios.

## Migration Plan / Rollback

No hay migración de datos: todo vive en memoria del cliente y no hay datos productivos. Cada fase es aditiva y se revierte con `git revert` de sus commits; las fases 1 y 2 no cambian el comportamiento visible y pueden mantenerse si se revierten las posteriores. Si se revierte la fase 4, el flujo vuelve a terminar en el paso de documento y el KYC; si se revierte la fase 3, vuelve la secuencia base de 3 etapas de `hu-01-onboarding-mvp`. No se introduce bandera de configuración: duplicaría la matriz de pruebas por un riesgo que el control de fases ya cubre.

## Preguntas abiertas que bloquean fases

- Número del Issue y estado de aprobación de Compliance sobre los datos complementarios (fase 0).
- Aceptación del contrato `PUT .../identity` por Backend y Arquitectura (fase 4).
- Relación del DNI con el `document` `TEST-` si se decidiera reemplazar el paso de documento (fase 3).
- Estándar y nivel de accesibilidad exigibles (fase 5).
- Uso final de la edad: si no se necesita, se retira su visualización de la revisión (fase 4).
