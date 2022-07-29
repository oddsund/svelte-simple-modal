<!--
Svelte Simple Modal - Demos
https://github.com/oddsund/svelte-simple-modal-slots
-->
<script lang="ts">
	import Modal from '$lib/Modal.svelte';
	import CloseButton from '../demo/CloseButton.svelte';
	import Dialog from '../demo/Dialog.svelte';
	import Popup from '../demo/Popup.svelte';
	import PopupLong from '../demo/PopupLong.svelte';
	import { fly } from 'svelte/transition';

	let showModal = false;
	let showPopupLong = false;
	let showPopupCustom = false;
	let showDialog = false;

	let opening = false;
	let opened = false;
	let closing = false;
	let closed = false;

	const popupCustomStyling = {
		styleBg: {
			background: 'rgba(200, 255, 0, 0.66)'
		},
		styleWindowWrap: {
			margin: '4rem'
		},
		styleWindow: {
			border: '2px solid #00beff',
			boxShadow: 'inset 0 0 0 2px white, 0 0 0 2px white',
			background: '#ff7000'
		},
		styleContent: {
			color: '#9500ff',
			fontFamily: 'Comic Sans',
			fontStyle: 'italic'
		},
		styleCloseButton: {
			borderRadius: 0,
			boxShadow: '0 0 0 2px white',
			background: 'pink'
		},
		transitionWindow: fly,
		transitionWindowProps: {
			y: 100,
			duration: 1000
		}
	};

	const popupCustomCallbacks = {
		onOpen: () => {
			opening = true;
		},
		onOpened: () => {
			opening = false;
			opened = true;
		},
		onClose: () => {
			opened = false;
			opening = false;
			closing = true;
		},
		onClosed: () => {
			closing = false;
			closed = true;
			setTimeout(() => {
				closed = false;
			}, 1000);
		}
	};

	let name: string;
	let status = 0;

	const onCancel = () => {
		name = '';
		status = -1;
	};

	const onOkay = (text: string) => {
		name = text;
		status = 1;
	};
</script>

<button on:click={() => (showModal = true)}>Show a popup!</button>
<br />
<button on:click={() => (showPopupLong = true)}>Show a popup with long text!</button>
<br />
<button on:click={() => (showPopupCustom = true)}>
	Show a customized popup and listen to events!
</button>
<br />
<button on:click={() => (showDialog = true)}>Show a dialog!</button>

<Modal bind:showModal>
	<Popup slot="modalContent" message="It's a popup!" />
</Modal>
<Modal bind:showModal={showPopupLong}>
	<PopupLong slot="modalContent" message="It's a popup with long text!" />
</Modal>
<Modal
	showModal={showPopupCustom}
	{...popupCustomStyling}
	callbacks={popupCustomCallbacks}
	on:close={() => (showPopupCustom = false)}
>
	<Popup slot="modalContent" />
	<CloseButton slot="closeButton" bind:showModal={showPopupCustom} />
</Modal>
<Modal
	showModal={showDialog}
	closeButton={false}
	closeOnEsc={false}
	closeOnOuterClick={false}
	on:close={() => (showDialog = false)}
>
	<Dialog slot="modalContent" message="What is your name?" hasForm={true} {onCancel} {onOkay} />
</Modal>

{#if status === 1}
	<p>Hi {name}! =</p>
{:else if status === -1}
	<p><em>Dialog was canceled</em></p>
{/if}

<div id="state">
	{#if opening}
		<p>opening modal...</p>
	{:else if opened}
		<p>opened modal!</p>
	{:else if closing}
		<p>closing modal...</p>
	{:else if closed}
		<p>closed modal!</p>
	{/if}
</div>

<style>
	button {
		margin-bottom: 1rem;
	}
</style>
