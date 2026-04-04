# Testing Axios Requests

When writing automated tests for units or components that make HTTP requests via Axios, you **never** want to hit the actual live API. Real network requests make tests slow, create flakey results due to network dependency, and can mutate real production or staging data.

There are three primary approaches to testing Axios. 

---

## 1. The Modern Standard: Mock Service Worker (MSW)

Instead of mocking the `axios` library itself, the industry standard has shifted towards intercepting network requests at the node level using **MSW**. This means your code uses Axios exactly as it would in production, but MSW catches the request and returns a mock handler.

**Why is it better?** It makes your tests library-agnostic. If you later swap Axios for `fetch`, your tests don't break!

### Setup (Jest/Vitest Environment)

```javascript
// handlers.js
import { rest } from 'msw';

export const handlers = [
  rest.get('https://api.example.com/user', (req, res, ctx) => {
    // Return a mocked 200 OK response with JSON
    return res(
      ctx.status(200),
      ctx.json({
        id: 1,
        name: 'John Doe',
      })
    );
  }),
  
  // Example of testing a server error
  rest.post('https://api.example.com/login', (req, res, ctx) => {
    return res(
      ctx.status(401),
      ctx.json({ message: 'Unauthorized' })
    );
  }),
];
```

### The Test

```javascript
// user-api.test.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
import axios from 'axios';

// 1. Setup the MSW server
const server = setupServer(...handlers);

beforeAll(() => server.listen()); // Start intercepting requests
afterEach(() => server.resetHandlers()); // Reset handlers between tests
afterAll(() => server.close()); // Clean up

test('fetches user data successfully', async () => {
  // Axios makes an actual request, but MSW intercepts it instantly
  const response = await axios.get('https://api.example.com/user');
  
  expect(response.status).toBe(200);
  expect(response.data.name).toBe('John Doe');
});

test('handles 401 unauthorized errors', async () => {
  // We expect this axios call to cleanly throw an error caught by MSW
  await expect(axios.post('https://api.example.com/login')).rejects.toThrow();
  
  try {
     await axios.post('https://api.example.com/login');
  } catch (error) {
     expect(error.response.status).toBe(401);
     expect(error.response.data.message).toBe('Unauthorized');
  }
});
```

---

## 2. Using `axios-mock-adapter` 

If you don't want to use MSW or heavily rely on custom Axios instances, `axios-mock-adapter` is a specialized library built exactly for mocking Axios instances.

It is brilliant for testing complex interceptors attached to customized Axios instances.

```javascript
import axios from 'axios';
import MockAdapter from 'axios-mock-adapter';
import { fetchUsers } from './userService'; // Your actual source code file

// 1. Create a mock bound to the default axios instance
const mock = new MockAdapter(axios);

describe('UserService', () => {
  afterEach(() => {
    // Always clean up after every test
    mock.reset();
  });

  it('returns an array of users', async () => {
    const mockData = [{ id: 1, name: 'Alice' }];

    // 2. Instruct the adapter what to return when GET /users is called
    mock.onGet('/users').reply(200, mockData);

    // 3. Call your function (which uses axios internally)
    const users = await fetchUsers();

    expect(users).toEqual(mockData);
  });

  it('handles network timeouts', async () => {
    // Easily test how your code responds to a network timeout
    mock.onGet('/users').networkError();

    await expect(fetchUsers()).rejects.toThrow();
  });
});
```

---

## 3. The Classic Approach: `jest.mock('axios')`

If you want zero additional dependencies, you can mock the Axios module entirely using native Jest functionality. This is common but slightly more fragile because you bypass all of Axios's internal mechanics.

```javascript
// user-service.test.js
import axios from 'axios';
import { getUserName } from './userService';

// 1. Completely hijack the axios module
jest.mock('axios');

describe('getUserName', () => {
  it('fetches successfully data from an API', async () => {
    const mockResponse = {
      data: { name: 'Dinesh' }
    };
    
    // 2. We dictate what the 'get' function returns
    axios.get.mockResolvedValueOnce(mockResponse);

    const result = await getUserName();
    
    // 3. Assert correct output
    expect(result).toBe('Dinesh');
    
    // 4. Assert axios was called correctly
    expect(axios.get).toHaveBeenCalledWith('/api/user');
    expect(axios.get).toHaveBeenCalledTimes(1);
  });

  it('handles failures gracefully', async () => {
    const errorMessage = 'Network Error';
    
    // Mock the rejection 
    axios.get.mockRejectedValueOnce(new Error(errorMessage));

    await expect(getUserName()).rejects.toThrow('Network Error');
  });
});
```

### Summary Recommendation:
1. Use **MSW (Mock Service Worker)** for standard React/Node applications. It's the most robust and realistic.
2. Use **`axios-mock-adapter`** if you are specifically testing complex Axios Interceptors and custom instances.
3. Use **`jest.mock('axios')`** only for tiny utility scripts where adding an external mocking library is overkill.
