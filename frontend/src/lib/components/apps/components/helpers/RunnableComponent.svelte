<script lang="ts">
	import type { Schema } from '$lib/common'
	import Alert from '$lib/components/common/alert/Alert.svelte'
	import LightweightSchemaForm from '$lib/components/LightweightSchemaForm.svelte'
	import Popover from '$lib/components/Popover.svelte'
	import TestJobLoader from '$lib/components/TestJobLoader.svelte'
	import { AppService, type CompletedJob } from '$lib/gen'
	import { classNames, defaultIfEmptyString, emptySchema, sendUserToast } from '$lib/utils'
	import { deepEqual } from 'fast-equals'
	import { Bug } from 'lucide-svelte'
	import { createEventDispatcher, getContext, onDestroy } from 'svelte'
	import type { AppInputs, Runnable } from '../../inputType'
	import type { Output } from '../../rx'
	import type { AppViewerContext, CancelablePromise, InlineScript } from '../../types'
	import { computeGlobalContext, eval_like } from './eval'
	import InputValue from './InputValue.svelte'
	import RefreshButton from './RefreshButton.svelte'
	import { selectId } from '../../editor/appUtils'

	// Component props
	export let id: string
	export let fields: AppInputs
	export let runnable: Runnable
	export let transformer: (InlineScript & { language: 'frontend' }) | undefined
	export let extraQueryParams: Record<string, any> = {}
	export let autoRefresh: boolean = true
	export let result: any = undefined
	export let forceSchemaDisplay: boolean = false
	export let wrapperClass = ''
	export let wrapperStyle = ''
	export let initializing: boolean | undefined = undefined
	export let render: boolean
	export let outputs: { result: Output<any>; loading: Output<boolean> }
	export let extraKey = ''
	export let recomputeOnInputChanged: boolean = true
	export let loading = false
	export let recomputableByRefreshButton: boolean = true
	export let refreshOnStart: boolean = false

	const {
		worldStore,
		runnableComponents,
		workspace,
		appPath,
		isEditor,
		jobs,
		noBackend,
		errorByComponent,
		mode,
		stateId,
		state,
		componentControl,
		initialized,
		selectedComponent,
		app,
		connectingInput
	} = getContext<AppViewerContext>('AppViewerContext')

	const dispatch = createEventDispatcher()

	let donePromise: (() => void) | undefined = undefined

	const cancellableRun: (inlineScript?: InlineScript) => CancelablePromise<void> = (
		inlineScript?: InlineScript
	) => {
		let rejectCb: (err: Error) => void
		let p: Partial<CancelablePromise<void>> = new Promise<void>((resolve, reject) => {
			rejectCb = reject
			donePromise = resolve
			executeComponent(true, inlineScript).catch(reject)
		})
		p.cancel = () => {
			testJobLoader?.cancelJob()
			loading = false
			rejectCb(new Error('Canceled'))
		}

		return p as CancelablePromise<void>
	}

	$runnableComponents[id] = {
		autoRefresh: autoRefresh && recomputableByRefreshButton,
		refreshOnStart,
		cb: cancellableRun
	}

	if (!$initialized.initializedComponents.includes(id)) {
		$initialized.initializedComponents = [...$initialized.initializedComponents, id]
	}

	onDestroy(() => {
		$initialized.initializedComponents = $initialized.initializedComponents.filter((c) => c !== id)
	})

	$runnableComponents = $runnableComponents

	let args: Record<string, any> | undefined = undefined
	let testIsLoading = false
	let runnableInputValues: Record<string, any> = {}
	let executeTimeout: NodeJS.Timeout | undefined = undefined

	$: outputs.loading?.set(loading)

	function setDebouncedExecute() {
		executeTimeout && clearTimeout(executeTimeout)
		executeTimeout = setTimeout(() => {
			executeComponent(true)
		}, 200)
	}

	function computeStaticValues() {
		return Object.entries(fields ?? {})
			.filter(([k, v]) => v.type == 'static')
			.map(([name, field]) => {
				return [name, field['value']]
			})
	}

	let lazyStaticValues = computeStaticValues()
	let currentStaticValues = lazyStaticValues

	$: fields && (currentStaticValues = computeStaticValues())
	$: if (!deepEqual(currentStaticValues, lazyStaticValues)) {
		lazyStaticValues = currentStaticValues
		refreshIfAutoRefresh('static changed')
	}

	$: (runnableInputValues || extraQueryParams || args) &&
		testJobLoader &&
		refreshIfAutoRefresh('arg changed')

	$: refreshOn =
		runnable && runnable.type === 'runnableByName' ? runnable.inlineScript?.refreshOn ?? [] : []

	function refreshIfAutoRefresh(_src: string) {
		const refreshEnabled =
			autoRefresh && ((recomputeOnInputChanged ?? true) || refreshOn?.length > 0)
		if (refreshEnabled && $initialized.initialized) {
			setDebouncedExecute()
		}
	}

	// Test job internal state
	let testJob: CompletedJob | undefined = undefined
	let testJobLoader: TestJobLoader | undefined = undefined

	let schemaStripped: Schema | undefined =
		autoRefresh || forceSchemaDisplay ? emptySchema() : undefined

	$: (autoRefresh || forceSchemaDisplay) &&
		Object.keys(fields ?? {}).length > 0 &&
		(schemaStripped = stripSchema(fields, $stateId))

	function stripSchema(inputs: AppInputs, s: any): Schema {
		if (inputs === undefined) {
			return emptySchema()
		}
		let schema =
			runnable?.type == 'runnableByName' ? runnable.inlineScript?.schema : runnable?.schema
		try {
			schemaStripped = JSON.parse(JSON.stringify(schema))
		} catch (e) {
			console.warn('Error loading schema')
			return emptySchema()
		}

		// Remove hidden static inputs
		Object.keys(inputs ?? {}).forEach((key: string) => {
			const input = inputs[key]

			if (input.type === 'static' && schemaStripped !== undefined) {
				delete schemaStripped.properties[key]
			}

			if (input.type === 'connected' && schemaStripped !== undefined) {
				delete schemaStripped.properties[key]
			}
		})
		return schemaStripped as Schema
	}

	function generateNextFrontendJobId() {
		const prefix = 'Frontend: '
		let nextJobNumber = 1
		while ($jobs.find((j) => j.job === `${prefix}#${nextJobNumber}`)) {
			nextJobNumber++
		}
		return `${prefix}#${nextJobNumber}`
	}

	async function executeComponent(noToast = false, inlineScriptOverride?: InlineScript) {
		console.debug('execute', id)

		if (runnable?.type === 'runnableByName' && runnable.inlineScript?.language === 'frontend') {
			loading = true
			try {
				const r = await eval_like(
					runnable.inlineScript?.content,
					computeGlobalContext($worldStore, {}),
					false,
					$state,
					$mode == 'dnd',
					$componentControl,
					$worldStore,
					$runnableComponents
				)
				await setResult(r)

				$state = $state
			} catch (e) {
				sendUserToast(`Error running frontend script ${id}: ` + e.message, true)

				// Manually add a fake job to the job list to show the error

				const job = generateNextFrontendJobId()
				const error = e.body ?? e.message

				$errorByComponent[job] = {
					error,
					componentId: id
				}

				$jobs = [{ job, component: id, error }, ...$jobs]
			}
			loading = false
			return
		} else if (noBackend) {
			if (!noToast) {
				sendUserToast('This app is not connected to a windmill backend, it is a static preview')
			}
			return
		}
		if (runnable?.type === 'runnableByName' && !runnable.inlineScript) {
			return
		}

		loading = true

		try {
			let njob = await testJobLoader?.abstractRun(() => {
				const nonStaticRunnableInputs = {}
				const staticRunnableInputs = {}
				Object.keys(fields ?? {}).forEach((k) => {
					let field = fields[k]
					if (field?.type == 'static' && fields[k]) {
						staticRunnableInputs[k] = field.value
					} else if (field?.type == 'user') {
						nonStaticRunnableInputs[k] = args?.[k]
					} else {
						nonStaticRunnableInputs[k] = runnableInputValues[k]
					}
				})

				const requestBody = {
					args: nonStaticRunnableInputs,
					force_viewer_static_fields: !isEditor ? undefined : staticRunnableInputs
				}

				if (runnable?.type === 'runnableByName') {
					const { inlineScript } = inlineScriptOverride
						? { inlineScript: inlineScriptOverride }
						: runnable

					if (inlineScript) {
						requestBody['raw_code'] = {
							content: inlineScript.content,
							language: inlineScript.language,
							path: inlineScript.path
						}
					}
				} else if (runnable?.type === 'runnableByPath') {
					const { path, runType } = runnable
					requestBody['path'] = runType !== 'hubscript' ? `${runType}/${path}` : `script/${path}`
				}

				return AppService.executeComponent({
					workspace,
					path: defaultIfEmptyString(appPath, 'newapp'),
					requestBody
				})
			})
			if (njob) {
				$jobs = [{ job: njob, component: id }, ...$jobs]
			}
		} catch (e) {
			setResult({ error: e.body ?? e.message })
			loading = false
		}
	}

	export async function runComponent() {
		try {
			await executeComponent()
		} catch (e) {
			setResult({ error: e.body ?? e.message })
		}
	}

	let lastStartedAt: number = Date.now()

	function recordError(error: string) {
		if (testJob) {
			$errorByComponent[testJob.id] = {
				error: error,
				componentId: id
			}
		}
	}

	async function setResult(res: any) {
		const hasRes = res !== undefined && res !== null

		if (transformer) {
			$worldStore.newOutput(id, 'raw', res)
			res = await eval_like(
				transformer.content,
				computeGlobalContext($worldStore, { result: res }),
				false,
				$state,
				$mode == 'dnd',
				$componentControl,
				$worldStore,
				$runnableComponents
			)

			if (hasRes && res === undefined) {
				res = {
					error: {
						name: 'TransformerError',
						message: 'An error occured in the transformer',
						stack: 'Transformer returned undefined'
					}
				}
			}
		}

		outputs.result?.set(res)

		result = res
		if (res?.error) {
			recordError(res.error)
		} else {
			dispatch('success')
		}

		const previousJobId = Object.keys($errorByComponent).find(
			(key) => $errorByComponent[key].componentId === id
		)

		if (previousJobId && !result?.error) {
			delete $errorByComponent[previousJobId]
			$errorByComponent = $errorByComponent
		}

		donePromise?.()
	}

	function handleInputClick(e: CustomEvent) {
		const event = e as unknown as PointerEvent
		!$connectingInput.opened && selectId(event, id, selectedComponent, $app)
	}
</script>

{#each Object.entries(fields ?? {}) as [key, v] (key)}
	{#if v.type != 'static' && v.type != 'user'}
		<InputValue
			key={key + extraKey}
			{id}
			input={fields[key]}
			bind:value={runnableInputValues[key]}
		/>
	{/if}
{/each}

{#if runnable?.type == 'runnableByName' && runnable.inlineScript?.language == 'frontend'}
	{#each runnable.inlineScript.refreshOn ?? [] as { id: tid, key } (`${tid}-${key}`)}
		{@const fkey = `${tid}-${key}${extraKey}`}
		<InputValue
			{id}
			key={fkey}
			input={{ type: 'connected', connection: { componentId: tid, path: key }, fieldType: 'any' }}
			bind:value={runnableInputValues[fkey]}
		/>
	{/each}
{/if}

<TestJobLoader
	workspaceOverride={workspace}
	on:done={(e) => {
		if (testJob) {
			const startedAt = new Date(testJob.started_at).getTime()
			if (startedAt > lastStartedAt) {
				lastStartedAt = startedAt
				setResult(e.detail.result)
			}
		}
		loading = false
	}}
	bind:isLoading={testIsLoading}
	bind:job={testJob}
	bind:this={testJobLoader}
/>

{#if render}
	<div class="h-full flex relative flex-row flex-wrap {wrapperClass}" style={wrapperStyle}>
		{#if (autoRefresh || forceSchemaDisplay) && schemaStripped && Object.keys(schemaStripped?.properties ?? {}).length > 0}
			<div class="px-2 h-fit min-h-0">
				<LightweightSchemaForm
					schema={schemaStripped}
					bind:args
					on:inputClicked={handleInputClick}
				/>
			</div>
		{/if}

		{#if !runnable && autoRefresh}
			<Alert type="warning" size="xs" class="mt-2 px-1" title="Missing runnable">
				Please select a runnable
			</Alert>
		{:else if result?.error && $mode === 'preview'}
			<div
				title="Error"
				class={classNames(
					'text-red-500 px-1 text-2xs py-0.5 font-bold w-fit absolute border border-red-500 -bottom-2  shadow left-1/2 transform -translate-x-1/2 z-50 cursor-pointer',
					'bg-red-100/80'
				)}
			>
				<Popover notClickable placement="bottom" popupClass="!bg-white border w-96">
					<Bug size={14} />
					<span slot="text">
						<div class="bg-white">
							<Alert type="error" title="Error during execution">
								<div class="flex flex-col gap-2">
									An error occured, please contact the app author.
									<span class="font-semibold">Job id: {testJob?.id}</span>
								</div>
							</Alert>
						</div>
					</span>
				</Popover>
			</div>
			<div class="block grow w-full max-h-full border border-red-300 bg-red-50 relative">
				<slot />
			</div>
		{:else}
			<div class="block grow w-full max-h-full">
				<slot />
			</div>
		{/if}
		{#if !initializing && autoRefresh === true}
			<div class="flex absolute top-1 right-1 z-50">
				<RefreshButton {loading} componentId={id} />
			</div>
		{/if}
	</div>
{/if}
