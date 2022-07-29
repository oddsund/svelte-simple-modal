<script lang="ts">
	import { getContext } from 'svelte';

	export let showModal: boolean;

	const { close } = getContext('simple-modal-slots');

	const onClose = () => {
		close();
		showModal = false;
	};
</script>

<button on:click={onClose}>Custom Close Button</button>

<style>
	button {
		position: absolute;
		top: -3rem;
		right: 0;
		color: white;
		background: purple;
		border: 2px solid pink;
	}
</style>
