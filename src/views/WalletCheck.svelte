<script lang="ts">
  import { get } from 'svelte/store'
  import { onDestroy, onMount } from 'svelte'
  import { fade } from 'svelte/transition'
  import BigNumber from 'bignumber.js'

  import { getBlocknative } from '../services'
  import {
    app,
    state,
    stateSyncStatus,
    walletInterface,
    wallet,
    address,
    network,
    balance
  } from '../stores'
  import { validateModal, validateWalletCheckModule } from '../validation'
  import { isPromise, openLink } from '../utilities'

  import Modal from '../components/Modal.svelte'
  import ModalHeader from '../components/ModalHeader.svelte'
  import Button from '../elements/Button.svelte'
  import Spinner from '../elements/Spinner.svelte'

  import {
    WalletCheckModule,
    WalletSelectFunction,
    WalletCheck,
    AppState,
    WalletCheckModal,
    UserState,
    Connect
  } from '../interfaces'

  export let walletCheck: WalletCheck
  export let walletSelect: WalletSelectFunction
  export let modules: WalletCheckModule[] | undefined

  const blocknative = getBlocknative()

  let currentState: UserState

  let activeModal: WalletCheckModal | undefined = undefined
  let currentModule: WalletCheckModule | undefined = undefined
  let errorMsg: string
  let pollingInterval: any
  let checkingModule: boolean = false
  let actionResolved: boolean | undefined = undefined
  let loading: boolean = false
  let loadingModal: boolean = false

  const unsubscribe = walletInterface.subscribe(currentInterface => {
    if (currentInterface === null) {
      handleExit()
      unsubscribe()
    }
  })

  // recheck modules if below conditions
  $: if (!activeModal && !checkingModule) {
    renderModule()
  }

  function lockScroll() {
    window.scrollTo(0, 0)
  }

  let originalOverflowValue: string

  const unsubscribeCurrentState = state.subscribe(
    store => (currentState = store)
  )

  onMount(() => {
    originalOverflowValue = window.document.body.style.overflow
    window.document.body.style.overflow = 'hidden'
    window.addEventListener('scroll', lockScroll)
  })

  onDestroy(() => {
    unsubscribeCurrentState()
    window.removeEventListener('scroll', lockScroll)
    window.document.body.style.overflow = originalOverflowValue
  })

  async function renderModule() {
    checkingModule = true

    let checkModules = modules || get(app).checkModules

    if (isPromise(checkModules)) {
      checkModules = await checkModules
      checkModules.forEach(validateWalletCheckModule)
      app.update(store => ({ ...store, checkModules }))
    }

    const currentWallet = get(wallet).name

    // loop through and run each module to check if a modal needs to be shown
    runModules(checkModules).then(
      (result: {
        modal: WalletCheckModal | undefined
        module: WalletCheckModule | undefined
      }) => {
        // no result then user has passed all conditions
        if (!result.modal) {
          blocknative &&
            blocknative.event({
              categoryCode: 'onboard',
              eventCode: 'onboardingCompleted'
            })

          handleExit(true)

          return
        }

        // set that UI has been displayed, so that timeouts can be added for UI transitions
        app.update(store => ({ ...store, walletCheckDisplayedUI: true }))

        activeModal = result.modal
        currentModule = result.module

        // log the event code for this module
        blocknative &&
          blocknative.event({
            eventCode: activeModal.eventCode,
            categoryCode: 'onboard'
          })

        // run any actions that module require as part of this step
        if (activeModal.action) {
          doAction()
        }

        // poll to automatically to check if condition has been met
        pollingInterval = setInterval(async () => {
          if (currentModule) {
            const result = await invalidState(currentModule, get(state))
            if (!result && actionResolved !== false) {
              resetState()

              // delayed for animations
              setTimeout(() => {
                checkingModule = false
              }, 250)
            } else {
              activeModal = result && result.modal ? result.modal : activeModal
            }
          }
        }, 100)
      }
    )
  }

  function doAction() {
    actionResolved = false
    loading = true

    activeModal &&
      activeModal.action &&
      (activeModal.action as Connect)()
        .then(() => {
          actionResolved = true
          loading = false
        })
        .catch((err: { message: string }) => {
          errorMsg = err.message
          loading = false
        })
  }

  function handleExit(
    completed: boolean = false,
    { switchingWallets }: Partial<AppState> = {}
  ) {
    resetState()
    app.update((store: AppState) => ({
      ...store,
      switchingWallets,
      walletCheckInProgress: false,
      walletCheckCompleted: completed,
      accountSelectInProgress: false
    }))
  }

  function resetState() {
    clearInterval(pollingInterval)
    errorMsg = ''
    actionResolved = undefined
    activeModal = undefined
    currentModule = undefined
  }

  function runModules(modules: WalletCheckModule[]) {
    return new Promise(
      async (
        resolve: (result: {
          modal: WalletCheckModal | undefined
          module: WalletCheckModule | undefined
        }) => void
      ) => {
        for (const module of modules) {
          const result = await invalidState(module, currentState)

          if (result) {
            return resolve(result)
          }
        }

        return resolve({ modal: undefined, module: undefined })
      }
    )
  }

  async function invalidState(
    module: WalletCheckModule,
    state: UserState
  ): Promise<
    { module: WalletCheckModule; modal: WalletCheckModal } | undefined
  > {
    const result = module({
      ...state,
      BigNumber,
      walletSelect,
      walletCheck,
      exit: handleExit,
      wallet: get(wallet),
      stateSyncStatus,
      stateStore: {
        address,
        network,
        balance
      }
    })

    if (result) {
      if (isCheckModal(result)) {
        validateModal(result)
        return {
          module,
          modal: result
        }
      } else {
        // module returned a promise, so await it for val
        // show loading spinner if promise doesn't resolve within 150ms
        let modal
        await new Promise(resolve => {
          let completed: boolean = false

          result.then(res => {
            loadingModal = false
            completed = true
            modal = res
            resolve(undefined)
          })

          setTimeout(() => {
            if (!completed) {
              loadingModal = true
            }
          }, 650)
        })

        if (modal) {
          validateModal(modal)
          return {
            module,
            modal
          }
        }
      }
    }
  }

  function isCheckModal(
    val: WalletCheckModal | Promise<WalletCheckModal | undefined>
  ): val is WalletCheckModal {
    return (val as WalletCheckModal).heading !== undefined
  }
</script>

<style>
  /* .bn-onboard-prepare-description */
  p {
    font-size: 0.889em;
    padding: 0 1rem;
    color: #3da8f5;
    padding-bottom: 1rem;
    border-bottom: 1px solid #002886;
    font-family: inherit;
    margin: 1em 0;
  }

  /* .bn-onboard-prepare-error */
  span {
    color: #e2504a;
    font-size: 0.889em;
    font-family: inherit;
    display: block;
    margin: 0 1rem 0.75em 1rem;
    padding: 0.5em;
    border: 1px solid #e2504a;
    border-radius: 5px;
  }

  /* .bn-onboard-prepare-button-container */
  div {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    position: relative;
  }

  /* .bn-onboard-get-help-container */
  .bn-onboard-get-help-container {
    flex-direction: row;
  }
  section {
    display: flex;
    justify-content: center;
    flex-direction: column;
    align-items: center;
    margin-bottom: 1rem;
  }

  .loading {
    margin: 30px auto 20px;
    width: 177px;
    height: 92px;
  }
</style>

{#if loadingModal}
  <Modal closeable={false}>
    <Spinner description="Checking wallet" />
  </Modal>
{/if}

{#if activeModal}
  <Modal closeModal={() => handleExit()}>
    <div class="loading">
      <svg
        id="eaGBD5e1KEP1"
        xmlns="http://www.w3.org/2000/svg"
        xmlns:xlink="http://www.w3.org/1999/xlink"
        viewBox="688 0 620 267.400000"
        shape-rendering="geometricPrecision"
        text-rendering="geometricPrecision"
      >
        <style>
          <![CDATA[#eaGBD5e1KEP4_to {animation: eaGBD5e1KEP4_to__to 1000ms linear infinite normal forwards}@keyframes eaGBD5e1KEP4_to__to { 0% {transform: translate(577.321785px,-59.500000px)} 50% {transform: translate(638.185892px,-59.500000px)} 100% {transform: translate(579.050000px,-59.500000px)}} #eaGBD5e1KEP6_to {animation: eaGBD5e1KEP6_to__to 1000ms linear infinite normal forwards}@keyframes eaGBD5e1KEP6_to__to { 0% {transform: translate(579.050000px,-59.500000px)} 50% {transform: translate(519.050000px,-59.500000px)} 100% {transform: translate(579.050000px,-59.500000px)}}]]>
        </style>
        <g id="eaGBD5e1KEP2">
          <g id="eaGBD5e1KEP3" />
          <g id="eaGBD5e1KEP4_to" transform="translate(577.321785,-59.500000)">
            <polygon
              id="eaGBD5e1KEP4"
              points="667.700000,59.500000 436.100000,193.200000 667.700000,326.900000 667.700000,291.900000 496.900000,193.300000 667.700000,94.700000"
              transform="translate(0,0)"
              stroke="none"
              stroke-width="1"
            />
          </g>
          <rect
            id="eaGBD5e1KEP5"
            width="30.400000"
            height="267.400000"
            rx="0"
            ry="0"
            transform="matrix(1 0 0 1 984.75000000000000 0.00000000000003)"
            stroke="none"
            stroke-width="1"
          />
          <g id="eaGBD5e1KEP6_to" transform="translate(579.050000,-59.500000)">
            <polygon
              id="eaGBD5e1KEP6"
              points="174.200000,94.700000 344.900000,193.300000 174.200000,291.900000 174.200000,326.900000 405.700000,193.200000 174.200000,59.500000"
              transform="translate(0,0)"
              stroke="none"
              stroke-width="1"
            />
          </g>
        </g>
      </svg>
    </div>
    <div>{activeModal.description}</div>

    <!--    <ModalHeader icon={activeModal.icon || ''} heading={activeModal.heading} />-->
    <!--    <p class="bn-onboard-custom bn-onboard-prepare-description">-->
    <!--      {@html activeModal.description}-->
    <!--    </p>-->
    <!--    {#if errorMsg}-->
    <!--      <span-->
    <!--        class:bn-onboard-dark-mode-background={$app.darkMode}-->
    <!--        class="bn-onboard-custom bn-onboard-prepare-error"-->
    <!--        in:fade-->
    <!--      >-->
    <!--        {errorMsg}-->
    <!--      </span>-->
    <!--    {/if}-->

    <!--    {#if activeModal.html}-->
    <!--      <section class="bn-onboard-custom bn-onboard-wallet-check-section">-->
    <!--        {@html activeModal.html}-->
    <!--      </section>-->
    <!--    {/if}-->

    <!--    <div class="bn-onboard-custom bn-onboard-prepare-button-container">-->
    <!--      {#if activeModal.button}-->
    <!--        <Button position="right" onclick={activeModal.button.onclick}>-->
    <!--          {activeModal.button.text}-->
    <!--        </Button>-->
    <!--      {/if}-->
    <!--      {#if errorMsg}-->
    <!--        <Button onclick={doAction}>Try Again</Button>-->
    <!--      {:else}-->
    <!--        <div />-->
    <!--      {/if}-->
    <!--      {#if loading}-->
    <!--        <Spinner />-->
    <!--        {#if activeModal.getHelpLink}-->
    <!--          <section class="bn-onboard-custom bn-onboard-get-help-container">-->
    <!--            Having trouble?-->
    <!--            <Button-->
    <!--              help={true}-->
    <!--              primary={true}-->
    <!--              onclick={() => openLink(activeModal?.getHelpLink)}-->
    <!--              >Get Help</Button-->
    <!--            >-->
    <!--          </section>-->
    <!--        {/if}-->
    <!--      {/if}-->
    <!--    </div>-->
  </Modal>
{/if}
