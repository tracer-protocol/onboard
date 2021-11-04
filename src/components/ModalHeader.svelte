<script lang="ts">
  export let heading: string
  import { app } from "../stores";
</script>

<style>
  .wallet-icon {
    margin: 20px auto;
    width: 50px;
    height: 50px;
  }

  .wallet-heading {
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 1.2em;
    padding-bottom: 20px;
    border-bottom: 1px solid #e6e6e6;
  }
  .wallet-heading-dark {
    border-bottom: 1px solid #374151;
  }
</style>

<header class="bn-onboard-custom bn-onboard-modal-content-header">
  <div class="wallet-icon">
    <svg
      width="50"
      height="50"
      viewBox="0 0 50 50"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
    >
      <rect width="50" height="50" rx="25" fill="#D1FAE5" />
      <path
        d="M31.25 13.625C31.25 13.194 31.0788 12.7807 30.774 12.476C30.4693 12.1712 30.056 12 29.625 12C29.194 12 28.7807 12.1712 28.476 12.476C28.1712 12.7807 28 13.194 28 13.625V18.5H31.25V13.625ZM33.6875 20.125H15.8125C15.597 20.125 15.3903 20.2106 15.238 20.363C15.0856 20.5153 15 20.722 15 20.9375V22.5625C15 22.778 15.0856 22.9847 15.238 23.137C15.3903 23.2894 15.597 23.375 15.8125 23.375H16.625V25C16.6253 26.8729 17.2723 28.6882 18.4567 30.1391C19.641 31.5899 21.2901 32.5873 23.125 32.9625V38H26.375V32.9625C28.2099 32.5873 29.859 31.5899 31.0433 30.1391C32.2277 28.6882 32.8747 26.8729 32.875 25V23.375H33.6875C33.903 23.375 34.1097 23.2894 34.262 23.137C34.4144 22.9847 34.5 22.778 34.5 22.5625V20.9375C34.5 20.722 34.4144 20.5153 34.262 20.363C34.1097 20.2106 33.903 20.125 33.6875 20.125ZM21.5 13.625C21.5 13.194 21.3288 12.7807 21.024 12.476C20.7193 12.1712 20.306 12 19.875 12C19.444 12 19.0307 12.1712 18.726 12.476C18.4212 12.7807 18.25 13.194 18.25 13.625V18.5H21.5V13.625Z"
        fill="#0E9A6F"
      />
    </svg>
  </div>
  <div
    class="wallet-heading bn-onboard-custom bn-onboard-modal-content-header-heading"
    class:wallet-heading-dark={$app.darkMode}
  >
    {heading}
  </div>
</header>
