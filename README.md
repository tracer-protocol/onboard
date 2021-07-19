# Onboard

JavaScript library to easily onboard users to ethereum apps by enabling wallet selection, connection, wallet checks and real time state updates.

## Install

`npm install bnc-onboard`

## Quick Start

```javascript
import Onboard from 'bnc-onboard'
import Web3 from 'web3'

// set a variable to store instantiated web3
let web3

// head to blocknative.com to create a key
const BLOCKNATIVE_KEY = 'blocknative-api-key'

// the network id that your dapp runs on
const NETWORK_ID = 1

// initialize onboard
const onboard = Onboard({
  dappId: BLOCKNATIVE_KEY,
  networkId: NETWORK_ID,
  subscriptions: {
    wallet: wallet => {
      // instantiate web3 when the user has selected a wallet
      web3 = new Web3(wallet.provider)
      console.log(`${wallet.name} connected!`)
    }
  }
})

// Prompt user to select a wallet
await onboard.walletSelect()

// Run wallet checks to make sure that user is ready to transact
await onboard.walletCheck()
```

## Documentation

For detailed documentation head to [docs.blocknative.com](https://docs.blocknative.com/onboard)


## Updates Made
Updates made from the original onboard package include
- addition of getHelpLink to both walletSelect props as well as walletCheck props
- styling of buttons
- styling of modal itself
- add custom loading icon instead of spinner


### Custom Colour Scheme Example
```
  /** BNC-Onboarding styles */
    .bn-onboard-modal-content {
        background: var(--color-background-secondary)!important;
        color: var(--color-text)!important;
    }

    /** Header Styles */
    .bn-onboard-modal-content-header h3 {
        color: var(--color-text)!important;
    }

    /** Wallet Styles */
    .bn-onboard-modal-select-wallets li {
        .bn-onboard-selected-wallet {
            background: var(--color-background)!important;
        }
    }
    .bn-onboard-modal-select-wallets li:hover {
        background: var(--color-accent);
    }

    /** Close Styles */
    .bn-onboard-modal-content-close {
        border: 1px solid var(--color-primary);
        svg {
            fill: var(--color-primary)!important;
        }
    }

    /** Loadinng */
    .bn-onboard-loading {
        svg {
            fill: var(--color-text);
        }
    }
```