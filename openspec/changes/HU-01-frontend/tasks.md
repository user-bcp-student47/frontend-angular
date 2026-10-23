# Tasks

Unidad de trabajo de producto: [`docs/alcance/HU-01-frontend.md`](../../../docs/alcance/HU-01-frontend.md). Contrato observable: `specs/onboarding-identity-capture/spec.md` y `specs/onboarding-guided-experience/spec.md`. Decisiones: `design.md` (D1-D8).

Reglas de ejecución:
- Cada fase termina con la suite de pruebas y `npm run build` en verde, y es reversible por separado (`design.md`, Migration Plan).
- Las pruebas de caracterización (fase 0) se escriben sobre el código actual y deben pasar sin modificar código de producción.
- Cualquier cambio a una prueba existente fuera de los listados en 3.4 es señal de regresión y debe justificarse.
- Todos los valores de prueba, mocks y stubs son ficticios.

## Divergencias entre `design.md` y el repositorio

`design.md` se redactó contra `output/frontend-angular/`; este repositorio difiere. Las tareas usan el estado real. Confirmar la divergencia en 0.1 antes de empezar.

| `design.md` asume | Estado real observado | Efecto en las tareas |
|-------------------|-----------------------|----------------------|
| `npm test` con 20 pruebas Karma/Jasmine | No hay script `test`, target `test` en `angular.json`, `tsconfig.spec.json` ni dependencias Karma/Jasmine; el spec existente tiene 10 pruebas | Tarea 0.2 habilita el runner antes de caracterizar |
| Señales (`stage`, `busy`) y `@switch` | Propiedades de clase, `stage: 'intro' \| 'otp' \| 'kyc' \| 'result' \| 'finished'`, `*ngIf` por etapa, `requestInProgress` | Las tareas usan los nombres reales |
| `POST .../kyc` | `POST .../submit` (`documentType`, `document`) | "KYC existente" = `submitKyc` hacia `/submit` |
| Estilos en `src/styles.scss` con presupuesto de 2/4 kB | Estilos en `onboarding.component.scss`; `src/styles.css` global; sin presupuestos en `angular.json` | Se vigila el tamaño de build sin asumir el presupuesto |
| `onboarding.models.ts`, `dev-mock-backend.mjs`, `README.md` existentes | No existen | Se crean solo si la tarea lo exige y está aprobado en `design.md` |
| Resultado desconocido, `PENDING` y rechazo cubiertos por pruebas | Solo hay pruebas de `ACTIVE`, `PENDING` (tras reintento) y `KYC_UNAVAILABLE` | Se añaden en la fase 0 |
| Cierre de flujo y reporte de mediana no mencionados | Existen `finishFlow`, `closeWithBeacon` y `getReport` | Se preservan sin cambios y se protegen en la fase 0 |

## 0. Puertas y pruebas de caracterización

- [ ] 0.1 **Puerta de entrada.** Registrar en este archivo el número del GitHub Issue (hoy PENDIENTE en `proposal.md`) y la respuesta de Compliance sobre la lista de datos complementarios; actualizar la tabla de divergencias si el estado real cambió. Verificación: ambos datos figuran por escrito antes de iniciar la fase 1; sin ellos, no continuar.
- [ ] 0.2 Habilitar el runner de pruebas (Karma/Jasmine en `ChromeHeadless`, compatible con Angular 17.3): script `test`, target `test` en `angular.json`, `tsconfig.spec.json` y dependencias de desarrollo necesarias, sin tocar `src/app/`. Verificación: `npm test` ejecuta las 10 pruebas existentes en verde sin depender de backend ni mock, y `npm run build` sigue compilando.
- [ ] 0.3 Caracterizar el inicio: si `POST /api/onboarding/flows` falla, se muestra `globalError`, `requestInProgress` vuelve a `false` y un nuevo `startFlow()` crea el flujo. Verificación: prueba nueva en verde sobre el código actual.
- [ ] 0.4 Caracterizar el OTP: un `OTP_INVALID` del backend deja la etapa en `otp`, muestra el mensaje de código no coincidente y conserva el valor; un error de red muestra `globalError` y no avanza. Verificación: pruebas nuevas en verde.
- [ ] 0.5 Caracterizar el envío KYC: el cuerpo enviado a `/submit` es solo `documentType: 'TEST'` y `document` recortado; un error de red o inesperado muestra un mensaje recuperable, mantiene la etapa `kyc` y no presenta resultado KYC; el reintento tras `503` reutiliza el mismo documento y no llama de nuevo a `/otp`. Verificación: pruebas nuevas en verde.
- [ ] 0.6 Caracterizar los resultados: `REJECTED` y `SUBMISSION_UNKNOWN` no presentan la wallet como activa; `SUBMISSION_UNKNOWN` no ofrece reintento automático (ninguna solicitud adicional a `/submit`); `ACTIVE` es el único estado con `walletActive` verdadero. Verificación: pruebas nuevas en verde.
- [ ] 0.7 Caracterizar la revisión manual: desde `PENDING`, `resolveDecision('ACTIVE' | 'REJECTED')` actualiza el estado visible desde la misma vista; mientras la decisión está en curso no se emiten solicitudes duplicadas y los botones de resolución quedan deshabilitados; no hay consola independiente. Verificación: pruebas nuevas en verde.
- [ ] 0.8 Caracterizar cierre y reporte: `finishFlow()` conserva el comportamiento actual de éxito y de error (`closeError`, sin afirmar registro) y el reporte de mediana pasa por `loading`, `available`, `empty` y `error`. Verificación: pruebas nuevas en verde.
- [ ] 0.9 Caracterizar la protección de datos: durante un recorrido completo (inicio, OTP, KYC, resultado) no se invoca `Storage.prototype.setItem` de `localStorage` ni `sessionStorage`, no se escribe `document.cookie` y no se llama a `console.log|info|warn|error|debug`. Verificación: prueba con espías en verde.
- [ ] 0.10 Caracterizar accesibilidad base: cada control visible del flujo tiene etiqueta asociada, el paso actual expone `aria-current="step"` y el paso mostrado coincide con `currentStep`. Verificación: prueba de DOM en verde.
- [ ] 0.11 Revisar las pruebas existentes y las nuevas: las que fijan estado mediante asignación directa de propiedades (`stage`, `flowId`) quedan documentadas como candidatas a cambio deliberado en 3.4; ninguna otra se marca como modificable. Verificación: lista registrada en el mensaje del commit o en este archivo.

## 1. Fundaciones puras (sin cambios de interfaz)

- [ ] 1.1 Crear `onboarding.validators.ts` con validadores de DNI (8 dígitos numéricos), correo (formato válido, con `@` y dominio), fecha de nacimiento (existente y no futura), nombres y apellidos (caracteres admitidos), texto obligatorio con longitud, y reglas de domicilio con referencia opcional. Verificación: pruebas unitarias sin DOM para cada regla, incluidos valores vacíos, solo espacios, DNI de 7 y 9 dígitos o con letras, correo sin `@` o sin dominio, fecha `30 de febrero` y fecha futura.
- [ ] 1.2 Crear `onboarding.age.ts` con `calculateAge(birthDate, today)` y `today` inyectable. Verificación: pruebas del cumpleaños el mismo día, un día antes y un día después, año bisiesto (29 de febrero) y menores de 18 o mayores de 21 sin que ninguna edad bloquee.
- [ ] 1.3 Crear `onboarding.masking.ts` con enmascarado de DNI y correo (solo últimos caracteres del DNI; parte local del correo parcial, dominio visible). Verificación: pruebas con DNI y correos de longitud mínima y larga; el resultado nunca contiene el valor completo.
- [ ] 1.4 Crear `onboarding.messages.ts` con el diccionario único `código de error -> mensaje accionable`: DNI indica "8 dígitos numéricos", correo muestra ejemplo de formato válido, fecha indica ingresar una fecha de nacimiento válida, cada obligatorio indica qué completar, nombres indican caracteres admitidos, PEP y consentimiento indican qué hacer. Verificación: prueba que cada código de error de 1.1 tiene mensaje y que ningún mensaje incluye el valor ingresado.
- [ ] 1.5 Crear `onboarding.catalogs.ts` con nacionalidades y ocupaciones **provisionales de demo** y el texto de términos y tratamiento de datos **provisional de demo**, rotulados como tales y como reemplazables por Compliance/Legal. Verificación: prueba de que cada constante lleva el rótulo provisional.
- [ ] 1.5.1 Verificar la fase: `npm test` y `npm run build` en verde, sin diferencias en la interfaz ni en las pruebas de la fase 0.

## 2. Infraestructura de pasos (Bloque A, secuencia aún de 3 etapas)

- [ ] 2.1 Generalizar el stepper a datos (`steps: StepInfo[]`) y derivar de `steps.length` los textos "Paso N de M" y el recuento de pasos, sin añadir etapas. Verificación: las pruebas de la fase 0 siguen en verde y el stepper muestra los mismos 3 pasos.
- [ ] 2.2 Conservar los valores válidos al navegar entre pasos (A2): el estado de formularios vive en el contenedor mientras dura el flujo, sin servicios singleton ni almacenamiento del navegador. Verificación: prueba que regresar a un paso anterior y volver conserva los valores válidos; la prueba de 0.9 sigue en verde.
- [ ] 2.3 Marcar como tocados solo los controles del paso actual cuando un avance falla (A3). Verificación: prueba que un intento fallido no muestra errores en pasos no visitados y conserva los valores válidos.
- [ ] 2.4 Crear el componente presentacional de resumen de errores que lista cada error con su corrección usando `onboarding.messages.ts` y que lleva el foco al primer control inválido (extender `focusFirstInvalid`). Verificación: prueba con varios errores simultáneos (bloqueo, lista con correcciones, foco en el primer campo inválido) y que los mensajes en línea existentes (`role="alert"`, `aria-invalid`, `aria-describedby`) no cambian.
- [ ] 2.5 Mostrar el estado de carga y bloquear el segundo envío en cada paso que llama al backend (A5), reutilizando `requestInProgress`. Verificación: prueba que un segundo envío con la solicitud en curso no emite solicitudes y que se muestra el indicador de carga.
- [ ] 2.6 Ofrecer en cada paso una acción de reintento ante fallo de red que conserve los datos y no muestre resultado KYC (A5). Verificación: pruebas por paso con error de red simulado.
- [ ] 2.7 Añadir stepper compacto para 360 px ("Paso N de M" con el nombre del paso actual y puntos) y mantener las etiquetas en pantallas amplias, con objetivos táctiles de al menos 44 px. Verificación: revisión manual a 360 px sin scroll horizontal; actualizar `onboarding.component.responsive-checklist.ts` solo con lo realmente ejecutado.
- [ ] 2.7.1 Verificar la fase: `npm test` y `npm run build` en verde; todas las pruebas de caracterización pasan sin modificación.

## 3. Pasos de identificación y perfil

- [ ] 3.1 Crear el paso **Identificación** (presentacional, recibe el `FormGroup` por `input`): DNI (`inputmode="numeric"`, sin `autocomplete`), fecha de nacimiento (`type="date"` con `max` igual a la fecha actual), correo (`type="email"`, `autocomplete="email"`), nombres, apellido paterno y apellido materno (`autocomplete` estándar), con etiquetas visibles asociadas y aviso persistente "Demo: usa solo datos de prueba". Verificación: pruebas de bloqueo por cada campo vacío o inválido con su mensaje, aceptación de valores válidos, y que no se muestra "verificado" ni se consulta servicio alguno ni se solicita código.
- [ ] 3.2 Crear el paso **Perfil y domicilio**: nacionalidad y ocupación (`<select>` con catálogos de 1.5), departamento, provincia, distrito y dirección (texto), referencia opcional, declaración PEP sí/no sin selección inicial, con el mismo aviso de demo. Verificación: pruebas de bloqueo por obligatorio ausente, PEP sin responder con ninguna opción seleccionada por defecto, y que la referencia vacía no bloquea.
- [ ] 3.3 Ampliar la secuencia a **Celular -> Identificación -> Perfil y domicilio -> Documento de prueba -> Revisión y confirmación -> Resultado**, con las plantillas existentes de celular, documento y resultado sin reescribir, y con el paso de revisión como marcador hasta la fase 4. Verificación: el stepper muestra todos los pasos y señala el actual; avanzar solo ocurre con datos válidos; el paso de documento conserva `documentType: 'TEST'`, prefijo `TEST-` y el cuerpo de `/submit` sin campos nuevos.
- [ ] 3.4 Actualizar de forma deliberada solo las pruebas que asumen "Paso N de 3", el recuento de pasos y los atajos que fijan `stage` para recorrer los pasos nuevos (lista de 0.11). Verificación: el resto de pruebas de la fase 0 pasan sin cambios; cada modificación queda justificada.
- [ ] 3.5 Verificar que la identidad no viaja al KYC: el único cuerpo enviado a `/submit` sigue siendo `{ documentType, document }` y no hay solicitudes al mock KYC desde el frontend. Verificación: prueba con `HttpTestingController` sobre el recorrido completo.
- [ ] 3.5.1 Verificar la fase: `npm test` y `npm run build` en verde; revisión manual a 360 px de los pasos nuevos (etiquetas, teclado por campo, errores y foco).

## 4. Revisión, consentimiento y envío

> **Puerta:** no iniciar sin que Backend y Arquitectura acepten o sustituyan el contrato provisional `PUT /api/onboarding/flows/{id}/identity` (`design.md`, D5 y R2). Registrar la decisión aquí.

- [ ] 4.0 Registrar la aceptación o sustitución del contrato de identidad por Backend y Arquitectura, y la respuesta sobre el uso final de la edad (si no se necesita, no se muestra en la revisión). Verificación: decisión escrita; si el contrato cambia, el alcance se limita a `OnboardingApiService` y al orquestador.
- [ ] 4.1 Añadir `submitIdentity(flowId, payload)` a `OnboardingApiService` (`PUT .../identity`, campos de D5: `dni`, `birthDate` ISO, `email`, `firstNames`, `paternalSurname`, `maternalSurname`, `nationality`, `address`, `occupation`, `isPep`, `consentAccepted`) y tipos del payload. Verificación: prueba del servicio de método, URL y cuerpo; los métodos existentes no cambian.
- [ ] 4.2 Crear el paso **Revisión y confirmación** con los datos agrupados, DNI y correo enmascarados (1.3) calculados al renderizar sin almacenarlos, la edad solo informativa si 4.0 la mantiene, sin OTP, sin teléfono y sin etiquetas "verificado". Verificación: pruebas de DOM de enmascarado, ausencia de OTP y teléfono, y ausencia de la palabra "verificado" en DNI y correo.
- [ ] 4.3 Añadir la edición dirigida: "Editar" por grupo lleva al paso correspondiente con los valores presentes y, al guardar, vuelve a la revisión (`returnToReview`). Verificación: prueba de que editar un dato y regresar deja los demás intactos.
- [ ] 4.4 Añadir la casilla de aceptación de términos y tratamiento de datos, sin marcar por defecto y con el texto provisional de 1.5 rotulado como tal. Verificación: pruebas de estado inicial sin marcar, bloqueo del envío sin aceptar con mensaje accionable, y visibilidad del rótulo provisional.
- [ ] 4.5 Orquestar el envío en la confirmación: validar el formulario completo; ejecutar `PUT .../identity` y, solo si resulta correcto, el envío KYC existente (`submitKyc` hacia `/submit`, sin cambios). Verificación: pruebas de orden de llamadas, de que un fallo de identidad no llama al KYC y conserva los datos con acción de reintento, y de que no se emiten solicitudes duplicadas con el envío en curso.
- [ ] 4.6 Separar los reintentos con un indicador interno `identitySent`: ante `503 KYC_UNAVAILABLE` solo se reintenta el KYC sin reenviar la identidad; `SUBMISSION_UNKNOWN` conserva el comportamiento actual sin reintento automático. Verificación: pruebas de ambos casos; las pruebas de la fase 0 de resultados siguen en verde.
- [ ] 4.7 Si se necesita ver la interfaz sin backend, crear un stub local mínimo para `PUT .../identity` que no registre cuerpos y use solo datos ficticios. Verificación: el stub no escribe cuerpos en consola ni disco; omitir la tarea si Backend ya ofrece el endpoint.
- [ ] 4.7.1 Verificar la fase: `npm test` y `npm run build` en verde; los resultados `ACTIVE`, `PENDING`, `REJECTED`, `KYC_UNAVAILABLE` y `SUBMISSION_UNKNOWN` se comportan igual que en la fase 0.

## 5. Cierre y verificación

- [ ] 5.1 Verificar la protección de datos con el flujo completo ampliado: la prueba 0.9 sigue en verde con todos los campos nuevos, y una búsqueda en el código confirma que no se registran ni se almacenan DNI, correo, nombres, domicilio, fecha de nacimiento, OTP, teléfono ni documento.
- [ ] 5.2 Revisión manual responsive en 360 px y escritorio: sin scroll horizontal, etiquetas asociadas, objetivos táctiles de al menos 44 px, foco visible, teclado adecuado por campo, aviso de demo visible y errores con foco al primer campo inválido. Registrar solo lo realmente ejecutado en `onboarding.component.responsive-checklist.ts`; el estándar de accesibilidad exigible sigue pendiente y no se declara conformidad.
- [ ] 5.3 Medir y reportar la duración del recorrido ampliado frente a la referencia de 3 minutos, sin afirmar impacto de lanzamiento; si se aleja, documentar la opción de fusionar pasos (D1) sin aplicarla. Coordinar con las tareas 3.x de `hu-01-onboarding-mvp` el momento del evento de envío correcto (R3); no implementar eventos ni medición aquí.
- [ ] 5.4 Documentar en un README del frontend: pasos, datos de prueba, aviso de demo y contrato provisional de identidad. Verificación: los comandos documentados se ejecutan tal como están escritos.
- [ ] 5.5 Reconciliar con `hu-01-onboarding-mvp`: actualizar su `spec.md`, `design.md` y `tasks.md` (secuencia de 3 etapas y tarea 2.5) o dejar la reconciliación anotada para el archivo de ambos cambios; revisar el solapamiento con `wallet-onboarding`. Verificación: nota de reconciliación registrada.
- [ ] 5.6 Ejecutar `npm test` y `npm run build`, registrar los resultados reales y corregir cualquier fallo; confirmar que `docs/openapi.json` no cambió. Verificación: salidas registradas.

## Trazabilidad

| Requisito de las specs | Tareas |
|------------------------|--------|
| Captura de datos DAC con validación de formato | 1.1, 1.2, 1.4, 3.1 |
| Captura de datos complementarios | 1.1, 1.4, 1.5, 3.2 |
| Consentimiento explícito | 1.5, 4.4 |
| Protección de datos en el cliente | 0.9, 2.2, 5.1 |
| Límites de la captura en la demo | 3.1, 3.2, 3.5 |
| Secuencia guiada ampliada | 2.1, 3.3 |
| Revisión previa al envío | 1.3, 4.2, 4.3 |
| Conservación de datos válidos | 2.2, 2.3 |
| Resumen de errores accionable | 2.4 |
| Usabilidad en smartphone y errores de red | 0.10, 2.5, 2.6, 2.7, 5.2 |
| Preservación del comportamiento de HU-01 | 0.3-0.8, 3.5, 4.6, 4.7.1 |
| Medición de duración (solo reporte) | 5.3 |

## Fuera de esta lista (no aprobado)

Verificación de DNI o correo, detección de duplicados, restricciones por edad, catálogos oficiales (ubigeo, países, ocupaciones), textos legales definitivos, enrutamiento, estado global, librerías de UI, pruebas e2e, eventos y medición de duración (`hu-01-onboarding-mvp`, bloque 3), pagos, MAU y CAC.
