<script>
  import { getContext, setContext } from "svelte"

  export let labelPosition = "above"

  const { styleable } = getContext("sdk")
  const component = getContext("component")
  setContext("fieldGroup", { labelPosition })
</script>

<div class="wrapper" use:styleable={$component.styles}>
  <div
    class="spectrum-Form"
    class:spectrum-Form--labelsAbove={labelPosition === "above"}
  >
    <slot />
  </div>
</div>

<style>
  .wrapper {
    width: 100%;
    position: relative;
  }
  .spectrum-Form {
    width: 100%;
  }
  .spectrum-Form--labelsAbove {
    gap: var(--spectrum-global-dimension-size-100);
  }
</style>
