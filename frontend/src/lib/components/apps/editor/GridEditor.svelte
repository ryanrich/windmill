<script lang="ts">
	import { getContext } from 'svelte'
	import type { AppEditorContext, AppViewerContext } from '../types'
	import { columnConfiguration, isFixed, toggleFixed } from '../gridUtils'
	import { twMerge } from 'tailwind-merge'

	import RecomputeAllComponents from './RecomputeAllComponents.svelte'
	import type { Policy } from '$lib/gen'
	import HiddenComponent from '../components/helpers/HiddenComponent.svelte'
	import Component from './component/Component.svelte'
	import { push } from '$lib/history'
	import { dfs, expandGriditem, findGridItem } from './appUtils'
	import Grid from '../svelte-grid/Grid.svelte'
	import { deepEqual } from 'fast-equals'
	import ComponentWrapper from './component/ComponentWrapper.svelte'
	import { classNames } from '$lib/utils'

	export let policy: Policy

	const {
		selectedComponent,
		app,
		mode,
		connectingInput,
		summary,
		focusedGrid,
		parentWidth,
		breakpoint,
		allIdsInPath
	} = getContext<AppViewerContext>('AppViewerContext')

	const { history } = getContext<AppEditorContext>('AppEditorContext')

	let previousSelectedIds: string[] | undefined = undefined
	$: if (!deepEqual(previousSelectedIds, $selectedComponent)) {
		previousSelectedIds = $selectedComponent
		$allIdsInPath = ($selectedComponent ?? [])
			.flatMap((id) => dfs($app.grid, id, $app.subgrids ?? {}))
			.filter((x) => x != undefined) as string[]
	}
</script>

<div class="relative w-full z-20 overflow-visible">
	<div
		class="w-full sticky top-0 flex justify-between border-b {$connectingInput?.opened
			? ''
			: 'bg-gray-50 '} px-4 py-1 items-center gap-4"
		style="z-index: 1000;"
	>
		<h2 class="truncate">{$summary}</h2>
		{#if !$connectingInput.opened}
			<RecomputeAllComponents />
		{/if}
		<div class="flex text-2xs text-gray-600 gap-1 items-center">
			<div class="py-2 pr-2 text-gray-600 flex gap-2 items-center">
				Hide bar on view
				<input class="windmillapp" type="checkbox" bind:checked={$app.norefreshbar} />
			</div>
			<div>
				{policy.on_behalf_of ? `on behalf of ${policy.on_behalf_of_email}` : ''}
			</div>
		</div>
	</div>

	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<div
		style={$app.css?.['app']?.['grid']?.style}
		class={twMerge('px-4 pt-4 pb-2 overflow-visible', $app.css?.['app']?.['grid']?.class ?? '')}
		on:pointerdown={() => {
			$selectedComponent = undefined
			$focusedGrid = undefined
		}}
		bind:clientWidth={$parentWidth}
	>
		<div class={!$focusedGrid && $mode !== 'preview' ? 'border-gray-400 border border-dashed' : ''}>
			<Grid
				allIdsInPath={$allIdsInPath}
				selectedIds={$selectedComponent}
				items={$app.grid}
				on:redraw={(e) => {
					push(history, $app)
					$app.grid = e.detail
				}}
				let:dataItem
				rowHeight={36}
				cols={columnConfiguration}
				fastStart={true}
				gap={[4, 2]}
			>
				<ComponentWrapper
					id={dataItem.id}
					type={dataItem.data.type}
					class={classNames(
						'h-full w-full center-center',
						Boolean($selectedComponent?.includes(dataItem.id)) ? 'active-grid-item' : ''
					)}
				>
					<Component
						render={true}
						component={dataItem.data}
						selected={Boolean($selectedComponent?.includes(dataItem.id))}
						locked={isFixed(dataItem)}
						on:lock={() => {
							const gridItem = findGridItem($app, dataItem.id)
							if (gridItem) {
								toggleFixed(gridItem)
							}
							$app = $app
						}}
						on:expand={() => {
							push(history, $app)
							$selectedComponent = [dataItem.id]
							expandGriditem($app.grid, dataItem.id, $breakpoint)
							$app = $app
						}}
					/>
				</ComponentWrapper>
			</Grid>
		</div>
	</div>
</div>

{#if $app.hiddenInlineScripts}
	{#each $app.hiddenInlineScripts as script, index}
		{#if script}
			<HiddenComponent
				id={`bg_${index}`}
				inlineScript={script.inlineScript}
				name={script.name}
				fields={script.fields}
				recomputeOnInputChanged={script.recomputeOnInputChanged ?? true}
				recomputableByRefreshButton={script.autoRefresh ?? false}
			/>
		{/if}
	{/each}
{/if}

<style global>
	.svlt-grid-shadow {
		/* Back shadow */
		background: #93c4fdd0 !important;
	}
	.svlt-grid-active {
		opacity: 1 !important;
	}
</style>
