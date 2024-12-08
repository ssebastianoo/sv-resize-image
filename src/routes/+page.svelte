<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { cn } from '$lib/utils';

	let fileElement = $state<HTMLInputElement>();
	let fileSize = $state(0);
	let expectedFileSize = $state(0);
	let fileDimension = $state({ width: 0, height: 0 });
	let originalDimension = { width: 0, height: 0 };
	let canSave = $state(false);
	let lockAspectRatio = $state(true);
	let fileURL = $state<string | null>(null);
	let a = $state<HTMLAnchorElement>();
	let buttonText = $state('choose file');
	let dropping = $state(false);
	let dragCounter = 0;

	onMount(() => {
		const handleDragOver = (e: DragEvent) => {
			e.preventDefault();
		};

		const handleDragEnter = (e: DragEvent) => {
			e.preventDefault();
			dragCounter++;
			dropping = true;
		};

		const handleDragLeave = (e: DragEvent) => {
			e.preventDefault();
			dragCounter--;
			if (dragCounter === 0) {
				dropping = false;
			}
		};

		const handleDrop = (e: DragEvent) => {
			e.preventDefault();
			dragCounter = 0;
			dropping = false;

			if (fileElement && e.dataTransfer && e.dataTransfer.files) {
				fileElement.files = e.dataTransfer.files;
			}

			checkImage();
		};

		window.addEventListener('dragover', handleDragOver);
		window.addEventListener('dragenter', handleDragEnter);
		window.addEventListener('dragleave', handleDragLeave);
		window.addEventListener('drop', handleDrop);

		return () => {
			window.removeEventListener('dragover', handleDragOver);
			window.removeEventListener('dragenter', handleDragEnter);
			window.removeEventListener('dragleave', handleDragLeave);
			window.removeEventListener('drop', handleDrop);
		};
	});

	function handleFile() {
		if (!fileElement || !fileElement.files || !fileURL) return;

		if (a) {
			a.click();
			return;
		}

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

				if (a) URL.revokeObjectURL(a.href);

				a = document.createElement('a');
				const resizedFileURL = URL.createObjectURL(resizedFile);
				a.href = resizedFileURL;
				a.download = newFileName;
				a.click();
				a.remove();
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
		if (!fileElement || !fileElement.files) {
			canSave = false;
			return;
		}

		if (!fileElement.files[0].type.startsWith('image/')) {
			canSave = false;
			alert('only images are supported');
			// @ts-ignore
			fileElement.value = null;
			return;
		}

		buttonText = fileElement.files[0].name;

		if (buttonText.length > 30) buttonText = buttonText.slice(0, 30) + '...';

		const file = fileElement.files[0];

		if (fileURL) URL.revokeObjectURL(fileURL);
		fileURL = URL.createObjectURL(file);

		if (a) {
			URL.revokeObjectURL(a.href);
			a.remove();
			a = undefined;
		}

		canSave = true;
		fileSize = fileElement.files[0].size;

		const img = new Image();
		img.src = fileURL;
		img.onload = () => {
			fileDimension.width = img.width;
			fileDimension.height = img.height;

			originalDimension = { ...fileDimension };

			updateExpectedFileSize();
		};

		img.remove();
	}

	function updateExpectedFileSize() {
		const originalArea = originalDimension.width * originalDimension.height;
		const newArea = fileDimension.width * fileDimension.height;
		expectedFileSize = Math.floor((newArea / originalArea) * fileSize);
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
		updateExpectedFileSize();
	}

	onDestroy(() => {
		if (fileURL) URL.revokeObjectURL(fileURL!);
		if (a) URL.revokeObjectURL(a.href);
	});
</script>

<div class="flex h-full flex-col justify-between">
	<div class="h-8 w-full"></div>
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="relative flex h-full w-full flex-col items-center justify-center gap-3">
		<div
			class="flex w-5/6 max-w-96 flex-col gap-3 rounded-lg border border-neutral-800 bg-neutral-950 p-4"
		>
			{#if fileSize}
				<p>{humanSize(fileSize)}</p>
			{/if}
			<input
				onchange={checkImage}
				bind:this={fileElement}
				accept="image/*"
				type="file"
				class="hidden"
			/>
			<button
				onclick={() => {
					if (fileElement) fileElement.click();
				}}
				class={cn(
					'w-full rounded-md border p-2 outline-none hover:bg-neutral-900',
					canSave ? 'border-neutral-800 text-sm' : 'border-transparent'
				)}>{buttonText}</button
			>
			{#if canSave}
				<div class="relative flex justify-center">
					<img
						src={fileURL}
						class="absolute left-0 top-0 z-10 h-40 w-full opacity-50 blur-sm"
						alt="background"
					/>
					<img src={fileURL} alt="Uploaded" class="z-20 h-40 w-auto rounded-md" />
				</div>
			{/if}
		</div>
		{#if canSave}
			<div
				class="flex w-5/6 max-w-96 flex-col gap-3 rounded-lg border border-neutral-800 bg-neutral-950 p-4"
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
					onclick={handleFile}>save (≈ {humanSize(expectedFileSize)})</button
				>
			</div>
		{/if}
		{#if dropping}
			<div
				class="absolute flex h-full w-full items-center justify-center rounded-3xl border border-dashed border-neutral-500 bg-neutral-700 bg-opacity-40 backdrop-blur-[2px]"
			>
				<p>drop your image here</p>
			</div>
		{/if}
	</div>
	<footer class="flex h-8 items-end justify-center gap-4">
		<a href="https://gir8.it" target="_blank" class="text-sm text-neutral-400 hover:underline"
			>made by seb</a
		>
		<a
			href="https://github.com/ssebastianoo/sv-resize-image"
			target="_blank"
			class="text-sm text-neutral-400 hover:underline">github</a
		>
	</footer>
</div>
