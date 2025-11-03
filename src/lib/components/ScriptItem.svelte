<script>
	import { createEventDispatcher } from 'svelte';

	/** @type {File} */
	export let file;

	const fileName = file?.name ?? '';
	const dispatch = createEventDispatcher();

	function handleRemove() {
		dispatch('remove');
	}
</script>

<div class="card">
	<div class="meta">
		<div class="name" title={fileName}>{fileName}</div>
	</div>
	<button type="button" class="remove-btn" aria-label={`Remove ${fileName}`} on:click={handleRemove}
		>×</button
	>
</div>

<style>
	.card {
		min-width: 300px;
		display: flex;
		gap: 1rem;
		padding: 1rem;
		border: 0.1rem solid var(--panel-border);
		border-radius: 0.5rem;
		background: var(--panel-bg);
		position: relative;
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
