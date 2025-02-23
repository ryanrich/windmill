<script lang="ts">
	import { Button } from '$lib/components/common'
	import { faUser } from '@fortawesome/free-solid-svg-icons'
	import { getContext } from 'svelte'
	import { Icon } from 'svelte-awesome'
	import { initConfig, initOutput } from '../../editor/appUtils'
	import { components } from '../../editor/component'
	import type { AppInput } from '../../inputType'
	import type { AppViewerContext, ComponentCustomCSS, RichConfigurations } from '../../types'
	import { concatCustomCss } from '../../utils'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import ResolveConfig from '../helpers/ResolveConfig.svelte'
	import type RunnableComponent from '../helpers/RunnableComponent.svelte'
	import RunnableWrapper from '../helpers/RunnableWrapper.svelte'

	export let id: string
	export let componentInput: AppInput | undefined
	export let configuration: RichConfigurations
	export let recomputeIds: string[] | undefined = undefined
	export let extraQueryParams: Record<string, any> = {}
	export let horizontalAlignment: 'left' | 'center' | 'right' | undefined = undefined
	export let customCss: ComponentCustomCSS<'formcomponent'> | undefined = undefined
	export let render: boolean

	export const staticOutputs: string[] = ['loading', 'result']

	const { app, worldStore, stateId } = getContext<AppViewerContext>('AppViewerContext')

	let outputs = initOutput($worldStore, id, {
		result: undefined,
		loading: false
	})

	let resolvedConfig = initConfig(
		components['formcomponent'].initialData.configuration,
		configuration
	)
	let runnableComponent: RunnableComponent

	let isLoading: boolean = false

	$: noInputs =
		$stateId != undefined &&
		(componentInput?.type != 'runnable' || Object.keys(componentInput?.fields ?? {}).length == 0)

	$: css = concatCustomCss($app.css?.formcomponent, customCss)

	let loading = false
</script>

{#each Object.keys(components['formcomponent'].initialData.configuration) as key (key)}
	<ResolveConfig
		{id}
		{key}
		bind:resolvedConfig={resolvedConfig[key]}
		configuration={configuration[key]}
	/>
{/each}

<RunnableWrapper
	{recomputeIds}
	{render}
	bind:runnableComponent
	bind:loading
	{componentInput}
	{id}
	doOnSuccess={resolvedConfig.onSuccess}
	{extraQueryParams}
	autoRefresh={false}
	forceSchemaDisplay={true}
	runnableClass="!block"
	runnableStyle={css?.container?.style}
	{outputs}
>
	<AlignWrapper {horizontalAlignment}>
		<div
			class="flex flex-col gap-2 px-4 w-full {css?.container?.class ?? ''}"
			style={css?.container?.style ?? ''}
		>
			<div>
				{#if noInputs}
					<div class="text-gray-600 italic text-sm my-4">
						Run forms are associated with a runnable that has user inputs.
						<br />
						Once a script or flow is chosen, set some <strong>Runnable Inputs</strong> to
						<strong>
							User Input
							<Icon data={faUser} scale={1.3} class="rounded-sm bg-gray-200 p-1 ml-0.5" />
						</strong>
					</div>
				{/if}
			</div>
			<div class="flex justify-end">
				{#if !noInputs}
					<Button
						loading={isLoading}
						btnClasses="my-1 {css?.button?.class ?? ''}"
						style={css?.button?.style ?? ''}
						on:pointerdown={(e) => {
							e?.stopPropagation()
						}}
						on:click={() => {
							runnableComponent?.runComponent()
						}}
						size={resolvedConfig.size}
						color={resolvedConfig.color}
					>
						{resolvedConfig.label}
					</Button>
				{/if}
			</div>
		</div>
	</AlignWrapper>
</RunnableWrapper>
