# Advanced Axios Patterns in React

This guide covers advanced, real-world patterns for using Axios within React applications.

## 1. Centralized Axios Instance with Interceptors (Token Refresh)

Creating a customized Axios instance allows you to set base URLs, default headers, and intercept requests/responses. This pattern is crucial for handling authentication (like attaching JWTs and automatically refreshing them when they expire).

```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'https://api.example.com',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request Interceptor: Attach Token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response Interceptor: Handle Token Refresh
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // If error is 401 Unauthorized and we haven't retried yet
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      try {
        const refreshToken = localStorage.getItem('refreshToken');
        // Make a request to your refresh token endpoint
        const { data } = await axios.post('https://api.example.com/auth/refresh', {
          token: refreshToken,
        });

        localStorage.setItem('accessToken', data.accessToken);
        
        // Update authorization header and retry original request
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        // Handle refresh logic failure (e.g., redirect to login)
        localStorage.clear();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }
    
    return Promise.reject(error);
  }
);

export default api;
```

## 2. AbortController for Request Cancellation in `useEffect`

Preventing race conditions and memory leaks when components unmount by cancelling pending Axios requests using the modern `AbortController` API.

```javascript
import React, { useState, useEffect } from 'react';
import api from './api'; // Your customized instance
import axios from 'axios';

const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    // 1. Create the AbortController
    const controller = new AbortController();
    
    const fetchUser = async () => {
      setLoading(true);
      setError(null);
      try {
        // 2. Pass the signal to your request
        const response = await api.get(`/users/${userId}`, {
          signal: controller.signal,
        });
        setUser(response.data);
      } catch (err) {
        // 3. Ignore errors caused by cancellation
        if (axios.isCancel(err)) {
          console.log('Request canceled arbitrarily');
        } else {
          setError(err.message || 'An error occurred');
        }
      } finally {
        setLoading(false);
      }
    };

    fetchUser();

    // 4. Trigger cancel when component unmounts or userId changes
    return () => {
      controller.abort();
    };
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
};

export default UserProfile;
```

## 3. Creating a Generic Custom Hook (`useAxios`)

For smaller applications not using robust libraries (like React Query), a reusable hook simplifies component logic heavily.

```javascript
import { useState, useEffect, useCallback } from 'react';
import api from './api';
import axios from 'axios';

export const useAxios = (url, method = 'get', initialData = null) => {
  const [data, setData] = useState(initialData);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const execute = useCallback(
    async (body = null, customConfig = {}) => {
      setLoading(true);
      setError(null);
      
      const controller = new AbortController();
      
      try {
        const response = await api({
          url,
          method,
          data: body,
          signal: controller.signal,
          ...customConfig,
        });
        
        setData(response.data);
        return response.data;
      } catch (err) {
        if (!axios.isCancel(err)) {
          setError(err.response?.data?.message || err.message);
        }
        throw err;
      } finally {
        setLoading(false);
      }
    },
    [url, method]
  );

  return { data, loading, error, execute };
};

// Usage Example:
// const { data: users, loading, error, execute: fetchUsers } = useAxios('/users');
// useEffect(() => { fetchUsers(); }, [fetchUsers]);
```

## 4. Integrating Axios with TanStack React Query

For everyday production apps, pairing Axios with **TanStack React Query** is the golden standard. Axios handles the fetching logic; React Query handles caching, background fetching, and UI state.

```javascript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import api from './api';

// 1. Define API Fetcher functions
const fetchTodos = async () => {
  const { data } = await api.get('/todos');
  return data;
};

const createTodo = async (newTodo) => {
  const { data } = await api.post('/todos', newTodo);
  return data;
};

// 2. Custom hooks wrapping React Query
export const useTodos = () => {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });
};

export const useAddTodo = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: createTodo,
    onSuccess: () => {
      // Invalidate and refetch specific caches when a mutation succeeds
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
};

// 3. Clean Component
const TodoList = () => {
  const { data: todos, isLoading, isError } = useTodos();
  const { mutate: addTodo, isPending } = useAddTodo();

  if (isLoading) return <p>Loading...</p>;
  if (isError) return <p>Could not fetch todos.</p>;

  return (
    <div>
      <button 
        disabled={isPending} 
        onClick={() => addTodo({ title: 'New task', completed: false })}
      >
        {isPending ? 'Adding...' : 'Add Todo'}
      </button>
      
      <ul>
        {todos?.map((todo) => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </div>
  );
};

export default TodoList;
```

## 5. Type-Safe Axios with TypeScript (Generic Wrappers)

If you use TypeScript, passing explicit types to Axios standardizes the shape of your responses.

```typescript
import axios, { AxiosRequestConfig, AxiosResponse } from 'axios';

// Create generic wrapper to strictly type your API returns
async function request<T>(config: AxiosRequestConfig): Promise<T> {
  const response: AxiosResponse<T> = await axios(config);
  return response.data;
}

// User types
interface User {
  id: number;
  name: string;
}

// Strictly Typed API Calls
const API = {
  getUsers: () => request<User[]>({ method: 'GET', url: '/users' }),
  getUserById: (id: number) => request<User>({ method: 'GET', url: `/users/${id}` }),
  createUser: (userData: Omit<User, 'id'>) => request<User>({ 
    method: 'POST', 
    url: '/users', 
    data: userData 
  }),
};

// Usage (fully autocompleted and typed!)
const fetchAndLog = async () => {
  const users = await API.getUsers(); // 'users' is strictly typed as User[]
  console.log(users[0].name); 
};
```
