# Axios

Promise Based http Client for browser and node.js
- Uses XMLHttpRequest in browser
- Uses http module in node.js

## Features

- Automatic JSON data handling in response
- Intercept request and response
- Automatic request body serialization to:
    - JSON (application/json)
    - Multipart / FormData (multipart/form-data)
    - URL encoded form (application/x-www-form-urlencoded)

etc...

## Installing

```bash
npm install axios
```

## Example

```js
async function getUser() {
  try {
    const response = await axios.get('/user?ID=12345');
    console.log(response);
  } catch (error) {
    console.error(error);
  }
}
```

```js
axios.post('/user', {
    firstName: 'Fred',
    lastName: 'Flintstone'
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
```

## The Axios Instance

```js
const instance = axios.create({
  baseURL: 'https://some-domain.com/api/',
  timeout: 1000,
  headers: {'X-Custom-Header': 'foobar'}
});
```

```js
instance.interceptors.response.use(undefined, async (error) => {
  if (error.response?.status === 401) {
    await refreshToken();
    return instance(error.config); // Retry original request
  }

  throw error;
});
```
## Request Config

| Property | Description | Default / Notes |
| :--- | :--- | :--- |
| `url` | The server URL used for the request. | Required |
| `method` | The request method to be used. | `get` |
| `baseURL` | Prepended to `url` unless `url` is absolute. | |
| `allowAbsoluteUrls` | If true, absolute URLs override `baseURL`. | `true` |
| `transformRequest` | Functions to change request data before sending. | Only for PUT, POST, PATCH, DELETE |
| `transformResponse` | Functions to change response data before then/catch. | |
| `headers` | Custom headers to be sent. | |
| `params` | URL parameters to be sent (object or URLSearchParams). | `null`/`undefined` are not rendered |
| `paramsSerializer` | Optional config for serializing `params`. | Supports custom encoder/serializer |
| `data` | Data to be sent as the request body. | PUT, POST, DELETE, PATCH |
| `timeout` | Number of milliseconds before the request times out. | `0` (no timeout) |
| `withCredentials` | Whether cross-site requests use credentials. | `false` |
| `adapter` | Allows custom handling of requests (e.g., for testing). | |
| `auth` | HTTP Basic auth credentials (username/password). | Sets `Authorization` header |
| `responseType` | Type of data the server will respond with. | `json` |
| `responseEncoding` | Encoding for decoding responses (Node.js only). | `utf8` |
| `xsrfCookieName` | Name of the cookie to use for XSRF token. | `XSRF-TOKEN` |
| `xsrfHeaderName` | HTTP header name that carries the XSRF token. | `X-XSRF-TOKEN` |
| `onUploadProgress` | Progress event handler for uploads (browser only). | |
| `onDownloadProgress` | Progress event handler for downloads (browser only). | |
| `maxContentLength` | Max size of HTTP response content allowed (Node.js). | |
| `maxBodyLength` | Max size of HTTP request content allowed (Node.js). | |
| `validateStatus` | Determines whether to resolve/reject based on status. | `status >= 200 && < 300` |
| `maxRedirects` | Maximum number of redirects to follow (Node.js). | `21` |
| `beforeRedirect` | Function called before a redirect occurs. | |
| `socketPath` | UNIX Socket to be used (Node.js). | |
| `transport` | Transport method used to make the request. | |
| `httpAgent` | Custom agent for HTTP requests (Node.js). | |
| `httpsAgent` | Custom agent for HTTPS requests (Node.js). | |
| `proxy` | Proxy server hostname, port, and protocol. | |
| `signal` | AbortController signal to cancel the request. | |
| `cancelToken` | (Deprecated) Cancel token to cancel the request. | |
| `decompress` | Whether to decompress response body (Node.js). | `true` |
| `insecureHTTPParser` | Use an insecure HTTP parser (Node.js). | `undefined` |
| `transitional` | Options for backward compatibility. | |
| `env` | Environment specific settings (e.g., FormData class). | |
| `formSerializer` | Configuration for form serialization. | |
| `maxRate` | Rate limiting for upload and download (Node.js). | |

## Response Schema

| Property | Description | Notes |
| :--- | :--- | :--- |
| `data` | The response body provided by the server. | |
| `status` | The HTTP status code from the server response. | e.g. `200` |
| `statusText` | The HTTP status message from the server response. | Blank in HTTP/2 (RFC 7540) |
| `headers` | The HTTP headers the server responded with. | Names are lower-cased and accessible by bracket notation |
| `config` | The configuration provided to the axios request. | |
| `request` | The request that generated this response. | Node: `ClientRequest`; Browser: `XMLHttpRequest` |

## Config Defaults

### Global axios defaults

```js
axios.defaults.baseURL = 'https://api.example.com';
axios.defaults.headers.common['Authorization'] = AUTH_TOKEN;
axios.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded';
```

### Custom instance defaults

```js
// Set config defaults when creating the instance
const instance = axios.create({
  baseURL: 'https://api.example.com'
});

// Alter defaults after instance has been created
instance.defaults.headers.common['Authorization'] = AUTH_TOKEN;
```

### Config order of precedence

Config will be merged with an order of precedence. The order is library defaults found in lib/defaults/index.js, 
then defaults property of the instance, and finally config argument for the request.

The latter will take precedence over the former.
 
Here's an example.

```js
// Create an instance using the config defaults provided by the library
// At this point the timeout config value is `0` as is the default for the library
const instance = axios.create();

// Override timeout default for the library
// Now all requests using this instance will wait 2.5 seconds before timing out
instance.defaults.timeout = 2500;

// Override timeout for this request as it's known to take a long time
instance.get('/longRequest', {
  timeout: 5000
});
```

## Interceptors

- You can intercept requests or responses before they are handled by then or catch.

- The use function adds a handler to the list of handlers to be run when the Promise is fulfilled or rejected. The handler is defined by the fulfilled and rejected functions.

- There is an optional options object that can be passed in as the third parameter. synchronous if the synchronous option is true. The handler is defined as asynchronous if the synchronous option is false. If the synchronous option is not provided, the handler is defined as asynchronous. runWhen will control when the provided interceptor will run. Provide a function that will return true or false on whether it should run, defaults to always true.

```js
// Add a request interceptor
axios.interceptors.request.use(function (config) {
    // Do something before request is sent
    return config;
  }, function (error) {
    // Do something with request error
    return Promise.reject(error);
  },
  { synchronous: true, runWhen: () => /* This function returns true */}
);

// Add a response interceptor
axios.interceptors.response.use(function onFulfilled(response) {
    // Any status code that lie within the range of 2xx cause this function to trigger
    // Do something with response data
    return response;
  }, function onRejected(error) {
    // Any status codes that falls outside the range of 2xx cause this function to trigger
    // Do something with response error
    return Promise.reject(error);
  });
```

- In normal circumstances the onFulfilled response interceptor is only called for responses in the 2xx range, and onRejected is called otherwise. However, this behavior depends on validateStatus. For instance, if validateStatus is setup to always return true, then onFulfilled will be called for all responses.

- If you need to remove an interceptor later you can.

```js
const myInterceptor = axios.interceptors.request.use(function () {/*...*/});
axios.interceptors.request.eject(myInterceptor);
```

You can add interceptors to a custom instance of axios.

```js
const instance = axios.create();
instance.interceptors.request.use(function () {/*...*/});
```

## Handling Errors

The general structure of axios errors is as follows:

- message - A quick summary of the error message and the status it failed with.
- name - This defines where the error originated from. For axios, it will always be an 'AxiosError'.
- stack - Provides the stack trace of the error.
- config - An axios config object with specific instance configurations defined by the user from when the request was made.
- code - Represents an axios identified error. The table below lists out specific definitions for internal axios error.
- status - HTTP response status code. See here for common HTTP response status code meanings.

```js
axios.get('/user/12345')
  .catch(function (error) {
    if (error.response) {
      // The request was made and the server responded with a status code
      // that falls out of the range of 2xx
      console.log(error.response.data);
      console.log(error.response.status);
      console.log(error.response.headers);
    } else if (error.request) {
      // The request was made but no response was received
      // `error.request` is an instance of XMLHttpRequest in the browser and an instance of
      // http.ClientRequest in node.js
      console.log(error.request);
    } else {
      // Something happened in setting up the request that triggered an Error
      console.log('Error', error.message);
    }
    console.log(error.config);
  });
```

Using the validateStatus config option, you can define HTTP code(s) that should throw an error.

```js
axios.get('/user/12345', {
  validateStatus: function (status) {
    return status < 500; // Resolve only if the status code is less than 500
  }
})
```

Using toJSON you get an object with more information about the HTTP error.

```js
axios.get('/user/12345')
  .catch(function (error) {
    console.log(error.toJSON());
  });
```

## Cancellation

### Cancelling requests

Setting the timeout property in an axios call handles response related timeouts.

In some cases (e.g. network connection becomes unavailable) an axios call would benefit from cancelling the connection early. Without cancellation, the axios call can hang until the parent code/stack times out (might be a few minutes in a server-side applications).

To terminate an axios call you can use following methods:

- signal
- cancelToken (deprecated)
- Combining timeout and cancellation method (e.g. signal) should cover response related timeouts AND connection related timeouts.

### signal: AbortController

Starting from v0.22.0 Axios supports AbortController to cancel requests in fetch API way:

```js
const controller = new AbortController();

axios.get('/foo/bar', {
   signal: controller.signal
}).then(function(response) {
   //...
});
// cancel the request
controller.abort()
```

Example with a timeout using latest AbortSignal.timeout() API [nodejs 17.3+]:

```js
axios.get('/foo/bar', {
   signal: AbortSignal.timeout(5000) //Aborts request after 5 seconds
}).then(function(response) {
   //...
});
```

Example with a timeout helper function:

```js
function newAbortSignal(timeoutMs) {
  const abortController = new AbortController();
  setTimeout(() => abortController.abort(), timeoutMs || 0);

  return abortController.signal;
}

axios.get('/foo/bar', {
   signal: newAbortSignal(5000) //Aborts request after 5 seconds
}).then(function(response) {
   //...
});
```

### CancelToken deprecated

You can also cancel a request using a CancelToken.

The axios cancel token API is based on the withdrawn cancelable promises proposal.

This API is deprecated since v0.22.0 and shouldn't be used in new projects

You can create a cancel token using the CancelToken.source factory as shown below:

```js
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

axios.get('/user/12345', {
  cancelToken: source.token
}).catch(function (thrown) {
  if (axios.isCancel(thrown)) {
    console.log('Request canceled', thrown.message);
  } else {
    // handle error
  }
});

axios.post('/user/12345', {
  name: 'new name'
}, {
  cancelToken: source.token
})

// cancel the request (the message parameter is optional)
source.cancel('Operation canceled by the user.');
```


You can also create a cancel token by passing an executor function to the CancelToken constructor:

```js
const CancelToken = axios.CancelToken;
let cancel;

axios.get('/user/12345', {
  cancelToken: new CancelToken(function executor(c) {
    // An executor function receives a cancel function as a parameter
    cancel = c;
  })
});

// cancel the request
cancel();
```

Note: you can cancel several requests with the same cancel token / signal.

During the transition period, you can use both cancellation APIs, even for the same request:

```js
const controller = new AbortController();

const CancelToken = axios.CancelToken;
const source = CancelToken.source();

axios.get('/user/12345', {
  cancelToken: source.token,
  signal: controller.signal
}).catch(function (thrown) {
  if (axios.isCancel(thrown)) {
    console.log('Request canceled', thrown.message);
  } else {
    // handle error
  }
});

axios.post('/user/12345', {
  name: 'new name'
}, {
  cancelToken: source.token
})

// cancel the request (the message parameter is optional)
source.cancel('Operation canceled by the user.');
// OR
controller.abort(); // the message parameter is not supported
```

## URL-Encoding Bodies

By default, axios serializes JavaScript objects to JSON. To send data in the application/x-www-form-urlencoded format instead, you can use one of the following approaches.

Browser
In a browser, you can use the URLSearchParams API as follows:

```js
const params = new URLSearchParams();
params.append('param1', 'value1');
params.append('param2', 'value2');
axios.post('/foo', params);
```

Note that URLSearchParams is not supported by all browsers (see caniuse.com), but there is a polyfill available (make sure to polyfill the global environment).

Alternatively, you can encode data using the qs library:

```js
const qs = require('qs');
axios.post('/foo', qs.stringify({ 'bar': 123 }));
```

Or in another way (ES6),
```js
import qs from 'qs';
const data = { 'bar': 123 };
const options = {
  method: 'POST',
  headers: { 'content-type': 'application/x-www-form-urlencoded' },
  data: qs.stringify(data),
  url,
};
axios(options);
```

Node.js
Query string
In node.js, you can use the querystring module as follows:

```js
const querystring = require('querystring');
axios.post('http://something.com/', querystring.stringify({ foo: 'bar' }));
```

or 'URLSearchParams' from 'url module' as follows:

```js
const url = require('url');
const params = new url.URLSearchParams({ foo: 'bar' });
axios.post('http://something.com/', params.toString());
```
You can also use the qs library.

Note: The qs library is preferable if you need to stringify nested objects, as the querystring method has known issues with that use case (https://github.com/nodejs/node-v0.x-archive/issues/1665).

🆕 Automatic serialization
Axios will automatically serialize the data object to urlencoded format if the content-type header is set to application/x-www-form-urlencoded.

This works both in the browser and in node.js:

```js
const data = {
  x: 1,
  arr: [1, 2, 3],
  arr2: [1, [2], 3],
  users: [{name: 'Peter', surname: 'Griffin'}, {name: 'Thomas', surname: 'Anderson'}],
};

await axios.post('https://postman-echo.com/post', data,
  {headers: {'content-type': 'application/x-www-form-urlencoded'}}
);
```

The server will handle it as
```js
  {
    x: '1',
    'arr[]': [ '1', '2', '3' ],
    'arr2[0]': '1',
    'arr2[1][0]': '2',
    'arr2[2]': '3',
    'arr3[]': [ '1', '2', '3' ],
    'users[0][name]': 'Peter',
    'users[0][surname]': 'griffin',
    'users[1][name]': 'Thomas',
    'users[1][surname]': 'Anderson'
  }
```

If your server framework's request body parser (like body-parser of express.js) supports nested objects decoding, you will automatically receive the same server object that you submitted.

Echo server example (express.js) :

```js
  var app = express();
  
  app.use(bodyParser.urlencoded({ extended: true })); // support url-encoded bodies
  
  app.post('/', function (req, res, next) {
     res.send(JSON.stringify(req.body));
  });

  server = app.listen(3000);
```

## Multipart Bodies
Posting data as multipart/form-data
Using FormData API
Browser
```js
const form = new FormData();
form.append('my_field', 'my value');
form.append('my_buffer', new Blob([1,2,3]));
form.append('my_file', fileInput.files[0]);

axios.post('https://example.com', form)
```

The same result can be achieved using the internal Axios serializer and corresponding shorthand method:

```js
axios.postForm('https://httpbin.org/post', {
  my_field: 'my value',
  my_buffer: new Blob([1,2,3]),
  my_file:  fileInput.files // FileList will be unwrapped as separate fields
});
```

HTML form can be passed directly as a request payload.

Node.js
```js
import axios from 'axios';

const form = new FormData();
form.append('my_field', 'my value');
form.append('my_buffer', new Blob(['some content']));

axios.post('https://example.com', form)
```

Since node.js does not currently support creating a Blob from a file, you can use a third-party package for this purpose.

```js
import {fileFromPath} from 'formdata-node/file-from-path'

form.append('my_field', 'my value');
form.append('my_file', await fileFromPath('/foo/bar.jpg'));

axios.post('https://example.com', form)
```
For Axios older than v1.3.0 you must import form-data package.

```js
const FormData = require('form-data');

const form = new FormData();
form.append('my_field', 'my value');
form.append('my_buffer', new Buffer(10));
form.append('my_file', fs.createReadStream('/foo/bar.jpg'));

axios.post('https://example.com', form)
```

🆕 Automatic serialization
Starting from v0.27.0, Axios supports automatic object serialization to a FormData object if the request Content-Type header is set to multipart/form-data.

The following request will submit the data in a FormData format (Browser & Node.js):
```js
import axios from 'axios';

axios.post('https://httpbin.org/post', {
  user: {
    name: 'Dmitriy'
  },
  file: fs.createReadStream('/foo/bar.jpg')
}, {
  headers: {
    'Content-Type': 'multipart/form-data'
  }
}).then(({data})=> console.log(data));
```

Axios FormData serializer supports some special endings to perform the following operations:

{} - serialize the value with JSON.stringify
[] - unwrap the array-like object as separate fields with the same key
NOTE: unwrap/expand operation will be used by default on arrays and FileList objects

FormData serializer supports additional options via config.formSerializer: object property to handle rare cases:

visitor: Function - user-defined visitor function that will be called recursively to serialize the data object to a FormData object by following custom rules.

dots: boolean = false - use dot notation instead of brackets to serialize arrays and objects;

metaTokens: boolean = true - add the special ending (e.g user{}: '{"name": "John"}') in the FormData key. The back-end body-parser could potentially use this meta-information to automatically parse the value as JSON.

indexes: null|false|true = false - controls how indexes will be added to unwrapped keys of flat array-like objects

null - don't add brackets (arr: 1, arr: 2, arr: 3)
false(default) - add empty brackets (arr[]: 1, arr[]: 2, arr[]: 3)
true - add brackets with indexes (arr[0]: 1, arr[1]: 2, arr[2]: 3)
Let's say we have an object like this one:

```js
const obj = {
  x: 1,
  arr: [1, 2, 3],
  arr2: [1, [2], 3],
  users: [{name: 'Peter', surname: 'Griffin'}, {name: 'Thomas', surname: 'Anderson'}],
  'obj2{}': [{x:1}]
};
```

The following steps will be executed by the Axios serializer internally:

```js
const formData= new FormData();
formData.append('x', '1');
formData.append('arr[]', '1');
formData.append('arr[]', '2');
formData.append('arr[]', '3');
formData.append('arr2[0]', '1');
formData.append('arr2[1][0]', '2');
formData.append('arr2[2]', '3');
formData.append('users[0][name]', 'Peter');
formData.append('users[0][surname]', 'Griffin');
formData.append('users[1][name]', 'Thomas');
formData.append('users[1][surname]', 'Anderson');
formData.append('obj2{}', '[{"x":1}]');
```
```js
import axios from 'axios';

axios.post('https://httpbin.org/post', {
  'myObj{}': {x: 1, s: "foo"},
  'files[]': document.querySelector('#fileInput').files 
}, {
  headers: {
    'Content-Type': 'multipart/form-data'
  }
}).then(({data})=> console.log(data));
```

Axios supports the following shortcut methods: postForm, putForm, patchForm which are just the corresponding http methods with the content-type header preset to multipart/form-data.

FileList object can be passed directly:
```js
await axios.postForm('https://httpbin.org/post', document.querySelector('#fileInput').files)
```

All files will be sent with the same field names: files[];



