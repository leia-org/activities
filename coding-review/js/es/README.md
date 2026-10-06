# Actividad de revisión de código JavaScript con Leia

## Sobre esta actividad

Esta es una landing para probar una actividad con **LEIAs** en la que el alumno debe enviar un fragmento de código y será evaluado en función de los conocimientos que demuestre durante la revisión.

Una vez iniciada la actividad, lo primero que hay que hacer es pegar un fragmento de código en el editor. Después, hay que avisar a LEIA de que se está listo para comenzar la revisión con un mensaje como:

> Hola, ya estoy listo para la revisión, he copiado mi código en el editor.

Para facilitar la prueba de esta actividad, a continuación se presenta un conjunto de ejemplos pensados para simular una situación en la que un alumno realiza un *Pull Request* sobre una temática determinada.

Los ejemplos están ordenados de menor a mayor complejidad, de forma que se pueda empezar por casos sencillos y avanzar progresivamente hacia situaciones que requieren un análisis más profundo.

**[ACCEDER A LA ACTIVIDAD](https://workbench.leia.ovh/?code=RMUJZIEPCN1G6FQED&email=_test_cfia)**

---

# Ejemplos de revisión de Pull Requests en JavaScript

El material propone cinco ejercicios de revisión de código JavaScript. Cada uno simula un *Pull Request* que modifica un único archivo.

La idea es revisar cada caso como si hubiera sido enviado por otra persona del equipo, buscando posibles errores, problemas de diseño, casos límite, problemas de legibilidad, rendimiento, asincronía o mantenibilidad.

Los ejercicios aparecen en orden creciente de dificultad.

> **Nota:** los fragmentos de código incluidos a continuación son ejemplos equivalentes creados para esta actividad. Mantienen la temática y los tipos de problemas que se quieren revisar, pero no reproducen literalmente el código del README de referencia.

---

# 1. Resumen de un carrito de compra

## Contexto

El primer ejercicio gira en torno a una función que calcula el resumen de un carrito: subtotal, descuento opcional y número total de unidades.

### Archivo: `cart.js`

```js
function getCartSummary(products, discountPercent) {
  let subtotal = 0;
  let units = 0;

  for (const product of products) {
    subtotal += product.price * product.quantity;
    units += product.quantity;
  }

  let discountAmount = 0;

  if (discountPercent) {
    discountAmount = subtotal * discountPercent / 100;
  }

  const finalPrice = subtotal - discountAmount;

  return {
    subtotal: subtotal.toFixed(2),
    discount: discountAmount.toFixed(2),
    total: finalPrice.toFixed(2),
    units,
  };
}

const exampleCart = [
  {
    name: "Mechanical Keyboard",
    price: 84.95,
    quantity: 1,
  },
  {
    name: "Wireless Mouse",
    price: 34.5,
    quantity: 2,
  },
];

console.log(getCartSummary(exampleCart, 10));

module.exports = {
  getCartSummary,
};
```

## Aspectos para revisar

Conviene prestar atención a la validación de datos, los tipos devueltos, los descuentos fuera de rango, la precisión de los cálculos monetarios y los posibles efectos secundarios al importar el módulo.

Algunas preguntas que pueden ayudar durante la revisión:

- ¿Qué ocurre si `products` no es un array?
- ¿Qué pasa si `price` o `quantity` contienen valores no numéricos?
- ¿Tiene sentido permitir descuentos negativos o superiores al 100 %?
- ¿Qué tipo devuelve `toFixed()`?
- ¿Es recomendable trabajar con números de coma flotante para cantidades monetarias?
- ¿Debería ejecutarse un `console.log()` simplemente por importar este archivo?

---

# 2. Agrupación de pedidos por cliente

## Contexto

Este ejercicio construye un resumen de pedidos por usuario, incluyendo número de pedidos, gasto acumulado y fecha del pedido más reciente.

### Archivo: `orders.js`

```js
function createCustomerReport(orders) {
  const customers = {};

  orders.sort((left, right) => {
    return new Date(left.createdAt) - new Date(right.createdAt);
  });

  for (const order of orders) {
    if (!customers[order.customerId]) {
      customers[order.customerId] = {
        customerId: order.customerId,
        orderCount: 0,
        amountSpent: 0,
        lastOrderAt: null,
      };
    }

    const customer = customers[order.customerId];

    customer.orderCount += 1;
    customer.amountSpent += order.total;
    customer.lastOrderAt = order.createdAt;
  }

  const report = [];

  for (const customerId in customers) {
    const customer = customers[customerId];

    customer.amountSpent = Number(
      customer.amountSpent.toFixed(2)
    );

    report.push(customer);
  }

  return report;
}

const exampleOrders = [
  {
    id: "order-101",
    customerId: "customer-a",
    total: 19.95,
    createdAt: "2026-04-10T09:30:00Z",
  },
  {
    id: "order-102",
    customerId: "customer-b",
    total: 42,
    createdAt: "2026-04-11T16:15:00Z",
  },
  {
    id: "order-103",
    customerId: "customer-a",
    total: 28.5,
    createdAt: "2026-04-13T08:00:00Z",
  },
];

console.log(createCustomerReport(exampleOrders));

module.exports = {
  createCustomerReport,
};
```

## Aspectos para revisar

La revisión puede centrarse en la mutación del array original, el uso del ordenamiento, el tratamiento de fechas, la estructura usada para agrupar información y la mezcla de varias responsabilidades dentro de una sola función.

Algunas preguntas que pueden ayudar durante la revisión:

- ¿`sort()` modifica el array recibido?
- ¿Es realmente necesario ordenar todos los pedidos para conocer la fecha más reciente?
- ¿Qué ocurre con fechas inválidas?
- ¿Qué pasa si `total` llega como string?
- ¿Sería más apropiado utilizar un `Map` para agrupar por cliente?
- ¿La función tiene demasiadas responsabilidades?
- ¿Cómo se comportaría con cientos de miles de pedidos?

---

# 3. Carga de perfiles de usuario desde una API

## Contexto

Aquí se cargan varios perfiles de usuario mediante peticiones HTTP en paralelo y se devuelven únicamente los perfiles recuperados correctamente.

### Archivo: `profiles.js`

```js
async function requestProfile(userId) {
  const response = await fetch(
    `https://api.example.com/profiles/${userId}`
  );

  if (!response.ok) {
    throw new Error(
      `Profile request failed for ${userId}: ${response.status}`
    );
  }

  return response.json();
}

async function loadUserProfiles(userIds) {
  const pendingRequests = userIds.map(async (userId) => {
    try {
      const profile = await requestProfile(userId);

      return {
        id: profile.id,
        name: profile.name,
        email: profile.email,
      };
    } catch (error) {
      console.error(
        "Unable to load profile",
        userId,
        error.message
      );

      return null;
    }
  });

  const profiles = await Promise.all(pendingRequests);

  return profiles.filter(Boolean);
}

async function main() {
  const userIds = [
    "user-100",
    "user-101",
    "user-102",
    "user-103",
    "user-104",
  ];

  const profiles = await loadUserProfiles(userIds);

  console.log(`Profiles loaded: ${profiles.length}`);
  console.log(profiles);
}

main();

module.exports = {
  requestProfile,
  loadUserProfiles,
};
```

## Aspectos para revisar

Es útil analizar la concurrencia, el manejo y la pérdida de información de errores, los posibles *timeouts*, las peticiones duplicadas, la dependencia de `fetch` y los efectos secundarios producidos al importar el archivo.

Algunas preguntas que pueden ayudar durante la revisión:

- ¿Qué ocurre si se reciben 5 usuarios? ¿Y si se reciben 50.000?
- ¿Es conveniente lanzar todas las peticiones al mismo tiempo?
- ¿El código permite saber qué perfiles fallaron?
- ¿Se distingue entre un error HTTP, un problema de red y una respuesta JSON inválida?
- ¿Existe algún tiempo máximo de espera?
- ¿Qué ocurre si `userIds` contiene identificadores repetidos?
- ¿Cómo se podría sustituir `fetch` en un test unitario?
- ¿Es deseable que `main()` se ejecute automáticamente al importar el módulo?

---

# 4. Caché asíncrona con expiración

## Contexto

Este caso implementa una pequeña caché en memoria con tiempo de vida configurable para evitar repetir operaciones costosas.

### Archivo: `async-cache.js`

```js
class ExpiringCache {
  constructor(ttlMs = 5000) {
    this.ttlMs = ttlMs;
    this.values = new Map();
  }

  async get(key, loader) {
    const entry = this.values.get(key);

    if (entry && entry.expiresAt > Date.now()) {
      return entry.value;
    }

    const value = await loader();

    this.values.set(key, {
      value,
      expiresAt: Date.now() + this.ttlMs,
    });

    return value;
  }

  has(key) {
    const entry = this.values.get(key);

    if (!entry) {
      return false;
    }

    return entry.expiresAt > Date.now();
  }

  remove(key) {
    return this.values.delete(key);
  }

  clear() {
    this.values.clear();
  }

  count() {
    return this.values.size;
  }
}

async function fetchProduct(productId) {
  const response = await fetch(
    `https://api.example.com/products/${productId}`
  );

  if (!response.ok) {
    throw new Error(`Unable to load product ${productId}`);
  }

  return response.json();
}

const productCache = new ExpiringCache(10_000);

async function getProduct(productId) {
  return productCache.get(
    productId,
    () => fetchProduct(productId)
  );
}

module.exports = {
  ExpiringCache,
  getProduct,
};
```

## Aspectos para revisar

Los puntos principales son las peticiones concurrentes para una misma clave, la limpieza de entradas expiradas, el crecimiento de memoria, la validación del TTL y el tratamiento de promesas rechazadas.

Algunas preguntas que pueden ayudar durante la revisión:

- ¿Qué ocurre si dos llamadas solicitan la misma clave exactamente al mismo tiempo?
- ¿Se ejecutará `loader()` una o varias veces?
- ¿Se eliminan de memoria las entradas expiradas?
- ¿`count()` representa realmente el número de entradas válidas?
- ¿Qué ocurre si `ttlMs` es negativo, cero o no es un número?
- ¿Qué ocurre si `loader` no es una función?
- ¿Sería útil almacenar temporalmente la propia `Promise`?
- Si se almacenasen promesas, ¿qué habría que hacer con una promesa rechazada?

---

# 5. Procesador concurrente de trabajos con reintentos

## Contexto

El último ejercicio introduce un procesador capaz de ejecutar varios trabajos en paralelo, limitar la concurrencia y reintentar operaciones fallidas.

### Archivo: `job-runner.js`

```js
async function processJobs(
  jobs,
  handler,
  options = {}
) {
  const concurrency = options.concurrency || 3;
  const maxRetries = options.maxRetries || 2;

  const queue = [...jobs];
  const completed = [];
  const failed = [];

  async function runWorker(workerId) {
    while (queue.length > 0) {
      const job = queue.shift();

      if (!job) {
        return;
      }

      let attempt = 0;
      let done = false;

      while (attempt <= maxRetries && !done) {
        try {
          const result = await handler(job, {
            workerId,
            attempt,
          });

          completed.push({
            jobId: job.id,
            result,
          });

          done = true;
        } catch (error) {
          attempt += 1;

          if (attempt > maxRetries) {
            failed.push({
              jobId: job.id,
              error: error.message,
            });

            continue;
          }

          await new Promise((resolve) => {
            setTimeout(resolve, attempt * 150);
          });
        }
      }
    }
  }

  const workers = [];

  for (let index = 0; index < concurrency; index++) {
    workers.push(runWorker(index + 1));
  }

  await Promise.all(workers);

  return {
    completed,
    failed,
  };
}

module.exports = {
  processJobs,
};
```

## Aspectos para revisar

Este ejemplo permite analizar límites de concurrencia, estado compartido, orden de resultados, estrategia de reintentos, cancelación, operaciones bloqueadas y diferenciación entre errores recuperables y no recuperables.

Algunas preguntas que pueden ayudar durante la revisión:

- ¿Qué ocurre con `concurrency = 0` o `maxRetries = 0`?
- ¿Se validan valores negativos, decimales o excesivamente altos?
- ¿El orden de `completed` coincide necesariamente con el orden original de `jobs`?
- ¿Qué implicaciones tiene compartir y modificar `queue` entre varios workers?
- ¿Cuál es el coste de utilizar `shift()` repetidamente sobre arrays grandes?
- ¿Es intuitivo que el primer intento tenga el número `0`?
- ¿Es suficiente un retraso lineal entre reintentos?
- ¿Debería incorporarse *jitter*?
- ¿Todos los errores deberían volver a intentarse?
- ¿Qué ocurre si `handler()` nunca termina?
- ¿Cómo se cancelaría el procesamiento?
- ¿Qué ocurre si el valor lanzado no es una instancia de `Error`?
- ¿Sería útil devolver información sobre duración, número de intentos o worker que procesó cada trabajo?

---

# Uso sugerido

Una forma práctica de completar los ejercicios es revisar cada ejemplo como si fuese un *Pull Request* real y clasificar los comentarios por gravedad o naturaleza, por ejemplo:

- **Error:** comportamiento incorrecto o casos límite que generan resultados erróneos.
- **Mantenibilidad:** código difícil de comprender, ampliar o modificar.
- **Rendimiento:** decisiones que pueden degradarse cuando aumenta el volumen de datos.
- **Fiabilidad:** problemas relacionados con errores, reintentos, tiempos de espera o concurrencia.
- **Mejora menor:** cambios pequeños de estilo, claridad o legibilidad.

Después de cada revisión también puede ser útil proponer una implementación corregida o mejorada.

---

## Flujo recomendado para probar la actividad

Para realizar una prueba completa de la actividad:

1. Acceder a la actividad mediante el enlace indicado al principio de esta página.
2. Elegir uno de los cinco ejemplos.
3. Copiar únicamente el contenido del bloque de código correspondiente.
4. Pegar el código en el editor de la actividad.
5. Enviar a LEIA un mensaje como:

   > Hola, ya estoy listo para la revisión, he copiado mi código en el editor.

6. Responder a las preguntas de LEIA justificando qué problemas se han detectado y cómo podrían solucionarse.
7. Repetir el proceso con el siguiente ejemplo para incrementar progresivamente la dificultad.

La intención no es únicamente encontrar errores, sino demostrar comprensión sobre las decisiones de implementación, sus posibles consecuencias y las alternativas disponibles.
