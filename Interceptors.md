# Axios Interceptors Guide

Interceptors allow you to run your code or modify the request/response before it is handled by `then` or `catch`. They act as middleware for your HTTP requests.

This document covers common patterns and advanced use cases for Axios interceptors.

---

## 1. Request Interceptors

Request interceptors are useful for attaching authentication tokens, adding common headers, or logging outgoing requests before they leave the browser.

```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.example.com'
});

// Add a request interceptor
api.interceptors.request.use(
  (config) => {
    // 1. You can modify the config object before it is sent
    const token = localStorage.getItem('auth_token');
    
    if (token) {
      // 2. Attach Authorization header dynamically
      config.headers.Authorization = `Bearer ${token}`;
    }
    
    // 3. Log the outgoing request
    console.log(`[Request] ${config.method.toUpperCase()} ${config.url}`);
    
    // You MUST return the config object, otherwise the request will hang!
    return config;
  },
  (error) => {
    // Handle errors that occurred during request setup
    return Promise.reject(error);
  }
);
```

---

## 2. Response Interceptors

Response interceptors are executed before the final `then` or `catch` block in your application. They are ideal for global error handling, data transformation, and token refreshing.

```javascript
// Add a response interceptor
api.interceptors.response.use(
  (response) => {
    // 1. Any status code that lies within the range of 2xx causes this function to trigger
    
    // Example: Simplify data structure (strip axios wrapper)
    // Only do this if you don't need headers/status in your components
    // return response.data; 
    
    return response;
  },
  (error) => {
    // 2. Any status codes that fall outside the range of 2xx cause this function to trigger
    
    // Centralized Error Logging
    console.error('API Error Response:', error.response?.status, error.response?.data);

    // Common Global 401 Unauthorized handling
    if (error.response?.status === 401) {
      // Redirect to login or clear local state
      console.warn("User session expired. Redirecting to login...");
      // window.location.href = '/login';
    }

    return Promise.reject(error);
  }
);
```

---

## 3. Advanced: JWT Token Refresh Pattern

A very common use case for response interceptors is automatically refreshing access tokens when you receive a `401 Unauthorized` response.

```javascript
// A simple flag to determine if a refresh is already in progress
let isRefreshing = false;
// Queue of requests waiting for the new token
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });  
  failedQueue = [];
}

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // If 401 and we haven't already retried this request
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      if (isRefreshing) {
        // If refreshing, add this request to the queue to wait for the new token
        return new Promise(function(resolve, reject) {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers['Authorization'] = 'Bearer ' + token;
          return api(originalRequest);
        }).catch(err => {
          return Promise.reject(err);
        });
      }

      isRefreshing = true;

      try {
        // Attempt to get a new token from refresh endpoint
        const refreshToken = localStorage.getItem('refresh_token');
        const { data } = await axios.post('https://api.example.com/auth/refresh', { 
           refreshToken 
        });

        const newAccessToken = data.accessToken;
        localStorage.setItem('auth_token', newAccessToken);

        // Update the header of the failed request
        originalRequest.headers['Authorization'] = 'Bearer ' + newAccessToken;
        
        // Let the queue know we have a new token and retry suspended requests
        processQueue(null, newAccessToken);
        
        // Retry the original request
        return api(originalRequest);
        
      } catch (refreshError) {
        processQueue(refreshError, null);
        // If refresh fails, log the user out
        localStorage.removeItem('auth_token');
        localStorage.removeItem('refresh_token');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

---

## 4. Ejecting (Removing) Interceptors

If you are using Axios dynamically or inside React components (e.g., attaching context/hooks variables), you might need to clean up interceptors when you no longer need them to prevent memory leaks or duplicate executions.

```javascript
// 1. When adding an interceptor, it returns an ID
const myInterceptorId = api.interceptors.request.use(function (config) {
  /*...*/
  return config;
});

// 2. Later, you can remove it using the stored ID
api.interceptors.request.eject(myInterceptorId);
```

### Ejecting in React (useEffect)

```javascript
import { useEffect } from 'react';
import api from './api';

const useAuthInterceptor = (token) => {
  useEffect(() => {
    const requestInterceptor = api.interceptors.request.use(
      (config) => {
        if (token) config.headers.Authorization = `Bearer ${token}`;
        return config;
      }
    );

    // Cleanup function runs on unmount or before token changes
    return () => {
      api.interceptors.request.eject(requestInterceptor);
    };
  }, [token]);
};
```

---

## 5. Execution Order

When you add multiple interceptors, it's important to understand the order in which they run:

- **Request Interceptors:** Execute in **Reverse Order** (Last added runs first).
- **Response Interceptors:** Execute in **Forward Order** (First added runs first).

```javascript
// Request 1
axios.interceptors.request.use(config => {
  console.log('Request Interceptor 1');
  return config;
});
// Request 2
axios.interceptors.request.use(config => {
  console.log('Request Interceptor 2');
  return config;
});

// Response 1
axios.interceptors.response.use(response => {
  console.log('Response Interceptor 1');
  return response;
});
// Response 2
axios.interceptors.response.use(response => {
  console.log('Response Interceptor 2');
  return response;
});

axios.get('/api');

// Output order:
// "Request Interceptor 2"
// "Request Interceptor 1"
// "-- actual request happens --"
// "Response Interceptor 1"
// "Response Interceptor 2"
```
