<script lang="ts">
	import Button from '$lib/components/common/button/Button.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import Tooltip from '$lib/components/Tooltip.svelte'
	import { faClose } from '@fortawesome/free-solid-svg-icons'
	import { getContext } from 'svelte'
	import type { ResultAppInput } from '../../inputType'
	import type { AppEditorContext, AppViewerContext } from '../../types'
	import { clearResultAppInput } from '../../utils'
	import type { AppComponent } from '../component'
	import ComponentTriggerList from './triggerLists/ComponentTriggerList.svelte'

	const { app } = getContext<AppViewerContext>('AppViewerContext')
	const { selectedComponentInEditor } = getContext<AppEditorContext>('AppEditorContext')

	export let appInput: ResultAppInput
	export let appComponent: AppComponent

	function detach() {
		if (appInput.runnable?.type === 'runnableByName' && appInput.runnable.inlineScript) {
			$app.unusedInlineScripts.push({
				name: appInput.runnable.name,
				inlineScript: appInput.runnable.inlineScript
			})
			$app = $app
			appInput = clearResultAppInput(appInput)
		}
	}

	function clear() {
		appInput = clearResultAppInput(appInput)
	}

	$: {
		if (appInput.recomputeOnInputChanged === undefined) {
			appInput.recomputeOnInputChanged = true
		}
		//TODO: remove after migration is done
		if (appInput.doNotRecomputeOnInputChanged != undefined) {
			appInput.recomputeOnInputChanged = !appInput.doNotRecomputeOnInputChanged
			appInput.doNotRecomputeOnInputChanged = undefined
		}
	}
</script>

<div class="flex justify-between w-full items-center gap-1">
	<span class="text-xs font-semibold truncate">
		{#if appInput.runnable?.type === 'runnableByName'}
			{appInput.runnable.name}
		{:else if appInput.runnable?.type === 'runnableByPath'}
			{appInput.runnable.path}
		{/if}
	</span>
	<div class="flex gap-1 justify-center">
		{#if appInput.runnable?.type === 'runnableByName' && appInput.runnable.inlineScript}
			<Button size="xs" color="light" variant="border" on:click={detach}>
				Detach&nbsp;
				<Tooltip>
					Detaching an inline script keep it for later to be reused by another component
				</Tooltip>
			</Button>
		{/if}
		<Button size="xs" color="red" variant="border" startIcon={{ icon: faClose }} on:click={clear}>
			Clear
		</Button>
	</div>
</div>
<div>
	<div class="w-full"><div class="mx-auto w-0">&downarrow;</div></div>
	<div class="flex gap-1 justify-between items-center">
		<span class="text-xs font-semibold truncate">
			Transformer &nbsp;
			<Tooltip>
				A transformer is an optional frontend script that is executed right after the component's
				script whose purpose is to do lightweight transformation in the browser. It takes the
				previous computation's result as `result`
			</Tooltip>
		</span>
		<div class="flex gap-1">
			{#if !appInput.transformer}
				<div>
					<Button
						variant="border"
						color="light"
						size="xs"
						on:click={() => {
							appInput.transformer = {
								language: 'frontend',
								content: 'return result'
							}
							$selectedComponentInEditor = appComponent.id + '_transformer'
						}}
					>
						Add Transformer
					</Button>
				</div>
			{:else}
				<Button
					size="xs"
					color="red"
					variant="border"
					startIcon={{ icon: faClose }}
					on:click={() => {
						appInput.transformer = undefined
					}}
				>
					Clear
				</Button>
			{/if}
		</div></div
	>
</div>

{#if !['buttoncomponent', 'formbuttoncomponent', 'formcomponent'].includes(appComponent.type)}
	<div class="flex items-center">
		<Toggle
			size="xs"
			bind:checked={appInput.recomputeOnInputChanged}
			options={{ right: 'recompute on any input changes' }}
		/>
	</div>
{/if}
{#if appInput.runnable?.type === 'runnableByName'}
	<div>
		<ComponentTriggerList
			bind:runnable={appInput.runnable}
			{appComponent}
			fields={appInput.fields}
			recomputeOnInputChanged={appInput.recomputeOnInputChanged}
		/>
	</div>
{/if}
{#if appInput.runnable?.type === 'runnableByName' && !appInput.runnable.inlineScript}
	<span class="text-xs text-gray-500">
		Please configure the language in the inline script panel
	</span>
{/if}
