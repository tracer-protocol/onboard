<script lang="ts">
  import { app } from '../stores'
  export let heading: string
  export let icon: string
</script>

<style>
  /* .bn-onboard-modal-header */
  /*header {*/
  /*  display: flex;*/
  /*  align-items: center;*/
  /*  font-size: inherit;*/
  /*  font-family: inherit;*/
  /*  margin-bottom: 1em;*/
  /*  padding: 0 1rem;*/
  /*}*/

  /* .bn-onboard-modal-content-header-icon */
  /*div {*/
  /*  justify-content: center;*/
  /*  align-items: center;*/
  /*  font-size: inherit;*/
  /*  font-family: inherit;*/
  /*  padding: 0.6em;*/
  /*  border-radius: 30px;*/
  /*  background: #eeeeee;*/

  /*  display: none; !** to add back display: flex*!*/
  /*}*/

  /* .bn-onboard-modal-content-header-heading */

  .wallet-icon {
    margin: 30px auto 20px;
    width: 50px;
    height: 50px;
    background-color: #3da8f5;
  }

  .wallet-heading {
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 1.2em;
    font-weight: bold;
    padding-bottom: 10px;
    border-bottom: 1px solid #e6e6e6;
  }
</style>

<header class="bn-onboard-custom bn-onboard-modal-content-header">
<!--  <div-->
<!--    class:bn-onboard-dark-mode-background={$app.darkMode}-->
<!--    class="bn-onboard-custom bn-onboard-modal-content-header-icon"-->
<!--  >-->
<!--    {@html icon}-->
<!--  </div>-->
  <div class="wallet-icon"></div>
  <div class="wallet-heading bn-onboard-custom bn-onboard-modal-content-header-heading">
    {heading}
  </div>
</header>
