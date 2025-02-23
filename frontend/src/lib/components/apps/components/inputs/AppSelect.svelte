<script lang="ts">
	import { getContext } from 'svelte'
	import Select from 'svelte-select'

	import type { AppViewerContext, ComponentCustomCSS, RichConfigurations } from '../../types'
	import { concatCustomCss } from '../../utils'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import InputValue from '../helpers/InputValue.svelte'
	import { SELECT_INPUT_DEFAULT_STYLE } from '../../../../defaults'
	import { initOutput } from '../../editor/appUtils'
	import InitializeComponent from '../helpers/InitializeComponent.svelte'

	export let id: string
	export let configuration: RichConfigurations
	export let horizontalAlignment: 'left' | 'center' | 'right' | undefined = undefined
	export let verticalAlignment: 'top' | 'center' | 'bottom' | undefined = undefined
	export let customCss: ComponentCustomCSS<'selectcomponent'> | undefined = undefined
	export let render: boolean

	const { app, worldStore, connectingInput, selectedComponent } =
		getContext<AppViewerContext>('AppViewerContext')
	let items: { label: string; value: any; created?: boolean }[]
	let placeholder: string = 'Select an item'

	let outputs = initOutput($worldStore, id, {
		result: undefined as string | undefined
	})

	let value: string | undefined = outputs?.result.peak()

	$: items && handleItems()

	let listItems: { label: string; value: string; created?: boolean }[] = []

	function handleItems() {
		listItems = Array.isArray(items)
			? items.map((item) => {
					return {
						label: item.label,
						value: JSON.stringify(item.value)
					}
			  })
			: []
		let rawValue
		if (defaultValue !== undefined) {
			rawValue = defaultValue
		} else if (listItems.length > 0) {
			rawValue = items[0].value
		}
		if (rawValue !== undefined) {
			value = JSON.stringify(rawValue)
			outputs?.result.set(rawValue)
		}
	}

	function onChange(e: CustomEvent) {
		e?.stopPropagation()

		if (create) {
			listItems = listItems.map((i) => {
				delete i.created
				return i
			})
		}

		let result: any = undefined
		try {
			result = JSON.parse(e.detail?.['value'])
		} catch (_) {}
		value = e.detail?.['value']
		outputs?.result.set(result)
	}

	$: css = concatCustomCss($app.css?.selectcomponent, customCss)

	let defaultValue: any = undefined

	function handleFilter(e) {
		if (create) {
			if (e.detail.length === 0 && filterText.length > 0) {
				const prev = listItems.filter((i) => !i.created)
				listItems = [
					...prev,
					{ value: JSON.stringify(filterText), label: filterText, created: true }
				]
			}
		}
	}

	$: defaultValue && handleDefault()

	function handleDefault() {
		if (defaultValue) {
			value = JSON.stringify(defaultValue)
			outputs?.result.set(defaultValue)
		}
	}
	let create = false
	let filterText = ''
</script>

<InputValue {id} input={configuration.items} bind:value={items} />
<InputValue {id} input={configuration.placeholder} bind:value={placeholder} />
<InputValue {id} input={configuration.defaultValue} bind:value={defaultValue} />
<InputValue {id} input={configuration.create} bind:value={create} />

<InitializeComponent {id} />

<AlignWrapper {render} {horizontalAlignment} {verticalAlignment}>
	<div
		class="app-select w-full"
		style="height: 34px;"
		on:pointerdown={(e) => {
			if (!e.shiftKey) {
				e.stopPropagation()
			}
		}}
	>
		<Select
			--border-radius="0"
			--border-color="#999"
			bind:filterText
			on:filter={handleFilter}
			on:clear={onChange}
			on:change={onChange}
			items={listItems}
			inputStyles={SELECT_INPUT_DEFAULT_STYLE.inputStyles}
			containerStyles={'border-color: #999;' +
				SELECT_INPUT_DEFAULT_STYLE.containerStyles +
				css?.input?.style}
			{value}
			{placeholder}
			on:focus={() => {
				if (!$connectingInput.opened) {
					$selectedComponent = [id]
				}
			}}
		>
			<div slot="item" let:item>
				{#if create}
					{item.created ? 'Add new: ' : ''}
				{/if}
				{item.label}
			</div>
		</Select>
	</div>
</AlignWrapper>

<style global>
	.app-select .value-container {
		padding: 0 !important;
	}
	.svelte-select-list {
		z-index: 1000 !important;
	}
</style>
