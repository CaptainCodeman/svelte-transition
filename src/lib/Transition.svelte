<script lang="ts" context="module">
  import type { Readable } from 'svelte/store'

  // unique key for context
  const key = {}

  // interface for context
  interface Context {
    // store provides observable state to child transitions
    show: Readable<boolean>

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

  // state of element (shown or hidden), if null this we are treated as a child
  // transition and will get the state from our parent, coordinating with it
  export let show: boolean = null

  // apply transition when element is first rendered (i.e. animate in)
  export let appear: boolean = false

  // whether the element should be removed from the DOM (vs hidden)
  export let unmount: boolean = false

  // classes to apply when entering (showing)
  export let enter: string = ''
  export let enterFrom: string = ''
  export let enterTo: string = ''

  // classes toi apply when leaving (hiding)
  export let leave: string = null
  export let leaveFrom: string = null
  export let leaveTo: string = null

  // convert class strings to arrays, for easier use with DOM elements
  $: enterClasses = classes(enter)
  $: enterFromClasses = classes(enterFrom)
  $: enterToClasses = classes(enterTo)

  // if leave, leaveFrom, or leaveTo are not specified then use enter values
  // as a convenient way to avoid repeating definitions (but reverse To & From)
  $: leaveClasses = classes(leave === null ? enter : leave)
  $: leaveFromClasses = classes(leaveFrom === null ? enterTo : leaveFrom)
  $: leaveToClasses = classes(leaveTo === null ? enterFrom : leaveTo)

  // initial state
  let initial = show && !appear
  let mounted = !unmount || show

  // get parent context if we're a child
  const parent = show === null ? getContext<Context>(key) : null

  // create our own context (which will also become parent for any children)
  // we keep the writable part (using set) and give a readable store to them
  const { subscribe, set } = writable(show)
  const context: Context = {
    count: 0,
    show: { subscribe },
    completed: () => {},
  }

  // set context for children to use
  setContext(key, context)

  const dispatch = createEventDispatcher()

  // use action that hooks into our wrapper div and manages everything
  function transition(node: HTMLDivElement, show: boolean) {
    // the child element that we will be applying classes to
    // let el: HTMLElement = node.firstElementChild as HTMLElement

    let el: HTMLElement
    function addClasses(classes: string[]) {
      el.classList.add(...classes)
    }

    function removeClasses(classes: string[]) {
      el.classList.remove(...classes)
    }

    function transitionEnd(transitions: string[]) {
      // return a promise that transitions are complete (resolve immediately if no transitions)
      return transitions.length
        ? new Promise<void>(resolve => el.addEventListener('transitionend', e => {
            e.stopPropagation()
            resolve()
          }, { once: true }))
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

      let resolveCompleted: Function
      const completed = new Promise<void>(resolve => {
        resolveCompleted = resolve
      })

      const children = childrenCompleted(completed)

      // set state for any child transitions
      set(show)

      addClasses(base)
      addClasses(from)

      const transitioned = transitionEnd(base)

      await nextFrame()

      removeClasses(from)
      addClasses(to)

      await Promise.all([
        transitioned,
        children,
      ])

      if (parent) {
        await parent.completed()
      }

      removeClasses(base)
      removeClasses(to)

      resolveCompleted()
    }

    async function ensureMountedElement() {
      if (unmount && mounted === false) {
        mounted = true
        await tick() // give slot chance to render
      }
      return node.firstElementChild as HTMLElement
    }

    async function enter() {
      dispatch('before-enter')

      node.classList.toggle('show', true)

      await apply(true, enterClasses, enterFromClasses, enterToClasses)

      dispatch('after-enter')
    }

    async function leave() {
      dispatch('before-leave')

      await apply(false, leaveClasses, leaveFromClasses, leaveToClasses)

      node.classList.toggle('show', false)

      if (unmount) {
        mounted = false
      }

      dispatch('after-leave')
    }

    // execute is always called, even for the initial render, so we use a flag
    // to prevent a transition running unless appear is set for animating in
    let run = appear

    async function execute(show: boolean) {
      // TODO: handle state changing while previous transition is still in progress
      // option is to complete immediately before playing new one, or to wait for it
      // to complete before switching

      if (run) {
        // run appropriate transition
        show ? enter() : leave()
      }

      // play transitions on all subsequent calls ...
      run = true
    }

    // to unsubscribe from parent when we're destroyed (if we're a child)
    let unsubscribe: Function

    // if we're a child transition, increment the count on the parent and listen for state notifications
    if (parent) {
      parent.count++
      // child updates happen here, as show propery is updated by the parent, which triggers the transition
      unsubscribe = parent.show.subscribe(show => execute(show))
    } else {
      // otherwise, first run through to set initial state (and possibly, 'appear' transition)
      execute(show)
    }

    return {
      update(show: boolean) {
        // top-level updates happen here, as show property is updated, which triggers the transition
        execute(show)
      },
      destroy() {
        // if we're a child and being removed, notify our parent and stop listening for updates
        if (parent) {
          unsubscribe()
          parent.count--
        }
      }
    }
  }
</script>

<div class:show={initial} use:transition={show}>{#if mounted}<slot />{/if}</div>

<style>
  /* default to *not* rendering, so content is initially hidden (to prevent flash of content before we can hide it if appear is set) */
  div { display: none; }

  /* when showing the component, we don't want to add anything to the box model, it's like we we're not even there ... */
  .show { display: contents; }
</style>
