<script>
  import { getContext } from "svelte"

  export let row

  const { Provider, ContextScopes } = getContext("sdk")
</script>

<!-- Exposes the row to the child components rendered in the custom column -->
<Provider data={row} scope={ContextScopes.Local}>
  <slot />
</Provider>
