# Spec Delta

## Purpose

Permite capturar en el onboarding de demo los datos que identifican a la persona (datos DAC y datos complementarios para una fintech regulada por la SBS), validando solo su formato, protegiéndolos en el cliente y dejando explícito qué no hace esta capacidad.

## ADDED Requirements

### Requirement: Captura de datos DAC con validación de formato
La interfaz SHALL solicitar el DNI, la fecha de nacimiento y el correo electrónico, SHALL validar únicamente su formato y SHALL bloquear el avance con un mensaje que indique cómo corregir cuando el dato falte o sea inválido.

#### Scenario: DNI con formato inválido
- **WHEN** el usuario intenta avanzar con un DNI que no tiene 8 dígitos numéricos
- **THEN** el avance se bloquea y el mensaje indica el formato esperado de 8 dígitos numéricos

#### Scenario: DNI con formato válido
- **WHEN** el usuario ingresa un DNI de 8 dígitos numéricos
- **THEN** el campo se acepta sin consultar ningún servicio de verificación ni de duplicados y sin etiquetarlo como verificado

#### Scenario: Correo mal escrito
- **WHEN** el usuario intenta avanzar con un correo sin `@` o sin dominio
- **THEN** el avance se bloquea y el mensaje muestra un ejemplo de formato válido

#### Scenario: Correo bien escrito
- **WHEN** el usuario ingresa un correo con formato válido
- **THEN** el campo se acepta, no se solicita ningún código y no se etiqueta como verificado

#### Scenario: Fecha de nacimiento inexistente o futura
- **WHEN** el usuario ingresa una fecha inexistente o posterior a la fecha actual
- **THEN** el avance se bloquea y el mensaje indica que debe ingresar una fecha de nacimiento válida

#### Scenario: Cualquier edad puede continuar
- **WHEN** el usuario ingresa una fecha de nacimiento válida que corresponde a una edad menor de 18 o mayor de 21 años
- **THEN** el avance no se bloquea por la edad y el resultado KYC no cambia por ella

#### Scenario: Cálculo de edad en fechas límite
- **WHEN** se calcula la edad a partir de una fecha de nacimiento cuyo cumpleaños es el mismo día, un día antes o un día después de la fecha actual
- **THEN** la edad calculada es la correcta para cada caso

#### Scenario: Datos obligatorios ausentes
- **WHEN** el usuario intenta avanzar con el DNI, la fecha de nacimiento o el correo vacíos
- **THEN** el avance se bloquea y cada campo ausente muestra un mensaje que indica que debe completarse

### Requirement: Captura de datos complementarios
La interfaz SHALL solicitar nombres, apellido paterno, apellido materno, nacionalidad, domicilio (departamento, provincia, distrito y dirección), ocupación o actividad económica declarada y la declaración sí/no de condición de persona expuesta políticamente (PEP), y SHALL bloquear el avance cuando un campo obligatorio falte o sea inválido. La lista final está sujeta a validación de Compliance.

#### Scenario: Campos complementarios obligatorios ausentes
- **WHEN** el usuario intenta avanzar sin completar un dato complementario obligatorio
- **THEN** el avance se bloquea y el mensaje indica qué dato falta y cómo completarlo

#### Scenario: Declaración PEP sin responder
- **WHEN** el usuario intenta avanzar sin responder la declaración PEP
- **THEN** el avance se bloquea y la declaración no tiene una opción seleccionada por defecto

#### Scenario: Referencia del domicilio opcional
- **WHEN** el usuario deja vacía la referencia del domicilio y completa los demás datos del domicilio
- **THEN** el avance no se bloquea por la referencia

#### Scenario: Nombres con caracteres no válidos
- **WHEN** el usuario ingresa en nombres o apellidos caracteres que no corresponden a un nombre de persona
- **THEN** el avance se bloquea y el mensaje indica los caracteres admitidos

### Requirement: Consentimiento explícito de términos y tratamiento de datos
La interfaz SHALL exigir la aceptación explícita de los términos y del tratamiento de datos antes del envío final, SHALL mostrarla sin marcar por defecto y SHALL obtener el texto de una fuente aprobada por Legal/Compliance.

#### Scenario: Aceptación sin marcar
- **WHEN** el usuario llega al paso de aceptación
- **THEN** la aceptación aparece sin marcar

#### Scenario: Envío sin aceptación
- **WHEN** el usuario intenta enviar sin aceptar los términos y el tratamiento de datos
- **THEN** el envío se bloquea y el mensaje indica que debe aceptarlos para continuar

#### Scenario: Texto legal no aprobado
- **WHEN** el texto de términos y tratamiento de datos aún no está aprobado por Legal/Compliance
- **THEN** la interfaz identifica el texto como provisional de demo y no lo presenta como contenido legal definitivo

### Requirement: Protección de los datos de identificación en el cliente
Los datos de identificación SHALL NOT persistir en almacenamiento del navegador tras cerrar o recargar la página, y SHALL NOT aparecer en logs ni en eventos de onboarding.

#### Scenario: Recarga del navegador
- **WHEN** el usuario cierra o recarga el navegador a mitad del flujo
- **THEN** los datos de identificación ingresados no están disponibles en almacenamiento persistente del navegador

#### Scenario: Eventos y logs sin datos de identificación
- **WHEN** el flujo registra eventos o logs durante la captura
- **THEN** ninguno contiene DNI, correo, nombres, domicilio, fecha de nacimiento, OTP, teléfono ni documento

### Requirement: Límites de la captura en la demo
La capacidad SHALL ser explícita en que opera con datos de prueba, SHALL NOT verificar el DNI ni el correo y SHALL NOT impedir un registro por duplicidad; esas capacidades corresponden a otra HU.

#### Scenario: Aviso de datos de prueba
- **WHEN** el usuario llega a un paso que solicita datos de identificación
- **THEN** la interfaz indica de forma visible que es una demo y que debe ingresar solo datos de prueba, no datos reales

#### Scenario: Sin consulta de duplicados ni verificación
- **WHEN** el usuario avanza con DNI y correo de formato válido
- **THEN** el frontend no consulta si existe un registro previo ni solicita un código de verificación

#### Scenario: Datos KYC de prueba preservados
- **WHEN** el flujo envía el documento para el KYC
- **THEN** el tipo de documento sigue siendo `TEST` y el documento conserva el prefijo `TEST-`, y no se envía al mock ninguno de los campos nuevos
