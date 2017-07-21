
## 1. Uso básico de service workers

Un _service worker_ actúa como un proxy de red ejecutándose en el navegador: intercepta las peticiones HTTP que salen de nuestro sitio web hacia la red y puede contestar con cualquier tipo de contenido.

En esta primera lección, vamos a preparar nuestra aplicación para que funcione incluso sin conexión a la red.

### Creación del fichero y registro

Lo primero que vamos a hacer en Glitch es crear un nuevo archivo cuyo nombre será `public/service-worker.js`. En él escribiremos el código que controla la intercepción de las peticiones.

![En Glitch, al especificar el nombre, especificamos también la ruta]()

Visita tu aplicación haciendo click en `Show Live`, abre las herramientas de desarrollo y haz clic sobre la pestaña _Application_.

![La pestaña aplicación]

En la lista de la izquierda, haz clic en el item _Service Workers_ para comprobar que no hay ninguno asociado a ese origen.

Recuerda que un origen no es lo mismo que un dominio. El origen es el protocolo junto con el nombre de dominio y el puerto. Así, `http://mozilla.org` y `https://mozilla.org` son orígenes distintos, como también lo son `localhost:8000` y `localhost:3333`.

Un service worker debe registrarse desde el código del cliente. Edita `public/client.js` y añade el siguiente código:

```js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js')
  .then(function () { console.log('¡Service Worker registrado!'); })
  .catch(function (e) { console.error('Parece que hubo algún problema:', e)});
}
```

El condicional omite el proceso de registro si el navegador no soporta _service workers_. En caso de soportarlos, el método [`register`](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register) devuelve una promesa que, en caso de cumplirse, garantiza que el _service worker_ está instalándose.

Por supuesto, has de incluir el script en el punto de entrada HTML. En este caso edita `views/index.html` y antes de la etiqueta de cierre de la cabecera `</head>`, añade la siguiente línea:

```html
<script src="/client.js"></script>
```

Cuando tu aplicación se actualice, comprueba que en la consola aparece el mensaje de que todo ha ido bien y que en la sección _Service workers_ de la pestaña _Application_ se muestra el service worker como aparece en la captura siguiente:

![Si todo ha ido bien, el status del service worker indica _activated and is running_]()

Un service worker sólo puede registrarse desde un origen seguro, que utilice [HTTPS](https://es.wikipedia.org/wiki/Hypertext_Transfer_Protocol_Secure). Puedes comprobar otros requerimientos rápidamente en la infografía [_Service Workers 101_](https://github.com/delapuente/service-workers-101#service-workers-101).

### Instalación y activación del _service worker_

Vamos a modificar el _service worker_ &mdash;de ahora en adelante _SW_&mdash; múltiples veces y cada vez que lo hagamos, Glitch relanzará nuestra aplicación. Es conveniente, por tanto, activar la opción _Update on reload_ del menú _Service Workers_ del panel _Application_ para garantizar que el _SW_ se actualiza en cada recarga.

![Detalle del interruptor para activar la recarga al refrescar]()

El panel de aplicación debería indicar que nuestro _SW_ funciona correctamente aunque todavía no haga nada. Durante la instalación, el _SW_ pasa por tres estados (que puedes revisar en la inforgrafía [_Service Workers 101_](https://github.com/delapuente/service-workers-101#service-workers-101)):

  1. Instalando: pensado para preparar la infraestructura necesaria para el funcionamiento del _SW_.
  2. Activando: pensado para retirar la infraestructura de alguna versión anterior del _SW_.
  3. Activo: listo para interceptar peticiones a la red.

Añade el siguiente código al fichero del _service worker_ `public/service-worker.js`:

```js
var VERSION = 1;
var PREFIX = '__pwa-workshop';
var CACHE_NAME = `${PREFIX}-assets-v${VERSION}`;
var ASSETS = [
  'https://cdn.glitch.com/aa6a5f34-4aee-4eae-807f-ca86f623e58a%2Fplus-black-symbol.svg?1499350618348',
  'https://cdn.glitch.com/aa6a5f34-4aee-4eae-807f-ca86f623e58a%2Ftick-sign.svg?1499363273037',
  'https://cdn.glitch.com/aa6a5f34-4aee-4eae-807f-ca86f623e58a%2Flike.svg?1499363275514',
  'https://cdn.glitch.com/aa6a5f34-4aee-4eae-807f-ca86f623e58a%2Fcloud-backup-up-arrow.svg?1499366672437',
  'https://fonts.googleapis.com/css?family=Poppins',
  'https://fonts.gstatic.com/s/poppins/v2/HUuNgGR31mqIHE6zs0BlBgLUuEpTyoUstqEm5AMlJo4.woff2',
  '/client.js',
  '/style.css',
  '/error-page.html',
  '/'
];

self.addEventListener('install', event => {
  event.waitUntil(Promise.all([addAssets(), self.skipWaiting()]));
});

self.addEventListener('activate', event => {
  event.waitUntil(Promise.all([clearOldCaches(), self.clients.claim()]));
});
```

En el contexto de un _service worker_ o de cualquier otro _worker_, `self` hace referencia siempre al objeto global. Con `addEventListener` podemos suscribirnos a los cambios en el ciclo de vida del _SW_.

El método [`waitUntil`](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil) de los eventos `install` y `activate` permite extender las fases de instalación y activación respectivamente, hasta que la promesa pasada como parámetro se resuelva.

Entre la instalación y la activación, el navegador espera a que todos los clientes actualmente controlados por un _service worker_ se cierren, antes de activar el siguiente. Con el método [`skipWaiting`](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope/skipWaiting) podemos acelerar el proceso evitando la espera del _SW_.

Durante la activación, utilizamos el método [`claim`](https://developer.mozilla.org/en-US/docs/Web/API/Clients/claim) para hacer que el _SW_ intercepte todas las peticiones originadas en los clientes activos (pestañas ya abiertas, otros _workers_&hellip;) a partir de ahora.

Aun no hemos escrito las funciones `addAssets` y `clearOldCaches`, por lo que el proceso de instalación falla en tiempo de ejecución y la instalación se interrumpe. Podéis ver los errores relacionados con la instalación del _SW_ bajo el estado, en la vista _Service Workers_, en la pestaña _Application_.

![Detalle de los errores de instalación por la ausencia de las funciones]()

Antes de continuar, **borra los errores** y luego añade el siguiente listado con la implementación de las funciones que faltan:

```js
function addAssets() {
  return self.caches.open(CACHE_NAME)
  .then(cache => cache.addAll(ASSETS));
}

function clearOldCaches() {
  return self.caches.keys()
  .then(allCaches => {
    return allCaches.filter(cacheName => {
      var isMine = cacheName.indexOf(PREFIX) === 0;
      var isNotTheNewest = cacheName !== CACHE_NAME;
      return isMine && isNotTheNewest;
    });
  })
  .then(oldCaches => {
    return oldCaches.map(cacheName => {
      return self.caches.delete(cacheName);
    });
  })
  .then(deletingTasks => Promise.all(deletingTasks));
}
```

Ls función `addAssets` acepta una lista de recursos (_assets_), y con el método [`open`](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/open), abre una nueva caché (creándola si no existía) donde añadir estos recursos. Para ello utiliza el método [`addAll`](https://developer.mozilla.org/en-US/docs/Web/API/Cache/addAll) de la caché.

La lista de recursos `ASSETS` contiene los iconos de la aplicación, la hoja de estilos, el código JavaScript del cliente (exceptuando el _SW_), las fuentes, la página de error y el índice.

La función `clearOldCaches` obtiene todas los nombres de las caches en el origen con el método [`keys`](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/keys), reconoce las propias y borra las antiguas con el método [`delete`](https://developer.mozilla.org/en-US/docs/Web/API/Cache/delete).

Si todo ha ido bien, no deberías ver nuevos errores bajo el estado del _service worker_. Si además haces click en el elemento _Cache storage_, deberías poder ver un listado de las caches y su contenido (haz click en el icono de refrescar si no aparece nada). Prueba a cambiar el número de versión para comprobar que el código funciona correctamente.

![Cache offline y su contenido]

### Estrategias de cache

Gracias a las cachés sin conexión ([_Offline Caches_](https://developer.mozilla.org/en-US/docs/Web/API/Cache)) tenemos los recursos para recrear la interfaz de usuario sin necesidad de estar conectados a la red, pero aun no le hemos dicho al _SW_ cuándo debe servir estos recursos. Por el momento, todas las peticiones a la red alcanzan la red.

Puedes ir al panel _Network_ en las herramientas de desarrollador y activar el interruptor _Offline_ para simular que no hay red. Realiza alguna acción o recarga y verás como la aplicación falla estrepitosamente.

![Detalle del interruptor offline que simula la ausencia de red]()

Sin modificar una sola línea en el código de la UI, nuestra intención es crear una capa de red, en el _service worker_, que responda de forma diferente según el tipo de petición y el estado de la conexión.

Empieza añadiendo el siguiente listado, tras registrar el _listener_ del evento `activate`:

```js
self.addEventListener('fetch', event => {
  var result = handleRequest(event.request);
  var response = result[0];
  var completion = result[1];
  event.respondWith(response);
  event.waitUntil(completion);
});
```

La función `handleRequest` será la encargada de implementar esta capa de red y devolverá una lista con dos promesas. El método [`respondWith`](https://developer.mozilla.org/en-US/docs/Web/API/FetchEvent/respondWith) consume la primera, que debe resolverse con un objeto del tipo [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) y será entregado al navegador para que sirva de respuesta a la petición. De nuevo, utilizaremos [`waitUntil`](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil) para extender la vida del _SW_ hasta que la segunda promesa de la lista se resuelva.

Los _service workers_ están pensados para realizar una tarea concreta y cerrarse de forma que no consuman recursos innecesarios. Dada su naturaleza asíncrona, no se puede determinar de antemano cuándo un _service worker_ ha terminado. Es por ello que utilizamos `waitUntil` con una promesa. Tal promesa expresa que todas las acciones que queremos realizar han terminado.

Por el mismo motivo, no se puede confiar en el estado global de un _service worker_ dado que, tarde o temprano, el navegador eliminará al _SW_ y el estado global se perderá.

#### Estrategia _sólo caché_

Antes de continuar, comprueba que has desactivado el modo _Offline_ y refresca la pestaña.

Comencemos de forma sencilla. Añade el siguiente listado al final del fichero del _SW_:

```js
function handleRequest(request) {
  if (isAsset(request)) {
    return only(fromCache(request));
  }
  return only(fetch(request));
}

function isAsset(request) {
  var url = new URL(request.url);
  return !isIndex(request) &&
         (ASSETS.indexOf(url.href) >= 0 ||
         ASSETS.indexOf(url.pathname) >= 0);
}

function isIndex(request) {
  var url = new URL(request.url);
  return url.pathname === '/';
}

function fromCache(request) {
  return self.caches.open(CACHE_NAME)
  .then(cache => cache.match(request));
}

function only(promise) {
  return [promise, Promise.resolve()];
}
```

Ve ahora a la pestaña _Network_ y limpia los logs. Recarga el tab manualmente y observa los resultados en la lista de peticiones. Verás como los recursos se sirven desde el _SW_:

![En la columna size se lee "from service worker" indicando que el recurso se ha servidor desde un service worker]()

Date cuenta de que aunque se indique que el recurso se ha servido desde el _SW_, esto **no significa que se haya servido desde una caché**.

Si activas el modo _Offline_ y recargas, verás cómo la aplicación sigue fallando. Esto es porque el índice no se considera un recurso (_asset_) y, por tanto, no se sirve desde una caché.

Sin embargo, podrías consultar la página de error `/error-page.html` dado que esta sí se considera un recurso y se servirá desde la caché. Lo mismo ocurre con cualquier recurso que incluyeras en la variable `ASSETS`.

El código anterior se explica por sí mismo: si la petición es un recurso, lo servimos desde la caché con la función `fromCache`. Si no, lo servimos desde la red con [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch). La nueva API fetch tiende a reemplazar la famosa interfaz [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) y además, es la única forma de realizar una petición desde un _SW_. Ninguna de las peticiones originadas en un _SW_ será jamás interceptada.

La función `fromCache` usa el método [`match`](https://developer.mozilla.org/en-US/docs/Web/API/Cache/match) de las cachés para buscar una respuesta a la petición pasada como parámetro.

Desactiva el modo _Offline_ y recarga el índice antes de continuar.

#### Red más actualización o alternativa offline

Veamos ahora cómo tratar las acciones. Modifica la función `handleRequest` para que quede de la siguiente forma:

```js
function handleRequest(request) {
  if (isAsset(request)) {
    return only(fromCache(request));
  }
  if (isAction(request)) {
    var offlinePage = isIndex(request) ? cachedIndex() : errorPage();
    return fetchAndUpdateIndex(request, offlinePage);  
  }
  return only(fetch(request));
}
```

En pocas palabras, si la petición no es un recurso, pero es una operación, lo que queremos es que llegue a la red. De no haber red, queremos poder dar una alternativa sin conexión. Esta alternativa dependerá de si estamos visitando la lista de recomendaciones o realizando una operación. En el primer caso devolveremos la lista más actualizada desde la caché. En el segundo mostraremos una pantalla de error dando la opción de volver a la lista. En caso de que la petición no sea ni un recurso, ni una acción, dejaremos que alcance la red normalmente.

Añade el siguiente listado al final del archivo:

```js
function isAction() {
  var url = new URL(request.url);
  return url.pathname.indexOf('/recommendations') === 0;
}

function cachedIndex() {
  return fromCache('/');
}

function errorPage() {
  return fromCache('/error-page.html');
}

function fetchAndUpdateIndex(request, offlineAlternative) {
  var done;
  var completion = new Promise(resolve => done = resolve);
  var response = doRequest(request, offlineAlternative, done);
  return [response, completion];

  function doRequest(request, offlineAlternative, done) {
    return fetch(request)
    .then(response => {
      if (!response.ok) {
        done();
        return offlineAlternative;
      }
      updateIndex(response.clone()).then(done);
      return response;
    })
    .catch(reason => {
      done();
      return offlineAlternative;
    });
  }
}

function updateIndex(response) {
  return self.caches.open(CACHE_NAME)
  .then(cache => cache.put('/', response));
}
```

La función `fetchAndUpdateIndex` implementa toda la lógica de caché. Respondemos con la alternativa sin conexión cuando `fetch` falla o el servidor devuelve una respuesta _no OK_, lo que quiere decir que no está en el rango [`2XX`](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes#2xx_Success). Si todo ha salido bien, usamos la respuesta para actualizar la entrada de caché que corresponde con el listado de recomendaciones (usando para ello el método [`put`](https://developer.mozilla.org/en-US/docs/Web/API/Cache/put)) y para responder a la petición.

El cuerpo de una respuesta **sólo puede utilizarse una vez**, o se utiliza para representarse en el cliente o se utiliza para guardarse en la caché. Es por ello que copiamos la respuesta con [`clone`](https://developer.mozilla.org/en-US/docs/Web/API/Response/clone) antes de actualizar la caché.

Con estos cambios ya puedes pasar a modo _Offline_ y probar a visitar el índice y a realizar alguna operación para alcanzar la página informativa.

### Conclusión



## 2. Escribiendo el manifiesto web

El manifiesto web es un fichero JSON con un objeto de [claves bien conocidas](https://w3c.github.io/manifest/#manifest-and-its-members) que se enlaza desde nuestra página web de manera que el navegador obtenga la información contenida y la use para aquellos fines que crea convenientes. Entre ellos, una mejor integración con el sistema como veremos a continuación.

No obstante, el manifiesto web también puede ser utilizado por los buscadores web que beneficiarse de la información contenida para mejorar la clasificación de estas páginas web.

Crea un nuevo fichero en Glitch llamado `public/manifest.json` y añade el siguiente contenido:

```js
{
  "name": "Recommendations",
  "short_name": "Recommendations",
  "description": "Keep track of everything you're recommended",
  "theme_color": "#FFDD60",
  "background_color": "#FFDD60",
  "display": "standalone",
  "icons": [
    {
      "src": "https://cdn.glitch.com/6f34aba2-54dc-445d-bebc-39255a1b8c6b%2Ficon196.png?1500627723764",
      "sizes": "196x196",
      "type": "image/png"
    }
  ]
}
```

Antes de profundizar en cada clave, enlaza el manifiesto con tu página web añadiendo el siguiente elemento HTML antes del cierre de la etiqueta `head`:

```html
<link rel="manifest" href="/manifest.json">
```

Las claves `name`, `short_name` y `description` son autoexplicativas. La clave `short_name` se prefiere cuando el espacio en pantalla para mostrar el nombre es limitado. Por ejemplo, en el caso del nombre bajo el icono en la pantalla de inicio.

Los campos `theme_color` y `background_color` colorean la interfaz de usuario del navegador. En particular, `background_color` se refiere al color de fondo del navegador mientras se carga la página web.

El campo `display` permite seleccionar una experiencia de navegación óptima, siendo `standalone` el valor que corresponde a la ausencia total de elementos de navegación.

La lista `icons` contiene una lista de iconos con entradas para las distintas resoluciones de pantalla y densidades de píxeles.

Para una descripción completa y con ejemplos de los campos del manifiesto, puedes consultar [la página sobre el manifiesto web de la MDN](https://developer.mozilla.org/en-US/docs/Web/Manifest#Members).

Visita tu aplicación desde el móvil usando Opera, Chrome o Samsung Internet, navegadores todos que ponen especial énfasis en la integración con el manifiesto y añade tu aplicación a la pantalla de inicio:

![Añadir la aplicación a la pantalla de inicio desde Chrome]()

Observa la _splashpage_ que genera Chrome gracias al color de fondo y la lista de iconos:

![Splashpage generada por Chrome]()

Y fíjate en la ausencia de interfaz de usuario relacionada con el navegador:

![En el modo standalone no hay interfaz del navegador, como barras de navegación o herramientas]()

Lo que también se manifiesta en el cambiador de tareas:

![En el modo standalone, la página web aparece como un elemento distinto del cambiador de tareas]()

Con estas modificaciones, tu sitio web ya entra en las definiciones de _progressive web apps_ más extendidas. No obstante recuerda que las [_PWA_ no son una receta sino una herramienta](https://medium.com/samsung-internet-dev/progressive-web-apps-are-a-toolkit-not-a-recipe-b2fd68613de5).

Las herramientas de desarrollador de Chrome incluyen, en la pestaña _Application_ y sección _Manifest_ una visualización de los distintos campos del mismo.

### Conclusión

## 3. Uso básico de notificaciones push

Las notificaciones push tienen una interpretación doble: desde el punto de vista de red, una notificación push es una comunicación desde el servidor al cliente, iniciada por el servidor sin que el cliente haya realizado una petición previa.

![Diagrama de una comunicación push del servidor al cliente]()

Por otro lado, desde el punto de vista de la experiencia de usuario, una notificación push es un elemento de interfaz cuya finalidad es interrumpir levemente la actividad del usuario para comunicar cierta información.

![Diagrama de una comunicación push del cliente al usuario]()

No es extraño encontrar una tercera definición que reúne estas dos y establece que cualquier comunicación iniciada en el servidor debe manifestarse ante el usuario. De hecho, los navegadores actuales fuerzan esta dependencia de forma que si no se muestra una notificación al usuario como respuesta a una notificación push proveniente del servidor, el navegador muestra una genérica.

Algunos navegadores estás experimentando con permitir algunas notificaciones silenciosas, que no interrumpan la acción del usuario.

De todas formas, conviene conocer la diferencia entre el protocolo de red y la metáfora visual.

### Estableciendo la comunicación cliente-servidor

**Nota**: configurar las notificaciones push para que funcionen con Chrome, Samsung Internet y Opera requiere conocer los valores `gcm_sender_id` y `gcm_api_key` de una aplicación [Firebase](https://firebase.google.com/). Aunque se te propocionarán unos valores de prueba durante el talle, puedes crear tu propia aplicación Firebase y consultar esta [guía de TapJoy para saber dónde encontrarlos](http://dev.tapjoy.com/faq/how-to-find-sender-id-and-api-key-for-gcm/).

---

Comenzar a enviar notificaciones, sin el permiso explícito del usuario, podría resultar demasiado intrusivo. Es por ello que los navegadores obligan a pedir una subscripción explícitamente. En el momento en que el código del cliente pide una subscripción, el navegador pide permiso al usuario para recibir notificaciones.

Edita el fichero `public/client.js` y modifícalo para que quede así:

```js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js')
  .then(function (registration) {
    console.log('¡Service Worker registrado!');
  })
  .catch(function (e) { console.error('Parece que hubo algún problema:', e)});

  navigator.serviceWorker.ready.then(function (registration) {
    return subscribeToNotifications(registration.pushManager);
  });
}

function subscribeToNotifications(pushManager) {
  return pushManager.getSubscription(function (subscription) {
    if (subscription) {
      return subscription;
    }
    return pushManager.subscribe();
  })
  .then(sendDetailsToTheServer);
}

function sendDetailsToTheServer(subscription) {
  return fetch('/recommendations/subscribe', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      endpoint: subscription.endpoint,
      key: toBase64(subscription.key),
      auth: toBase64(subscription.auth)
    })
  });
}

function toBase64(target) {
  return !target ? '' :
         btoa(String.fromCharCode.apply(null, new Uint8Array(target)));
}
```

Edita también el fichero `public/manifest.json` y añade la clave `gcm_sender_id` con el valor que te proporcionarán en el taller o el que obtengas de tu aplicación Firebase.

Lo que estamos haciendo ahora es pedir al servicio de notificaciones del navegador que nos proporcione una subscripción. Primero comprobamos que no exista ninguna anterior, que pudiéramos reciclar, con el método [`getSubscription`](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/getSubscription) de la interfaz [`PushManager`](https://developer.mozilla.org/en-US/docs/Web/API/PushManager). En caso de no existir, pediremos una nueva subscripción con el método [`subscribe`](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe).

No siempre es bueno pedir permiso nada más detectemos que no hay una subscripción. Si trabajáramos en un sitio web real, quizá debiéramos pedir una subscripción cuando sepamos algo más sobre el usuario. Igual cuando detectemos que no ha marcado ninguna recomendación durante semanas o, si se tratara de un blog, al detectar que el usuario termina de leer un artículo o al finalizar una compra con éxito si se tratase de un sitio de comercio _online_.

La subscripción incluye una _URL_ o _endpoint_ al que realizaremos las peticiones de envío de las notificaciones y unas claves necesarias para cifrar el contenido de las mismas.

Fíjate en que la interfaz JavaScript que devuelve la subscripción sólo se comunica con el cliente y es necesario que el cliente comunique a su servidor los detalles de la subscripción, que es lo que se hace en la función `sendDetailsToTheServer`. Para ello utilizamos la interfaz [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) de la que ya habíamos hablado anteriormente.

Edita ahora el fichero del servidor `server.js` para que guarde los detalles de la subscripción. Añade la siguiente ruta justo antes de la declaración de `recommendations`:

```js
app.post('/subscribe', function (request, response) {
  subscriptions[request.body.endpoint] = {
    key: request.body.key,
    auth: request.body.auth
  };
  request.sendStatus(201);
});

var subscriptions = {};
```

### Enviando notificaciones a los clientes

¿Recuerdas la definición de notificación push? En este punto vamos a implementar la primera interpretación. Es decir, la comunicación que ocurre desde el servidor hacia el cliente.

Para ello modifica el código del archivo `server.js` para que haga uso de la biblioteca `WebPush` que abstrae los detalles acerca de cómo enviar notificaciones a través de una interfaz más amable. Comienza solicitando la biblioteca añadiendo la siguiente línea al comienzo del archivo:

```js
var webPush = require('web-push');
webPush.setGCMAPIKey('XXXXXXXXX'); // Reemplaza esta valor por el que te
                                   // proporcionen en el taller o por el que
                                   // obtengas de tu aplicación Firebase.
```

Ahora modifica la declaración de `listener` para llamar al planificador de notificaciones:

```js
var listener = app.listen(process.env.PORT, function () {
  console.log('Your app is listening on port ' + listener.address().port);
  scheduleNotification();
});

var NOTIFICATION_PERIOD = 60 * 1000; // 1 min

function scheduleNotification() {
  setTimeout(function () {
    const total = getUncheckedRecommendations();
    if (total > 0) {
      const summary = getSummary(total);
      Object.keys(subscriptions).forEach(sendNotification.bind(undefined, summary));
    }
    scheduleNotification();
  }, NOTIFICATION_PERIOD);
}

function getUncheckedRecommendations() {
  return recommendations.reduce(function (total, recommendation) {
    return !recommendation.checked ? total + 1 : total;
  }, 0);
}

function getSummary(total) {
  return `You still have ${total} unchecked recommendation${total > 1 ? 's' : ''}`;
}

function sendNotification(summary, endpoint) {
  const subscription = {
    endpoint,
    keys: {
      auth: subscriptions[endpoint].auth,
      p256dh: subscriptions[endpoint].key
    },
    ttl: 60 * 60
  };
  webPush.sendNotification(subscription, summary);
}
```

Las funciones `getUncheckedRecommendations` y `getSummary` no tienen misterio. La primera cuenta el número de recomendaciones que aun no hemos comprobado y la segunda compone un mensaje con este número.

La función `sendNotification` utiliza la biblioteca `webPush` para enviar una notificación mediante el método [`sendNotification`](https://github.com/web-push-libs/web-push#sendnotificationpushsubscription-payload-options).

La constante `NOTIFICATION_PERIOD` establece cada cuánto se lanzará una nueva notificación. Cuando hayamos comprobado que todo funciona podemos elevar este número a 48 horas o el tiempo que consideremos oportuno.

### Mostrando la notificación al usuario

Es difícil comprobar que, efectivamente, la notificación ha alcanzado al cliente porque este no la trata, no hay aún una asociación entre recibir una notificación desde el servidor y usar la interfaz de usuario para mostrarla.

Esto lo arreglaremos en el _service worker_. Lo que haremos será añadir un manejador para el evento [`push`](). Modifica el fichero del _SW_ `public/service-worker.js` y añade esto al final del mismo:

```js
self.addEventListener('push', function (event) {
  var payload = event.data.text();
  event.waitUntil(self.registration.showNotification('Recommendations', {
    body: payload
  }));
});
```

El método [`showNotification`]() mostrará la interfaz de usuario del sistema operativo asociada a una notificación. Como cuerpo de la notificación, utilizaremos el contenido de la notificación _push_ que está disponible a través de la propiedad [`data`](https://developer.mozilla.org/en-US/docs/Web/API/PushMessageData) del evento.

### Acción al pulsar en la notificación

Las notificaciones pueden recibirse sin tener la pestaña abierta e incluso, sin tener el navegador abierto. Por ello, al hacer click sobre la notificación sería conveniente recuperar la pestaña con la aplicación o, en su defecto, abrir una vista nueva.

El evento [`notificationclick`](https://developer.mozilla.org/en-US/docs/Web/Events/notificationclick) nos permite responder a la pulsación sobre una notificación. Añade el siguiente listado al final del _SW_:

```js
self.addEventListener('notificationclick', function (event) {
  event.waitUntil(focusOnOpenTabOrNew());
});

function focusOnOpenTabOrNew() {
  return self.clients.matchAll({ type: 'window' })
  .then(clientList => {
    if (clientList.length > 0) {
      return clientList[0].focus();
    }
    return self.clients.openWindow('/');
  });
}
```

Como en otras ocasiones, utilizamos `waitUntil` para mantener el _SW_ corriendo hasta que todas las operaciones asíncronas hayan terminado. Esta vez, utilizamos el método [`matchAll`](https://developer.mozilla.org/en-US/docs/Web/API/Clients/matchAll) para recuperar un listado completo de todas las ventanas/pestañas controladas por el _SW_. Si no hay ninguna, gracias al método [`openWindow`](https://developer.mozilla.org/en-US/docs/Web/API/Clients/openWindow) podemos abrir una pestaña nueva dónde indiquemos como parámetro. Si existe alguna, bastará con darle el foco de atención mediante su método [`focus`](https://developer.mozilla.org/en-US/docs/Web/API/Window/focus).

### Conclusión

https://serviceworke.rs/web-push.html
