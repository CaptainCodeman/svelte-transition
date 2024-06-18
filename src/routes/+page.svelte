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
	$: result = groups ? removeEmptyLines(`<Transition {show}
  enter=${groups.enter}
  enterFrom=${groups.enterFrom}
  enterTo=${groups.enterTo}
  ${groups.leave !== groups.enter ? `leave=${groups.leave}` : ''}
  ${groups.leaveFrom !== groups.enterTo ? `leaveFrom=${groups.leaveFrom}` : ''}
  ${groups.leaveTo !== groups.enterFrom ? `leaveTo=${groups.leaveTo}` : ''}
>
</Transition>`) : ''
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
	{/if}
</main>