# convex-svelte

Receive live updates to Convex query subscriptions and call mutations and actions from Svelte with `convex-svelte`.

To install:

```
bun install convex convex-svelte
```

Run `bunx convex init` to get started with Convex.

See the [example app live](https://convex-svelte.vercel.app/).

`convex-svelte` provides a `setupConvex()` function which takes a Convex deployment URL,
a `useConvexClient()` which returns a [ConvexClient](https://docs.convex.dev/api/classes/browser.ConvexClient)
used to set authentication credentials and run Convex mutations and actions,
and a `useQuery()` function for subscribing to Convex queries.

### Example

Call `setupConvex()` in a component above the components that need to Convex queries
and use `useQuery()` components where you need to listen to the query.

See [+layout.svelte](src/routes/+layout.svelte) for `setupConvex()`

```svelte
<script>
	import { PUBLIC_CONVEX_URL } from '$env/static/public';

	setupConvex(PUBLIC_CONVEX_URL);
</script>
```

and [Chat.svelte](src/routes/Chat.svelte) for how to use `useQuery()`

```svelte
<script>
	const query = useQuery(api.messages.list, () => ({ muteWords }), {
		useResultFromPreviousArguments: true
	});
</script>

...
{#if query.isLoading}
	Loading...
{:else if query.error != null}
	failed to load: {query.error.toString()}
{:else}
	<ul>
		{#each query.data as message}
			<li>
				<span>{message.author}</span>
				<span>{message.body}</span>
			</li>
		{/each}
	</ul>
{/if}
```

Running a mutation looks like

```svelte
<script lang="ts">
	import { api } from '../../convex/_generated/api.js'; // depending on file location
	import { useConvexClient } from 'convex-svelte';
	const client = useConvexClient();

	let toSend = $state('');
	let author = $state('me');

	function handleSubmit(event: SubmitEvent) {
		event.preventDefault();

		const data = Object.fromEntries(new FormData(event.target as HTMLFormElement).entries());
		client.mutation(api.messages.send, {
			author: data.author as string,
			body: data.body as string
		});
	}
</script>

<form onsubmit={handleSubmit}>
	<input type="text" name="author" bind:value={author} />
	<input type="text" name="body" bind:value={toSend} />
	<button type="submit" disabled={!toSend}>Send</button>
</form>
```

### Conditionally skipping queries

You can conditionally skip a query by returning the string `'skip'` from the arguments function.
This is useful when a query depends on some condition, like authentication state or user input.

```svelte
<script lang="ts">
import { useQuery } from "convex-svelte";
import { api } from "../convex/_generated/api.js";

let auth = $state({ isAuthenticated: true });

const activeUserResponse = useQuery(
  api.users.queries.getActiveUser,
  () => (auth.isAuthenticated ? {} : 'skip')
);
</script>

{#if activeUserResponse.isLoading}
  Loading user...
{:else if activeUserResponse.error}
  Error: {activeUserResponse.error}
{:else if activeUserResponse.data}
  Welcome, {activeUserResponse.data.name}!
{/if}
```

When a query is skipped, `isLoading` will be `false`, `error` will be `null`, and `data` will be `undefined`.

### Server-side rendering

`useQuery()` accepts an `initialData` option in its third argument.
By defining a `load()` function in a +page.server.ts file
that uses the `ConvexHttpClient` to request the same query to get initial data
and passing that through to the `initialData` option of a useQuery call you can avoid an initial loading state.

```ts
// +page.server.ts
import { ConvexHttpClient } from 'convex/browser';
import type { PageServerLoad } from './$types.js';
import { PUBLIC_CONVEX_URL } from '$env/static/public';
import { api } from '../convex/_generated/api.js';

export const load = (async () => {
	const client = new ConvexHttpClient(PUBLIC_CONVEX_URL!);
	return {
		messages: await client.query(api.messages.list, { muteWords: [] })
	};
}) satisfies PageServerLoad;
```

```svelte
<script lang="ts">
	// +page.svelte
	import type { PageData } from './$types.js';
	let { data }: { data: PageData } = $props();

	import { useQuery, useConvexClient } from '$lib/client.svelte.js';
	import { api } from '../convex/_generated/api.js';

	const messages = useQuery(
		api.messages.list,
		() => args,
		() => ({ initialData: data.messages })
	);
</script>
```

Combining specifying `initialData` and either setting the `keepPreviousData` option to true or never modifying the arguments passed to a query should be enough to avoid ever seeing a loading state for a `useQuery()`.

### Troubleshooting

#### effect_in_teardown Error

If you encounter `effect_in_teardown` errors when using `useQuery` in components that can be conditionally rendered (like dialogs, modals, or popups), this is caused by wrapping `useQuery` in a `$derived` block that depends on reactive state.

When `useQuery` is wrapped in `$derived`, state changes during component cleanup can trigger re-evaluation of the `$derived`, which attempts to create a new `useQuery` instance. Since `useQuery` internally creates a `$effect`, and effects cannot be created during cleanup, this throws an error.

Use [Conditionally skipping queries](#conditionally-skipping-queries) instead. By calling `useQuery` unconditionally at the top level and passing a function that returns `'skip'`, the function is evaluated inside `useQuery`'s own effect tracking, preventing query recreation during cleanup.