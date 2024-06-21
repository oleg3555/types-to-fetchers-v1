# types-to-fetchers-v1

[![NPM version](https://img.shields.io/npm/v/types-to-fetchers-v1.svg?style=flat)](https://www.npmjs.com/package/types-to-fetchers)
![Tests](https://github.com/neruchev/types-to-fetchers/workflows/Tests/badge.svg)

Automatically creates fetchers from declarative descriptions of APIs and types. Based on [axios](https://www.npmjs.com/package/axios).

## Installation

Install it with yarn:

```sh
yarn add types-to-fetchers-v2
```

Or with npm:

```sh
npm install types-to-fetchers-v2
```

## Usage

Use types that describe your API. You can use e.g. [fastify-extract-definitions](https://www.npmjs.com/package/fastify-extract-definitions) to automatically generate them.

Write or extract types:

```ts
interface Error {
  code: number;
  error: string;
}

interface API {
  '/': {
    GET: {
      Reply: {
        version: string;
        mode: 'production' | 'development';
      };
    };
  };
  '/foo/:bar': {
    GET: {
      Params: { bar: string };
      Reply: string;
    };
    POST: {
      Params: { bar: string };
      Body: { baz: string };
      Reply: Error | string;
    };
  };
}
```

Make fetchers:

```ts
const api = makeApi<API, Error>(
  {
    '/': ['GET'],
    '/foo/:bar': ['GET', 'POST'],
  },
  { baseURL: 'https://my-api.example.com/' }
);
```

Use it:

```ts
// POST `{ baz: 'def' }` to `/foo/abc`
const reply = await api['/foo/:bar'].POST({
  Params: { bar: 'abc' },
  Body: { baz: 'def' },
});

console.log(reply); // Error | string
```

### Abort request

```ts
const abortController = new AbortController();

const reply = await api['/foo/:bar'].POST({
  Params: { bar: 'abc' },
  Body: { baz: 'def' },
  Axios: { signal: abortController.signal },
});

// ...

abortController.abort();
```

### Handle File Progress

```ts
const reply = await api['/foo/:bar'].POST({
  Params: { bar: 'abc' },
  Body: { baz: 'def' },
  Axios: {
    onUploadProgress: (progressEvent) => {
      const percentCompleted = Math.round(
        (progressEvent.loaded * 100) / progressEvent.total
      );
      console.log(`Upload progress: ${percentCompleted}%`);
    },
    onDownloadProgress: (progressEvent) => {
      const percentCompleted = Math.round(
        (progressEvent.loaded * 100) / progressEvent.total
      );
      console.log(`Download progress: ${percentCompleted}%`);
    },
  },
});
```

### Effects

You can use a callback to apply some effect to each request. For example, we will use the `createEffect` from the [effector](https://www.npmjs.com/package/effector) library.

_Note:_ the callback fires at the time of generation, not at the time of the call.

Write custom output types (if necessary) and make fetchers:

```ts
import { Effect, createEffect } from 'effector';
import { Payload, makeApi } from 'types-to-fetchers-v1';

type Reply<PayloadRecord extends Payload> = Exclude<
  PayloadRecord['Reply'],
  Error
>;

type Methods<MethodsRecord extends object> = {
  [Method in keyof MethodsRecord]: Effect<
    Omit<MethodsRecord[Method], 'Reply'> & AxiosOptions,
    Reply<MethodsRecord[Method]>
  >;
};

type Endpoints<EndpointsRecord extends object> = {
  [Endpoint in keyof EndpointsRecord]: Methods<EndpointsRecord[Endpoint]>;
};

type Output = Endpoints<API>;

const api = makeApi<API, Error, Output>(
  {
    '/': ['GET'],
    '/foo/:bar': ['GET', 'POST'],
  },
  {
    baseURL: 'https://my-api.example.com/',
    effect: (action) => createEffect(action),
  }
);
```

Use it:

```ts
import { createStore } from 'effector';

type State = {
  version: string | null;
};

const initialState: State = {
  version: null,
};

const $app = createStore<State>(initialState).on(
  api['/'].GET.doneData,
  (state, payload): State => ({
    ...state,
    version: payload.version,
  })
);
```

```ts
import React, { useEffect } from 'react';

const ComponentName: React.FC = () => {
  useEffect(() => {
    const abortController = new AbortController();

    api['/'].GET({ Axios: { signal: abortController.signal } });

    return () => abortController.abort();
  }, []);

  return <>...</>;
};
```

## License

MIT
