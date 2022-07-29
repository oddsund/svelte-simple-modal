<!--
Svelte Simple Modal - Demos
https://github.com/oddsund/svelte-simple-modal-slots
-->
<script lang="ts">
	import Modal from '$lib/Modal.svelte';
	import Content from 'src/demo/Content.svelte';
	import ContentOutside from 'src/demo/ContentOutside.svelte';

	let showModal = false;
</script>

<Modal bind:showModal>
	<Content />
</Modal>

<ContentOutside />
