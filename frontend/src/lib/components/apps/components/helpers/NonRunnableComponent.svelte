<script lang="ts">
	import { getContext } from 'svelte'
	import type { AppInput } from '../../inputType'
	import type { Output } from '../../rx'
	import type { AppViewerContext } from '../../types'
	import InputValue from './InputValue.svelte'
	import InitializeComponent from './InitializeComponent.svelte'

	export let componentInput: AppInput
	export let id: string
	export let result: any
	export let render: boolean

	// Sync the result to the output
	const { worldStore } = getContext<AppViewerContext>('AppViewerContext')

	$: outputs = $worldStore?.outputsById?.[id] as {
		loading: Output<boolean>
		result: Output<any>
	}

	function setOutput(v: any) {
		outputs?.result?.set(v, true)
	}

	$: result && outputs && setOutput(result)
</script>

<InitializeComponent {id} />

{#if componentInput.type !== 'runnable'}
	<InputValue {id} input={componentInput} bind:value={result} />
{/if}

{#if render}
	<slot />
{:else}
	<div class="w-full h-full" />
{/if}
