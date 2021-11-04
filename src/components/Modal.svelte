<script lang="ts">
  import { fade } from 'svelte/transition'
  import { app } from '../stores'
  import Branding from '../elements/Branding.svelte'
  export let closeModal: () => void = () => {}
  export let closeable: boolean = true
</script>

<style>
  /* === TARGET BY ELEMENT TO ALLOW CUSTOM OVERRIDES TO HAVE ADEQUATE SPECIFICITY ===*/
  /* .bn-onboard-modal */
  aside {
    display: flex;
    font-family: 'Helvetica Neue', 'Helvetica', 'Arial', sans-serif;
    justify-content: center;
    align-items: center;
    position: fixed;
    font-size: 16px;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: rgba(0, 0, 0, 0.3);
  }

  /* .bn-onboard-modal-content  */
  section {
    display: block;
    background-color: #ffffff;
    box-sizing: content-box;
    border-radius: 10px;
    box-shadow: 0 1px 5px 0 rgba(0, 0, 0, 0.1);
    font-family: inherit;
    font-size: inherit;
    position: relative;
    overflow: hidden;
    width: 422px;
    padding: 20px;
  }

  @media screen and (max-width: 680px) {
    aside {
      font-size: 14px;
    }

    section {
      width: 300px;
    }
  }

  svg {
    position: absolute;
    top: 20px;
    right: 20px;
    width: 20px;
    height: 20px;
  }
  svg:hover {
    cursor: pointer;
  }

  .no-padding-branding {
    padding-bottom: 0;
  }
</style>

<aside
  transition:fade
  class="bn-onboard-custom bn-onboard-modal"
  on:click={closeModal}
>
  <section
    class:bn-onboard-dark-mode={$app.darkMode}
    class:no-padding-branding={$app.displayBranding}
    class="bn-onboard-custom bn-onboard-modal-content"
    on:click={e => e.stopPropagation()}
  >
    <slot />
    {#if $app.displayBranding}
      <Branding darkMode={$app.darkMode} />
    {/if}
    {#if closeable}
      <div
        class="bn-onboard-custom bn-onboard-modal-content-close"
        class:bn-onboard-dark-mode-close-background={$app.darkMode}
        on:click={closeModal}
      >
        <svg
          aria-hidden="true"
          focusable="false"
          data-prefix="fas"
          data-icon="xmark"
          class="svg-inline--fa fa-xmark fa-w-10"
          role="img"
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 320 512"
          ><path
            fill="currentColor"
            d="M310.6 361.4c12.5 12.5 12.5 32.75 0 45.25C304.4 412.9 296.2 416 288 416s-16.38-3.125-22.62-9.375L160 301.3L54.63 406.6C48.38 412.9 40.19 416 32 416S15.63 412.9 9.375 406.6c-12.5-12.5-12.5-32.75 0-45.25l105.4-105.4L9.375 150.6c-12.5-12.5-12.5-32.75 0-45.25s32.75-12.5 45.25 0L160 210.8l105.4-105.4c12.5-12.5 32.75-12.5 45.25 0s12.5 32.75 0 45.25l-105.4 105.4L310.6 361.4z"
          /></svg
        >
      </div>
    {/if}
  </section>
</aside>
