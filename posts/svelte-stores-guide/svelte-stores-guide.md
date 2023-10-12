---
title: The Svelte Stores Guide
description: Svelte stores are a simple way to share reactive state across multiple components and handle global state.
slug: svelte-stores-guide
published: '2023-10-12'
category: svelte
draft: true
---

## Table of Contents

## Sharing Reactive State Across Components

Let's say you have some unrelated state like a `count` value you want to share across multiple components, or regular JavaScript modules.

```ts:lib/counter.ts showLineNumbers
export let count = 0
```

Even if you import `count` and try to change it, you can't because imports are read-only.

```html:+page.svelte showLineNumbers
<script lang="ts">
  import { count } from '$lib/counter'

  count += 1 // Cannot assign to 'count' because it is an import
</script>
```

How can you update and share the `count` value then? 🤔

It would be great if there was a way that anyone interested in `count` changing can get notified by subscribing to `counter`.

```ts:lib/counter.ts showLineNumbers
function createCounter(count) {
  // only keep track of unique subscribers
	const subscribers = new Set()

  // add subscriber
	function subscribe(subscriber) {
		subscribers.add(subscriber)
	}

  // notify the subscribers when you update the value
	function update(updater) {
    count = updater(count)
		subscribers.forEach((subscriber) => subscriber(count))
	}

	return { subscribe, update }
}

// create the counter
export const counter = createCounter(0)
```

This is also known as the [observer pattern](https://www.patterns.dev/posts/observer-pattern) which is just an object with subscribers who get notified when a value they're subscribed to updates — you might have heard of it as the **publish-subscribe**, or **pub/sub** pattern.

You can subscribe to the `counter` inside a component or regular JavaScript module.

```html:+page.svelte {2,8,10,13} showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	import Increment from './increment.svelte'
	import Decrement from './decrement.svelte'
	import Reset from './reset.svelte'

	let count = 0

	counter.subscribe(value => count = value)
</script>

<h1>The count is {count}</h1>

<Increment />
<Decrement />
<Reset />
```

You can update the `count` value from anywhere, and it's going to be reactive.

```html:increment.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	function increment() {
		counter.update(count => count + 1)
	}
</script>

<button on:click={increment}>+</button>
```

```html:decrement.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	function decrement() {
		counter.update(count => count - 1)
	}
</script>

<button on:click={decrement}>-</button>
```

```html:reset.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	function reset() {
		counter.set(0)
	}
</script>

<button on:click={reset}>
	Reset
</button>
```

You might have noticed how `createCounter` has nothing specific to a counter inside of it — let's rename it to something more generic like `writable`, add a `set` method, and return a cleanup function inside the `subscribe` method.

```ts:lib/counter.ts {1,5-8,11-13,18-21,24} showLineNumbers
export function writable(value) {
	const subscribers = new Set()

  // set the value directly
	function set(newValue) {
		value = newValue
		subscribers.forEach((subscriber) => subscriber(value))
	}

	function update(updater) {
		set(updater(value))
	}

	function subscribe(subscriber) {
		subscribers.add(subscriber)

		// return cleanup function
		return () => {
			subscribers.delete(subscriber)
		}
	}

	return { set, update, subscribe }
}

export const counter = writable(0)
```

I'm going to include the TypeScript types separately, to focus on what's important. Here is the typed version if you're interested.

<details>
  <summary>Typed version</summary>

```ts:lib/counter.ts showLineNumbers
type Subscriber<T> = (value: T) => void
type Updater<T> = (value: T) => T

export function writable<T>(value: T) {
  const subscribers = new Set<Subscriber<T>>()

  function set(newValue: T) {
    value = newValue
    subscribers.forEach((subscriber) => subscriber(value))
  }

  function update(updater: Updater<T>) {
    set(updater(value))
  }

  function subscribe(subscriber: Subscriber<T>) {
    subscribers.add(subscriber)

    return () => {
      subscribers.delete(subscriber)
    }
  }

  return { set, update, subscribe }
}

export const counter = writable(0)
```

</details>

You just implemented a Svelte store! 😄

You don't have to understand the observer pattern to use a Svelte store, but knowing JavaScript makes you resilient to change — [signals are becoming adopted by every JavaScript framework](https://dev.to/this-is-learning/the-evolution-of-signals-in-javascript-8ob), and use the observer pattern.

## Svelte Stores

So what is a Svelte store?

A Svelte store is an object with a `subscribe`, `update`, and `set` method that allows you to manage and share reactive state across multiple components. It provides a straightforward way to handle data that needs to be accessed globally, enabling components to subscribe to changes and automatically re-render when the store's value changes.

You can import a `writable`, `readable`, or `derived` store from `svelte/store`.

```ts:lib/counter.ts showLineNumbers
import { writable } from 'svelte/store'

const counter = writable(0)
```

A store accepts a function as a second argument that only runs once after the first subscriber, and return a cleanup function when there are no subscriber anymore.

```ts:lib/counter.ts showLineNumbers
import { writable } from 'svelte/store'

const counter = writable(0, () => {
	console.log('Got a subscriber')

	return () => console.log('No more subscribers')
})

const unsubscribe = counter.subscribe(count => console.log(count)) // "Got a subscriber"

unsubscribe() // "No more subscribers"
```

The second argument also accepts a `set` and `update` method as arguments.

```ts:lib/counter.ts showLineNumbers
import { writable } from 'svelte/store'

const counter = writable(0, (set, update) => {
	set(10) // logs 10
	update(prevCount => prevCount * 2) // logs 20
})

counter.subscribe(count => console.log(count))
```

You can read and write to a `writable` store, but you might want a read-only store, in which case you can use a `readable` store.

```ts:lib/counter.ts showLineNumbers
import { readable } from 'svelte/store'

const counter = readable(0)

// you can only subscribe to a readable store
counter.subscribe(count => console.log(count))
```

A `derived` store is useful if you need to create a store based on the value from other stores.

```ts:lib/counter.ts showLineNumbers
import { writable, derived } from 'svelte/store'

const counter = writable(0)
const doubled = derived(counter, (count) => count * 2)

doubled.subscribe(count => console.log(count)) // logs 0

counter.set(10) // logs 20
```

> 🐿️ You might see values in the second argument for `derived` stores use a `$` prefix like `$count => $count * 2` but it has no special meaning like auto-subscriptions.

You can derive a value from multiple stores.

```ts:lib/counter.ts showLineNumbers
import { writable, derived } from 'svelte/store'

const counter = writable(0)
const doubled = derived([counter, counter], ([a, b]) => a + b)

doubled.subscribe(count => console.log(count)) // logs 0

counter.set(10) // logs 10
```

If you need to retrieve the value from a store without subscribing to it, you can use the `get` method from Svelte which subscribes, reads the value, and unsubscribes from the store.

```ts:lib/counter.ts showLineNumbers
import { get, writable } from 'svelte/store'

const counter = writable(10)

const count = get(counter)
console.log(count) // 10
```

## Store Auto-Subscriptions

Stores are awesome, but having to subscribe and do cleanup for every store is tedious. 🥱

```html:+page.svelte {2,7,9} showLineNumbers
<script lang="ts">
	import { onDestroy } from 'svelte'
	import { counter } from '$lib/counter'

	let count = 0

	const unsubscribe = counter.subscribe((value) => (count = value))

	onDestroy(unsubscribe)
</script>
```

Inside Svelte components, you can reference a store using the `$` prefix which subscribes and unsubscribes to the store for you.

```html:+page.svelte {9} showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	import Increment from './increment.svelte'
	import Decrement from './decrement.svelte'
	import Reset from './reset.svelte'
</script>

<h1>The count is {$count}</h1>

<Increment />
<Decrement />
<Reset />
```

```html:increment.svelte {5} showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	function increment() {
		$counter += 1
	}
</script>

<button on:click={increment}>+</button>
```

```html:decrement.svelte {5} showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	function decrement() {
		$counter -= 1
	}
</script>

<button on:click={decrement}>-</button>
```

```html:reset.svelte {5} showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'

	function reset() {
		$counter = 0
	}
</script>

<button on:click={reset}>
	Reset
</button>
```

You can also bind the store value if it's writable.

```html:+page.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'
</script>

<input bind:value={$counter}>

<button on:click={() => $counter += 1}>
	Add
</button>
```

This is possible because Svelte is a compiler, and it desugars `$` into the code you wrote before, reducing the amount of boilerplate.

Using the `$` syntax only works inside `.svelte` components, because Svelte doesn't change the behavior of JavaScript outside componens.

## Custom Svelte Stores

Custom Svelte stores allow you to encapsulate related logic within a store, and expose a clear and specific API.

To create a custom store, you only have to return the `subscribe` method. You can use the `set` and `update` method to update the store value.

```ts:lib/counter.ts showLineNumbers
import { writable } from 'svelte/store'

function createCounter(count: number) {
	const { subscribe, set, update } = writable(count)

	function increment() {
		update(count => count + 1)
	}

	function decrement() {
		update(count => count - 1)
	}

	function reset() {
		set(0)
	}

	return { subscribe, increment, decrement, reset }
}

export const counter = createCounter(0)
```

Let's update the previous `counter` example.

```html:increment.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'
</script>

<button on:click={counter.increment}>+</button>
```

```html:decrement.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'
</script>

<button on:click={counter.decrement}>-</button>
```

```html:reset.svelte showLineNumbers
<script lang="ts">
	import { counter } from '$lib/counter'
</script>

<button on:click={counter.rest}>
	Reset
</button>
```

That's it! 😄

## Using Stores On The Server

Don't use stores on the server in SvelteKit.

Stores are designed to manage state on the client. On the server, each request is handled independently. This means that if you use a Svelte store on the server, the state you set could be shared across multiple requests.

At best you're going to run into weird behavior, and at the worst you might have data leakage, where one user's data is exposed to another user.

```ts:lib/user.ts showLineNumbers
import { writable } from 'svelte/store'

export const user = writable('')
```

```ts:+page.server.ts showLineNumbers
import { user } from '$lib/user'

export async function load({ fetch }) {
	const response = await fetch('/api/user')
	const userData = await response.json()

	// 💩 avoid using a store on the server
	user.set(userData)
}
```

```html:+page.svelte showLineNumbers
<script lang="ts">
	import { user } from '$lib/user'

	// 💩 this doesn't get updated
	console.log('$user: ' + $user)
</script>

<h1>Server</h1>
```

Instead of using stores on the server, pass the data to the component that needs it, or use the `$page` store.

```ts:+page.server.ts showLineNumbers
export async function load({ fetch }) {
	const response = await fetch('/api/user')
	const userData = await response.json()

	return {
		// 👍️ pass the data to the component
		user: userData,
	}
}
```

```html:+page.svelte showLineNumbers
<script lang="ts">
	import { page } from '$app/stores'

	export let data

	// 👍️ pass the data to the component
	console.log('data: ' + data.user.name)

	// 👍️ or use the `$page.data` store
	console.log('$page.data: ' + $page.data.user.name)
</script>

<h1>Server</h1>
```

Importing `$page.data` is useful if you have a component on the page that needs the data. Instead of passing the data as a prop like `<Component user={$page.data.user}`, you can get the value returned from the `load` function inside the component from `$page.data`.

## Usage With Other Libraries

Stores have great interoperability with most libraries that use observables like [XState](https://xstate.js.org/), or [RxJS](https://rxjs.dev/).

```html:example.html showLineNumbers
<script lang="ts">
	import { createMachine, interpret } from 'xstate'

	// machine
	const toggleMachine = createMachine({
    id: 'toggle',
    initial: 'inactive',
    states: {
      inactive: {
        on: { TOGGLE: 'active' }
      },
      active: {
        on: { TOGGLE: 'inactive' }
      }
    }
  })

	// custom store
	function useMachine(machine, options) {
		const service = interpret(machine).start()

		const state = readable(service.state, (set) => {
			service.subscribe(state => set(state))
			return () => service.stop()
		})

		return { state, send: service.send, service }
	}

	// usage
	const { state, send } = useMachine(toggleMachine)
</script>

<button on:click={() => send('TOGGLE')}>
  {$state.value === 'inactive' ? 'Activate' : 'Deactivate'}
</button>
```

Please use the [offical `@xstate/svelte` package](https://stately.ai/docs/xstate-svelte).

## Signals Are The Future

[Svelte recently introduced a preview for runes](https://svelte.dev/blog/runes) which are signals that unlock universal, fine-grained reactivity.

I wanted to explain how stores work, so you understand in the future that signals are almost the same, because they're both observables.

Here is how the previous `useCounter` custom store looks using signals.

```html:runes.svelte showLineNumbers
<script>
	function createCounter(initialCount) {
		let count = $state(initialCount)

		function increment() {
			count += 1
		}

		function decrement() {
			count -= 1
		}

		function reset() {
			count = 0
		}

		return {
			get count() { return count },
			increment,
			decrement,
			reset
		}
	}

	const counter = createCounter(0)
</script>

<h1>Count is {counter.count}</h1>

<button on:click={counter.increment}>+</button>
<button on:click={counter.decrement}>-</button>
<button on:click={counter.reset}>Reset</button>
```

You no longer have to remember where you can use `$` for stores and other rules, because signals work in Svelte components and regular JavaScript modules.

The greatest tragedy of only knowing JavaScript frameworks instead of JavaScript is [mistaking runes for other frameworks instead of understanding that you can do anything](https://x.com/joyofcodedev/status/1711718344473129319).

I hope this inspires you to ask question how JavaScript framework work, and you realize it's not magic — you're going to be more resilient to changes, and gain a deeper understanding and passion for your craft.
