# type-cacheable

[![Build Status](https://travis-ci.org/joshuaslate/type-cacheable.svg?branch=master)](https://travis-ci.org/joshuaslate/type-cacheable) [![Node version](https://img.shields.io/npm/dm/@type-cacheable/core.svg?maxAge=43200&label=v6.2.0%20downloads)](https://www.npmjs.com/package/@type-cacheable/core) [![Code coverage](https://codecov.io/gh/joshuaslate/type-cacheable/branch/master/graph/badge.svg)](https://codecov.io/gh/joshuaslate/type-cacheable)

TypeScript-based caching decorators to assist with caching (and clearing cache for) async methods. Currently supports Redis (`redis`, `ioredis`) and `node-cache`. If you would like to see more adapters added, please open an issue or, better yet, a pull request with an implementation.

## Usage

### Installation

```bash
npm install --save @type-cacheable/core
```

or

```bash
yarn add @type-cacheable/core
```

### Setup Adapter

You will need to set up the appropriate adapter for your cache of choice.

Redis:

- `redis` - `@type-cacheable/redis-adapter` - https://github.com/joshuaslate/type-cacheable/tree/master/packages/redis-adapter
- `ioredis` - `@type-cacheable/ioredis-adapter` - https://github.com/joshuaslate/type-cacheable/tree/master/packages/ioredis-adapter

Node-Cache:

- `node-cache` - `@type-cacheable/node-cache-adapter` https://github.com/joshuaslate/type-cacheable/tree/master/packages/node-cache-adapter

### Change Global Options

Some options can be configured globally for all decorated methods. Here is an example of how you can change these options:

```ts
// Import and set adapter as above
import cacheManager, { CacheManagerOptions } from '@type-cacheable/core';
cacheManager.setOptions(<CacheManagerOptions>{
  excludeContext: false, // Defaults to true. If you don't pass a specific hashKey into the decorators, one will be generated by serializing the arguments passed in and optionally the context of the instance the method is being called on.
  ttlSeconds: 0, // A global setting for the number of seconds the decorated method's results will be cached for.
});
```

Currently, there are three decorators available in this library: `@Cacheable`, `@CacheClear`, and `@CacheUpdate`. Here is a sample of how they can be used:

```ts
import * as Redis from 'redis';
import { Cacheable, CacheClear } from '@type-cacheable/core';

const userClient = Redis.createClient();

class TestClass {
  private values: any[] = [1, 2, 3, 4, 5];
  private userRepository: Repository<User>;

  // This static method is being called to generate a cache key based on the given arguments.
  // Not featured here: the second argument, context, which is the instance the method
  // was called on.
  static setCacheKey = (args: any[]) => args[0];

  @Cacheable({ cacheKey: 'values' })
  public getValues(): Promise<number[]> {
    return Promise.resolve(this.values);
  }

  @Cacheable({ cacheKey: TestClass.setCacheKey })
  public getValue(id: number): Promise<number | undefined> {
    return Promise.resolve(this.values.find((num) => num === id));
  }

  // If incrementValue were called with id '1', this.values would be updated and the cached values for
  // the 'values' key would be cleared and the value at the cache key '1' would be updated to the return
  // value of this method call
  @CacheUpdate({
    cacheKey: (args, context, returnValue) => args[0],
    cacheKeysToClear: (args) => ['values'],
  })
  public async incrementValue(id: number): Promise<number> {
    let newValue = 0;

    this.values = this.values.map((value) => {
      if (value === id) {
        newValue = value + 1;
        return newValue;
      }

      return value;
    });

    return newValue;
  }

  @Cacheable({
    cacheKey: 'users',
    client: userClient,
    ttlSeconds: 86400,
  })
  public async getUserById(id: string): Promise<any> {
    return this.userRepository.findOne(id);
  }

  // If getUserById('123') were called, the return value would be cached
  // in a hash under user:123, which would expire in 86400 seconds
  @Cacheable({
    cacheKey: TestClass.setCacheKey,
    hashKey: 'user',
    client: userClient,
    ttlSeconds: 86400,
  })
  public async getUserById(id: string): Promise<any> {
    return this.userRepository.findOne(id);
  }

  // If getProp('123') were called, the return value would be cached
  // under 123 in this case for 10 seconds
  @Cacheable({ cacheKey: TestClass.setCacheKey, ttlSeconds: (args) => args[1] })
  public async getProp(id: string, cacheForSeconds: number): Promise<any> {
    return this.aProp;
  }

  // If setProp('123', 'newVal') were called, the value cached under
  // key 123 would be deleted in this case.
  @CacheClear({ cacheKey: TestClass.setCacheKey })
  public async setProp(id: string, value: string): Promise<void> {
    this.aProp = value;
  }
}
```

#### `@Cacheable`

The `@Cacheable` decorator first checks for the given key(s) in cache. If a value is available (and not expired), it will be returned. If no value is available, the decorated method will run, and the cache will be set with the return value of that method. It takes `CacheOptions` for an argument. The available options are:

```ts
interface CacheOptions {
  cacheKey?: string | CacheKeyBuilder; // Individual key the result of the decorated method should be stored on
  hashKey?: string | CacheKeyBuilder; // Set name the result of the decorated method should be stored on (for hashes)
  client?: CacheClient; // If you would prefer use a different cache client than passed into the adapter, set that here
  fallbackClient?: CacheClient; // If you would prefer use a different cache client than passed into the adapter as a fallback, set that here
  noop?: boolean; // Allows for consuming libraries to conditionally disable caching. Set this to true to disable caching for some reason.
  ttlSeconds?: number | TTLBuilder; // Number of seconds the cached key should live for
  strategy?: CacheStrategy | CacheStrategyBuilder; // Strategy by which cached values and computed values are handled
}
```

#### `@CacheClear`

The `@CacheClear` decorator first runs the decorated method. If that method does not throw, `@CacheClear` will delete the given key(s) in the cache. It takes `CacheClearOptions` for an argument. The available options are:

```ts
interface CacheClearOptions {
  cacheKey?: string | string[] | CacheKeyDeleteBuilder; // Individual key the result of the decorated method should be cleared from
  hashKey?: string | CacheKeyBuilder; // Set name the result of the decorated method should be stored on (for hashes)
  client?: CacheClient; // If you would prefer use a different cache client than passed into the adapter, set that here
  fallbackClient?: CacheClient; // If you would prefer use a different cache client than passed into the adapter as a fallback, set that here
  noop?: boolean; // Allows for consuming libraries to conditionally disable caching. Set this to true to disable caching for some reason.
  isPattern?: boolean; // Will remove pattern matched keys from cache (ie: a 'foo' cacheKey will remove ['foolish', 'foo-bar'] matched keys assuming they exist)
  strategy?: CacheClearStrategy | CacheClearStrategyBuilder; // Strategy by which cached values are cleared
}
```

#### `@CacheUpdate`

The `@CacheUpdate` decorator first runs the decorated method. If that method does not throw, `@CacheUpdate` will set the given key(s) in the cache, then clear any keys listed under `cacheKeysToClear`. It takes `CacheUpdateOptions` for an argument. The available options are:

```ts
interface CacheUpdateOptions {
  cacheKey?: string | CacheKeyBuilder | PostRunKeyBuilder; // Individual key the result of the decorated method should be stored on
  hashKey?: string | CacheKeyBuilder | PostRunKeyBuilder; // Set name the result of the decorated method should be stored on (for hashes)
  cacheKeysToClear?: string | string[] | CacheKeyDeleteBuilder; // Keys to be cleared from cache after a successful method call
  client?: CacheClient; // If you would prefer use a different cache client than passed into the adapter, set that here
  fallbackClient?: CacheClient; // If you would prefer use a different cache client than passed into the adapter as a fallback, set that here
  noop?: boolean; // Allows for consuming libraries to conditionally disable caching. Set this to true to disable caching for some reason.
  isPattern?: boolean; // Will remove pattern matched keys from cache (ie: a 'foo' cacheKey will remove ['foolish', 'foo-bar'] matched keys assuming they exist)
  strategy?: CacheUpdateStrategy | CacheUpdateStrategyBuilder; // Strategy by which cached values and computed values are handled
  clearStrategy?: CacheClearStrategy | CacheClearStrategyBuilder; // Strategy by which cached values are cleared
  clearAndUpdateInParallel?: boolean; // Whether or not to clear and update at the same time (can improve performance, but could create inconsistency)
}
```

##### CacheKeyBuilder

`CacheKeyBuilder` can be passed in as the value for cacheKey or hashKey on either `@Cacheable` or `@CacheClear`. This is a function that is passed two arguments, `args` and `context`, where `args` is the arguments the decorated method was called with, and `context` is the object (`this` value) the method was called on. This function must return a string.

For example, if you would like to cache a user, you might want to cache them by id. Refer to the sample above to see how this could be done.

##### Note

If no cacheKey is passed in, one will be generated by serializing and hashing the method name, arguments, and context in which the method was called. This will not allow you to reliably clear caches, but is available as a convenience.

##### CacheStrategy

A custom implementation of `CacheStrategy` can be passed in to replace the default strategy. The default strategy will always return the cached value, or call the method and cache the result otherwise. If you want to update the cache before its time to live ends, you can implement your own `CacheStrategy` like this:

```ts
import { CacheStrategy, CacheStrategyContext } from '@type-cacheable/core';

export class MyCustomStrategy implements CacheStrategy {
  async handle(context: CacheStrategyContext): Promise<any> {
    // Implement your caching logic here
  }
}
```

If you need more details you can check the implementation of the default stratergy [here](./packages/core/lib/strategies/DefaultStrategy.ts).

### Using the CacheManager Directly

It could be the case that you need to read/write data from the cache directly, without decorators. To achieve this you can use `cacheManager`. For example:

```ts
import cacheManager from '@type-cacheable/core';
import keyGenerator from './utils/cacheKeyGenerator';

class UserService {
  private userRepository: Repository<User>;
  private rolesRepository: Repository<Role>;

  public async blockUser(id: string): Promise<void> {
    await this.userRepository.update({ id }, { isBlocked: true });
    const key = keyGenerator([id], CacheKey.UserRoles);
    await cacheManager.client?.del(key);
  }

  public async getUserDetails(id: string): Promise<UserWithRoles> {
    const key = keyGenerator([id], CacheKey.UserRoles);

    let userRoles = await cacheManager.client?.get(key);
    if (!userRoles) {
      userRoles = await this.rolesRepository.find({ userId: id });
      await cacheManager.client?.set(key, userRoles, 3600);
    }

    const user = await this.userRepository.findOne(id);

    return { ...user, userRoles };
  }
}
```

### TypeScript Configuration

```ts
{
  "target": "es2015", // at least
  "experimentalDecorators": true
}
```

## Contribution

Feel free to contribute by forking this repository, making, testing, and building your changes, then opening a pull request. Please try to maintain a uniform code style.
