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
    localStorage.setItem('onboard.acceptedAllTerms', 'true')

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

  let agreedToTerms: boolean = localStorage.getItem('agreedToTerms') === 'true'
  $: if (agreedToTerms) {
    localStorage.setItem('agreedToTerms', 'true')
  } else if (agreedToTerms === false) {
    localStorage.setItem('agreedToTerms', 'false')
  }

  let agreedToAgreement: boolean =
    localStorage.getItem('agreedToAgreement') === 'true'
  $: if (agreedToAgreement) {
    localStorage.setItem('agreedToAgreement', 'true')
  } else if (agreedToAgreement === false) {
    localStorage.setItem('agreedToAgreement', 'false')
  }

  let showAgreement: boolean
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

  .info-container {
    display: flex;
    justify-content: space-between;
    padding: 20px 20px 0;
  }

  .icon {
    margin: 30px auto 20px;
    width: 50px;
    height: 50px;
  }

  .title {
    display: flex;
    justify-content: center;
    font-size: 18px;
    margin-bottom: 5px;
  }

  .subtitle {
    display: flex;
    justify-content: center;
    color: gray;
  }

  .terms-content {
    text-align: center;
    color: gray;
  }

  .agreement-content {
    text-align: center;
    color: gray;
    height: 300px;
    overflow-y: scroll;
  }

  .link {
    display: flex;
    justify-content: center;
    color: gray;
  }
  .link a {
    color: #3da8f5;
    text-decoration: underline;
  }

  button {
    width: 100%;
    height: 28px;
    display: flex;
    justify-content: center;
    align-items: center;
    color: #ffffff;
    background-color: #0210be;
    border: 1px solid #0210be;
    padding: 20px 0;
    border-radius: 10px;
    cursor: not-allowed;
    opacity: 0.5;
  }
  button.active {
    transition: 0.5s;
    opacity: 1;
    cursor: pointer;
  }
  button.active:hover {
    color: #0210be;
    background-color: #ffffff;
  }

  .progress {
    width: 80px;
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

  .check {
    height: 60px;
    margin-left: 10px;
    display: flex;
    align-items: center;
  }
  .check > * {
    cursor: pointer;
  }
  .checkbox-text {
    margin-left: 5px;
    color: gray;
  }
</style>

{#if modalData}
  <Modal closeModal={() => finish({ completed: false })}>
    {#if showConnection || localStorage.getItem('onboard.acceptedAllTerms') === 'true'}
      <ModalHeader icon={walletIcon} heading={modalData.heading} />
      {#if !selectedWalletModule}
        <Wallets
          {modalData}
          {handleWalletSelect}
          {loadingWallet}
          {showingAllWalletModules}
          {showAllWallets}
          {walletsDisabled}
        />
        {#if localStorage.getItem('onboard.acceptedAllTerms') !== 'true'}
          <br />
          <div class="progress">
            <div
              class="progress-indicator"
              on:click={() => {
                showConnection = false
                showAgreement = false
              }}></div>
            <div
              class="progress-indicator"
              on:click={() => (showConnection = false)}></div>
            <div class="progress-indicator-selected"></div>
          </div>
        {/if}
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
    {:else if showAgreement}
      <div class="icon">
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
      <div class="title">Connect Wallet</div>
      <div class="subtitle">Participation Agreement</div>
      <br />
      <div class="agreement-content">
        a) BY CLICKING THE ‘OK, LET’S CONNECT’ BUTTON BELOW; YOU WILL DIGITALLY
        SIGN THE TRACER DAO PARTICIPATION AGREEMENT.
        <br /><br />
        b) PLEASE READ THIS PARTICIPATION AGREEMENT ("AGREEMENT") CAREFULLY BEFORE
        CONFIRMING YOUR INTENT TO BE BOUND BY PARTICIPATING IN THE TRACER DAO, TRACER
        SMART CONTRACTS AND/OR TRACER TOKENS. THIS AGREEMENT INCLUDES THE TERMS OF
        PARTICIPATION IN THE TRACER DAO. YOU UNDERSTAND, AGREE AND CONFIRM THAT:
        <br /><br />
        1. LIMITATION OF LIABILITY
        <br /><br />
        YOU ACKNOWLEDGE AND AGREE THAT, TO THE FULLEST EXTENT PERMITTED BY APPLICABLE
        LAW, REGARDLESS OF THE FORM OF ACTION, WHETHER IN CONTRACT, TORT (INCLUDING
        NEGLIGENCE) OR OTHERWISE, IN NO EVENT WILL TRACER DAO OR ITS AFFILIATES,
        INCLUDING WITHOUT LIMITATION, THEIR RESPECTIVE OFFICERS, DIRECTORS, EMPLOYEES,
        SUCCESSORS AND ASSIGNS, BE LIABLE TO YOU OR ANY OTHER PARTY FOR ANY DIRECT
        OR INDIRECT LOSS,DAMAGE,COST, EXPENSE OR LIABILITY OF ANY KIND (“LOSS”) ARISING
        IN ANY WAY OUT OF OR IN CONNECTION WITH THE AVAILABILITY, USE, RELIANCE ON,
        OR INABILITY TO PARTICIPATE IN THE TRACER DAO WITHOUT LIMITATION; THE TRACER
        DAO IS AN EXPERIMENT IN THE FIELD OF DECENTRALISED GOVERNANCE STRUCTURES,
        IN WHICH PARTICIPATION IS ENTIRELY AT YOUR OWN RISK; YOU ALSO AGREE TO WAIVE
        AND LIMIT ANY POTENTIAL LIABILITY OF TRACER DAO SERVICE PROVIDERS OR OTHER
        TRACER DAO PARTICIPANTS.
        <br /><br />
        2. REPRESENTATIONS AND WARRANTIES
        <br /><br />
        YOU ARE SOPHISTICATED AND HAVE SUFFICIENT TECHNICAL UNDERSTANDING OF THE
        FUNCTIONALITY, USAGE, STORAGE, TRANSMISSION MECHANISMS, AND INTRICACIES ASSOCIATED
        WITH CRYPTOGRAPHIC TOKENS, TOKEN STORAGE FACILITIES (INCLUDING WALLETS),
        BLOCKCHAIN TECHNOLOGY, AND BLOCKCHAIN-BASED SOFTWARE SYSTEMS;
        <br />
        YOU UNDERSTAND THAT ALL GOVERNANCE TOKENS RELATED TO THE TRACER DAO ONLY
        ALLOW HOLDERS TO PARTICIPATE IN THE TRACER SYSTEM VIA ITS GOVERNANCE MECHANISM
        AND PROVIDE NO OWNERSHIP OR ECONOMIC RIGHTS OF ANY KIND;
        <br />
        YOU ARE NOT CLAIMING OR RECEIVING ANY GOVERNANCE TOKENS FOR A SPECULATIVE
        PURPOSE AND NOT ACQUIRING A TRACER DAO TOKEN AS AN INVESTMENT OR WITH THE
        AIM OF MAKING A PROFIT. YOU FURTHER REPRESENT AND WARRANT THAT YOU ARE AN
        ACTIVE USER OF BLOCKCHAIN TECHNOLOGY AND BLOCKCHAIN-BASED SOFTWARE SYSTEMS.
        IF YOU ARE CLAIMING OR HAVE CLAIMED A GOVERNANCE TOKEN YOU ARE OR HAVE DONE
        SO ONLY TO PARTICIPATE IN THE TRACER DAO EXPERIMENT AND TO PARTICIPATE IN
        TRACER DAO GOVERNANCE-RELATED DECISIONS;
        <br /><br />
        YOU REPRESENT AND WARRANT THAT PARTICIPATING IN THE TRACER DAO UNDER THIS
        AGREEMENT IS NOT PROHIBITED UNDER THE LAWS OF YOUR JURISDICTION OR UNDER
        THE LAWS OF ANY OTHER JURISDICTION TO WHICH YOU MAY BE SUBJECT AND YOU ARE
        AND WILL CONTINUE TO BE IN FULL COMPLIANCE WITH APPLICABLE LAWS (INCLUDING,
        BUT NOT LIMITED TO, IN COMPLIANCE WITH ANY TAX OR DISCLOSURE OBLIGATIONS
        TO WHICH YOU MAY BE SUBJECT IN ANY APPLICABLE JURISDICTION);
        <br /><br />
        FURTHER YOU REPRESENT AND WARRANT THAT YOU HAVE THE FULL RIGHT, CAPACITY
        AND POWER TO ENTER INTO AND FULLY PERFORM THIS AGREEMENT IN ACCORDANCE WITH
        THE TERMS AND THAT YOUR EXECUTION AND PERFORMANCE UNDER THIS AGREEMENT WILL
        NOT VIOLATE THE PROVISIONS OF ANY OTHER AGREEMENT TO WHICH YOU ARE A PARTY;
        AND
        <br /><br />
        YOU ACKNOWLEDGE AND AGREE THAT YOU PARTICIPATE AT YOUR OWN RISK. PARTICIPATION
        IN THE TRACER DAO IS PROVIDED AS IS WITHOUT ANY EXPRESS OR IMPLIED WARRANTIES.
        THE TRACER DAO DOES NOT GUARANTEE THAT PARTICIPATION WILL ALWAYS BE SAFE,
        SECURE, OR ERROR-FREE OR THAT THE TRACER DAO WILL ALWAYS FUNCTION WITHOUT
        DISRUPTIONS, DELAYS OR IMPERFECTIONS.
        <br /><br />
        3. DISPUTES
        <br /><br />
        IF A DISPUTE CANNOT BE RESOLVED AMICABLY WITHIN THE TRACER DAO, ALL CLAIMS
        ARISING OUT OF OR RELATING TO THIS AGREEMENT OR THE TRACER DAO SHALL BE SETTLED
        IN BINDING ARBITRATION IN ACCORDANCE WITH THE ARBITRATION CLAUSE CONTAINED
        HEREIN; BY ENTERING INTO THIS AGREEMENT YOU ARE AGREEING TO WAIVE YOUR RIGHT,
        IF ANY, TO A TRIAL BY JURY AND PARTICIPATION IN A CLASS ACTION LAWSUIT;
        <br />
        THIS AGREEMENT WILL BE DEEMED TO BE DIGITALLY SIGNED IF YOU SUBMIT ANY TRANSACTION
        TO THE TRACER SYSTEM ON THE ETHEREUM BLOCKCHAIN WHETHER VIA DIRECT INTERACTION
        WITH ANY SMART CONTRACT WHEREIN THIS AGREEMENT IS STATED, REFERENCED OR BY
        INTERACTION WITH ANY OTHER VOTE INTERFACE INCORPORATING THIS AGREEMENT. ANY
        SUCH DIGITAL SIGNATURE SHALL CONSTITUTE CONCLUSIVE EVIDENCE OF YOUR INTENT
        TO BE BOUND BY THIS AGREEMENT AND YOU WAIVE ANY RIGHT TO CLAIM THAT THE AGREEMENT
        IS UNENFORCEABLE OR OTHERWISE ARGUE AGAINST ITS ADMISSIBILITY OR AUTHENTICITY
        IN ANY LEGAL PROCEEDINGS; AND TRACER DAO IS GOVERNED BY THE LAWS OF AUSTRALIA
        WITHOUT REFERENCE TO ITS CHOICE OF LAW RULES.
        <br /><br />
        YOU HAVE READ, FULLY UNDERSTOOD, AND ACCEPT THIS DISCLAIMER AND ALL THE TERMS
        CONTAINED IN THE PARTICIPATION AGREEMENT.
        <br /><br />
        <div class="link">
          Read the&nbsp;<a
            href="https://gateway.pinata.cloud/ipfs/QmS161WXV2bEAWUtdecfS5FYPmHQZdhNnjVFAwQ5FTX3og"
            target="_blank">Participation Agreement</a
          >
        </div>
      </div>
      <br />
      <div class="progress">
        <div
          class="progress-indicator"
          on:click={() => (showAgreement = false)}></div>
        <div class="progress-indicator-selected"></div>
        <div
          class="progress-indicator"
          on:click={() => {
            if (agreedToAgreement) {
              showConnection = true
            }
          }}></div>
      </div>
      <div class="check">
        <input type="checkbox" bind:checked={agreedToAgreement} />
        <div
          class="checkbox-text"
          on:click={() => (agreedToAgreement = !agreedToAgreement)}
        >
          I have read and agree to the Participation Agreement
        </div>
      </div>
      <button
        class={`${agreedToAgreement ? 'active' : ''}`}
        on:click={agreedToAgreement ? () => (showConnection = true) : null}
        >Ok, let's connect</button
      >
    {:else}
      <div class="icon">
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
      <div class="title">Connect Wallet</div>
      <div class="subtitle">Terms of Use</div>
      <br />
      <div class="terms-content">
        By connecting your wallet, you accept Tracer’s Terms of Use and
        represent and warrant that you are not a resident of any of the
        following countries:
        <br /><br />
        China, the United States, Antigua and Barbuda, Algeria, Bangladesh, Bolivia,
        Belarus, Burundi, Myanmar (Burma), Cote D’Ivoire (Ivory Coast), Crimea and
        Sevastopol, Cuba, Democratic Republic of Congo, Ecuador, Iran, Iraq, Liberia,
        Libya, Magnitsky, Mali, Morocco, Nepal, North Korea, Somalia, Sudan, Syria,
        Venezuela, Yemen or Zimbabwe.
      </div>
      <br />
      <div class="link">
        Read the&nbsp;<a href="/terms-of-use" target="_blank">Terms of Use</a>
      </div>
      <br />
      <div class="progress">
        <div class="progress-indicator-selected"></div>
        <div
          class="progress-indicator"
          on:click={() => {
            if (agreedToTerms) {
              showAgreement = true
            }
          }}></div>
        <div
          class="progress-indicator"
          on:click={() => {
            if (agreedToTerms && agreedToAgreement) {
              showAgreement = true
              showConnection = true
            }
          }}></div>
      </div>
      <div class="check">
        <input type="checkbox" bind:checked={agreedToTerms} />
        <div
          class="checkbox-text"
          on:click={() => (agreedToTerms = !agreedToTerms)}
        >
          I have read and agree to the Terms of Use
        </div>
      </div>
      <button
        class={`${agreedToTerms ? 'active' : ''}`}
        on:click={agreedToTerms ? () => (showAgreement = true) : null}
        >Continue</button
      >
    {/if}
  </Modal>
{/if}
