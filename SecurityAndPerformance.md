# Security and Performance in Axios

Optimizing your Axios implementation involves two critical pillars: **Security** (protecting data and users) and **Performance** (ensuring speed and scalability). This guide covers best practices and built-in features to master both.

---

## 🛡️ Security Best Practices

### 1. XSRF/CSRF Protection
Axios provides built-in support for protecting against Cross-Site Request Forgery. It does this by reading a token from a cookie and including it in a header.

```javascript
// Default configuration
const instance = axios.create({
  xsrfCookieName: 'XSRF-TOKEN', // default
  xsrfHeaderName: 'X-XSRF-TOKEN', // default
});
```

> [!NOTE]
> This only works if the cookie is accessible to JavaScript. For `HttpOnly` cookies, you must handle CSRF tokens manually via meta tags or hidden inputs.

### 2. Handling Sensitive Information
Never log request headers or bodies that contain passwords, tokens, or PII (Personally Identifiable Information).

**Good Practice: Interceptor Sanitization**
```javascript
axios.interceptors.request.use(config => {
  const sanitizedConfig = { ...config };
  if (sanitizedConfig.headers['Authorization']) {
    console.log('Request sent with Authorization header');
  }
  return config;
});
```

### 3. Preventing Man-in-the-Middle (MITM)
Always use `https://`. In Node.js environments, you can also enforce custom SSL certificates.

```javascript
const https = require('https');

const instance = axios.create({
  httpsAgent: new https.Agent({  
    rejectUnauthorized: true // Enforce valid certificates
  })
});
```

### 4. CORS and Credentials
When making cross-origin requests that require cookies or Authorization headers, set `withCredentials` to `true`.

```javascript
axios.get('https://api.example.com/user', {
  withCredentials: true
});
```

---

## ⚡ Performance Optimization

### 1. Request Cancellation
Unfinished requests consume resources. Use `AbortController` to cancel requests when a user navigates away or a component unmounts.

```javascript
const controller = new AbortController();

axios.get('/data', {
  signal: controller.signal
}).catch(err => {
  if (axios.isCancel(err)) {
    console.log('Request canceled');
  }
});

// Cancel the request
controller.abort();
```

### 2. Timeouts
Never leave a request "hanging." Always set a reasonable timeout to free up socket connections.

```javascript
axios.create({
  timeout: 5000 // 5 seconds
});
```

### 3. Concurrent Requests
Use `Promise.all` to fetch multiple resources in parallel rather than sequentially.

```javascript
async function fetchData() {
  const [user, posts] = await Promise.all([
    axios.get('/user'),
    axios.get('/posts')
  ]);
}
```

### 4. Response Caching
Axios doesn't cache by default. For performance, use an interceptor to cache GET requests.

```javascript
const cache = new Map();

axios.interceptors.request.use(config => {
  if (config.method === 'get' && cache.has(config.url)) {
    return Promise.resolve(cache.get(config.url));
  }
  return config;
});

axios.interceptors.response.use(response => {
  if (response.config.method === 'get') {
    cache.set(response.config.url, response);
  }
  return response;
});
```

### 5. Compression (Node.js)
In Node.js, you can specify headers to request compressed data, significantly reducing bandwidth.

```javascript
axios.get('https://api.example.com', {
  headers: { 'Accept-Encoding': 'gzip, deflate, br' }
});
```

---

## 🛠️ Summary Checklist

| Feature | Category | Best Practice |
| :--- | :--- | :--- |
| **XSRF** | Security | Match cookie/header names with backend. |
| **HTTPS** | Security | Always use `https://` for API URLs. |
| **AbortController**| Performance | Use in React `useEffect` cleanups. |
| **Timeouts** | Performance | Set global defaults (e.g., 10s). |
| **Interceptors**| Both | Use for logging and caching logic. |
