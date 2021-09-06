<script lang="ts">
  import BigNumber from 'bignumber.js'
  import { get } from 'svelte/store'
  import { fade } from 'svelte/transition'
  import { onDestroy, onMount } from 'svelte'
  import { app, walletInterface, wallet, resetWalletState } from '../stores'

  import Modal from '../components/Modal.svelte'
  import ModalHeader from '../components/ModalHeader.svelte'
  import Wallets from '../components/Wallets.svelte'
  import SelectedWallet from '../components/SelectedWallet.svelte'
  import Button from '../elements/Button.svelte'
  import walletIcon from '../elements/walletIcon'

  import {
    getProviderName,
    createLegacyProviderInterface,
    createModernProviderInterface,
    getAddress,
    getBalance,
    getNetwork,
    networkName,
    openLink
  } from '../utilities'

  import {
    WalletSelectModalData,
    WalletModule,
    WalletSelectModule,
    WalletInterface
  } from '../interfaces'
  import { STORAGE_KEYS } from '../constants'

  export let module: WalletSelectModule = {
    heading: '',
    description: '',
    wallets: [],
    agreement: undefined
  }

  let modalData: WalletSelectModalData | null
  let showWalletDefinition: boolean
  let walletAlreadyInstalled: string | undefined
  let installMessage: string

  let selectedWalletModule: WalletModule | null

  const { mobileDevice, os } = get(app)
  let { heading, description, explanation, wallets, agreement, getHelpLink } =
    module

  const { termsUrl, privacyUrl, version } = agreement || {}
  const {
    terms: termsAgreed,
    privacy: privacyAgreed,
    version: versionAgreed
  } = JSON.parse(localStorage.getItem(STORAGE_KEYS.TERMS_AGREEMENT) || '{}')

  const showTermsOfService: boolean = !!(
    (termsUrl && !termsAgreed) ||
    (privacyUrl && !privacyAgreed) ||
    (version && version !== versionAgreed)
  )

  let walletsDisabled: boolean = showTermsOfService

  let agreed: boolean | undefined = undefined

  $: if (agreed) {
    localStorage.setItem(
      STORAGE_KEYS.TERMS_AGREEMENT,
      JSON.stringify({
        version,
        terms: !!termsUrl,
        privacy: !!privacyUrl
      })
    )
    walletsDisabled = false
  } else if (agreed === false) {
    localStorage.removeItem(STORAGE_KEYS.TERMS_AGREEMENT)
    walletsDisabled = true
  }

  let primaryWallets: WalletModule[]
  let secondaryWallets: WalletModule[] | undefined

  let loadingWallet: string | undefined = undefined

  let showingAllWalletModules = false
  const showAllWallets = () => (showingAllWalletModules = true)

  function lockScroll() {
    window.scrollTo(0, 0)
  }

  let originalOverflowValue: string

  onMount(() => {
    originalOverflowValue = window.document.body.style.overflow
    window.document.body.style.overflow = 'hidden'
    window.addEventListener('scroll', lockScroll)
  })

  onDestroy(() => {
    window.removeEventListener('scroll', lockScroll)
    window.document.body.style.overflow = originalOverflowValue
  })

  renderWalletSelect()

  async function renderWalletSelect() {
    const appState = get(app)
    wallets = await wallets

    const deviceWallets = (wallets as WalletModule[])
      .filter(wallet => wallet[mobileDevice ? 'mobile' : 'desktop'])
      .filter(wallet => {
        const { osExclusions = [] } = wallet
        return !osExclusions.includes(os.name)
      })

    if (deviceWallets.find(wallet => wallet.preferred)) {
      // if preferred wallets, then split in to preferred and not preferred
      primaryWallets = deviceWallets.filter(wallet => wallet.preferred)
      secondaryWallets = deviceWallets.filter(wallet => !wallet.preferred)
    } else {
      // otherwise make the first 4 wallets preferred
      primaryWallets = deviceWallets.slice(0, 4)
      secondaryWallets =
        deviceWallets.length > 4 ? deviceWallets.slice(4) : undefined
    }

    if (appState.autoSelectWallet) {
      const module = deviceWallets.find(
        (m: WalletModule) => m.name === appState.autoSelectWallet
      )

      app.update(store => ({ ...store, autoSelectWallet: '' }))

      if (module) {
        handleWalletSelect(module, true)
        return
      }
    }

    modalData = {
      heading,
      description,
      explanation,
      getHelpLink,
      primaryWallets,
      secondaryWallets
    }

    app.update(store => ({ ...store, walletSelectDisplayedUI: true }))
  }

  async function handleWalletSelect(
    module: WalletModule,
    autoSelected?: boolean
  ) {
    localStorage.setItem('onboard.acceptedTerms', 'true')

    const currentWalletInterface = get(walletInterface)
    const { browser, os } = get(app)

    if (currentWalletInterface && currentWalletInterface.name === module.name) {
      finish({ completed: true })
      return
    }

    loadingWallet = module.name

    const {
      provider,
      interface: selectedWalletInterface,
      instance
    } = await module.wallet({
      getProviderName,
      createLegacyProviderInterface,
      createModernProviderInterface,
      BigNumber,
      getNetwork,
      getAddress,
      getBalance,
      resetWalletState,
      networkName,
      browser,
      os
    })

    loadingWallet = undefined

    // if no interface then the user does not have the wallet they selected installed or available
    if (!selectedWalletInterface) {
      selectedWalletModule = module

      walletAlreadyInstalled = provider && getProviderName(provider)

      installMessage = module.installMessage
        ? module.installMessage({
            currentWallet: walletAlreadyInstalled,
            selectedWallet: selectedWalletModule.name
          })
        : ''

      // if it was autoSelected then we need to add modalData to show the modal
      if (autoSelected) {
        modalData = {
          heading,
          description,
          getHelpLink,
          explanation,
          primaryWallets,
          secondaryWallets
        }

        app.update(store => ({ ...store, walletSelectDisplayedUI: true }))
      }

      return
    }

    walletInterface.update((currentInterface: WalletInterface | null) => {
      if (currentInterface && currentInterface.disconnect) {
        currentInterface.disconnect()
      }

      return selectedWalletInterface
    })
    const { name, type, svg, iconSrc, iconSrcSet } = module
    wallet.set({
      provider,
      instance,
      dashboard: selectedWalletInterface.dashboard,
      name,
      connect: selectedWalletInterface.connect,
      type,
      icons: {
        svg,
        iconSrc,
        iconSrcSet
      }
    })

    finish({ completed: true })
  }

  function finish(options: { completed: boolean }) {
    modalData = null
    app.update(store => ({
      ...store,
      walletSelectInProgress: false,
      walletSelectCompleted: options.completed
    }))
  }

  let showConnection: boolean
</script>

<style>
  /* .bn-onboard-select-description, .bn-onboard-select-wallet-definition */
  p {
    font-size: 0.889em;
    padding: 0 1rem;
    color: #005ea4;
    margin: 1.6em 0 0 0;
    font-family: inherit;
  }

  /* .bn-onboard-select-info-container */
  .bn-onboard-select-info-container {
    display: flex;
    font-size: inherit;
    font-family: inherit;
    justify-content: space-between;
    padding: 0 1rem;
  }

  /* .bn-onboard-select-wallet-info */
  .bn-onboard-select-wallet-info {
    font-size: inherit;
    color: inherit;
    font-family: inherit;
    margin-top: 0.66em;
  }

  .bn-onboard-modal-terms-of-service {
    display: flex;
    align-items: center;
    padding: 0 1rem;
  }
  .bn-onboard-modal-terms-of-service-check-box {
    margin-right: 7px;
  }

  .terms-icon {
    margin: 30px auto 20px;
    width: 50px;
    height: 50px;
  }

  .terms-title {
    display: flex;
    justify-content: center;
    font-size: 1.2em;
    font-weight: bold;
    margin-bottom: 10px;
  }

  .terms-content {
    margin-bottom: 10px;
    text-align: center;
  }

  .terms-link {
    display: flex;
    justify-content: center;
    margin-bottom: 10px;
  }
  .terms-link a {
    color: #3da8f5;
  }

  button {
    width: 100%;
    height: 28px;
    display: flex;
    justify-content: center;
    align-items: center;
    color: #ffffff;
    background-color: #0210be;
    padding: 20px 0;
    border-radius: 10px;
    cursor: pointer;
  }

  .progress {
    width: 40px;
    height: 40px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin: 0 auto;
  }

  .progress-indicator {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background-color: #002886;
    cursor: pointer;
  }

  .progress-indicator-selected {
    width: 15px;
    height: 15px;
    border-radius: 50%;
    background-color: #002886;
    border: 3px solid #3da8f5;
  }
</style>

{#if modalData}
  <Modal closeModal={() => finish({ completed: false })}>
    {#if showConnection || localStorage.getItem('onboard.acceptedTerms') === 'true'}
      <ModalHeader icon={walletIcon} heading={modalData.heading} />
      {#if showTermsOfService}
        <p>
          <label class="bn-onboard-custom bn-onboard-modal-terms-of-service">
            <input
              class="bn-onboard-custom bn-onboard-modal-terms-of-service-check-box"
              type="checkbox"
              bind:checked={agreed}
            />
            <span>
              I agree to the
              {#if termsUrl}<a href={termsUrl} target="_blank"
                  >Terms & Conditions</a
                >{privacyUrl ? ' and' : '.'}
              {/if}
              {#if privacyUrl}<a href={privacyUrl} target="_blank"
                  >Privacy Policy</a
                >.{/if}
            </span>
          </label>
        </p>
      {/if}
      {#if !selectedWalletModule}
        <!--        <p class="bn-onboard-custom bn-onboard-select-description">-->
        <!--          {@html modalData.description}-->
        <!--        </p>-->
        <Wallets
          {modalData}
          {handleWalletSelect}
          {loadingWallet}
          {showingAllWalletModules}
          {showAllWallets}
          {walletsDisabled}
        />
        {#if localStorage.getItem('onboard.acceptedTerms') !== 'true'}
          <div class="progress">
            <div
              class="progress-indicator"
              on:click={() => (showConnection = false)}
            />
            <div class="progress-indicator-selected" />
          </div>
        {/if}
        <div class="bn-onboard-custom bn-onboard-select-info-container">
          <span class="bn-onboard-custom bn-onboard-select-wallet-info">
            Are you new?
            <Button
              help={true}
              primary={true}
              onclick={() => openLink(modalData?.getHelpLink)}
            >
              Get Started
            </Button>
          </span>
          {#if mobileDevice}
            <Button cta={false} onclick={() => finish({ completed: false })}
              >Dismiss</Button
            >
          {/if}
        </div>
        {#if showWalletDefinition}
          <p
            in:fade
            class="bn-onboard-custom bn-onboard-select-wallet-definition"
          >
            {@html modalData.explanation}
          </p>
        {/if}
      {:else}
        <SelectedWallet
          {selectedWalletModule}
          onBack={() => {
            selectedWalletModule = null
            walletAlreadyInstalled = undefined
          }}
          {installMessage}
        />
      {/if}
    {:else}
      <div class="terms-icon">
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
      <div class="terms-title">Connect Wallet</div>
      <div class="terms-content">
        By connecting your wallet, you accept Tracer’s Terms of Use and
        represent and warrant that you are not a resident of any of the
        following countries:
      </div>
      <div class="terms-content">
        China, the United States, Antigua and Barbuda, Algeria, Bangladesh,
        Bolivia, Belarus, Burundi, Myanmar (Burma), Cote D’Ivoire (Ivory Coast),
        Crimea and Sevastopol, Cuba, Democratic Republic of Congo, Ecuador,
        Iran, Iraq, Liberia, Libya, Magnitsky, Mali, Morocco, Nepal, North
        Korea, Somalia, Sudan, Syria, Venezuela, Yemen or Zimbabwe.
      </div>
      <div class="terms-link">
        Read the&nbsp;<a href="/terms-of-use" target="_blank">Terms of Use</a>
      </div>
      <div class="progress">
        <div class="progress-indicator-selected" />
        <div
          class="progress-indicator"
          on:click={() => (showConnection = true)}
        />
      </div>
      <button on:click={() => (showConnection = true)}>Ok, let's connect</button
      >
    {/if}
  </Modal>
{/if}
