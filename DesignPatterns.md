# Axios Design Patterns

Structuring your API layer correctly is the difference between a codebase that scales and one that becomes a "spaghetti" mess of `axios.get` calls scattered everywhere.

This guide covers modern design patterns for Axios in enterprise-scale applications.

---

## 1. The Centralized Instance (Base API)

Instead of importing the global `axios` package in every file, you should create a centralized instance. This allows you to set `baseURL`, global `headers`, and `interceptors` in one place.

```javascript
// src/api/client.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'https://api.example.com',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add global interceptors here...
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default apiClient;
```

---

## 2. The Repository (Service) Pattern

The Repository pattern organizes your API calls by "domain" or "resource". Instead of making requests directly in components, you call methods from a service object.

**Benefits:** Single point of change, cleaner components, and easy mocking for tests.

```javascript
// src/api/services/userService.js
import apiClient from '../client';

export const userService = {
  getProfile: () => apiClient.get('/users/profile'),
  
  updateProfile: (userData) => apiClient.put('/users/profile', userData),
  
  getSettings: (userId) => apiClient.get(`/users/${userId}/settings`),
  
  searchUsers: (query) => apiClient.get('/users/search', { params: { q: query } }),
};
```

**Usage in a Component:**
```javascript
import { userService } from './api/services/userService';

const Profile = () => {
  const handleUpdate = async (data) => {
    try {
      const response = await userService.updateProfile(data);
      // ... handle success
    } catch (err) {
      // ... handle error
    }
  };
};
```

---

## 3. The API Factory Pattern

If your application interacts with multiple different APIs (e.g., a Main API, a Search API, and a Stripe API), use a Factory pattern to generate pre-configured instances.

```javascript
// src/api/factory.js
import axios from 'axios';

const createInstance = (config) => {
  const instance = axios.create(config);
  
  // Apply standard interceptors to every instance created
  instance.interceptors.response.use(
    (res) => res,
    (err) => {
      console.error(`[API Error] ${err.config.url}`, err);
      return Promise.reject(err);
    }
  );
  
  return instance;
};

export const mainApi = createInstance({ baseURL: 'https://api.core.com' });
export const searchApi = createInstance({ baseURL: 'https://search.srv.com' });
```

---

## 4. The Custom Hook Pattern (React)

For React applications, wrapping Axios calls in a custom hook is the gold standard. This manages `loading`, `error`, and `data` states automatically.

```javascript
// src/hooks/useApi.js
import { useState, useCallback } from 'react';

export const useApi = (apiFunc) => {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);

  const request = useCallback(async (...args) => {
    setLoading(true);
    setError(null);
    try {
      const response = await apiFunc(...args);
      setData(response.data);
      return response.data;
    } catch (err) {
      setError(err || 'Unexpected Error');
      throw err;
    } finally {
      setLoading(false);
    }
  }, [apiFunc]);

  return { data, error, loading, request };
};
```

**Usage:**
```javascript
const { data, loading, request } = useApi(userService.getProfile);

useEffect(() => {
  request();
}, [request]);
```

---

## 5. Next.js SSR / Proxy Pattern

In SSR environments (like Next.js), you often need to forward user cookies or headers from the browser to your internal API.

```javascript
// src/api/ssrClient.js
import axios from 'axios';

export const getSsrClient = (context) => {
  const { req } = context;
  
  return axios.create({
    baseURL: process.env.INTERNAL_API_URL,
    headers: {
      // Forward cookies so the API knows who the user is
      Cookie: req.headers.cookie || '',
      // Forward original IP if needed for rate limiting
      'X-Forwarded-For': req.headers['x-forwarded-for'] || '',
    },
  });
};

// In getServerSideProps
export async function getServerSideProps(context) {
  const client = getSsrClient(context);
  const { data } = await client.get('/dashboard');
  
  return { props: { data } };
}
```

---

## Summary Checklist
- [ ] **Don't hardcode URLs.** Use environment variables.
- [ ] **Abstract the client.** Never use `axios` directly in your UI logic.
- [ ] **Group by Domain.** Use the Repository pattern for readable services.
- [ ] **Manage State Professionally.** Use hooks or libraries like React Query to handle loading/error states.
- [ ] **Handle SSR Headers.** If using Next.js, ensure you are forwarding authentication headers.
