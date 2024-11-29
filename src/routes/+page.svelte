<script lang="ts">
	import { onMount } from 'svelte';

	let fileElement = $state<HTMLInputElement>();
	let fileSize = $state<number>();
	let fileDimension = $state({ width: 0, height: 0 });
	let originalDimension = { width: 0, height: 0 };
	let canSave = $state(false);
	let lockAspectRatio = $state(true);
	let fileURL: string | null;

	function handleFile() {
		if (!fileElement || !fileElement.files || !fileURL) return;
		const file = fileElement.files[0];

		const canvas = document.createElement('canvas');
		const ctx = canvas.getContext('2d') as CanvasRenderingContext2D;

		const fileName = file.name.split('.')[0];
		const extension = file.name.split('.')[1];
		const newFileName = fileName + '-resized.' + extension;

		const img = new Image();
		img.src = fileURL;
		img.onload = () => {
			canvas.width = fileDimension.width;
			canvas.height = fileDimension.height;

			ctx.drawImage(img, 0, 0, img.width, img.height, 0, 0, canvas.width, canvas.height);

			canvas.toBlob((blob) => {
				if (!blob) return;
				const resizedFile = new File([blob], newFileName, { type: file.type });

				const a = document.createElement('a');
				const resizedFileURL = URL.createObjectURL(resizedFile);
				a.href = resizedFileURL;
				a.download = newFileName;
				a.click();

				URL.revokeObjectURL(fileURL!);
				URL.revokeObjectURL(resizedFileURL);
			}, file.type);
		};
		img.onerror = () => {
			console.error('Failed to load the image.');
			URL.revokeObjectURL(fileURL!);
		};
	}

	function humanSize(bytes: number) {
		const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
		if (bytes === 0) return '0 Byte';
		const i = Math.floor(Math.log(bytes) / Math.log(1024));
		return (bytes / Math.pow(1024, i)).toFixed(2) + ' ' + sizes[i];
	}

	function checkImage() {
		if (!fileElement || !fileElement.files || !fileElement.files[0].type.startsWith('image/')) {
			canSave = false;
			return;
		}

		const file = fileElement.files[0];

		if (fileURL) URL.revokeObjectURL(fileURL);
		fileURL = URL.createObjectURL(file);

		canSave = true;
		fileSize = fileElement.files[0].size;

		const img = new Image();
		img.src = fileURL;
		img.onload = () => {
			fileDimension.width = img.width;
			fileDimension.height = img.height;

			originalDimension = { ...fileDimension };
		};
	}

	function handleResize(edited: 'width' | 'height') {
		if (lockAspectRatio) {
			if (edited === 'width') {
				fileDimension.height = Math.floor(
					(fileDimension.width * originalDimension.height) / originalDimension.width
				);
			} else {
				fileDimension.width = Math.floor(
					(fileDimension.height * originalDimension.width) / originalDimension.height
				);
			}
		}
	}
</script>

<div class="flex h-full w-full flex-col items-center justify-center gap-3">
	<div
		class="flex w-2/3 max-w-96 flex-col gap-3 rounded-lg border border-neutral-800 bg-neutral-950 p-4"
	>
		{#if fileSize}
			<p>{humanSize(fileSize)}</p>
		{/if}
		<input onchange={checkImage} bind:this={fileElement} accept="image/*" type="file" />
	</div>
	{#if canSave}
		<div
			class="flex w-2/3 max-w-96 flex-col gap-3 rounded-lg border border-neutral-800 bg-neutral-950 p-4"
		>
			<div class="flex items-center gap-3">
				<input
					type="checkbox"
					id="lockAspectRatio"
					class="cursor-pointer"
					checked={lockAspectRatio}
					onclick={() => {
						lockAspectRatio = !lockAspectRatio;
						if (lockAspectRatio) {
							if (fileDimension.width > fileDimension.height) {
								handleResize('width');
							} else {
								handleResize('height');
							}
						}
					}}
				/>
				<label for="lockAspectRatio">lock aspect ratio</label>
			</div>
			<div class="flex items-center justify-between gap-2">
				<input
					type="number"
					class="w-full rounded-md border border-neutral-800 bg-neutral-900 px-2 py-1 outline-none"
					placeholder="width"
					bind:value={fileDimension.width}
					onchange={() => {
						handleResize('width');
					}}
				/>
				<p class="text-neutral-400">x</p>
				<input
					type="number"
					class="w-full rounded-md border border-neutral-800 bg-neutral-900 px-2 py-1 outline-none"
					placeholder="height"
					bind:value={fileDimension.height}
					onchange={() => {
						handleResize('height');
					}}
				/>
			</div>
			<button
				class="w-full rounded-lg border border-transparent bg-neutral-900 p-2 hover:border-neutral-600"
				onclick={handleFile}>save</button
			>
		</div>
	{/if}
</div>
