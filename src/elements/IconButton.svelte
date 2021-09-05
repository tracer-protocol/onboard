<script lang="ts">
  import Spinner from './Spinner.svelte'
  import { app } from '../stores'
  export let iconSrc: string
  export let iconSrcSet: string
  export let svg: string
  export let onclick: () => void = () => {}
  export let text: string
  export let loadingWallet: string | undefined
  export let currentlySelected: boolean = false
  export let disabled: boolean = false
</script>

<style>
  /* .bn-onboard-icon-button */
  button {
    display: flex;
    align-items: center;
    border: none;
    background: inherit;
    font-size: inherit;
    transition: box-shadow 150ms ease-in-out, background 200ms ease-in-out,
      opacity 200ms;
    cursor: pointer;
    color: inherit;
    line-height: 1.15;
    font-family: inherit;
    opacity: 1;

    width: 100%;
    border-radius: 0;
    padding: 0 1.25em;
    height: 100%;
    margin: 0;
  }

  .disabled {
    cursor: inherit;
    pointer-events: none;
    opacity: 0.4;
  }

  button:hover {
    box-shadow: 0 2px 10px 0 rgba(0, 0, 0, 0.1);
  }

  button:focus {
    outline: none;
  }

  div {
    display: flex;
    justify-content: center;
    align-items: center;
    text-align: center;
    height: 40px;
    width: 40px;
    line-height: 40px;
    font-family: inherit;
  }

  img {
    max-height: 100%;
    max-width: 100%;
    vertical-align: middle;
  }

  span {
    width: 100%;
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-size: inherit;
    margin-left: 0.66em;
    font-weight: bold;
    text-align: left;
    font-family: inherit;
  }

  i {
    font-size: 0.8rem;
    color: inherit;
    text-decoration: underline;
  }

  @media only screen and (max-width: 450px) {
    button {
      width: 100%;
    }
  }

  .bn-onboard-selected-wallet {
    background: #000240;
  }
</style>

<button
  on:click={onclick}
  {disabled}
  class:disabled
  class="bn-onboard-custom bn-onboard-icon-button"
  class:bn-onboard-dark-mode-background-hover={$app.darkMode}
  class:bn-onboard-selected-wallet={currentlySelected}
>
  <div>
    {#if loadingWallet === text}
      <Spinner icon={true} />
    {:else if svg}
      {@html svg}
    {:else}
      <img src={iconSrc} srcset={iconSrcSet} alt={text} />
    {/if}
  </div>
  <span>
    {text}
    {#if currentlySelected}
      <i>selected</i>
    {/if}
  </span>
</button>
