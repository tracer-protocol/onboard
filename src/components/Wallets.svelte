<script lang="ts">
  import { onDestroy } from 'svelte'
  import Button from '../elements/Button.svelte'
  import IconButton from '../elements/IconButton.svelte'
  import { wallet } from '../stores'
  import {
    WalletSelectModalData,
    WalletModule,
    WritableStore
  } from '../interfaces'
  export let modalData: WalletSelectModalData
  export let handleWalletSelect: (wallet: WalletModule) => void
  export let loadingWallet: string | undefined
  export let showingAllWalletModules: boolean = false
  export let showAllWallets: () => void
  export let walletsDisabled: boolean = false

  let selectedWallet: WritableStore

  const unsubscribe = wallet.subscribe(wallet => (selectedWallet = wallet))

  onDestroy(() => unsubscribe())
</script>

<style>
  /* .bn-onboard-modal-select-wallets */
  ul {
    display: flex;
    flex-flow: row wrap;
    align-items: center;
    list-style-type: none;
    padding: 0;
    font-family: inherit;
    font-size: inherit;
    line-height: 1.15;
    box-sizing: border-box;
  }

  ul li {
    width: 100%;
    padding: 0;
    height: 63px;
    transition: 0.1s;
  }

  ul li:hover {
    background: #3da8f5;
  }

  ::-webkit-scrollbar {
    display: none;
  }

  @media only screen and (max-width: 450px) {
    ul li {
      width: 100%;
    }

    ul {
      max-height: 66vh;
      overflow-y: scroll;
    }
  }

  .border-bottom {
    width: 100%;
    border-bottom: 1px solid #e6e6e6;
  }
</style>

<ul class="bn-onboard-custom bn-onboard-modal-select-wallets">
  <!--{#each modalData.primaryWallets as wallet, i (wallet.name)}-->
  <!--  <li>-->
  <!--    <IconButton-->
  <!--      disabled={walletsDisabled}-->
  <!--      onclick={() => handleWalletSelect(wallet)}-->
  <!--      iconSrc={wallet.iconSrc}-->
  <!--      iconSrcSet={wallet.iconSrcSet}-->
  <!--      svg={wallet.svg}-->
  <!--      text={wallet.name}-->
  <!--      currentlySelected={wallet.name === selectedWallet.name}-->
  <!--      {loadingWallet}-->
  <!--    />-->
  <!--  </li>-->
  <!--{/each}-->
  <!--{#if modalData.secondaryWallets && modalData.secondaryWallets.length && !showingAllWalletModules}-->
  <!--  <div>-->
  <!--    <Button disabled={walletsDisabled} onclick={showAllWallets}-->
  <!--      >Show More</Button-->
  <!--    >-->
  <!--  </div>-->
  <!--{/if}-->
  <!--{#if showingAllWalletModules}-->
  <!--  {#each modalData.secondaryWallets as wallet, i (wallet.name)}-->
  <!--    <li>-->
  <!--      <IconButton-->
  <!--        disabled={walletsDisabled}-->
  <!--        onclick={() => handleWalletSelect(wallet)}-->
  <!--        iconSrc={wallet.iconSrc}-->
  <!--        iconSrcSet={wallet.iconSrcSet}-->
  <!--        svg={wallet.svg}-->
  <!--        text={wallet.name}-->
  <!--        currentlySelected={wallet.name === selectedWallet.name}-->
  <!--        {loadingWallet}-->
  <!--      />-->
  <!--    </li>-->
  <!--  {/each}-->
  <!--    <div class="border-bottom"></div>-->
  <!--{/if}-->

  {#each modalData.primaryWallets as wallet, i (wallet.name)}
    <li>
      <IconButton
        disabled={walletsDisabled}
        onclick={() => handleWalletSelect(wallet)}
        iconSrc={wallet.iconSrc}
        iconSrcSet={wallet.iconSrcSet}
        svg={wallet.svg}
        text={wallet.name}
        currentlySelected={wallet.name === selectedWallet.name}
        {loadingWallet}
      />
    </li>
  {/each}
  <!--{#if modalData.secondaryWallets && modalData.secondaryWallets.length && !showingAllWalletModules}-->
  <!--  <div>-->
  <!--    <Button disabled={walletsDisabled} onclick={showAllWallets}-->
  <!--    >Show More</Button-->
  <!--    >-->
  <!--  </div>-->
  <!--{/if}-->
  <!--{#if showingAllWalletModules}-->
  <!--  {#each modalData.secondaryWallets as wallet, i (wallet.name)}-->
  <!--    <li>-->
  <!--      <IconButton-->
  <!--        disabled={walletsDisabled}-->
  <!--        onclick={() => handleWalletSelect(wallet)}-->
  <!--        iconSrc={wallet.iconSrc}-->
  <!--        iconSrcSet={wallet.iconSrcSet}-->
  <!--        svg={wallet.svg}-->
  <!--        text={wallet.name}-->
  <!--        currentlySelected={wallet.name === selectedWallet.name}-->
  <!--        {loadingWallet}-->
  <!--      />-->
  <!--    </li>-->
  <!--  {/each}-->
  <!--{/if}-->
</ul>
<div class="border-bottom" />
