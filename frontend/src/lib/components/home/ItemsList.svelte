<script lang="ts">
	import CenteredPage from '$lib/components/CenteredPage.svelte'
	import { Alert, Badge, Skeleton } from '$lib/components/common'
	import ShareModal from '$lib/components/ShareModal.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import { AppService, FlowService, ListableApp, Script, ScriptService, type Flow } from '$lib/gen'
	import { userStore, workspaceStore } from '$lib/stores'
	import { canWrite } from '$lib/utils'
	import type uFuzzy from '@leeoniya/ufuzzy'
	import { Code2, LayoutDashboard } from 'lucide-svelte'

	export let filter = ''

	import AppRow from '$lib/components/common/table/AppRow.svelte'
	import FlowRow from '$lib/components/common/table/FlowRow.svelte'
	import ScriptRow from '$lib/components/common/table/ScriptRow.svelte'

	import { HOME_SEARCH_SHOW_FLOW, HOME_SEARCH_PLACEHOLDER } from '$lib/consts'

	import ConfirmationModal from '../common/confirmationModal/ConfirmationModal.svelte'
	import MoveDrawer from '../MoveDrawer.svelte'
	import SearchItems from '../SearchItems.svelte'
	import ListFilters from './ListFilters.svelte'
	import NoItemFound from './NoItemFound.svelte'
	import ToggleButtonGroup from '../common/toggleButton-v2/ToggleButtonGroup.svelte'
	import ToggleButton from '../common/toggleButton-v2/ToggleButton.svelte'
	import FlowIcon from './FlowIcon.svelte'

	type TableItem<T, U extends 'script' | 'flow' | 'app'> = T & {
		canWrite: boolean
		marked?: string
		type?: U
		time?: number
		starred?: boolean
	}

	type TableScript = TableItem<Script, 'script'>
	type TableFlow = TableItem<Flow, 'flow'>
	type TableApp = TableItem<ListableApp, 'app'>

	let scripts: TableScript[] | undefined
	let flows: TableFlow[] | undefined
	let apps: TableApp[] | undefined

	let filteredItems: (TableScript | TableFlow | TableApp)[] = []

	let itemKind: 'script' | 'flow' | 'app' | 'all' = 'all'

	let shareModal: ShareModal
	let moveDrawer: MoveDrawer

	let loading = true

	let nbDisplayed = 30

	export let deleteConfirmedCallback: (() => void) | undefined = undefined

	async function loadScripts(): Promise<void> {
		const loadedScripts = await ScriptService.listScripts({
			workspace: $workspaceStore!,
			showArchived: archived ? true : undefined,
			perPage: 300
		})

		scripts = loadedScripts.map((script: Script) => {
			return {
				canWrite:
					canWrite(script.path, script.extra_perms, $userStore) &&
					script.workspace_id == $workspaceStore &&
					!$userStore?.operator,
				...script
			}
		})
		loading = false
	}

	async function loadFlows(): Promise<void> {
		flows = (
			await FlowService.listFlows({
				workspace: $workspaceStore!,
				showArchived: archived ? true : undefined
			})
		).map((x: Flow) => {
			return {
				canWrite:
					canWrite(x.path, x.extra_perms, $userStore) &&
					x.workspace_id == $workspaceStore &&
					!$userStore?.operator,
				...x
			}
		})
		loading = false
	}

	async function loadApps(): Promise<void> {
		apps = (await AppService.listApps({ workspace: $workspaceStore! })).map((app: ListableApp) => {
			return {
				canWrite:
					canWrite(app.path!, app.extra_perms!, $userStore) &&
					app.workspace_id == $workspaceStore &&
					!$userStore?.operator,
				...app
			}
		})
		loading = false
	}

	$: owners = Array.from(
		new Set(filteredItems?.map((x) => x.path.split('/').slice(0, 2).join('/')) ?? [])
	).sort()

	let combinedItems: (TableScript | TableFlow | TableApp)[] | undefined = undefined

	$: combinedItems =
		flows == undefined || scripts == undefined || apps == undefined
			? undefined
			: [
					...flows.map((x) => ({
						...x,
						type: 'flow' as 'flow',
						time: new Date(x.edited_at).getTime()
					})),
					...scripts.map((x) => ({
						...x,
						type: 'script' as 'script',
						time: new Date(x.created_at).getTime()
					})),
					...apps.map((x) => ({
						...x,
						type: 'app' as 'app',
						time: new Date(x.edited_at).getTime()
					}))
			  ].sort((a, b) =>
					a.starred != b.starred ? (a.starred ? -1 : 1) : a.time - b.time > 0 ? -1 : 1
			  )

	$: preFilteredItems =
		ownerFilter != undefined
			? combinedItems?.filter(
					(x) => x.path.startsWith(ownerFilter ?? '') && (x.type == itemKind || itemKind == 'all')
			  )
			: combinedItems?.filter((x) => x.type == itemKind || itemKind == 'all')

	let ownerFilter: string | undefined = undefined

	$: if ($workspaceStore) {
		ownerFilter = undefined
	}

	$: {
		if ($userStore && $workspaceStore) {
			loadScripts()
			loadFlows()
			if (!archived) {
				loadApps()
			} else {
				apps = []
			}
		}
	}

	const cmp = new Intl.Collator('en').compare

	const opts: uFuzzy.Options = {
		sort: (info, haystack, needle) => {
			let {
				idx,
				chars,
				terms,
				interLft2,
				interLft1,
				//	interRgt2,
				//	interRgt1,
				start,
				intraIns,
				interIns
			} = info

			const sortResult = idx
				.map((v, i) => i)
				.sort(
					(ia, ib) =>
						// most contig chars matched
						chars[ib] - chars[ia] ||
						// least char intra-fuzz (most contiguous)
						intraIns[ia] - intraIns[ib] ||
						// most prefix bounds, boosted by full term matches
						terms[ib] +
							interLft2[ib] +
							0.5 * interLft1[ib] -
							(terms[ia] + interLft2[ia] + 0.5 * interLft1[ia]) ||
						// highest density of match (least span)
						//	span[ia] - span[ib] ||
						// highest density of match (least term inter-fuzz)
						interIns[ia] - interIns[ib] ||
						// earliest start of match
						start[ia] - start[ib] ||
						// alphabetic
						cmp(haystack[idx[ia]], haystack[idx[ib]]) +
							(preFilteredItems?.[idx[ib]]?.starred ? 100 : 0) -
							(preFilteredItems?.[idx[ia]]?.starred ? 100 : 0)
				)
			return sortResult
		}
	}

	$: items = filter !== '' ? filteredItems : preFilteredItems

	function resetScroll() {
		const element = document.getElementsByTagName('svelte-virtual-list-viewport')
		const firstElement = element.item(0)
		if (firstElement) {
			firstElement.scrollTop = 0
		}
	}

	$: items && resetScroll()

	let archived = false
</script>

<SearchItems
	{filter}
	items={preFilteredItems}
	bind:filteredItems
	f={(x) => (x.summary ? x.summary + ' (' + x.path + ')' : x.path)}
	{opts}
/>

<ShareModal
	bind:this={shareModal}
	on:change={() => {
		loadScripts()
		loadApps()
		loadFlows()
	}}
/>

<MoveDrawer
	bind:this={moveDrawer}
	on:update={() => {
		loadScripts()
		loadApps()
		loadFlows()
	}}
/>

<CenteredPage>
	<div class="flex flex-wrap gap-2 items-center justify-between w-full">
		<div class="flex justify-start">
			<ToggleButtonGroup bind:selected={itemKind} class="h-10">
				<ToggleButton value="all" label="All" class="text-sm px-4 py-2" />
				<ToggleButton value="script" icon={Code2} label="Scripts" class="text-sm px-4 py-2" />
				{#if HOME_SEARCH_SHOW_FLOW}
					<ToggleButton
						value="flow"
						label="Flows"
						icon={FlowIcon}
						class="text-sm px-4 py-2"
						selectedColor="#14b8a6"
					/>
				{/if}
				<ToggleButton
					value="app"
					label="Apps"
					icon={LayoutDashboard}
					class="text-sm px-4 py-2"
					selectedColor="#fb923c"
				/>
			</ToggleButtonGroup>
		</div>

		<div class="relative text-gray-600 grow min-w-[100px]">
			<!-- svelte-ignore a11y-autofocus -->
			<input
				autofocus
				placeholder={HOME_SEARCH_PLACEHOLDER}
				bind:value={filter}
				class="bg-white !h-10 !px-4 !pr-10 !rounded-lg text-sm focus:outline-none"
			/>
			<button type="submit" class="absolute right-0 top-0 mt-3 mr-4">
				<svg
					class="h-4 w-4 fill-current"
					xmlns="http://www.w3.org/2000/svg"
					xmlns:xlink="http://www.w3.org/1999/xlink"
					version="1.1"
					id="Capa_1"
					x="0px"
					y="0px"
					viewBox="0 0 56.966 56.966"
					style="enable-background:new 0 0 56.966 56.966;"
					xml:space="preserve"
					width="512px"
					height="512px"
				>
					<path
						d="M55.146,51.887L41.588,37.786c3.486-4.144,5.396-9.358,5.396-14.786c0-12.682-10.318-23-23-23s-23,10.318-23,23  s10.318,23,23,23c4.761,0,9.298-1.436,13.177-4.162l13.661,14.208c0.571,0.593,1.339,0.92,2.162,0.92  c0.779,0,1.518-0.297,2.079-0.837C56.255,54.982,56.293,53.08,55.146,51.887z M23.984,6c9.374,0,17,7.626,17,17s-7.626,17-17,17  s-17-7.626-17-17S14.61,6,23.984,6z"
					/>
				</svg>
			</button>
		</div>
	</div>
	<div class="relative">
		<ListFilters bind:selectedFilter={ownerFilter} filters={owners} />
		{#if filteredItems?.length == 0}
			<div class="mt-10" />
		{/if}
		{#if !loading}
			<div class="absolute -bottom-2 right-0 bg-white/90">
				<Toggle size="xs" bind:checked={archived} options={{ right: 'Show archived' }} /></div
			>
		{/if}
	</div>
	<div>
		{#if filteredItems == undefined}
			<div class="mt-4" />
			<Skeleton layout={[[2], 1]} />
			{#each new Array(6) as _}
				<Skeleton layout={[[4], 0.5]} />
			{/each}
		{:else if filteredItems.length === 0}
			<NoItemFound />
		{:else}
			<div class="border rounded-md divide-y divide-gray-200">
				<!-- <VirtualList {items} let:item bind:start bind:end> -->
				{#each (items ?? []).slice(0, nbDisplayed) as item (item.type + '/' + item.path)}
					{#if item.type == 'script'}
						<ScriptRow
							bind:deleteConfirmedCallback
							starred={item.starred ?? false}
							marked={item.marked}
							on:change={loadScripts}
							script={item}
							{shareModal}
							{moveDrawer}
						/>
					{:else if item.type == 'flow'}
						<FlowRow
							bind:deleteConfirmedCallback
							starred={item.starred ?? false}
							marked={item.marked}
							on:change={loadFlows}
							flow={item}
							{shareModal}
							{moveDrawer}
						/>
					{:else if item.type == 'app'}
						<AppRow
							bind:deleteConfirmedCallback
							starred={item.starred ?? false}
							marked={item.marked}
							on:change={loadApps}
							app={item}
							{moveDrawer}
							{shareModal}
						/>
					{/if}
				{/each}
				<!-- </VirtualList> -->
			</div>
			{#if items && items?.length > 30}
				<span class="text-xs"
					>{nbDisplayed} items out of {items.length}
					<button class="ml-4" on:click={() => (nbDisplayed += 30)}>load 30 more</button></span
				>
			{/if}
			<div class="pb-80" />
		{/if}
	</div>
</CenteredPage>

<ConfirmationModal
	open={Boolean(deleteConfirmedCallback)}
	title="Remove"
	confirmationText="Remove"
	on:canceled={() => {
		deleteConfirmedCallback = undefined
	}}
	on:confirmed={() => {
		if (deleteConfirmedCallback) {
			deleteConfirmedCallback()
		}
		deleteConfirmedCallback = undefined
	}}
>
	<div class="flex flex-col w-full space-y-4">
		<span>Are you sure you want to remove it?</span>
		<Alert type="info" title="Bypass confirmation">
			<div>
				You can press
				<Badge color="dark-gray">SHIFT</Badge>
				while removing to bypass confirmation.
			</div>
		</Alert>
	</div>
</ConfirmationModal>
