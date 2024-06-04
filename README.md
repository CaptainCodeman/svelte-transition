# svelte-transition

Svelte component to make using CSS class based transitions easier - ideally suited for use with [TailwindCSS](https://tailwindcss.com/)

Loosely modeled on the [HeadlessUI Transition](https://headlessui.dev/react/transition).

About 3Kb / 1.5Kb gzipped

## Installation

Install using your package manager of choice:

```bash
pnpm i svelte-transition
```

## Usage

Import into your component and use a flag to control whether to show or hide an HTML Element.

```ts
import Transition from 'svelte-transition'

let show = false
```

Wrap the HTML Element with the `<Transition>` component, setting the flag to toggle visibility and the classes to apply when transitioning:

```html
<Transition
	{show}
	enter="ease-in-out duration-300"
	enterFrom="opacity-0"
	enterTo="opacity-100"
	leave="ease-in-out duration-300"
	leaveFrom="opacity-100"
	leaveTo="opacity-0"
></Transition>
```

## Shortcut

If the leave transition is the opposite of the enter transition it can be omitted to save bytes. This is identical to the previous example:

```html
<Transition {show} enter="ease-in-out duration-300" enterFrom="opacity-0" enterTo="opacity-100"></Transition>
```

i.e. `leave` will equal `enter`, `leaveFrom` will equal `enterTo`, and `leaveTo` will equal `enterFrom` unless you override them.

## Appear

Set `appear` to have the transition play on initial mount.

Default `false`.

## Unmount

Set `unmount` to have the transitioned element removed from the DOM when not shown (instead of just hidden).

Default `false`.

## Co-ordinating Transitions

If the `show` property is ommitted then the transition is treated as a child and will receive it's state from it's parent. The parent will automatically wait for it's children to finish transitioning before they are unmounted or hidden, so the animations can complete.

## Events

The component raises events to indicate when any transition is running:

- `before-enter` runs before the enter transition happens
- `after-enter` runs after the enter transition happens
- `before-leave` runs before the leave transition happens
- `after-leave` runs after the leave transition happens

## TailwindUI

If you're converting from TailwindUI markup, you can use this [handy converter](https://kuba1meow.github.io/svelte-transition-converter/) to convert the comments into `<Transition>` markup and classes.
