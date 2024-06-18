<script lang="ts">
  let source = /* html */`  <!--
    Modal panel, show/hide based on modal state.

    Entering: "ease-out duration-300"
      From: "opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"
      To: "opacity-100 translate-y-0 sm:scale-100"
    Leaving: "ease-in duration-200"
      From: "opacity-100 translate-y-0 sm:scale-100"
      To: "opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"
  -->`;
  const regex = /\<!--\s+(?<description>.*)\s+Entering:\s?(?<enter>.*)\s+From:\s?(?<enterFrom>.*)\s+To:\s?(?<enterTo>.*)\s+Leaving:\s?(?<leave>.*)\s+From:\s?(?<leaveFrom>.*)\s+To:\s?(?<leaveTo>.*)\s+--\>/mi;

	function removeEmptyLines(value: string) {
		return value.split('\n').filter(line => line !== '  ').join('\n')
	}

  $: groups = source.match(regex)?.groups
	$: result = groups ? removeEmptyLines(`<!-- ${groups.description} -->
<Transition
  enter=${groups.enter}
  enterFrom=${groups.enterFrom}
  enterTo=${groups.enterTo}
  ${groups.leave !== groups.enter ? `leave=${groups.leave}` : ''}
  ${groups.leaveFrom !== groups.enterTo ? `leaveFrom=${groups.leaveFrom}` : ''}
  ${groups.leaveTo !== groups.enterFrom ? `leaveTo=${groups.leaveTo}` : ''}
>
</Transition>`) : ''

	async function copy() {
		await navigator.clipboard.writeText(result)
	}
</script>

<main class="m-4">
	<h1 class="font-bold text-3xl text-gray-700 tracking-tighter">Svelte-Transition</h1>
	<p class="mt-2 text-gray-700">A Svelte component to transition elements via CSS classes</p>
	<p class="mt-2 text-gray-700">
		Examples:
		<a class="ml-1 text-blue-600 hover:underline" href="examples/basic">Basic</a>
		<a class="ml-1 text-blue-600 hover:underline" href="examples/nested">Nested</a>
	</p>
	<h2 class="mt-8 font-bold text-xl text-gray-700 tracking-tighter">Convert TailwindUI Comments</h2>

	<div class="mt-4 max-w-2xl">
		<label for="source" class="block text-sm font-medium leading-6 text-gray-900">Source comment:</label>
		<div class="mt-2">
			<textarea rows="10" name="source" id="source" class="block w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 font-mono text-sm leading-5" bind:value={source}></textarea>
		</div>
	</div>

	{#if groups}
	<div class="mt-4 max-w-2xl">
		<label for="source" class="block text-sm font-medium leading-6 text-gray-900">Svelte Transition:</label>
		<div class="mt-2">
			<pre class="rounded-md border px-4 py-1.5 text-gray-900 font-mono text-sm leading-5 bg-gray-50">{result}</pre>
		</div>
	</div>

	<button type="button" class="mt-2 flex items-center gap-1 rounded-md bg-white px-2.5 py-1.5 text-sm font-semibold text-gray-700 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50" on:click={copy}>
		<svg fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="text-gray-500 size-5">
			<path stroke-linecap="round" stroke-linejoin="round" d="M8.25 7.5V6.108c0-1.135.845-2.098 1.976-2.192.373-.03.748-.057 1.123-.08M15.75 18H18a2.25 2.25 0 0 0 2.25-2.25V6.108c0-1.135-.845-2.098-1.976-2.192a48.424 48.424 0 0 0-1.123-.08M15.75 18.75v-1.875a3.375 3.375 0 0 0-3.375-3.375h-1.5a1.125 1.125 0 0 1-1.125-1.125v-1.5A3.375 3.375 0 0 0 6.375 7.5H5.25m11.9-3.664A2.251 2.251 0 0 0 15 2.25h-1.5a2.251 2.251 0 0 0-2.15 1.586m5.8 0c.065.21.1.433.1.664v.75h-6V4.5c0-.231.035-.454.1-.664M6.75 7.5H4.875c-.621 0-1.125.504-1.125 1.125v12c0 .621.504 1.125 1.125 1.125h9.75c.621 0 1.125-.504 1.125-1.125V16.5a9 9 0 0 0-9-9Z" />
		</svg>
		Copy
	</button>

	{/if}
</main>
