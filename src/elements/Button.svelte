<script lang="ts">
  import { app } from '../stores'
  export let onclick: () => void = () => {}
  export let position: string = ''
  export let disabled: boolean = false
  // If this is a CTA Button then we want to emphasize it
  export let cta: boolean = true
  // If this is the help button
  export let help: boolean = false
  export let primary: boolean = false
</script>

<style>
  button {
    border: none;
    background: inherit;
    font-size: 0.889em;
    font-family: inherit;
    padding: 0 1.4em;
    cursor: pointer;
    color: #3da8f5;
    opacity: 1;
    transition: background 150ms ease-in-out, opacity 200ms;
    height: 28px;
    line-height: 28px;
  }

  .primary {
    color: #3da8f5;
    border: 1px solid #3da8f5;
  }

  .primary:hover,
  .cta.primary:hover {
    background: #3da8f5;
    color: #ffffff;
  }

  button:focus {
    outline: none;
  }

  .bn-onboard-prepare-button-right {
    position: absolute;
    margin-right: 1rem;
    right: 0;
  }

  .bn-onboard-prepare-button-left {
    position: absolute;
    margin-left: 1rem;
    left: 0;
  }

  .disabled {
    cursor: inherit;
    pointer-events: none;
    opacity: 0.4;
  }
  .cta {
    border: 1px solid #3da8f5;
    border-radius: 20px;
  }

  .cta:hover {
    background: #fff;
  }

  .help {
    margin-left: 0.5rem;
  }
</style>

<button
  on:click={onclick}
  {disabled}
  class:disabled
  class:cta
  class="bn-onboard-custom bn-onboard-prepare-button"
  class:bn-onboard-prepare-button-right={position === 'right'}
  class:bn-onboard-prepare-button-left={position === 'left'}
  class:bn-onboard-prepare-button-center={position !== 'left' &&
    position !== 'right'}
  class:bn-onboard-dark-mode-link={$app.darkMode}
  class:bn-onboard-dark-mode-background-hover={$app.darkMode}
  class:help
  class:primary
>
  <slot />
</button>
