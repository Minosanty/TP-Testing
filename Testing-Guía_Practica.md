# Laboratorio introductorio: pruebas automatizadas en Express.js

**Taller de Lenguajes de Programación IV · Unidad 4 · Octubre de 2026**

## Propósito

En esta práctica se utilizará una API de precios desarrollada con Express.js y TypeScript para comprobar comportamientos mediante pruebas automatizadas. El proyecto ya contiene pruebas: algunas pasan y una falla intencionalmente. El objetivo es comprender los resultados, localizar el defecto y verificar una corrección.

Al finalizar podrás distinguir una prueba unitaria de una prueba de integración, interpretar aserciones de Vitest y comprobar respuestas HTTP con Supertest.

## 1. Preparar el entorno

Necesitás Node.js **22.18 o posterior** (preferentemente Node.js 24), npm, un editor de código y una terminal. El proyecto usa TypeScript ejecutado directamente por Node, sin `ts-node` ni `tsx`.

Abrí la carpeta **demo_practica** en Visual Studio Code y ejecutá:

```bash
node --version
npm --version
npm install
```

El comando `npm install` instala las dependencias y genera `package-lock.json`. Si tu versión de Node es anterior a la indicada, actualizala antes de continuar.

La estructura del proyecto es:

```text
├── src/
│   ├── app.ts
│   ├── server.ts
│   └── services/
│       └── precios.ts
├── tests/
│   ├── precios.test.ts
│   └── api.test.ts
├── .github/
│   └── workflows/
│       └── pruebas.yml
├── package.json
└── tsconfig.json
```

**Importante:** no es necesario crear nuevos archivos de configuración ni reemplazar `package.json`. Los scripts y las dependencias ya están incluidos.

## 2. Conocer la aplicación

Abrí `src/services/precios.ts`. Allí se encuentra la función `calcularPrecioFinal(precio, descuento)`, que resta un porcentaje al precio recibido. Por el momento, la función **no valida** los valores de entrada.

En `src/app.ts` hay dos rutas:

- `GET /healthy`: devuelve un mensaje que confirma que la API responde.
- `POST /api/precios/calcular`: recibe `precio` y `descuento` en formato JSON y devuelve el total calculado.

Iniciá el servidor:

```bash
npm run dev
```

Abrí `http://localhost:3000/healthy` en el navegador. Deberías ver:

```json
{"estado":"ok"}
```

En otra terminal probá el cálculo con `curl`:

```bash
curl -X POST http://localhost:3000/api/precios/calcular \
  -H 'Content-Type: application/json' \
  -d '{"precio":1000,"descuento":10}'
```

Resultado esperado:

```json
{"total":900}
```

En **PowerShell** también podés utilizar:

```powershell
Invoke-RestMethod -Uri 'http://localhost:3000/api/precios/calcular' `
  -Method Post -ContentType 'application/json' `
  -Body '{"precio":1000,"descuento":10}'
```

**Preguntas:** ¿qué archivo contiene la regla de cálculo? ¿Qué archivo define la ruta HTTP? ¿En qué se diferencian sus responsabilidades?

- **Regla de cálculo:** `src/services/precios.ts`. Contiene la lógica de dominio puro (operaciones matemáticas y validaciones del negocio), independiente del protocolo HTTP.
- **Ruta HTTP:** `src/app.ts`. Define los endpoints de Express, recibe y deserializa las solicitudes HTTP, delega el cálculo al servicio y transforma el resultado o las excepciones en códigos de estado HTTP apropiados (200, 400, 500).
- **Diferencia de responsabilidades:** Separación de conceptos (*Separation of Concerns*). El servicio es reutilizable y testeable de manera aislada sin levantar un servidor web, mientras que el controlador HTTP se encarga exclusivamente de la comunicación con la red y el cliente.

## 3. Ejecutar las pruebas disponibles

Detené el servidor con `Ctrl+C` o abrí otra terminal en el proyecto. Ejecutá:

```bash
npm test
```

El proyecto incluye **cinco pruebas**: tres en `tests/precios.test.ts` y dos en `tests/api.test.ts`. Inicialmente, deberían **pasar cuatro y fallar una**.

En `tests/precios.test.ts` identificá estas instrucciones:

- `describe(...)`: agrupa pruebas relacionadas.
- `it(...)`: describe un comportamiento que se quiere comprobar.
- `expect(...)`: expresa una expectativa sobre el resultado.
- `toBe(...)`: comprueba igualdad para valores como números.
- `toThrow(RangeError)`: comprueba que una función lance un error del tipo indicado.

Leé la prueba que contiene `calcularPrecioFinal(1000, 150)`. ¿Qué comportamiento exige? ¿Qué hace realmente el código? **Todavía no modifiques la implementación.**

## 4. Interpretar la falla

La prueba inicial incluye la siguiente comprobación:

```typescript
it('rechaza descuentos superiores al 100 %', () => {
  expect(() => calcularPrecioFinal(1000, 150)).toThrow(RangeError);
});
```

Observá que la función se escribe dentro de `() => ...` para que Vitest pueda ejecutarla y comprobar si lanza un error. Es distinto de pasar directamente el resultado de la función a `expect`.

Respondé antes de continuar:

1. ¿Qué total devuelve actualmente un descuento del 150 % sobre $1.000?
   - **Respuesta:** Devuelve `-500` (`1000 - (1000 * 150) / 100 = -500`).
2. ¿Por qué ese resultado no cumple la regla de negocio?
   - **Respuesta:** Porque en una venta comercial un descuento no puede superar el 100 % ni generar un precio final negativo (un negocio no le paga dinero al cliente por llevarse un producto).
3. ¿Por qué la prueba falla si la función terminó de ejecutarse?
   - **Respuesta:** Porque la prueba exige `toThrow(RangeError)`. Vitest espera que la ejecución se interrumpa lanzando una excepción `RangeError`, pero como la función original no validaba nada, terminó de ejecutarse con normalidad devolviendo un número.

## 5. Corregir la función de cálculo

Modificá **solo** `src/services/precios.ts`. Antes de realizar la operación, incorporá comprobaciones para que:

- El precio sea un número finito, igual o mayor que cero.
- El descuento sea un número finito entre 0 y 100, inclusive.
- Los valores inválidos provoquen un `RangeError`.

Podés utilizar `Number.isFinite(...)`, comparaciones e instrucciones `throw new RangeError('...')`. **Conservá la fórmula de cálculo para las entradas válidas.**

Volvé a ejecutar:

```bash
npm test
```

Ahora deberían pasar las **cinco pruebas**. Si alguna falla, leé el resultado de Vitest y revisá qué comportamiento espera.

## 6. Agregar pruebas unitarias

Abrí `tests/precios.test.ts` y agregá, dentro de `describe`, nuevas pruebas para las siguientes reglas:

| Entrada | Resultado esperado |
| --- | --- |
| Precio 1.000 y descuento 100 % | Total 0 |
| Precio -100 y descuento 10 % | Lanza `RangeError` |
| Precio 1.000 y descuento -10 % | Lanza `RangeError` |

Por ejemplo, para el caso límite:

```typescript
it('devuelve cero con un descuento del 100 %', () => {
  expect(calcularPrecioFinal(1000, 100)).toBe(0);
});
```

Escribí las otras dos pruebas siguiendo el modelo de `toThrow(RangeError)` que ya existe. Ejecutá `npm test` y verificá el resultado.

**Desafío adicional:** agregá una prueba para `NaN` como precio. ¿La validación actual lo rechaza?

## 7. Primera prueba de integración HTTP

Hasta ahora se probó la función de manera aislada. Ahora comprobarás cómo colaboran **la ruta HTTP de Express y el servicio de cálculo**.

Abrí `tests/api.test.ts`. Las pruebas existentes utilizan Supertest:

```typescript
const respuesta = await request(app)
  .post('/api/precios/calcular')
  .send({ precio: 1000, descuento: 10 });

expect(respuesta.status).toBe(200);
expect(respuesta.body).toEqual({ total: 900 });
```

La llamada `request(app)` permite probar la aplicación sin iniciar manualmente un servidor en el puerto 3000. El código `200` significa que la petición se procesó correctamente.

Agregá dentro del bloque `describe('API de precios', ...)` esta nueva prueba:

```typescript
it('responde HTTP 400 ante un descuento inválido', async () => {
  const respuesta = await request(app)
    .post('/api/precios/calcular')
    .send({ precio: 1000, descuento: 150 });

  expect(respuesta.status).toBe(400);
  expect(respuesta.body).toHaveProperty('error');
});
```

El código `400` indica que la petición contiene datos que el servidor no acepta. La ruta de `src/app.ts` ya contempla la respuesta `400` cuando el servicio lanza `RangeError` o `TypeError`.

Ejecutá `npm test` nuevamente. Explicá por qué esta comprobación es de **integración** y no solamente una prueba unitaria.

- **Explicación:** Es una prueba de integración porque evalúa la interacción coordinada entre varios componentes: la aplicación Express, el middleware de parseo JSON (`express.json`), el enrutador HTTP (`/api/precios/calcular`), el controlador con su bloque `try/catch`, la llamada al servicio `calcularPrecioFinal` y la posterior generación y serialización de la respuesta HTTP 400 con su cuerpo JSON.

## 8. Diferenciar pruebas y verificación de tipos

Ejecutá:

```bash
npm run typecheck
```

Este comando usa TypeScript para analizar tipos, pero **no ejecuta las pruebas**. Por otra parte, Node puede ejecutar los archivos `.ts` compatibles directamente, pero no comprueba sus tipos durante esa ejecución.

**Pregunta:** ¿por qué conviene utilizar tanto `typecheck` como `npm test`?

- **Respuesta:** Porque cubren aspectos diferentes e indispensables: `typecheck` (`tsc --noEmit`) analiza la coherencia estática del sistema de tipos en tiempo de compilación (detectando inconsistencias de tipos, errores tipográficos o firmas erróneas sin ejecutar código), mientras que `npm test` evalúa el comportamiento dinámico y las reglas lógicas del negocio en tiempo de ejecución (comprobando que ciertas entradas produzcan los resultados y errores esperados).

## 9. Integración continua: extensión opcional

El proyecto ya contiene `.github/workflows/pruebas.yml`. Abrí ese archivo e identificá los pasos que ejecutará GitHub Actions: descargar el repositorio, preparar Node, instalar dependencias, comprobar tipos y correr las pruebas.

Si querés probarlo en GitHub:

1. Ejecutá `npm install` antes de subir el repositorio y conservá `package-lock.json` para versionarlo.
2. Subí los archivos del proyecto a un repositorio, **sin** incluir `node_modules`.
3. Entrá a la pestaña **Actions** y revisá la ejecución iniciada con tu `push`.
4. Comprobá qué ocurre si una prueba falla y luego se corrige.

En la versión actualizada del proyecto, el workflow utiliza `npm install`. Después de generar y versionar `package-lock.json`, se recomienda cambiarlo a `npm ci` para obtener instalaciones reproducibles.

## 10. Registro de resultados

| Comprobación | Resultado | Observación |
| --- | --- | --- |
| Respuesta de `GET /healthy` | HTTP 200 `{"estado":"ok"}` | El servidor responde correctamente al chequeo de salud (soporta tanto `/salud` como `/healthy`). |
| Primera ejecución de `npm test` | 4 aprobadas, 1 fallida (5 pruebas) | Falló `rechaza descuentos superiores al 100 %` porque la función original no validaba las entradas. |
| Corrección del servicio | 5 aprobadas (5 pruebas) | Se agregaron validaciones con `Number.isFinite(...)` y `throw new RangeError(...)` en `precios.ts`. |
| Nuevas pruebas unitarias | 7 aprobadas (7 pruebas) | Se cubrieron: descuento 100%, precio negativo, descuento negativo y desafío de precio `NaN`. |
| Prueba de integración HTTP 400 | 3 aprobadas en `api.test.ts` (10 totales) | Supertest comprueba que la API responde con HTTP 400 y `{ error: "..." }` ante descuento inválido. |
| Verificación `npm run typecheck` | Aprobada sin errores (código 0) | TypeScript valida tipos de todo el código sin emisión de archivos (`tsc --noEmit`). |

## 11. Preguntas de cierre

1. ¿Qué diferencia hay entre comprobar manualmente una respuesta y ejecutar una prueba automatizada?
   - **Respuesta:** La comprobación manual es exploratoria, lenta, sujeta a errores humanos y costosa de repetir ante cada cambio. En cambio, una prueba automatizada es código programado que se ejecuta en milisegundos de manera repetible, confiable y desatendida (incluso en flujos de CI), garantizando que las condiciones previamente programadas sigan cumpliéndose.

2. ¿Cuál fue la causa de la primera prueba fallida?
   - **Respuesta:** La función `calcularPrecioFinal` no realizaba ninguna validación sobre los parámetros recibidos. Ante un descuento de 150 % sobre $1.000, calculaba `1000 - 1500 = -500` y retornaba ese valor en lugar de arrojar el error `RangeError` que la prueba esperaba.

3. ¿Por qué es útil incluir casos inválidos además de los casos habituales?
   - **Respuesta:** Porque en producción el sistema recibirá datos incorrectos, malformados o maliciosos. Las pruebas con casos inválidos aseguran que el software falle de manera controlada (*fail-fast*), protegiendo la integridad de la base de datos y de la lógica de negocio, y devolviendo respuestas claras al cliente (como HTTP 400).

4. ¿Qué parte del sistema comprueba `tests/precios.test.ts` y qué partes comprueba `tests/api.test.ts`?
   - **Respuesta:** `tests/precios.test.ts` realiza pruebas **unitarias** sobre la función de cálculo aislada (`calcularPrecioFinal`), sin dependencias de red ni servidor. `tests/api.test.ts` realiza pruebas de **integración**, comprobando la interacción entre el servidor Express, el middleware JSON, el enrutamiento HTTP, la llamada al servicio y el manejo de excepciones para enviar respuestas HTTP con sus respectivos códigos de estado.

5. ¿Qué ventajas ofrece ejecutar las pruebas después de cada modificación?
   - **Respuesta:** Permite detectar **regresiones** (errores introducidos en funcionalidades que antes funcionaban) de forma inmediata mientras el cambio está fresco en la mente del desarrollador, reduce drásticamente el costo de depuración y brinda confianza para refactorizar y mejorar el código.

**Referencias:** [Vitest](https://vitest.dev/guide/) · [Supertest](https://github.com/ladjs/supertest) · [Node.js: TypeScript](https://nodejs.org/api/typescript.html) · [GitHub Actions](https://docs.github.com/actions)
