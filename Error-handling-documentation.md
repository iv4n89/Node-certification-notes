## Error

Los objetos de Error son lanzados cuando un error en tiempo de ejecución ocurre. El objeto Error puede también ser usado como base para excepciones creadas por el usuario. 

## Descripción

Errores en tiempo de ejecución resultan en un nuevo objeto de Error creado y lanzado.

Error es un objeto serializable, por lo que puede ser clonado con structuredClone() o copiado entre Workers usando postMessage().

## Tipos de Error

Más allá del constructor genérico de Error, hay otros core constructors en JS.

### EvalError

Los objetos EvalError indican un error relacionado con el método global eval(). Esta excepción ya no es lanzada por JS, pero se mantiene por compatibilidad.

EvalError es una subclase de Error

#### Propiedades de instancia

- name: Representa el nombre del tipo de error. En este caso 'EvalError'.

### RangeError

Indica un error cuando el valor no está dentro del rango permitido de valores.

Es lanzado cuando se trata de pasar un valor como argumento a una función que no permite un rango incluído en el valor pasado.

Se puede encontrar cuando:

- Pasamos un valor que no está permitido en String.prototype.normalize()
- Tratamos de crear un array de una longitud ilegal con el constructor de Array
- Pasamos valores numéricos no permitidos a Number.prototype.toExponential(), Number.prototype.toFixed() o Number.prototype.toPrecision().

#### Propiedades de instancia

- name: Representa el nombre del tipo de error. En este caso 'RangeError'.

### ReferenceError

Representa un error cuando una variable no existe (o no ha sido inicializada) en el scope referenciado.

#### Propiedades de instancia

- name: Representa el nombre del tipo de error. En este caso 'ReferenceError'

### SyntaxError

Representa un error cuando tratamos de interpretar sintácticamente código inválido. Es lanzado cuando el engine de JS encuentra tokens o un orden de éstos que no sigue la sintáxis del lenguaje cuando se está parseando el código.

#### Propiedades de instancia

- name: Referencia el nombre del tipo de error. En este caso 'SyntaxError'

### TypeError

Representa un error cuando una operación no pudo ser realizada, típicamente (pero no exclusivamente) cuando un valor no es del tipo esperado.

Un TypeError debe ser lanzado cuando:

- Un operando o argumento pasado a una función es incompatible con el tipo esperado por el operador o la función
- Se trata de modificar un valor que no puede ser modificado
- Se trata de usar un valor de una manera inapropiada

#### Propiedades de instancia

- name: Representa el nombre del tipo de error. En este caso 'TypeError'

### URIError

Representa un error cuando una función de manejo de URI global fue usado de una manera incorrecta.

#### Propiedades de instancia

- name: Representa el nombre del tipo de error. En este caso 'URIError'

### AggregateError

Representa un error cuando muchos errores necesitan ser envueltos en un simple error.l Es lanzado cuando múltiples errores necesitan ser reportados por un operando, por ejemplo Promise.any(), cuando todas las promesas pasadas son rechazadas.

#### Propiedades de instancia

- name: Representa el nombre del tipo de error. En este caso 'AggregateError'
- errors: Array de los errores que fueron agregados.

## Métodos estáticos

### Error.captureStackTrace()

Una función de V8 no estándard que crea una propiedad stack en una instancia de Error

### Error.stackTraceLimit

Una propiedad numérica no estándard de V8 que limita cuándos stack frames se incluirán en un error stacktrace

### Error.prepareStackTrace()

Una función no estándard de V8 que, si es provista por el código del usuario, es llamada por V8 para lanzar excepciones, permitiendo al usuario proveer formatos propios de stacktraces.

## Propiedades de instancia

- Error.prototype.name: Representa el nombre del tipo del error. El valor inicial es 'Error'. Las subclases lo cambian al suyo.
- Error.prototype.stack: Propiedad no estándard para stack trace.
- cause: Indica la razón por la cual el error es lanzado - usualmente otro error capturado. Para objetos Error creados por el usuario, éste es el valor provisto como la propiedad cause, segundo argumento del constructor.
- message: Mensaje del error. Primer argumento del constructor.
