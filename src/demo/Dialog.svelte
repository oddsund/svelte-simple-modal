<script lang="ts">
	import { getContext } from 'svelte';
	export let message: string;
	export let hasForm = false;
	export let onCancel: () => void;
	export let onOkay: (arg: string) => void;

	const { close } = getContext('simple-modal-slots');

	let value: string;

	function _onCancel() {
		onCancel();
		close();
	}

	function _onOkay() {
		onOkay(value);
		close();
	}
</script>

<h2>{message}</h2>

{#if hasForm}
	<input type="text" bind:value on:keypress={(e) => e.key === 'Enter' && _onOkay()} />
{/if}

<div class="buttons">
	<button on:click={_onCancel}> Cancel </button>
	<button on:click={_onOkay}> Okay </button>
</div>

<style>
	h2 {
		font-size: 2rem;
		text-align: center;
	}

	input {
		width: 100%;
	}

	.buttons {
		display: flex;
		justify-content: space-between;
	}
</style>
