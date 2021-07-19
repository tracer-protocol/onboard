<script lang="ts">
  import { app } from '../stores'
  export let heading: string
  export let icon: string
</script>

<style>
  /* .bn-onboard-modal-header */
  header {
    display: flex;
    align-items: center;
    font-size: inherit;
    font-family: inherit;
    margin-bottom: 1em;
    padding: 0 1rem;
  }

  /* .bn-onboard-modal-content-header-icon */
  div {
    justify-content: center;
    align-items: center;
    font-size: inherit;
    font-family: inherit;
    padding: 0.6em;
    border-radius: 30px;
    background: #eeeeee;

    display: none; /** to add back display: flex*/
  }

  /* .bn-onboard-modal-content-header-heading */

  h3 {
    font-weight: lighter;
    font-size: 1.33em;
    color: inherit;
    font-family: inherit;
    margin: 0;
  }
</style>

<header class="bn-onboard-custom bn-onboard-modal-content-header">
  <div
    class:bn-onboard-dark-mode-background={$app.darkMode}
    class="bn-onboard-custom bn-onboard-modal-content-header-icon"
  >
    {@html icon}
  </div>
  <h3 class="bn-onboard-custom bn-onboard-modal-content-header-heading">
    {heading}
  </h3>
</header>
