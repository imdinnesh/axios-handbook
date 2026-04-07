# Axios Error Handling Guide

Robust error handling is critical for building resilient applications. Unlike the native browser `fetch()` API—which only rejects the promise on network errors (and resolves even on 404 or 500 statuses)—**Axios automatically intercepts HTTP response codes outside the 2xx range and throws them as Javascript Errors.**

This document outlines the anatomy of an Axios error and patterns for handling them elegantly in modern environments (principally React and TypeScript).

---

## 1. Anatomy of the Axios Error Object

When an Axios request fails, the resulting error object contains three crucial layers you must check.

```javascript
axios.get('/api/user')
  .catch((error) => {
    if (error.response) {
      // 1. The server responded with a status code outside the 2xx range
      // This is the most common scenario (e.g., 400 Bad Request, 401 Unauthorized, 500 Internal Error)
      console.log('Data:', error.response.data);
      console.log('Status:', error.response.status);
      console.log('Headers:', error.response.headers);
    } else if (error.request) {
      // 2. The request was made, but no response was received
      // This means the user has lost internet connection, the server is down, or CORS blocked it.
      console.log('Request Object:', error.request);
    } else {
      // 3. Something happened in setting up the request that triggered an Error
      console.log('Error Message:', error.message);
    }
  });
```

---

## 2. Using `axios.isAxiosError` (TypeScript integration)

In modern TypeScript, the caught `error` inside a `try...catch` block defaults to type `unknown` or `any`. Axios provides a type guard called `isAxiosError` to safely narrow down the error type and gain full Intellisense.

```typescript
import axios, { AxiosError } from 'axios';

interface BackendErrorData {
  message: string;
  code: string;
  fieldErrors?: Record<string, string[]>;
}

const fetchUserProfile = async () => {
  try {
    const { data } = await axios.get('/api/profile');
    return data;
  } catch (err) {
    if (axios.isAxiosError<BackendErrorData>(err)) {
      // TS now knows `err` is an AxiosError, and `err.response.data` matches BackendErrorData
      if (err.response) {
        console.error('API Error:', err.response.data.message);
        
        // Handle specific status validation
        if (err.response.status === 422 && err.response.data.fieldErrors) {
           console.log('Validation failed:', err.response.data.fieldErrors);
        }
      } else if (err.request) {
        console.error('Network Error: Server is unreachable.');
      }
    } else {
      // Not an Axios error (e.g., a TypeError in your own code)
      console.error('Unexpected native error:', err);
    }
  }
}
```

---

## 3. Centralized View/UI Error Handling via Interceptors

Rather than writing `try...catch` blocks that handle generic server errors in every single React component, you can use a **Response Interceptor** to catch global errors centrally.

This is perfect for:
- Showing Toast notifications on 500 Server Errors.
- Automatically redirecting the user to a Login page on 401 Unauthorized.
- Displaying a generic "Offline" message when the server doesn't respond.

```javascript
import axios from 'axios';
import { toast } from 'react-hot-toast'; // Example toast library

const api = axios.create({
  baseURL: 'https://api.example.com',
});

api.interceptors.response.use(
  (response) => {
    // Return successful responses directly
    return response;
  },
  (error) => {
    if (axios.isAxiosError(error)) {
      // Global 401 Unauthorized redirect
      if (error.response?.status === 401) {
        window.location.href = '/login';
        return Promise.reject(error);
      }

      // Global Server Error handler
      if (error.response?.status >= 500) {
        toast.error("The server encountered an error. Our team has been notified.");
      }
      
      // Global Network / Timeout Error handler
      if (!error.response && error.request) {
         toast.error("Network error. Please check your internet connection.");
      }
    }

    // Still reject the promise so the calling component can handle specific field errors (e.g., 400 Bad Request)
    return Promise.reject(error);
  }
);

export default api;
```

---

## 4. Building a Generic Error Parser Utility

Different backends format their error messages differently. It is a best practice to create a utility function that parses the Axios error and extracts a human-readable string. You can use this everywhere in your app.

```javascript
import axios from 'axios';

/**
 * Extracts a safe, human-readable error string from ANY error object
 */
export const extractErrorMessage = (error) => {
  // 1. Is it an Axios Error?
  if (axios.isAxiosError(error)) {
    
    // Does the backend provide a custom message array? (e.g., NestJS / Express validation)
    if (error.response?.data?.message) {
      const backendMessage = error.response.data.message;
      return Array.isArray(backendMessage) 
        ? backendMessage.join(', ') // Combine multiple errors
        : backendMessage;
    }
    
    // Fallback based on HTTP status
    if (error.response?.status === 404) return "Requested resource was not found.";
    if (error.response?.status === 403) return "You do not have permission to perform this action.";
    
    // Network errors
    if (!error.response && error.request) return "Network Error: Could not reach the server.";
    
    // Generic axios message
    return error.message;
  }

  // 2. Is it a standard native JS Error?
  if (error instanceof Error) {
    return error.message;
  }

  // 3. Fallback when error shape is completely unknown
  return "An unexpected and unknown error occurred.";
};
```

### Usage Pattern in Components:

```javascript
import { extractErrorMessage } from './utils';

const submitForm = async (data) => {
  try {
    await api.post('/users', data);
    toast.success("User created!");
  } catch (err) {
    const readableMessage = extractErrorMessage(err);
    toast.error(readableMessage); // Always a clean string for the user!
  }
}
```

---

## 5. Changing When Axios Throws an Error (`validateStatus`)

By default, Axios throws an error for any status code >= 300. You can override this behavior per request if your backend returns data you want to handle successfully on a 400 or 404.

```javascript
axios.get('/users/status', {
  // The promise will now only reject if the status is >= 500
  validateStatus: function (status) {
    return status < 500; 
  }
}).then(response => {
  // If status is 404, the code will come here instead of the catch block!
  if (response.status === 404) {
    console.log('User not found, but we caught it in the then block!');
  }
}).catch(error => {
  console.log("This will only trigger on 500s or network failures.");
});
```
