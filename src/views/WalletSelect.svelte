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
    wallets: Promise.resolve([]),
    agreement: undefined
  }

  let modalData: WalletSelectModalData | null
  let showWalletDefinition: boolean
  let walletAlreadyInstalled: string | undefined
  let installMessage: string

  let selectedWalletModule: WalletModule | null

  const { mobileDevice, os } = get(app)
  let { heading, description, explanation, agreement, getHelpLink } = module

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
    const wallets = await module.wallets

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
    height: 300px;
    overflow-y: scroll;
  }
  .terms-content-dark {
      color: white;
  }

  .agreement-content {
    text-align: center;
    color: gray;
    height: 300px;
    overflow-y: scroll;
  }
  .agreement-content-dark {
      color: white;
  }

  @media (max-width: 640px) {
    .terms-content {
      height: 100px;
      overflow-y: scroll;
    }
    .agreement-content {
      height: 100px;
      overflow-y: scroll;
    }
  }

  .link {
    display: flex;
    justify-content: center;
    color: gray;
  }
  .link-dark {
    color: white;
  }
  .link a {
    color: inherit;
    text-decoration: underline;
  }

  button {
    width: 100%;
    height: 28px;
    display: flex;
    justify-content: center;
    align-items: center;
    color: var(--primary);
    border-color: var(--primary);
    border-width: 1px;
    background-color: transparent;
    padding: 20px 0;
    border-radius: 1px;
    cursor: not-allowed;
    opacity: 0.5;
  }


  button.active {
    transition: 0.5s;
    opacity: 1;
    cursor: pointer;
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
    width: 12px;
    height: 12px;
    border-radius: 50%;
    border-color: var(--primary);
    border-width: 2px;
    background-color: inherit;
    cursor: pointer;
  }
  .progress-indicator-dark {
      background-color: inherit;
  }

  .progress-indicator-selected {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    box-shadow: 0px 0px 10px rgba(28, 100, 242, 0.8);
    background-color: var(--primary);
    border: 2px solid var(--primary);
  }
  .progress-indicator-selected-dark {
      background-color: var(--primary);
      border: 2px solid var(--primary);
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
  .checkbox-text-dark {
    color: white;
  }

  .input-dark {
    accent-color: var(--primary);
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
              class:progress-indicator-dark={$app.darkMode}
              on:click={() => {
                showConnection = false
                showAgreement = false
              }}></div>
            <div
              class="progress-indicator"
              class:progress-indicator-dark={$app.darkMode}
              on:click={() => (showConnection = false)}></div>
            <div class="progress-indicator-selected" class:progress-indicator-selected-dark={$app.darkMode}></div>
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
      <div class="agreement-content" class:agreement-content-dark={$app.darkMode}>
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
        <div class="link" class:link-dark={$app.darkMode}>
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
          class:progress-indicator-dark={$app.darkMode}
          on:click={() => (showAgreement = false)}></div>
        <div class="progress-indicator-selected" class:progress-indicator-selected-dark={$app.darkMode}></div>
        <div
          class="progress-indicator"
          class:progress-indicator-dark={$app.darkMode}
          on:click={() => {
            if (agreedToAgreement) {
              showConnection = true
            }
          }}></div>
      </div>
      <div class="check">
        <input type="checkbox" bind:checked={agreedToAgreement} class:input-dark={$app.darkMode} />
        <div
          class="checkbox-text"
          class:checkbox-text-dark={$app.darkMode}
          on:click={() => (agreedToAgreement = !agreedToAgreement)}
        >
          I have read and agree to the Participation Agreement
        </div>
      </div>
      <button
        class={`${agreedToAgreement ? 'active' : ''}`}
        class:button-dark={$app.darkMode}
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
      <div class="terms-content" class:terms-content-dark={$app.darkMode}>
        Your use of a smart contract governed by Tracer DAO (Mycelium), including a
        Mycelium Perpetual Pools or Mycelium Perpetual Swaps contract and the Mycelium Factory (“Mycelium Suite”),
        involves various risks, including, but not limited to, the risks outlined below.
        By interacting with the Mycelium Suite, you accept those risks. If you choose to use the Mycelium Suite,
        this Disclaimer applies in addition to Mycelium’s Terms of Use, which are available here: https://mycelium.xyz/terms-of-use
        <br /><br />
        1. Price and Asset Risk
        <br />
        Asset losses are possible due to the fluctuation of prices of tokens.
        You acknowledge that financial contracts involving cryptocurrencies are inherently risky,
        some cryptocurrencies are not recognised legal tender in some countries,
        are unregulated by many central and government authorities, and may be subject to extreme price volatility.
        You warrant that you understand the risks associated with transactions, cryptocurrencies,
        financial contracts and any other goods, services or products in connection to the Mycelium protocol.
        Before using the Mycelium protocol, you should review the relevant documentation to
        make sure you understand how the Mycelium protocol works.
        <br /><br />
        2. Liquidation Risk
        <br />
        When interacting with the Mycelium Suite, your positions may be subject to liquidation risk.
        Despite having well defined liquidation penalties, your loss could be 100% of your position.
        This risk is not only theoretical, during the so-called “Black Thursday”,
        around $8M of user’s positions in MakerDAO was lost. The Mycelium Suite is non-custodial,
        and therefore can not help users to avoid liquidations, or recover funds following liquidation.
        <br /><br />
        3. Oracle Risk
        <br />
        Mycelium relies on oracles to provide spot price data for assets.
        This price data is used as part of the mint and burn functions (for Perpetual Pools)
        and liquidation flow (for Perpetual Swaps) and, as a result, has inherent risks.
        Should oracle prices not be updated, or should they be updated erroneously,
        there is risk a user may suffer loss, or be liquidated, unexpectedly. Currently,
        the Mycelium Suite uses Chainlink’s oracle network; this may be subject to change.
        <br /><br />
        4. Smart Contract and Software Risk
        <br />
        The Mycelium Suite has been audited (Perpetual Pools by Sigma Prima and
        Perpetual Swaps by Sigma Prime and Code 423n4).
        The contracts were checked for correctness of functionality and safety of user funds.
        However, an audit does not eliminate the risk of an exploit or bug being present in the Mycelium protocol.
        You must do your own research before interacting with the Mycelium protocol,
        and only supply funds that you are prepared to lose.
        <br /><br />
        5. Contract Upgrade Risk
        <br />
        Contract upgrades may be implemented by Mycelium, via proposal, to the Mycelium Suite.
        While these upgrades will likely be audited prior to implementation,
        the Mycelium Suite’s codebase is subject to change and, as such,
        risks of exploits and bugs are present. You must remain aware of updates being
        implemented by Mycelium to the Mycelium Suite, and ensure that, following any update,
        the Mycelium Suite is safe to use.
        <br /><br />
        6. Governance Risk
        <br />
        Mycelium owns and controls the Mycelium Suite. Upgrades and modifications to the protocol
        are managed in a decentralised manner by holders of TCR governance tokens.
        No entity involved in creating the Mycelium Suite (including developers) will
        be liable for any claims or damage whatsoever associated with your use, inability
        to use, direct, indirect, incidental, special, exemplary, punitive or consequential damages,
        or loss of profits, cryptocurrencies, tokens, or anything else of value.
        For the avoidance of doubt, Mycelium does not control user funds.
        <br /><br />
        7. Developer Must Comply with Applicable Laws
        <br />
        Mycelium is not responsible for the actions of Mycelium Suite users. Users of Mycelium Suite are
        solely responsible for ensuring compliance with the laws of their specific jurisdiction.
        Mycelium are curators of the Mycelium Suite and maintain no liability for the repercussions,
        actions and consequences of material created using the Mycelium Suite.
        <br /><br />
        8. Limitation of Liability
        <br />
        MYCELIUM, ITS AFFILIATES, THEIR RESPECTIVE OFFICERS, DIRECTORS, EMPLOYEES, AGENTS, SUPPLIERS,
        OR LICENSORS MAKE NO WARRANTIES OR REPRESENTATIONS ABOUT THE MYCELIUM SUITE, ITS CONTENTS,
        OR ANY CREATION OR ANY DERIVATIVE OF THE MYCELIUM FACTORY INCLUDING BUT NOT LIMITED TO,
        ITS ACCURACY, RELIABILITY, FUNCTIONALITY, COMPLETENESS, OR TIMELINESS. ANY CREATION BY A
        USER OF THE MYCELIUM SUITE IS NOT THE RESPONSIBILITY OF MYCELIUM AND THE USER INDEMNIFIES
        MYCELIUM AGAINST ANY AND ALL CLAIMS THAT MAY ARISE OUT OF USING THE MYCELIUM SUITE.
        USERS ACKNOWLEDGE AND AGREE THAT THEY TAKE SOLE RESPONSIBILITY FOR CREATIONS OR WORKS OF
        THE MYCELIUM SUITE AND ACKNOWLEDGE THAT THEY USE THE MYCELIUM SUITE AT THEIR OWN RISK.
        <br /><br />
        <div class="link" class:link-dark={$app.darkMode}>
          Read the&nbsp;<a href="/terms-of-use" target="_blank">Terms of Use</a>
        </div>
      </div>
      <br />
      <div class="progress">
        <div class="progress-indicator-selected" class:progress-indicator-selected-dark={$app.darkMode}></div>
        <div
          class="progress-indicator"
          class:progress-indicator-dark={$app.darkMode}
          on:click={() => {
            if (agreedToTerms) {
              showAgreement = true
            }
          }}></div>
        <div
          class="progress-indicator"
          class:progress-indicator-dark={$app.darkMode}
          on:click={() => {
            if (agreedToTerms && agreedToAgreement) {
              showAgreement = true
              showConnection = true
            }
          }}></div>
      </div>
      <div class="check">
        <input type="checkbox" bind:checked={agreedToTerms} class:input-dark={$app.darkMode} />
        <div
          class="checkbox-text"
          class:checkbox-text-dark={$app.darkMode}
          on:click={() => (agreedToTerms = !agreedToTerms)}
        >
          I have read and agree to the Terms of Use
        </div>
      </div>
      <button
        class={`${agreedToTerms ? 'active' : ''}`}
        class:button-dark={$app.darkMode}
        on:click={agreedToTerms ? () => (showAgreement = true) : null}
        >Continue</button
      >
    {/if}
  </Modal>
{/if}
