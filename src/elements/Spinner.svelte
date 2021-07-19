<script lang="ts">
  import Button from '../elements/Button.svelte'
  export let description: string = ''
</script>

<style>
  .bn-onboard-loading-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    font-family: inherit;
    font-size: inherit;
    color: inherit;
  }
  span {
    font-family: inherit;
    font-size: 0.889em;
    margin-top: 1rem;
  }
  :global(.bn-onboard-get-help) {
    color: inherit;
    border-color: #3da8f5;
    border-radius: 20px;
    background: #3da8f5;
    padding: 0 2em;
    height: 28px;
    text-align: center;
  }

  :global(.bn-onboard-loading) {
    display: inline-block;
    position: relative;
    min-width: 100px;
    height: 55px;
  }
  :global(.bn-onboard-loading) :global(div) {
    box-sizing: border-box;
    font-size: inherit;
    display: block;
    position: absolute;
    min-width: 100px;
    height: 55px;
    border: 3px solid;
    border-radius: 50%;
    animation: bn-onboard-loading 1.2s cubic-bezier(0.5, 0, 0.5, 1) infinite;
    border-color: currentColor transparent transparent transparent;
  }
</style>

<div class="bn-onboard-loading-container">
  <div class="bn-onboard-custom bn-onboard-loading">
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
  {#if description}
    <span>{description}</span>
  {/if}
</div>
