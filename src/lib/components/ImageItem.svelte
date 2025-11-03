<script>
	import { onMount, onDestroy, createEventDispatcher } from 'svelte';

	/** @type {File} */
	export let file;

	/** @type {string | null} */
	let objectUrl = null;
	const fileName = file?.name ?? '';
	const dispatch = createEventDispatcher();

	onMount(() => {
		if (file) objectUrl = URL.createObjectURL(file);
	});
	onDestroy(() => {
		if (objectUrl) URL.revokeObjectURL(objectUrl);
	});

	function handleRemove() {
		dispatch('remove');
	}
</script>

<div class="card">
	<img class="thumb" src={objectUrl ?? undefined} alt={fileName} />
	<div class="meta">
		<div class="name" title={fileName}>{fileName}</div>
	</div>
	<button type="button" class="remove-btn" aria-label={`Remove ${fileName}`} on:click={handleRemove}
		>×</button
	>
</div>

<style>
	.card {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		padding: 1rem;
		border: 0.1rem solid var(--panel-border);
		border-radius: 0.5rem;
		background: var(--panel-bg);
		align-items: flex-start;
		position: relative;
	}
	.thumb {
		width: 90%;
		max-width: 250px;
		object-fit: cover;
		border-radius: 0.5rem;
		flex: 0 0 auto;
	}
	.meta {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		flex: 1 1 auto;
		min-width: 0;
	}
	.name {
		font-weight: 600;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.remove-btn {
		position: absolute;
		top: 0.5rem;
		right: 0.5rem;
		border: none;
		background: var(--danger);
		color: #fff;
		width: 1.5rem;
		height: 1.5rem;
		border-radius: 9999px;
		cursor: pointer;
		line-height: 1.5rem;
		text-align: center;
		padding: 0;
	}
	.remove-btn:hover {
		background: var(--danger-hover);
	}
</style>
