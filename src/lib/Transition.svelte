<script lang="ts" context="module">
  import type { Writable } from 'svelte/store'

  // unique key for context
  const key = {}

  // interface for context
  interface Context {
    // store provides observable state to child transitions
    show: Writable<boolean>

    // number of child transitions that need to be waited before completed
    count: number

    // method to communicate child transition completion
    completed: () => Promise<void>
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
  import { getContext, setContext, createEventDispatcher } from 'svelte'
  import { writable } from 'svelte/store'

  // state of element (shown or hidden), or null if this is a child
  export let show: boolean = null

  // apply transition when element first appears
  export let appear: boolean = false

  // TODO: to handle this, we'll need to re-reference the wrapped element
  // _after_ it has been added back to the DOM (and hidden), really like
  // running the initialization again each time

  // whether the element should be unmounted (vs hidden)
  export let unmount: boolean = false

  // transition classes when entering (showing)
  export let enter: string = ''
  export let enterFrom: string = ''
  export let enterTo: string = ''

  // transition classes when leaving (hiding)
  export let leave: string = null
  export let leaveFrom: string = null
  export let leaveTo: string = null

  // if show isn't defined then this transition is a child and will receive
  // it's show state from, and coordinate it's transition completion with,
  // it's parent transition
  // let child = show === null

  // this allows us to control initial render - if we want to transition in
  // the html needs start hidden otherwise it will flash before re-appearing
  let render = show && !appear

  // convert class strings to arrays, for easier use with DOM elements
  $: enterClasses = classes(enter)
  $: enterFromClasses = classes(enterFrom)
  $: enterToClasses = classes(enterTo)

  // if leave, leaveFrom, or leaveTo are not specified then use enter values
  // as a convenient way to avoid repeating definitions (but reverse To & From)
  $: leaveClasses = classes(leave === null ? enter : leave)
  $: leaveFromClasses = classes(leaveFrom === null ? enterTo : leaveFrom)
  $: leaveToClasses = classes(leaveTo === null ? enterFrom : leaveTo)

  // get parent context if we're a child
  const parent = show === null ? getContext<Context>(key) : null

    // create our own context
  const context: Context = {
    count: 0,
    show: writable(show),
    completed: () => Promise.resolve(),
  }

  // set context for children to use
  setContext(key, context)

  const dispatch = createEventDispatcher()

  // use action that hooks into our wrapper div and manages everything
  function transition(node: HTMLDivElement, show: boolean) {
    // TODO: this check may be invalid if unmount is set
    // if (node.childElementCount !== 1 || node.firstElementChild.nodeType !== Node.ELEMENT_NODE) {
    //   console.warn('<Transition> must be applied to a single HTML element')
    // }

    // the child element that we will be applying classes to
    let el: HTMLElement = node.firstElementChild as HTMLElement

    function addClasses(val: string[]) {
      el.classList.add(...val)
    }

    function removeClasses(val: string[]) {
      el.classList.remove(...val)
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

    function childrenCompleted () {
      // return a promise that all children have completed (resolve immediately if no children)
      // sets the context completed method that children call to a promise that the parent has completed
      return context.count
        ? new Promise<void>(resolve => {
            let count = 0
            context.completed = () => new Promise<void>(resolve => {
              if (++count === context.count) {
                resolve()
              }
            }).then(resolve)
          })
        : Promise.resolve()
    }

    async function apply(base: string[], from: string[], to: string[]) {
      const completed = childrenCompleted()

      addClasses(base)
      addClasses(from)

      await nextFrame()

      const transitioned = transitionEnd(base)

      removeClasses(from)
      addClasses(to)

      await transitioned

      removeClasses(base)
      removeClasses(to)

      await completed
    }

    async function enter() {
      el.style.display = ''
      el.style.visibility = ''

      dispatch('before-enter')

      await apply(enterClasses, enterFromClasses, enterToClasses)

      if (parent) {
        await parent.completed()
      }

      dispatch('after-enter')
    }

    async function leave() {
      dispatch('before-leave')

      await apply(leaveClasses, leaveFromClasses, leaveToClasses)

      el.style.visibility = 'hidden'

      if (parent) {
        await parent.completed()
      }

      el.style.display = 'none'

      dispatch('after-leave')
    }

    let initialized = false

    async function execute(show: boolean) {
      // TODO: handle unmount - re-reference node.firstElementChild & set appropriate initial state
      if (unmount) {}

      // set initial state (fixed)
      if (!initialized) {
        render = true
        el.style.display = show
          ? appear ? 'none' : ''
          : appear ? '' : 'none'
      }

      // if previously initialized or animating in (appear)
      if (initialized || appear) {
        if (show) {
          enter()
        } else {
          leave()
        }
      }

      // set state for any child transitions
      context.show.set(show)

      // avoid re-initializing on subsequent changes
      initialized = true
    }

    // to unsubscribe when component destroyed
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
        // if (parent) {
        //   throw '<Transition> child cannot be updated directly'
        // }

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

<div class:render use:transition={show}>{#if show || !unmount}<slot />{/if}</div>

<style>
  /* default to not rendering, so content is hidden (to prevent flash of content before we can hide it) */
  div { display: none; }

  /* once rendering is active then showing and hiding is controlled on child element and we don't add our own CSS box */
  .render { display: contents; }
</style>