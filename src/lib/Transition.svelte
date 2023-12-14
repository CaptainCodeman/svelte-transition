<script lang="ts" context="module">
	import type { Readable } from 'svelte/store'

	// unique key for context
	const key = {}

	// interface for context
	interface Context {
		// appear setting
		appear: boolean

		// store provides observable state to child transitions
		show: Readable<boolean | null>

		// number of child transitions that need to be waited before completed
		count: number

		// method to communicate child transition completion
		completed: () => void
	}

	// convert a string of class names into an array, for use with DOM methods
	function classes(classes: string) {
		return classes ? classes.split(' ').filter(x => x) : []
	}

	// to wait until css changes have been appplied we use a double rAF
	function nextFrame() {
		const raf = requestAnimationFrame // help minification
		return new Promise(resolve => raf(() => raf(resolve)))
	}
</script>

<script lang="ts">
	import { getContext, setContext, createEventDispatcher, tick } from 'svelte'
	import { writable } from 'svelte/store'

	interface Props {
		// state of element (shown or hidden), if null this we are treated as a child
		// transition and will get the state from our parent, coordinating with it
		show?: boolean | null

		// apply transition when element is first rendered (i.e. animate in)
		appear?: boolean

		// whether the element should be removed from the DOM (vs hidden)
		unmount?: boolean

		// classes to apply when entering (showing)
		enter?: string
		enterFrom?: string
		enterTo?: string

		// classes to apply when leaving (hiding)
		leave?: string | null
		leaveFrom?: string | null
		leaveTo?: string | null

		children: any
	}

	let {
		show = null,
		appear = false,
		unmount = false,
		enter = '',
		enterFrom = '',
		enterTo = '',
		leave = null,
		leaveFrom = null,
		leaveTo = null,
		children,
	} = $props<Props>()

	// convert class strings to arrays, for easier use with DOM elements
	let enterClasses = $derived(classes(enter))
	let enterFromClasses = $derived(classes(enterFrom))
	let enterToClasses = $derived(classes(enterTo))

	// if leave, leaveFrom, or leaveTo are not specified then use enter values
	// as a convenient way to avoid repeating definitions (but reverse To & From)
	let leaveClasses = $derived(classes(leave === null ? enter : leave))
	let leaveFromClasses = $derived(classes(leaveFrom === null ? enterTo : leaveFrom))
	let leaveToClasses = $derived(classes(leaveTo === null ? enterFrom : leaveTo))

	// get parent context if we're a child
	const parent = show === null ? getContext<Context>(key) : null

	// create our own context (which will also become parent for any children)
	// we keep the writable part (using set) and give a readable store to them
	const { subscribe, set } = writable(show)

	const context: Context = {
		appear: parent ? parent.appear : appear,
		count: 0,
		show: { subscribe },
		completed: () => {},
	}

	// initial state
	let display = $state(show && !context.appear ? 'contents' : 'none')
	let mounted = $state(!unmount || show === true)

	$inspect({
		show, appear, unmount, enter, enterFrom, enterTo, leave, leaveFrom, leaveTo, children,
		enterClasses, enterFromClasses, enterToClasses,
		leaveClasses, leaveFromClasses, leaveToClasses,
		parent, context,
		display,
		mounted,
	})

	// set context for children to use
	setContext(key, context)

	const dispatch = createEventDispatcher()

	// use action that hooks into our wrapper div and manages everything
	function transition(node: HTMLElement, show: boolean | null) {
		// the child element that we will be applying classes to
		// let el: HTMLElement = node.firstElementChild as HTMLElement

		let el: HTMLElement
		function addClasses(...classes: string[]) {
			el.classList.add(...classes)
		}

		function removeClasses(...classes: string[]) {
			el.classList.remove(...classes)
		}

		function transitionEnd(transitions: string[]) {
			// return a promise that transitions are complete (resolve immediately if no transitions)
			return transitions.length
				? new Promise<void>(resolve =>
						el.addEventListener('transitionend', e => {
							e.stopPropagation()
							resolve()
						}, { once: true })
				  )
				: Promise.resolve()
		}

		function childrenCompleted(parentCompleted: Promise<void>) {
			// return a promise that all children have completed (resolve immediately if no children)
			// sets the context completed method that children call to a promise that the parent has completed
			return context.count
				? new Promise<void>(resolve => {
						let count = 0
						context.completed = () => {
							if (++count === context.count) {
								resolve()
							}
							return parentCompleted
						}
				  })
				: Promise.resolve()
		}

		async function apply(show: boolean, base: string[], from: string[], to: string[]) {
			el = await ensureMountedElement()

			let resolveCompleted = () => {}
			const completed = new Promise<void>(resolve => {
				resolveCompleted = resolve
			})

			const children = childrenCompleted(completed)

			// set state for any child transitions
			set(show)

			addClasses(...base, ...from)

			const transitioned = transitionEnd(base)

			await nextFrame()

			removeClasses(...from)
			addClasses(...to)

			await Promise.all([transitioned, children])

			if (parent) {
				await parent.completed()
			}

			removeClasses(...base, ...to)

			resolveCompleted()
		}

		async function ensureMountedElement() {
			if (unmount && !mounted) {
				mounted = true
				await tick() // give slot chance to render
			}
			return node.firstElementChild as HTMLElement
		}

		async function enter() {
			dispatch('before-enter')

			display = 'contents'

			await apply(true, enterClasses, enterFromClasses, enterToClasses)

			dispatch('after-enter')
		}

		async function leave() {
			dispatch('before-leave')

			await apply(false, leaveClasses, leaveFromClasses, leaveToClasses)

			display = 'none'

			if (unmount) {
				mounted = false
			}

			dispatch('after-leave')
		}

		// execute is always called, even for the initial render, so we use a flag
		// to prevent a transition running unless appear is set for animating in
		let run = context.appear

		function execute(show: boolean | null) {
			// run appropriate transition, set promise for completion
			executing = run
				? show
					? enter()
					: leave()
				: Promise.resolve()

			// play transitions on all subsequent calls ...
			run = true
		}

		// to unsubscribe from parent when we're destroyed (if we're a child)
		let unsubscribe: Function

		// to wait for in-progress transitions to complete
		let executing: Promise<void>

		// if we're a child transition, increment the count on the parent and listen for state notifications
		if (parent) {
			parent.count++
			// child updates happen here, as show propery is updated by the parent, which triggers the transition
			unsubscribe = parent.show.subscribe(execute)
		} else {
			// otherwise, first run through to set initial state (and possibly, 'appear' transition)
			execute(show)
		}

		return {
			update(show: boolean | null) {
				// top-level updates happen here, as show property is updated, which triggers the transition
				// wait for current transition to complete so state is consistent (may be state waiting on our events)
				executing.then(() => execute(show))
			},
			destroy() {
				// if we're a child and being removed, notify our parent and stop listening for updates
				if (parent) {
					unsubscribe()
					parent.count--
				}
			},
		}
	}
</script>

<div style:display use:transition={show}>{#if mounted}{@render children()}{/if}</div>