<script lang="ts">
	import.meta.hot?.invalidate();

	import WebGPU from "three/addons/capabilities/WebGPU.js";

	import type GameInterface from "./game/GameInterface";

	import GameWorker from "./game/GameWorker?worker";
	import NetWorker from "./game/networking/NetWorker?worker";

	let { gameInterface } = $props<{
		gameInterface: GameInterface;
	}>();

	let canvas = $state<HTMLCanvasElement>();

	$effect(() => {
		const gameWorker = new GameWorker();
		const netWorker = new NetWorker();
		const netChannel = new MessageChannel();

		gameWorker.onmessage = () => {
			gameWorker.onmessage = null;

			const offscreen = canvas!.transferControlToOffscreen();
			gameWorker.postMessage(
				{
					ev: "init",
					param: {
						canvas: offscreen,
						param: [
							window.innerWidth,
							window.innerHeight,
							window.devicePixelRatio,
						],
						netChannel: netChannel.port1,
					},
				},
				[offscreen, netChannel.port1],
			);

			document!.addEventListener("keydown", (e) => {
				if (e instanceof MouseEvent || document.activeElement != document.body)
					return;
				gameWorker.postMessage({
					ev: "INPUT_KEY_DOWN",
					param: e.key,
				});
			});

			document!.addEventListener("keyup", (e) => {
				if (e instanceof MouseEvent || document.activeElement != document.body)
					return;
				gameWorker.postMessage({
					ev: "INPUT_KEY_UP",
					param: e.key,
				});
			});

			let mousedown = false;
			canvas!.addEventListener("mousedown", (e) => {
				if (e.button == 1 || e.button > 2) return;
				mousedown = true;

				gameWorker.postMessage({
					ev: "INPUT_POINTER_PRESS",
					param: [e.button, e.clientX, e.clientY],
				});
			});
			canvas!.addEventListener("mouseup", (e) => {
				if (e.button == 1 || e.button > 2) return;
				mousedown = false;

				gameWorker.postMessage({
					ev: "INPUT_POINTER_RELEASE",
					param: [e.button, e.clientX, e.clientY],
				});

				if (document.pointerLockElement == canvas) {
					document.exitPointerLock();
				}
			});

			let lastX = 0;
			let lastY = 0;
			canvas!.addEventListener("mousemove", (e) => {
				if (lastX != e.clientX || lastY != e.clientY) {
					if (document.pointerLockElement != canvas && mousedown) {
						canvas!.requestPointerLock();
					}
				}

				lastX = e.clientX;
				lastY = e.clientX;

				if (!mousedown) return;
				gameWorker.postMessage({
					ev: "INPUT_POINTER_MOVE",
					param: [lastX, lastY, e.movementX, e.movementY],
				});
			});

			let lastLocked = false;
			const onPointerLockChange = () => {
				const locked = document.pointerLockElement == canvas;
				if (lastLocked == locked) return;
				lastLocked = locked;
				gameWorker.postMessage({
					ev: "INPUT_POINTER_LOCK",
					param: document.pointerLockElement == canvas,
				});
			};
			document!.addEventListener("pointerlockchange", onPointerLockChange);
			document!.addEventListener("pointerlockerror", onPointerLockChange);

			gameInterface.setGameWorker(gameWorker);
		};

		netWorker.onmessage = () => {
			netWorker.onmessage = null;

			netWorker.postMessage(
				{
					ev: "init",
					param: {
						gameChannel: netChannel.port2,
					},
				},
				[netChannel.port2],
			);

			gameInterface.setNetWorker(netWorker);
		};

		const onResize = () => {
			const { innerWidth: width, innerHeight: height } = window;
			gameWorker.postMessage({ ev: "dimensions", param: { width, height } });
		};
		window.addEventListener("resize", onResize);

		return () => {
			netWorker.postMessage({ ev: "destroy" });
			gameWorker.postMessage({ ev: "destroy" });
		};
	});
</script>

{#if WebGPU.isAvailable()}
	<canvas bind:this={canvas} class="w-screen h-screen absolute" />
{:else}
	<div class="m-auto">
		Your browser or device doesn't fulfill the <a
			href="https://caniuse.com/?search=web%20gpu"
			target="_blank"
			class="underline">requirments</a
		>.
	</div>
{/if}