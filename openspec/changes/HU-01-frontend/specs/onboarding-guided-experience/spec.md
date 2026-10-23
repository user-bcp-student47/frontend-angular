# Spec Delta

## Purpose

Mantiene guiado, corregible y utilizable en smartphone el onboarding ampliado con datos de identificación, y deja explícito qué comportamiento de HU-01 se preserva.

## ADDED Requirements

### Requirement: Secuencia guiada ampliada
La interfaz SHALL mostrar todos los pasos requeridos, incluidos los de captura de datos de identificación, y SHALL señalar el paso actual durante todo el recorrido.

#### Scenario: Usuario recorre los pasos ampliados
- **WHEN** el usuario inicia el onboarding
- **THEN** la interfaz muestra todos los pasos requeridos, incluidos los de identificación, y señala el paso actual

#### Scenario: Avance del paso actual
- **WHEN** el usuario completa un paso con datos válidos
- **THEN** el indicador de progreso pasa al siguiente paso

### Requirement: Revisión previa al envío
La interfaz SHALL mostrar una revisión con todos los datos ingresados antes del envío final, SHALL enmascarar parcialmente el DNI y el correo, SHALL omitir el OTP y SHALL permitir corregir cada dato sin perder los demás.

#### Scenario: Revisión con datos enmascarados
- **WHEN** el usuario llega a la revisión final
- **THEN** ve los datos ingresados, con DNI y correo parcialmente enmascarados, y no ve el OTP

#### Scenario: Corrección desde la revisión
- **WHEN** el usuario edita un dato desde la revisión
- **THEN** puede corregirlo y regresar a la revisión con los demás datos intactos

#### Scenario: Sin etiqueta de verificado
- **WHEN** se muestran DNI y correo en la revisión
- **THEN** la interfaz no los etiqueta como verificados

### Requirement: Conservación de datos válidos
La interfaz SHALL conservar los datos válidos ya ingresados cuando el usuario regresa a un paso anterior o cuando un intento de avance falla, dentro de la misma sesión de flujo.

#### Scenario: Regreso a un paso anterior
- **WHEN** el usuario regresa a un paso anterior y vuelve al siguiente
- **THEN** los datos válidos ingresados siguen presentes

#### Scenario: Avance bloqueado por otro dato
- **WHEN** el usuario intenta avanzar con un dato inválido y otros válidos
- **THEN** los datos válidos se conservan y solo se señalan los inválidos

### Requirement: Resumen de errores accionable
Cuando existan varios datos inválidos, la interfaz SHALL bloquear el avance, SHALL listar los errores con su corrección y SHALL llevar el foco al primer campo inválido.

#### Scenario: Varios errores a la vez
- **WHEN** el usuario intenta avanzar con varios datos obligatorios ausentes o inválidos
- **THEN** el avance se bloquea, se listan los errores con cómo corregirlos y el foco pasa al primer campo inválido

### Requirement: Usabilidad en smartphone y recuperación ante errores de red
La interfaz SHALL ser utilizable en una pantalla de 360 px de ancho y SHALL ofrecer una acción de reintento ante un fallo de red sin presentar un resultado KYC. El estándar de accesibilidad exigible está pendiente de confirmación.

#### Scenario: Pantalla de 360 px
- **WHEN** se recorre el flujo en una pantalla de 360 px de ancho
- **THEN** no hay desplazamiento horizontal, los controles táctiles miden al menos 44 px y cada campo tiene una etiqueta asociada

#### Scenario: Teclado adecuado por campo
- **WHEN** el usuario enfoca el campo de DNI, de celular o de correo en un smartphone
- **THEN** se ofrece el tipo de teclado correspondiente al dato

#### Scenario: Fallo de red
- **WHEN** ocurre un fallo de red en cualquier paso
- **THEN** el usuario ve un mensaje con una acción de reintento, conserva sus datos y no se presenta ningún resultado KYC

#### Scenario: Envío en curso
- **WHEN** el usuario envía y la solicitud sigue en curso
- **THEN** la interfaz muestra un estado de carga y no permite un segundo envío

### Requirement: Preservación del comportamiento de HU-01
El flujo ampliado SHALL preservar el OTP de demo, los resultados KYC, la resolución de revisión manual, la medición de duración y los límites de datos de prueba definidos en HU-01.

#### Scenario: OTP de demo
- **WHEN** el usuario envía el código `123456` para un celular de prueba
- **THEN** el backend lo valida, se permite continuar y no se envía ningún SMS; un código distinto no avanza el flujo

#### Scenario: Resultado aprobado
- **WHEN** el backend informa el estado `ACTIVE`
- **THEN** la interfaz informa la aprobación y presenta la wallet como activa

#### Scenario: Revisión manual pendiente
- **WHEN** el backend informa el estado `PENDING`
- **THEN** la interfaz informa que la revisión sigue pendiente, no presenta la wallet como activa y permite resolverla desde la misma vista de resultado con `ACTIVE` o `REJECTED`, sin consola independiente

#### Scenario: Resultado rechazado
- **WHEN** el backend informa el estado `REJECTED`
- **THEN** la interfaz informa el rechazo y no presenta la wallet como activa

#### Scenario: Proveedor KYC no disponible
- **WHEN** el backend informa que el proveedor KYC no está disponible
- **THEN** la interfaz informa que no pudo completar la verificación, no presenta la wallet como activa y permite reintentar

#### Scenario: Resultado de envío desconocido
- **WHEN** el backend informa que no pudo confirmar el resultado del envío
- **THEN** la interfaz no ofrece un reintento automático y no presenta la wallet como activa

#### Scenario: Medición de duración
- **WHEN** se reporta la duración del onboarding ampliado
- **THEN** se mide desde el inicio del flujo hasta el envío correcto de todos los pasos, sin incluir la revisión KYC ni la activación, y se compara con la referencia de 3 minutos sin afirmar impacto de lanzamiento

#### Scenario: Funcionalidad fuera de alcance
- **WHEN** se completa el onboarding de la demo
- **THEN** no se crea ni procesa un pago y no se reportan MAU ni CAC
