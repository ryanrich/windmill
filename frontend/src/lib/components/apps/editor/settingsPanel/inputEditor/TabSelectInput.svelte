<script lang="ts">
	import type { StaticInput } from '$lib/components/apps/inputType'
	import type { AppViewerContext } from '$lib/components/apps/types'
	import { allItems } from '$lib/components/apps/utils'
	import { getContext } from 'svelte'

	export let componentInput: StaticInput<{ id: string; index: number }>

	const { app } = getContext<AppViewerContext>('AppViewerContext')

	const tabComponents = allItems($app.grid, $app.subgrids).filter(
		(component) => component.data.type === 'tabscomponent'
	)
</script>

{#if componentInput.value && tabComponents.length > 0}
	<div>
		<div class="flex flex-row gap-2 w-full">
			<div class="flex flex-col">
				<label for="tabId" class="text-xs font-semibold">Tab ID</label>

				<select
					id="tabId"
					bind:value={componentInput.value.id}
					class="border border-gray-300 rounded-md p-1 !w-16"
				>
					{#each tabComponents as tabComponent}
						<option value={tabComponent.data.id}>
							{tabComponent.data.id}
						</option>
					{/each}
				</select>
			</div>
			<div class="flex flex-col">
				<label for="tabIndex" class="text-xs font-semibold">Tab Index</label>
				<select
					id="tabIndex"
					bind:value={componentInput.value.index}
					class="border border-gray-300 rounded-md p-1 !w-16"
				>
					{#each tabComponents as tabComponent}
						{#if tabComponent.data.id === componentInput.value.id}
							{#each Array(tabComponent.data.numberOfSubgrids) as _, i}
								<option value={i}>{i}</option>
							{/each}
						{/if}
					{/each}
				</select>
			</div>
		</div>
	</div>
{:else}
	<div class="text-xs text-gray-500"> No tab component found in the app </div>
{/if}
