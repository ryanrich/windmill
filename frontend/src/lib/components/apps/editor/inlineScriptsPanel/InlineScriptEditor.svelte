<script lang="ts">
	import Button from '$lib/components/common/button/Button.svelte'
	import type { Preview } from '$lib/gen'
	import { createEventDispatcher, getContext, onMount } from 'svelte'
	import type { AppViewerContext, CancelablePromise, InlineScript } from '../../types'
	import { CornerDownLeft, Loader2, Maximize2, Trash2 } from 'lucide-svelte'
	import InlineScriptEditorDrawer from './InlineScriptEditorDrawer.svelte'
	import { inferArgs } from '$lib/infer'
	import type { Schema } from '$lib/common'
	import Badge from '$lib/components/common/badge/Badge.svelte'
	import Editor from '$lib/components/Editor.svelte'
	import { emptySchema, getModifierKey, scriptLangToEditorLang } from '$lib/utils'
	import Popover from '../../../Popover.svelte'
	import { computeFields } from './utils'
	import { deepEqual } from 'fast-equals'
	import type { AppInput } from '../../inputType'
	import Kbd from '$lib/components/common/kbd/Kbd.svelte'
	import SimpleEditor from '$lib/components/SimpleEditor.svelte'
	import { buildExtraLib } from '../../utils'

	let inlineScriptEditorDrawer: InlineScriptEditorDrawer

	export let inlineScript: InlineScript | undefined
	export let name: string | undefined = undefined
	export let id: string
	export let defaultUserInput: boolean = false
	export let fields: Record<string, AppInput> = {}
	export let syncFields: boolean = false
	export let transformer: boolean = false

	const { runnableComponents, stateId, worldStore, state, appPath, app } =
		getContext<AppViewerContext>('AppViewerContext')

	let editor: Editor
	let simpleEditor: SimpleEditor
	let validCode = true

	async function inferInlineScriptSchema(
		language: Preview.language,
		content: string,
		schema: Schema
	): Promise<Schema> {
		try {
			await inferArgs(language, content, schema)
			validCode = true
		} catch (e) {
			console.error("Couldn't infer args", e)
			validCode = false
		}

		return schema
	}

	$: inlineScript && (inlineScript.path = `${appPath}/${name}`)

	onMount(async () => {
		if (inlineScript && !inlineScript.schema) {
			if (inlineScript.language != 'frontend') {
				inlineScript.schema = await inferInlineScriptSchema(
					inlineScript?.language,
					inlineScript?.content,
					emptySchema()
				)
			}
		}
		if (inlineScript?.schema && inlineScript.language != 'frontend') {
			loadSchemaAndInputsByName()
		}
	})

	const dispatch = createEventDispatcher()
	let runLoading = false

	async function loadSchemaAndInputsByName() {
		if (syncFields && inlineScript) {
			const newSchema = inlineScript.schema ?? emptySchema()
			const newFields = computeFields(newSchema, defaultUserInput, fields)

			if (!deepEqual(newFields, fields)) {
				fields = newFields
				$stateId++
			}
		}
	}

	$: extraLib =
		inlineScript?.language == 'frontend' && worldStore
			? buildExtraLib($worldStore?.outputsById ?? {}, id, false, $state, true)
			: undefined

	let cancelable: CancelablePromise<void> | undefined = undefined
</script>

{#if inlineScript}
	{#if inlineScript.language != 'frontend'}
		<InlineScriptEditorDrawer {editor} bind:this={inlineScriptEditorDrawer} bind:inlineScript />
	{/if}

	<div class="h-full flex flex-col gap-1">
		<div class="flex justify-between w-full gap-2 px-2 pt-1 flex-row items-center">
			{#if name !== undefined}
				{#if !transformer}
					<input
						on:keydown|stopPropagation
						bind:value={name}
						placeholder="Inline script name"
						class="!text-xs !rounded-xs"
					/>
				{:else}
					<span class="text-xs font-semibold truncate w-full">{name} of {id}</span>
				{/if}
			{/if}
			<div class="flex w-full flex-row gap-2 items-center justify-end">
				{#if validCode}
					<Badge color="green" baseClass="!text-2xs">Valid</Badge>
				{:else}
					<Badge color="red" baseClass="!text-2xs">Invalid</Badge>
				{/if}

				{#if id.startsWith('unused-') || id.startsWith('bg_')}
					<Popover notClickable placement="bottom">
						<Button
							size="xs"
							color="light"
							btnClasses="!px-2 !bg-red-100 hover:!bg-red-200"
							aria-label="Delete"
							on:click={() => dispatch('delete')}
						>
							<Trash2 size={14} class="text-red-800" />
						</Button>
						<svelte:fragment slot="text">Delete</svelte:fragment>
					</Popover>
				{/if}
				{#if inlineScript.language != 'frontend'}
					<Popover notClickable placement="bottom">
						<Button
							size="xs"
							color="light"
							btnClasses="!px-2 !bg-gray-100 hover:!bg-gray-200"
							aria-label="Open full editor"
							on:click={() => {
								inlineScriptEditorDrawer?.openDrawer()
							}}
						>
							Full Editor&nbsp;<Maximize2 size={14} />
						</Button>
						<svelte:fragment slot="text">Open full editor</svelte:fragment>
					</Popover>
				{/if}

				<Button
					variant="border"
					size="xs"
					color="light"
					btnClasses="!px-2 !py-1"
					on:click={async () => {
						editor?.format()
						simpleEditor?.format()
					}}
				>
					<div class="flex flex-row gap-1 items-center">
						Format

						<div class="flex flex-row items-center gap-1">
							<Kbd>{getModifierKey()}</Kbd>
							<Kbd>S</Kbd>
						</div>
					</div>
				</Button>
				{#if $runnableComponents[id] != undefined}
					{#if !runLoading}
						<Button
							loading={runLoading}
							size="xs"
							color="dark"
							variant="border"
							btnClasses="!px-2 !py-1 !bg-gray-700 !text-white hover:!bg-gray-900"
							on:click={async () => {
								runLoading = true
								try {
									cancelable = $runnableComponents[id]?.cb?.(
										!transformer ? inlineScript : undefined
									)
									await cancelable
								} catch {}
								runLoading = false
							}}
						>
							<div class="flex flex-row gap-1 items-center">
								Run

								<div class="flex flex-row items-center gap-1">
									<Kbd>{getModifierKey()}</Kbd>
									<Kbd>
										<div class="h-4 flex items-center justify-center">
											<CornerDownLeft size={10} />
										</div>
									</Kbd>
								</div>
							</div>
						</Button>
					{:else}
						<Button
							size="xs"
							color="red"
							variant="border"
							btnClasses="!px-2 !py-1.5"
							on:click={async () => {
								cancelable?.cancel()
								runLoading = false
							}}
						>
							<Loader2 size={14} class="animate-spin mr-2" />
							Cancel
						</Button>
					{/if}
				{/if}
			</div>
		</div>

		<!-- {inlineScript.content} -->

		<div class="border h-full">
			{#if inlineScript.language != 'frontend'}
				<Editor
					path={inlineScript.path}
					bind:this={editor}
					class="flex flex-1 grow h-full"
					lang={scriptLangToEditorLang(inlineScript?.language)}
					bind:code={inlineScript.content}
					fixedOverflowWidgets={true}
					cmdEnterAction={async () => {
						if (inlineScript) {
							inlineScript.content = editor?.getCode() ?? ''
						}
						runLoading = true
						await $runnableComponents[id]?.cb?.(inlineScript)
						runLoading = false
					}}
					on:change={async (e) => {
						if (inlineScript && inlineScript.language != 'frontend') {
							const oldSchema = JSON.stringify(inlineScript.schema)
							if (inlineScript.schema == undefined) {
								inlineScript.schema = emptySchema()
							}
							await inferInlineScriptSchema(inlineScript?.language, e.detail, inlineScript.schema)
							if (JSON.stringify(inlineScript.schema) != oldSchema) {
								inlineScript = inlineScript
								loadSchemaAndInputsByName()
							}
						}
						$app = $app
					}}
				/>
			{:else}
				<SimpleEditor
					bind:this={simpleEditor}
					class="h-full"
					{extraLib}
					bind:code={inlineScript.content}
					lang="javascript"
					cmdEnterAction={async () => {
						runLoading = true
						await $runnableComponents[id]?.cb?.(!transformer ? inlineScript : undefined)
						runLoading = false
					}}
					on:change={() => {
						$app = $app
					}}
				/>
			{/if}
		</div>
	</div>
{/if}
