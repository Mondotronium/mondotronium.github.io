# State hierarchy

## Application level state

The most basic form of application level state is the active screen. Nowadays, your router abstracts this for you, but it is still there. The information (which screen is being displayed) is stored in the url, so it can be bookmarked and shared.

Most apps also have an authenticated user as well, and that is very likely needed across all screens, so that is also application level state. You can store it a Vuex store, or in a reactive variable that you import when needed.


## Screen level state

This is where it gets a bit tricky. Some page level state, like 'add user popup is open' is perfectly fine to be stored in a reactive variable. But there are a lot of cases where the page level state should be pushed to the query string. Good examples are pagination, sorting, or any kind of filtering. That way, back and forward buttons, bookmarking and sharing will work as expected.


## Component level state

The internal values of form fields and dropdown states are good examples to state that you don't expect to be preserved when arriving on a shared url. These can be states of the individual components and do not need to be stored in the query string.


# How to store state in the query string?

The most basic implementation would be to just use the Vue Router:

```ts
// store some state
router.push({ query: { foo: 'bar' } });

// read some state
console.log(router.currentRoute.query.foo); // 'bar'
```

It is very easy to push a string onto the query string, since it is made out of strings, but what happens if you want to push a number onto it?

```ts
// store some number
router.push({ query: { foo: 123 } });

// read it back
console.log(router.currentRoute.query.foo); // '123'
```

It comes back as a string! If you happen to know that `foo` can only be number, you need to manually decode it when reading from the query string like so:

```ts
// read it back
const decodeNumber = (value: string): number => parseInt(value, 10);
console.log(decodeNumber(router.currentRoute.query.foo)); // 123
```

With a number, you can rely on the Vue Router to turn it into a string when pushing it to the query string, but what about pushing something more complex, like an array of numbers? To make it work, you need to encode and decode the values yourself:

```ts
// store an array of numbers
const encodeNumber = (value: number): string => number.toString();
const encodeStringArray = (value: string[]): string => value.join(',');
router.push({ query: { foo: encodeStringArray([10,20,30].map(encodeNumber)) } });

// read it back
const decodeNumber = (value: string): number => parseInt(value, 10);
const decodeStringArray = (value: string): string[] => value.split(',');
console.log(decodeStringArray(router.currentRoute.query.foo).map(decodeNumber)); // [10, 20, 30];
```

The eagle eyed among you could have already spotted that the type for a query string value on retrieval is not `string` but `string | (string | null)[]`. It can be for example `?foo=bar&foo=baz&foo` in which case it returns `['bar', 'baz', null]`. This already shows us how important it is to handle invalid incoming values.

## Error handling

As you can see, you can store arbitrary complex structures in a query string variable as long as it can be serialized into a string and back. The bigger question is: what should happen when decoding fails? What if you end up on a page with the query string `?foo=10,20,bar`? 

The two options are:

- silently fail by using a fallback value or just ignoring invalid values
- display a big error message letting the users know that the state they are trying to access is not something your application can handle

Personally I think the latter option is preferable, as this is no different than any API braking contract. The incoming data is not in the expected format and your application might not be able to handle it.

## Making it more convenient

Of course, interacting with the router for state change might be too tedious. Wouldn't it be better if you could just somehow have a variable to store state and not have to worry about how it gets stored in the query string? You can do exactly that with a computed variable:

```ts
import { computed } from '@vue/composition-api';
import { encode, decode } from './codec' // some encoder

const foo = computed<T>({
    get: (): T => decode(router.currentRoute.query.foo),
    set: (value: T) => {
        router.push({ query: {
            ...router.currentRoute.query,
            foo: encode(value),
        } });
    },
  });
```

This way you can interact with your variable just like if it was any other kind of state while all changes would be reflected in the query string.

## Avoiding too many history entries

Some page level state can change rapidly, like search field values that change on every keyup. Pushing a new history entry on every keypress will break your back and forward buttons in a different way: they will work, but there will be too many history entries to traverse.

The are two solutions:

- Make the internal field value a component level state, and only update the page level filter value when the user is done typing, by for example displaying a submit button
- Debounce all changes to the state stored in the query string

While the former is more reliable, the latter can result in a better user experience in some cases.