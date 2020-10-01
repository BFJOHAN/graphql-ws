<div align="center">
  <br />

![GraphQLOverWebSocket](https://user-images.githubusercontent.com/25294569/94527042-172dba00-023f-11eb-944b-88c0bd58a8d2.gif)

  <h6>Coherent, zero-dependency, lazy, simple, <a href="PROTOCOL.md">GraphQL over WebSocket Protocol</a> compliant server and client.</h6>

[![Continuous integration](https://github.com/enisdenjo/graphql-transport-ws/workflows/Continuous%20integration/badge.svg)](https://github.com/enisdenjo/graphql-transport-ws/actions?query=workflow%3A%22Continuous+integration%22) [![graphql-transport-ws](https://img.shields.io/npm/v/graphql-transport-ws.svg?label=graphql-transport-ws&logo=npm)](https://www.npmjs.com/package/graphql-transport-ws)

  <br />
</div>

## Getting started

#### Install

```shell
$ yarn add graphql-transport-ws
```

#### Create a GraphQL schema

```ts
import { buildSchema } from 'graphql';

// Construct a schema, using GraphQL schema language
const schema = buildSchema(`
  type Query {
    hello: String
  }
  type Subscription {
    greetings: String
  }
`);

// The roots provide resolvers for each GraphQL operation
const roots = {
  query: {
    hello: () => 'Hello World!',
  },
  subscription: {
    greetings: async function* sayHiIn5Languages() {
      for (const hi of ['Hi', 'Bonjour', 'Hola', 'Ciao', 'Zdravo']) {
        yield { greetings: hi };
      }
    },
  },
};
```

#### Start the server

```ts
import http from 'http';
import { execute, subscribe } from 'graphql';
import { createServer } from 'graphql-transport-ws';

const server = http.createServer(function weServeSocketsOnly(_, res) {
  res.writeHead(404);
  res.end();
});

createServer(
  {
    schema, // from the previous step
    roots, // from the previous step
    execute,
    subscribe,
  },
  {
    server,
    path: '/graphql',
  },
);

server.listen(443);
```

#### Use the client

```ts
import { createClient } from 'graphql-transport-ws';

const client = createClient({
  url: 'wss://welcomer.com/graphql',
});

// query
(async () => {
  const result = await new Promise((resolve, reject) => {
    let result;
    client.subscribe(
      {
        query: '{ hello }',
      },
      {
        next: (data) => (result = data),
        error: reject,
        complete: () => resolve(result),
      },
    );
  });

  expect(result).toEqual({ hello: 'Hello World!' });
})();

// subscription
(async () => {
  const onNext = () => {
    /**/
  };

  await new Promise((resolve, reject) => {
    client.subscribe(
      {
        query: 'subscription { greetings }',
      },
      {
        next: onNext,
        error: reject,
        complete: resolve,
      },
    );
  });

  expect(onNext).toBeCalledTimes(5); // we say "Hi" in 5 languages
})();
```

## Recipes

<details>
<summary>Client usage with Promise</summary>

```ts
import { createClient, SubscribePayload } from 'graphql-transport-ws';

const client = createClient({
  url: 'wss://hey.there/graphql',
});

async function execute<T>(payload: SubscribePayload) {
  return new Promise((resolve, reject) => {
    let result: T;
    client.subscribe<T>(payload, {
      next: (data) => (result = data),
      error: reject,
      complete: () => resolve(result),
    });
  });
}

// use
(async () => {
  try {
    const result = await execute({
      query: '{ hello }',
    });
    // complete
    // next = result = { data: { hello: 'Hello World!' } }
  } catch (err) {
    // error
  }
})();
```

</details>

<details>
<summary>Client usage with <a href="https://github.com/tc39/proposal-observable">Observable</a></summary>

```ts
import { Observable } from 'relay-runtime';
// or
import { Observable } from '@apollo/client';
// or
import { Observable } from 'rxjs';
// or
import Observable from 'zen-observable';
// or any other lib which implements Observables as per the ECMAScript proposal: https://github.com/tc39/proposal-observable

const client = createClient({
  url: 'wss://graphql.loves/observables',
});

function toObservable(operation) {
  return new Observable((observer) => client.subscribe(operation, observer));
}

const observable = toObservable({ query: `subscription { ping }` });

const subscription = observable.subscribe({
  next: (data) => {
    expect(data).toBe({ data: { ping: 'pong' } });
  },
});

// ⏱
subscription.unsubscribe();
```

</details>

<details>
<summary>Client usage with <a href="https://relay.dev">Relay</a></summary>

```ts
import {
  Network,
  Observable,
  RequestParameters,
  Variables,
} from 'relay-runtime';
import { createClient } from 'graphql-transport-ws';

const subscriptionsClient = createClient({
  url: 'wss://i.love/graphql',
  connectionParams: () => {
    const session = getSession();
    if (!session) {
      return {};
    }
    return {
      Authorization: `Bearer ${session.token}`,
    };
  },
});

// yes, both fetch AND subscribe handled in one implementation
function fetchOrSubscribe(operation: RequestParameters, variables: Variables) {
  return Observable.create((sink) => {
    if (!operation.text) {
      return sink.error(new Error('Operation text cannot be empty'));
    }
    return subscriptionsClient.subscribe(
      {
        operationName: operation.name,
        query: operation.text,
        variables,
      },
      {
        ...sink,
        error: (err) => {
          if (err instanceof Error) {
            sink.error(err);
          } else if (err instanceof CloseEvent) {
            sink.error(
              new Error(
                `Socket closed with event ${err.code}` + err.reason
                  ? `: ${err.reason}` // reason will be available on clean closes
                  : '',
              ),
            );
          } else {
            // GraphQLError[]
            sink.error(new Error(err.map(({ message }) => message).join(', ')));
          }
        },
      },
    );
  });
}

export const network = Network.create(fetchOrSubscribe, fetchOrSubscribe);
```

</details>

<details>
<summary>Client usage with <a href="https://www.apollographql.com">Apollo</a></summary>

```typescript
import { ApolloLink, Operation, FetchResult, Observable } from '@apollo/client';
import { createClient, Config, Client } from 'graphql-transport-ws';

class WebSocketLink extends ApolloLink {
  private client: Client;

  constructor(config: Config) {
    super();
    this.client = createClient(config);
  }

  public request(operation: Operation): Observable<FetchResult> {
    return new Observable((sink) => {
      return this.client.subscribe<FetchResult>(operation, {
        ...sink,
        error: (err) => {
          if (err instanceof Error) {
            sink.error(err);
          } else if (err instanceof CloseEvent) {
            sink.error(
              new Error(
                `Socket closed with event ${err.code}` + err.reason
                  ? `: ${err.reason}` // reason will be available on clean closes
                  : '',
              ),
            );
          } else {
            // GraphQLError[]
            sink.error(new Error(err.map(({ message }) => message).join(', ')));
          }
        },
      });
    });
  }
}

const link = new WebSocketLink({
  url: 'wss://where.is/graphql',
  connectionParams: () => {
    const session = getSession();
    if (!session) {
      return {};
    }
    return {
      Authorization: `Bearer ${session.token}`,
    };
  },
});
```

</details>

<details>
<summary>Client usage in Node</summary>

```ts
const WebSocket = require('ws');
const Crypto = require('crypto');
const { createClient } = require('graphql-transport-ws');

const client = createClient({
  url: 'wss://no.browser/graphql',
  webSocketImpl: WebSocket,
  /**
   * Generates a v4 UUID to be used as the ID.
   * Reference: https://stackoverflow.com/a/2117523/709884
   */
  generateID: () =>
    ([1e7] + -1e3 + -4e3 + -8e3 + -1e11).replace(/[018]/g, (c) =>
      (c ^ (Crypto.randomBytes(1)[0] & (15 >> (c / 4)))).toString(16),
    ),
});

// consider other recipes for usage inspiration
```

</details>

<details>
<summary>Server usage with <a href="https://github.com/graphql/express-graphql">Express GraphQL</a></summary>

```typescript
import http from 'http';
import express from 'express';
import { graphqlHTTP } from 'express-graphql';
import { createServer } from 'graphql-transport-ws';
import { execute, subscribe } from 'graphql';
import { schema } from 'my-graphql-schema';

// create express and middleware
const app = express();
app.use('/graphql', graphqlHTTP({ schema }));

// create a http server using express
const server = http.createServer(app);

server.listen(PORT, () => {
  createServer(
    {
      schema,
      execute,
      subscribe,
    },
    {
      server,
      path: '/graphql', // you can use the same path too, just use the `ws` schema
    },
  );
});
```

</details>

## [Documentation](docs/)

Check the [docs folder](docs/) out for [TypeDoc](https://typedoc.org) generated documentation.

## [How does it work?](PROTOCOL.md)

Read about the exact transport intricacies used by the library in the [GraphQL over WebSocket Protocol document](PROTOCOL.md).

## [Want to help?](CONTRIBUTING.md)

File a bug, contribute with code, or improve documentation? Read up on our guidelines for [contributing](CONTRIBUTING.md) and drive development with `yarn test --watch` away!
