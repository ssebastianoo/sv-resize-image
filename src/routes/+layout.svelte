<script lang="ts">
	import '@fontsource/roboto';
	import '../app.css';

	let { children } = $props();
</script>

<div class="h-svh p-4">
	{@render children()}
</div>
