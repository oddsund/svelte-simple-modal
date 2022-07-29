<script lang="ts">
	import { type BlurParams, fade, type TransitionConfig } from 'svelte/transition';
	import { createEventDispatcher, setContext, onDestroy, onMount } from 'svelte';

	type State = {
		ariaLabel: string | null;
		ariaLabelledBy: string | null;
		closeButton: boolean;
		closeOnEsc: boolean;
		closeOnOuterClick: boolean;
		styleBg: Record<string, string | number>;
		styleWindowWrap: Record<string, string | number>;
		styleWindow: Record<string, string | number>;
		styleContent: Record<string, string | number>;
		styleCloseButton: Record<string, string | number>;
		classBg: string | undefined;
		classWindowWrap: string | undefined;
		classWindow: string | undefined;
		classContent: string | undefined;
		classCloseButton: string | undefined;
		transitionBg: (node: HTMLElement, parameters: BlurParams) => TransitionConfig;
		transitionBgProps: BlurParams;
		transitionWindow: (node: HTMLElement, parameters: BlurParams) => TransitionConfig;
		transitionWindowProps: BlurParams;
		disableFocusTrap: boolean;
		isTabbable: (node: HTMLElement) => boolean;
		unstyled: boolean;
	};

	type Callbacks = {
		onOpen?: (event: Event) => void;
		onClose?: (event: Event) => void;
		onOpened?: (event: Event) => void;
		onClosed?: (event: Event) => void;
	};

	const dispatch = createEventDispatcher();

	const baseSetContext = setContext;

	/**
	 * A basic function that checks if a node is tabbale
	 */
	// TODO: Må se på typer her
	const baseIsTabbable = (node: HTMLElement) =>
		node.tabIndex >= 0 &&
		!node.hidden &&
		!(node as HTMLInputElement).disabled &&
		node.style.display !== 'none' &&
		(node as HTMLInputElement).type !== 'hidden' &&
		Boolean(node.offsetWidth || node.offsetHeight || node.getClientRects().length);

	/**
	 * A function to determine if an HTML element is tabbable
	 */
	export let isTabbable: State['isTabbable'] = baseIsTabbable;

	/**
	 * If the modal is to be displayed(true) or not
	 */
	export let showModal = false;

	/**
	 * Svelte context key to reference the simple modal context
	 */
	export let key = 'simple-modal-slots';

	/**
	 * Accessibility label of the modal
	 * @see https://www.w3.org/TR/wai-aria-1.1/#aria-label
	 */
	export let ariaLabel: State['ariaLabel'] = null;

	/**
	 * Element ID holding the accessibility label of the modal
	 * @see https://www.w3.org/TR/wai-aria-1.1/#aria-labelledby
	 */
	export let ariaLabelledBy: State['ariaLabelledBy'] = null;

	/**
	 * Whether to show a close button or not
	 */
	export let closeButton: State['closeButton'] = true;

	/**
	 * Whether to close the modal on hitting the escape key or not
	 */
	export let closeOnEsc: State['closeOnEsc'] = true;

	/**
	 * Whether to close the modal upon an outside mouse click or not
	 */
	export let closeOnOuterClick: State['closeOnOuterClick'] = true;

	/**
	 * CSS for styling the background element
	 */
	export let styleBg: State['styleBg'] = {};

	/**
	 * CSS for styling the window wrapper element
	 */
	export let styleWindowWrap: State['styleWindowWrap'] = {};

	/**
	 * CSS for styling the window element
	 */
	export let styleWindow: State['styleWindow'] = {};

	/**
	 * CSS for styling the content element
	 */
	export let styleContent: State['styleContent'] = {};

	/**
	 * CSS for styling the close element
	 */
	export let styleCloseButton: State['styleCloseButton'] = {};

	/**
	 * Class name for the background element
	 */
	export let classBg: State['classBg'] = undefined;

	/**
	 * Class name for window wrapper element
	 */
	export let classWindowWrap: State['classWindowWrap'] = undefined;

	/**
	 * Class name for window element
	 */
	export let classWindow: State['classWindow'] = undefined;

	/**
	 * Class name for content element
	 */
	export let classContent: State['classContent'] = undefined;

	/**
	 * Class name for close element
	 */
	export let classCloseButton: State['classCloseButton'] | undefined = undefined;

	/**
	 * Do not apply default styles to the modal
	 */
	export let unstyled: State['unstyled'] = false;

	/**
	 * The setContext() function associated with this library
	 * @description If you want to bundle simple-modal with its own version of
	 * Svelte you have to pass `setContext()` from your main app to simple-modal
	 * using this parameter
	 * @see https://svelte.dev/docs#run-time-svelte-setcontext
	 */
	export let customSetContext: typeof setContext = baseSetContext;

	/**
	 * Transition function for the background element
	 * @see https://svelte.dev/docs#transition_fn
	 */
	export let transitionBg: State['transitionBg'] = fade;

	/**
	 * Parameters for the background element transition
	 */
	export let transitionBgProps: State['transitionBgProps'] = { duration: 250 };

	/**
	 * Transition function for the window element
	 * @see https://svelte.dev/docs#transition_fn
	 */
	export let transitionWindow: State['transitionWindow'] = transitionBg;

	/**
	 * Parameters for the window element transition
	 */
	export let transitionWindowProps: State['transitionWindowProps'] = transitionBgProps;

	/**
	 * If `true` elements outside the modal can be focused
	 */
	export let disableFocusTrap: State['disableFocusTrap'] = false;

	/**
	 * Optional callbacks that can be called at the start of or after the opening or closing of the modal
	 */
	export let callbacks: Callbacks | null = null;

	const defaultState: State = {
		ariaLabel,
		ariaLabelledBy,
		closeButton,
		closeOnEsc,
		closeOnOuterClick,
		styleBg,
		styleWindowWrap,
		styleWindow,
		styleContent,
		styleCloseButton,
		classBg,
		classWindowWrap,
		classWindow,
		classContent,
		classCloseButton,
		transitionBg,
		transitionBgProps,
		transitionWindow,
		transitionWindowProps,
		disableFocusTrap,
		isTabbable,
		unstyled
	};

	let state = { ...defaultState };

	let background: HTMLDivElement;
	let wrap: HTMLDivElement;
	let modalWindow: HTMLDivElement;
	let scrollY: number;
	let cssBg: string;
	let cssWindowWrap: string;
	let cssWindow: string;
	let cssContent: string;
	let cssCloseButton: string;
	let currentTransitionBg: (node: HTMLElement, parameters: BlurParams) => TransitionConfig;
	let currentTransitionWindow: (node: HTMLElement, parameters: BlurParams) => TransitionConfig;
	let prevBodyPosition: string;
	let prevBodyOverflow: string;
	let prevBodyWidth: string;
	let outerClickTarget: EventTarget;

	const camelCaseToDash = (str: string): string =>
		str.replace(/([a-zA-Z])(?=[A-Z])/g, '$1-').toLowerCase();

	// eslint-disable-next-line @typescript-eslint/no-explicit-any
	const toCssString = (props: any): string =>
		props
			? Object.keys(props).reduce(
					(str, key) => `${str}; ${camelCaseToDash(key)}: ${props[key]}`,
					''
			  )
			: '';

	const updateStyleTransition = () => {
		cssBg = toCssString(
			Object.assign(
				{},
				{
					width: window.innerWidth,
					height: window.innerHeight
				},
				state.styleBg
			)
		);
		cssWindowWrap = toCssString(state.styleWindowWrap);
		cssWindow = toCssString(state.styleWindow);
		cssContent = toCssString(state.styleContent);
		cssCloseButton = toCssString(state.styleCloseButton);
		currentTransitionBg = state.transitionBg;
		currentTransitionWindow = state.transitionWindow;
	};

	let displayModalContent = false;

	const open = (options: Partial<State> = {}) => {
		state = { ...defaultState, ...options };
		displayModalContent = true;
		updateStyleTransition();
		disableScroll();
	};

	const close = () => {
		displayModalContent = false;
		showModal = false;
		enableScroll();
	};

	const handleKeydown = (event: KeyboardEvent) => {
		if (state.closeOnEsc && displayModalContent && event.key === 'Escape') {
			event.preventDefault();
			close();
		}

		if (displayModalContent && event.key === 'Tab' && !state.disableFocusTrap) {
			// trap focus
			const nodes = modalWindow.querySelectorAll<HTMLElement>('*');
			const tabbable = Array.from(nodes)
				.filter(state.isTabbable)
				.sort((a, b) => a.tabIndex - b.tabIndex);

			let index = tabbable.indexOf(document.activeElement as HTMLElement);
			if (index === -1 && event.shiftKey) index = 0;

			index += tabbable.length + (event.shiftKey ? -1 : 1);
			index %= tabbable.length;

			tabbable[index].focus();
			event.preventDefault();
		}
	};

	const handleOuterMousedown = (event: MouseEvent) => {
		if (state.closeOnOuterClick && (event.target === background || event.target === wrap))
			outerClickTarget = event.target;
	};

	const handleOuterMouseup = (event: MouseEvent) => {
		if (state.closeOnOuterClick && event.target === outerClickTarget) {
			event.preventDefault();
			close();
		}
	};

	const disableScroll = () => {
		scrollY = window.scrollY;
		prevBodyPosition = document.body.style.position;
		prevBodyOverflow = document.body.style.overflow;
		prevBodyWidth = document.body.style.width;
		document.body.style.position = 'fixed';
		document.body.style.top = `-${scrollY}px`;
		document.body.style.overflow = 'hidden';
		document.body.style.width = '100%';
	};

	const enableScroll = () => {
		document.body.style.position = prevBodyPosition || '';
		document.body.style.top = '';
		document.body.style.overflow = prevBodyOverflow || '';
		document.body.style.width = prevBodyWidth || '';
		window.scrollTo(0, scrollY);
	};

	customSetContext(key, { open, close });

	$: onOpen = (event: Event) => {
		if (callbacks?.onOpen) callbacks.onOpen(event);
		/**
		 * The open event is fired right before the modal opens
		 * @event {void} open
		 */
		dispatch('open');
		/**
		 * The opening event is fired right before the modal opens
		 * @event {void} opening
		 * @deprecated Listen to the `open` event instead
		 */
		dispatch('opening'); // Deprecated. Do not use!
	};

	$: onClose = (event: Event) => {
		if (callbacks?.onClose) callbacks.onClose(event);
		/**
		 * The close event is fired right before the modal closes
		 * @event {void} close
		 */
		dispatch('close');
		/**
		 * The closing event is fired right before the modal closes
		 * @event {void} closing
		 * @deprecated Listen to the `close` event instead
		 */
		dispatch('closing'); // Deprecated. Do not use!
	};

	$: onOpened = (event: Event) => {
		if (callbacks?.onOpened) callbacks.onOpened(event);
		/**
		 * The opened event is fired after the modal's opening transition
		 * @event {void} opened
		 */
		dispatch('opened');
	};

	$: onClosed = (event: Event) => {
		if (callbacks?.onClosed) callbacks.onClosed(event);
		/**
		 * The closed event is fired after the modal's closing transition
		 * @event {void} closed
		 */
		dispatch('closed');
	};

	let isMounted = false;

	$: {
		if (isMounted) {
			if (showModal) {
				open();
			} else {
				close();
			}
		}
	}

	onDestroy(() => {
		if (isMounted) close();
	});

	onMount(() => {
		isMounted = true;
	});
</script>

<svelte:window on:keydown={handleKeydown} />

{#if displayModalContent}
	<div
		class={state.classBg}
		class:bg={!unstyled}
		on:mousedown={handleOuterMousedown}
		on:mouseup={handleOuterMouseup}
		bind:this={background}
		transition:currentTransitionBg={state.transitionBgProps}
		style={cssBg}
	>
		<div
			class={state.classWindowWrap}
			class:wrap={!unstyled}
			bind:this={wrap}
			style={cssWindowWrap}
		>
			<div
				class={state.classWindow}
				class:window={!unstyled}
				role="dialog"
				aria-modal="true"
				aria-label={state.ariaLabelledBy ? null : state.ariaLabel || null}
				aria-labelledby={state.ariaLabelledBy || null}
				bind:this={modalWindow}
				transition:currentTransitionWindow={state.transitionWindowProps}
				on:introstart={onOpen}
				on:outrostart={onClose}
				on:introend={onOpened}
				on:outroend={onClosed}
				style={cssWindow}
			>
				{#if state.closeButton}
					<slot name="closeButton">
						<button
							class={state.classCloseButton}
							class:close={!unstyled}
							aria-label="Close modal"
							on:click={close}
							style={cssCloseButton}
							type="button"
						/>
					</slot>
				{/if}
				<div class={state.classContent} class:content={!unstyled} style={cssContent}>
					<slot name="modalContent" />
				</div>
			</div>
		</div>
	</div>
{/if}
<slot name="alwaysOn" />

<style>
	* {
		box-sizing: border-box;
	}

	.bg {
		position: fixed;
		z-index: 1000;
		top: 0;
		left: 0;
		display: flex;
		flex-direction: column;
		justify-content: center;
		width: 100vw;
		height: 100vh;
		background: rgba(0, 0, 0, 0.66);
	}

	.wrap {
		position: relative;
		margin: 2rem;
		max-height: 100%;
	}

	.window {
		position: relative;
		width: 40rem;
		max-width: 100%;
		max-height: 100%;
		margin: 2rem auto;
		color: black;
		border-radius: 0.5rem;
		background: white;
	}

	.content {
		position: relative;
		padding: 1rem;
		max-height: calc(100vh - 4rem);
		overflow: auto;
	}

	.close {
		display: block;
		box-sizing: border-box;
		position: absolute;
		z-index: 1000;
		top: 1rem;
		right: 1rem;
		margin: 0;
		padding: 0;
		width: 1.5rem;
		height: 1.5rem;
		border: 0;
		color: black;
		border-radius: 1.5rem;
		background: white;
		box-shadow: 0 0 0 1px black;
		transition: transform 0.2s cubic-bezier(0.25, 0.1, 0.25, 1),
			background 0.2s cubic-bezier(0.25, 0.1, 0.25, 1);
		-webkit-appearance: none;
	}

	.close:before,
	.close:after {
		content: '';
		display: block;
		box-sizing: border-box;
		position: absolute;
		top: 50%;
		width: 1rem;
		height: 1px;
		background: black;
		transform-origin: center;
		transition: height 0.2s cubic-bezier(0.25, 0.1, 0.25, 1),
			background 0.2s cubic-bezier(0.25, 0.1, 0.25, 1);
	}

	.close:before {
		-webkit-transform: translate(0, -50%) rotate(45deg);
		-moz-transform: translate(0, -50%) rotate(45deg);
		transform: translate(0, -50%) rotate(45deg);
		left: 0.25rem;
	}

	.close:after {
		-webkit-transform: translate(0, -50%) rotate(-45deg);
		-moz-transform: translate(0, -50%) rotate(-45deg);
		transform: translate(0, -50%) rotate(-45deg);
		left: 0.25rem;
	}

	.close:hover {
		background: black;
	}

	.close:hover:before,
	.close:hover:after {
		height: 2px;
		background: white;
	}

	.close:focus {
		border-color: #3399ff;
		box-shadow: 0 0 0 2px #3399ff;
	}

	.close:active {
		transform: scale(0.9);
	}

	.close:hover,
	.close:focus,
	.close:active {
		outline: none;
	}
</style>
