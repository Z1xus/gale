<script lang="ts">
	import { setConfigEntry } from '$lib/config';
	import type { ConfigValue, ConfigNum, ConfigEntryId, ConfigRange } from '$lib/models';
	import ResetConfigButton from './ResetConfigButton.svelte';

	export let entryId: ConfigEntryId;

	let content = entryId.entry.value.content as ConfigNum;
	let range = content.range as ConfigRange;
	let type = entryId.entry.value.type as 'int' | 'float';

	$: fillPercent = clamp(((content.value - range.start) / (range.end - range.start)) * 100, 0, 100);
	$: decimals = type === 'int' ? 0 : 1;

	let element: HTMLDivElement;
	let fill: HTMLDivElement;
	let handle: HTMLDivElement;

	let isDragging = false;
	let inputString = content.value.toString();

	function onReset(newValue: ConfigValue) {
		content = newValue.content as ConfigNum;
		inputString = content.value.toString();
	}

	function submitValue() {
		setConfigEntry(entryId, { type, content });
	}

	const DECIMAL_STEP = 0.1;
	const X_OFFSET = 16;

	function calculateNewValue(clientX: number) {
		let rect = element.getBoundingClientRect();
		rect.width -= X_OFFSET;
		rect.x += X_OFFSET / 2;

		let x = clientX - rect.left;
		let newValue = range.start + (range.end - range.start) * (x / rect.width);

		if (type === 'float') {
			newValue = Math.round(newValue / DECIMAL_STEP) * DECIMAL_STEP;
		} else if (type === 'int') {
			newValue = Math.round(newValue);
		}

		newValue = clamp(newValue, range.start, range.end);
		inputString = newValue.toFixed(decimals);
		content.value = newValue;
	}

	function clamp(value: number, min: number, max: number) {
		return Math.max(min, Math.min(max, value));
	}
</script>

<svelte:window
	on:mousemove={(evt) => {
		if (isDragging) {
			calculateNewValue(evt.clientX);
		}
	}}
	on:mouseup={(evt) => {
		if (isDragging) {
			isDragging = false;
			calculateNewValue(evt.clientX);
			submitValue();
		}
	}}
/>

<div
	class="group bg-primary-900 h-5 grow rounded-full py-1 pr-2 pl-1"
	role="slider"
	aria-valuemin={range.start}
	aria-valuemax={range.end}
	aria-valuenow={content.value}
	tabindex="0"
	bind:this={element}
	on:keydown={(e) => {
		if (e.key === 'ArrowLeft' || e.key === 'ArrowDown') {
			content.value = Math.max(range.start, content.value - 1);
		} else if (e.key === 'ArrowRight' || e.key === 'ArrowUp') {
			content.value = Math.min(range.end, content.value + 1);
		}

		inputString = content.value.toFixed(decimals);
	}}
	on:mousedown={(evt) => {
		isDragging = true;
		calculateNewValue(evt.clientX);
	}}
>
	<div
		class="group-hover:bg-primary-600 relative h-full min-w-1 rounded-l-full"
		style="width: {fillPercent}%;"
		class:bg-primary-700={!isDragging}
		class:bg-primary-600={isDragging}
		bind:this={fill}
	>
		<div
			class="absolute right-[-0.5rem] h-3 w-3 rounded-full"
			class:bg-primary-400={!isDragging}
			class:bg-primary-300={isDragging}
			bind:this={handle}
			draggable="false"
		/>
	</div>
</div>

<input
	type="number"
	bind:value={inputString}
	on:change={() => {
		let newValue = parseFloat(inputString);

		if (!isNaN(newValue)) {
			newValue = clamp(newValue, range.start, range.end);

			if (type === 'int') {
				newValue = Math.round(newValue);
			}

			content.value = newValue;
			submitValue();
		}

		inputString = content.value.toString();
	}}
	class="focus:ring-accent-400 bg-primary-900 text-primary-300 placeholder-primary-400 hover:border-primary-500 hover:text-primary-200 ml-3 w-1/6 min-w-0 shrink rounded-lg border border-transparent px-3 py-1 focus:border-transparent focus:ring-2 focus:outline-hidden"
/>
<ResetConfigButton {entryId} {onReset} />

<style>
	input[type='number']::-webkit-inner-spin-button,
	input[type='number']::-webkit-outer-spin-button {
		-webkit-appearance: none;
		margin: 0;
	}
</style>
