<script lang="ts">
	import '@fontsource/press-start-2p';
	import { Button } from '$lib/components/ui/button';
	import {
		displaySize,
		FileDropZone,
		MEGABYTE,
		type FileDropZoneProps
	} from '$lib/components/ui/file-drop-zone';
	import { Progress } from '$lib/components/ui/progress';
	import { sleep } from '$lib/utils/sleep';
	import XIcon from '@lucide/svelte/icons/x';
	import { onDestroy, onMount } from 'svelte';
	import { SvelteDate } from 'svelte/reactivity';
	import * as THREE from 'three';
	import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader.js';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

	const onUpload: FileDropZoneProps['onUpload'] = async (files) => {
		await Promise.allSettled(files.map((file) => uploadFile(file)));
	};
	const onFileRejected: FileDropZoneProps['onFileRejected'] = async ({ reason, file }) => {
	};
	const uploadFile = async (file: File) => {
		if (files.find((f) => f.name === file.name)) return;
		const urlPromise = new Promise<string>(async (resolve) => {
			await sleep(2000);
			resolve(URL.createObjectURL(file));
		});
		files.push({
			name: file.name,
			type: file.type,
			size: file.size,
			uploadedAt: Date.now(),
			url: urlPromise
		});
		await urlPromise;
	};
	type UploadedFile = {
		name: string;
		type: string;
		size: number;
		uploadedAt: number;
		url: Promise<string>;
	};
	let files = $state<UploadedFile[]>([]);
	let date = new SvelteDate();
	let processing = $state(false);
	let processingStartTime = $state(0);
	let avatarCreated = $state(false);
	let container = $state<HTMLDivElement | undefined>(undefined);
	
	let allFilesLoaded = $state(false);
	
	let currentStep = $state(0);
	let stepProgress = $state(0);
	
	let wireframeMode = $state(false);
	let meshColor = $state('#888888');
	let currentModel: THREE.Object3D | null = null;
	
	const modelUrl = '/src/lib/assets/face_center.obj';

	async function createAvatar() {
		processingStartTime = Date.now();
		processing = true;
		currentStep = 0;
		stepProgress = 0;
		
		currentStep = 1;
		await sleep(2000);
		
		currentStep = 2;
		await sleep(2000);
		
		currentStep = 3;
		await sleep(1000);
		
		processing = false;
		avatarCreated = true;
		currentStep = 0;
		stepProgress = 0;
	}

	let scene: THREE.Scene;
	let camera: THREE.PerspectiveCamera;
	let renderer: THREE.WebGLRenderer;
	let controls: OrbitControls;
	let resizeObserver: ResizeObserver;

	function initializeThreeJS() {
		if (!container) {
			console.warn('Three.js container not available');
			return;
		}

		try {
			scene = new THREE.Scene();
			camera = new THREE.PerspectiveCamera(
				75,
				container.clientWidth / container.clientHeight,
				0.1,
				1000
			);
			renderer = new THREE.WebGLRenderer({ antialias: true });
			renderer.setSize(container.clientWidth, container.clientHeight);
			renderer.setClearColor(0xf5f5f5, 1.0);
			container.appendChild(renderer.domElement);

			const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
			scene.add(ambientLight);
			
			const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
			directionalLight.position.set(5, 5, 5);
			scene.add(directionalLight);

			const loader = new OBJLoader();
			loader.load(
				modelUrl,
				(obj) => {
					currentModel = obj;
					
					const material = new THREE.MeshStandardMaterial({ 
						color: meshColor,
						metalness: 0.1,
						roughness: 0.3,
						side: THREE.DoubleSide,
						wireframe: wireframeMode
					});
					
					obj.traverse((child) => {
						if (child instanceof THREE.Mesh) {
							child.material = material;
						}
					});
					
					scene.add(obj);
					
					const box = new THREE.Box3().setFromObject(obj);
					const center = box.getCenter(new THREE.Vector3());
					const size = box.getSize(new THREE.Vector3());
					obj.position.sub(center);
					const maxDim = Math.max(size.x, size.y, size.z);
					const scale = 2 / maxDim;
					obj.scale.multiplyScalar(scale);
					
					console.log('OBJ model loaded successfully');
				},
				(progress) => {
					console.log('Loading progress:', (progress.loaded / progress.total * 100) + '%');
				},
				(error) => {
					console.error('Error loading OBJ model:', error);
					const geometry = new THREE.BoxGeometry(1, 1, 1);
					const material = new THREE.MeshLambertMaterial({ color: 0x808080 });
					const cube = new THREE.Mesh(geometry, material);
					scene.add(cube);
					console.log('Using fallback cube instead');
				}
			);

			camera.position.z = 4;

			controls = new OrbitControls(camera, renderer.domElement);
			controls.enableDamping = true;
			controls.dampingFactor = 0.25;
			controls.enableZoom = true;
			controls.enablePan = true;
			controls.minDistance = 1.5;
			controls.maxDistance = 8;

			function animate() {
				requestAnimationFrame(animate);
				controls.update();
				renderer.render(scene, camera);
			}
			animate();

			resizeObserver = new ResizeObserver(() => {
				if (container) {
					camera.aspect = container.clientWidth / container.clientHeight;
					camera.updateProjectionMatrix();
					renderer.setSize(container.clientWidth, container.clientHeight);
				}
			});
			resizeObserver.observe(container);

			console.log('Three.js scene initialized successfully');
		} catch (error) {
			console.error('Error initializing Three.js:', error);
		}
	}

	function cleanupThreeJS() {
		if (renderer) {
			renderer.dispose();
		}
		if (resizeObserver) {
			resizeObserver.disconnect();
		}
	}

	onMount(() => {
		initializeThreeJS();
		
		return () => {
			cleanupThreeJS();
		};
	});

	$effect(() => {
		if (avatarCreated && container) {
			cleanupThreeJS();
			setTimeout(() => {
				initializeThreeJS();
			}, 100);
		}
	});

	onDestroy(async () => {
		for (const file of files) {
			URL.revokeObjectURL(await file.url);
		}
	});
	$effect(() => {
		const interval = setInterval(() => {
			date.setTime(Date.now());
		}, 10);
		return () => {
			clearInterval(interval);
		};
	});

	$effect(() => {
		if (files.length === 0) {
			allFilesLoaded = false;
			return;
		}
		
		Promise.all(files.map(file => file.url)).then(() => {
			allFilesLoaded = true;
		}).catch(() => {
			allFilesLoaded = false;
		});
	});
	$effect(() => {
		if (!processing) return;
		
		const interval = setInterval(() => {
			const elapsed = Date.now() - processingStartTime;
			const stepDuration = currentStep === 1 ? 2000 : currentStep === 2 ? 2000 : 1000;
			const stepStart = currentStep === 1 ? 0 : currentStep === 2 ? 2000 : 4000;
			const stepElapsed = elapsed - stepStart;
			
			stepProgress = Math.min((stepElapsed / stepDuration) * 100, 100);
		}, 50);
		
		return () => clearInterval(interval);
	});

	$effect(() => {
		if (!currentModel) {
			console.log('No current model available for material update');
			return;
		}
		
		console.log('Updating material:', { meshColor, wireframeMode });
		
		currentModel.traverse((child) => {
			if (child instanceof THREE.Mesh) {
				console.log('Found mesh, updating material');
				
				let newMaterial;
				if (wireframeMode) {
					newMaterial = new THREE.MeshBasicMaterial({ 
						color: meshColor,
						wireframe: true,
						side: THREE.DoubleSide
					});
				} else {
					newMaterial = new THREE.MeshStandardMaterial({ 
						color: meshColor,
						metalness: 0.1,
						roughness: 0.3,
						side: THREE.DoubleSide,
						wireframe: false
					});
				}
				
				if (child.material) {
					child.material.dispose();
				}
				
				child.material = newMaterial;
				console.log('New material created and applied:', newMaterial);
			}
		});
	});
</script>

<div class="flex flex-col justify-center h-screen max-w-2xl gap-2 p-6 mx-auto">
	<h1 class="mb-4 text-4xl font-semibold text-center">Digital Avatar</h1>
	{#if !avatarCreated}
		<FileDropZone
			{onUpload}
			{onFileRejected}
			maxFileSize={2 * MEGABYTE}
			accept="image/*"
			maxFiles={4}
			fileCount={files.length}
		/>
		<div class="flex flex-col gap-2">
			{#each files as file, i (file.name)}
				<div class="flex justify-between gap-2 place-items-center">
					<div class="flex gap-2 place-items-center">
						{#await file.url then src}
							<div class="relative size-9 overflow-clip">
								<img
									{src}
									alt={file.name}
									class="absolute -translate-x-1/2 -translate-y-1/2 top-1/2 left-1/2 overflow-clip"
								/>
							</div>
						{/await}
						<div class="flex flex-col">
							<span>{file.name}</span>
							<span class="text-xs text-muted-foreground">{displaySize(file.size)}</span>
						</div>
					</div>
					{#await file.url}
						<Progress
							class="w-full h-2 grow"
							value={Math.min(((date.getTime() - file.uploadedAt) / 2000) * 100, 100)}
							max={100}
						/>
					{:then url}
						<Button
							variant="outline"
							size="icon"
							onclick={() => {
								URL.revokeObjectURL(url);
								files = [...files.slice(0, i), ...files.slice(i + 1)];
							}}
						>
							<XIcon />
						</Button>
					{/await}
				</div>
			{/each}
		</div>
		{#if allFilesLoaded && !processing}
			<Button
				class="mx-auto mt-6 transform rounded-none bg-black px-8 py-3 font-['Press_Start_2P'] text-lg font-medium text-white shadow-[2px_2px_0px_0px_#000] transition-all duration-150 hover:-translate-x-[2px] hover:-translate-y-[2px] hover:border-blue-400 hover:shadow-[2px_2px_0px_0px_#1e40af]"
				onclick={createAvatar}
			>
				Create Avatar
			</Button>
		{/if}
		{#if processing}
			<div class="mt-6 space-y-6 text-center">
				<h2 class="mb-6 font-['Press_Start_2P'] text-xl tracking-wider text-blue-600">
					CREATING AVATAR...
				</h2>
				<div class="flex justify-center mb-6 space-x-4">
					{#each [
						{ id: 1, label: 'Analyzing', icon: '🔍' },
						{ id: 2, label: 'Generating', icon: '🎨' },
						{ id: 3, label: 'Finalizing', icon: '✨' }
					] as step}
						<div class="flex flex-col items-center">
							<div class="w-12 h-12 rounded-full border-2 flex items-center justify-center text-lg transition-all duration-300 {
								currentStep >= step.id ? 'bg-blue-500 border-blue-500 text-white' : 'bg-gray-200 border-gray-300 text-gray-500'
							}">
								{step.icon}
							</div>
							<span class="mt-2 font-mono text-xs text-gray-600">{step.label}</span>
						</div>
					{/each}
				</div>

				<div class="mx-auto w-80">
					<div class="h-3 overflow-hidden bg-gray-200 rounded-full">
						<div 
							class="h-full transition-all duration-300 ease-out rounded-full bg-gradient-to-r from-blue-500 to-purple-600"
							style="width: {stepProgress}%"
						></div>
					</div>
					<div class="mt-2 font-mono text-sm text-gray-600">
						{#if currentStep === 1}
							Analyzing uploaded images... {Math.round(stepProgress)}%
						{:else if currentStep === 2}
							Generating 3D mesh... {Math.round(stepProgress)}%
						{:else if currentStep === 3}
							Applying textures and finalizing... {Math.round(stepProgress)}%
						{/if}
					</div>
				</div>

				<div class="flex justify-center mt-4">
					<div class="flex space-x-1">
						{#each Array(3) as _, i}
							<div 
								class="w-2 h-2 bg-blue-500 rounded-full animate-bounce"
								style="animation-delay: {i * 0.2}s;"
							></div>
						{/each}
					</div>
				</div>
			</div>
		{/if}
	{:else}
		<div class="w-full h-[600px]" bind:this={container}></div>
		
		<div class="flex flex-col gap-4 mt-4 p-4 bg-gray-50 rounded-lg border">
			<h3 class="text-lg font-semibold text-gray-700">Model Controls</h3>
			
			<div class="flex flex-col sm:flex-row gap-4 items-start sm:items-center">
				<div class="flex items-center gap-2">
					<label for="wireframe-toggle" class="text-sm font-medium text-gray-600">Wireframe:</label>
					<Button
						id="wireframe-toggle"
						variant={wireframeMode ? "default" : "outline"}
						size="sm"
						onclick={() => {
							console.log('Wireframe button clicked, current state:', wireframeMode);
							wireframeMode = !wireframeMode;
							console.log('Wireframe state changed to:', wireframeMode);
						}}
						class="px-3 py-1 text-xs {wireframeMode ? 'bg-red-500 text-white' : ''}"
					>
						{wireframeMode ? 'ON' : 'OFF'}
					</Button>
					{#if wireframeMode}
						<span class="text-xs text-red-500 font-semibold">●</span>
					{/if}
				</div>
				
				<div class="flex items-center gap-2">
					<label for="color-picker" class="text-sm font-medium text-gray-600">Color:</label>
					<input
						id="color-picker"
						type="color"
						bind:value={meshColor}
						onchange={() => console.log('Color changed to:', meshColor)}
						class="w-8 h-8 rounded border border-gray-300 cursor-pointer"
						title="Choose mesh color"
					/>
					<span class="text-xs text-gray-500 font-mono">{meshColor}</span>
				</div>
			</div>
		</div>
		
		<Button
			class="mx-auto mt-4 rounded-none border-2 border-gray-300 bg-gray-100 px-8 py-3 font-['Press_Start_2P'] text-lg font-medium text-gray-700 shadow-md transition-all duration-200 hover:bg-gray-200 hover:shadow-lg"
			variant="outline"
			onclick={() => {
				avatarCreated = false;
				processing = false;
				files = [];
				allFilesLoaded = false;
				currentModel = null;
				wireframeMode = false;
				meshColor = '#888888';
			}}
		>
			<XIcon class="inline w-5 h-5 mr-2" />
			New Avatar
		</Button>
	{/if}
</div>

<style>
	h1 {
		font-family: 'Press Start 2P', system-ui;
	}
	:global(body) {
		background-color: #f9f9f9;
		color: #3333ff;
	}


	:global(.w-22) {
		width: 5.5rem;
	}
	:global(.w-19) {
		width: 4.75rem;
	}
</style>
