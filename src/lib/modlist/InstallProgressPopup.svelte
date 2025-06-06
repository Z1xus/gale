<script lang="ts">
	import Popup from '$lib/components/Popup.svelte';
	import { invokeCommand } from '$lib/invoke';
	import type { InstallProgress } from '$lib/models';
	import { refreshProfiles } from '$lib/stores';
	import { formatTime, shortenFileSize } from '$lib/util';

	import { listen } from '@tauri-apps/api/event';

	import { Dialog, Progress } from 'bits-ui';
	import { onMount } from 'svelte';

	let open = false;

	let progress: InstallProgress = {
		durationSecs: 0,
		totalProgress: 0,
		installedMods: 0,
		totalMods: 0,
		currentName: '',
		canCancel: false,
		task: {
			kind: 'installing'
		}
	};

	$: currentName = progress.currentName.replace('_', ' ');
	$: estimatedTimeLeft =
		progress.durationSecs > 1
			? formatTime(progress.durationSecs * (1 / progress.totalProgress - 1))
			: '---';

	onMount(() => {
		listen<InstallProgress>('install_progress', (event) => {
			progress = event.payload;

			switch (progress.task.kind) {
				case 'done':
					progress.totalProgress = 1;
					progress.installedMods = progress.totalMods;
					refreshProfiles();
					setTimeout(() => {
						open = false;
					}, 250);
					break;

				case 'error':
					open = false;
					break;

				default:
					open = true;
					break;
			}
		});
	});
</script>

<Popup
	bind:open
	title="Installing mods ({progress.installedMods}/{progress.totalMods})"
	canClose={progress.canCancel}
	onClose={() => invokeCommand('cancel_install')}
	confirmClose={{
		message: 'Are you sure you want to abort the installation?'
	}}
>
	<Dialog.Description class="text-primary-400">
		{#if progress.task.kind == 'done'}
			Done!
		{:else}
			<div>
				{#if progress.task.kind == 'downloading'}
					Downloading {currentName} ({shortenFileSize(
						progress.task.payload.downloaded
					)}/{shortenFileSize(progress.task.payload.total)})
				{:else if progress.task.kind == 'extracting'}
					Extracting {currentName}
				{:else if progress.task.kind == 'installing'}
					Installing {currentName}
				{/if}
			</div>

			<div>
				Estimated time remaining: {estimatedTimeLeft}
			</div>
		{/if}
	</Dialog.Description>

	<Progress.Root
		value={progress.totalProgress}
		max={1}
		class="bg-primary-900 relative mt-2 h-4 w-full overflow-hidden rounded-full"
	>
		<div
			class="bg-accent-600 absolute top-0 left-0 h-full rounded-l-full transition-all"
			style="width: {progress.totalProgress * 100}%"
		/>
	</Progress.Root>
</Popup>
