<script lang="ts">
	/**
	 * 地图信息窗体组件
	 *
	 * 模仿 GridPostCard 的样式，提供"液态玻璃"视觉效果的地图弹窗。
	 * 包含封面图（Placeholder）、标题、日期、描述和关闭按钮。
	 */
	import { formatDate } from '$lib/utils/datetime/date';
	import LiquidGlass from '$lib/components/ui/effect/LiquidGlass.svelte';
	import { ChevronLeft, ChevronRight, Images, X } from 'lucide-svelte';
	import { onDestroy } from 'svelte';
	import { fade } from 'svelte/transition';

	let { place, onClose } = $props<{
		place: any;
		onClose: () => void;
	}>();

	// 格式化日期
	let dateStr = $derived(place.visitDate ? formatDate(place.visitDate, 'zh-CN') : '');
	let imageList = $derived.by(() => {
		if (Array.isArray(place.images) && place.images.length > 0) return place.images;
		return place.image ? [place.image] : [];
	});
	let imageAlt = $derived(place.imageAlt || place.title || 'Footprint photo');
	let activeImageIndex = $state(0);
	let galleryOpen = $state(false);
	let carouselTimer: ReturnType<typeof setInterval> | null = null;

	$effect(() => {
		if (carouselTimer) {
			clearInterval(carouselTimer);
			carouselTimer = null;
		}

		activeImageIndex = 0;

		if (imageList.length > 1 && !galleryOpen) {
			carouselTimer = setInterval(() => {
				activeImageIndex = (activeImageIndex + 1) % imageList.length;
			}, 3200);
		}

		return () => {
			if (carouselTimer) {
				clearInterval(carouselTimer);
				carouselTimer = null;
			}
		};
	});

	onDestroy(() => {
		if (carouselTimer) clearInterval(carouselTimer);
	});

	function openGallery() {
		if (imageList.length > 0) galleryOpen = true;
	}

	function closeGallery() {
		galleryOpen = false;
	}

	function prevImage(event?: MouseEvent) {
		event?.stopPropagation();
		activeImageIndex = (activeImageIndex - 1 + imageList.length) % imageList.length;
	}

	function nextImage(event?: MouseEvent) {
		event?.stopPropagation();
		activeImageIndex = (activeImageIndex + 1) % imageList.length;
	}

	function handleGalleryKeydown(event: KeyboardEvent) {
		if (event.key === 'Escape') closeGallery();
		if (imageList.length <= 1) return;
		if (event.key === 'ArrowLeft') prevImage();
		if (event.key === 'ArrowRight') nextImage();
	}
</script>

<div class="relative w-[300px] origin-bottom animate-in duration-300 zoom-in-95">
	<LiquidGlass
		opaque={false}
		class="flex w-full flex-col gap-0 !p-0 shadow-xl"
		showLighting={true}
		showGloss={true}
	>
		<!-- 关闭按钮 -->
		<button
			class="absolute top-2 right-2 z-50 rounded-full p-1.5 text-muted-foreground transition-all hover:bg-black/5 hover:text-foreground"
			onclick={onClose}
			aria-label="关闭"
		>
			<X size={14} strokeWidth={2.5} />
		</button>

		<div class="relative flex w-full flex-col">
			{#if imageList.length > 0}
				<button
					type="button"
					class="group relative aspect-[16/9] w-full overflow-hidden rounded-t-[inherit] bg-muted text-left"
					onclick={openGallery}
					aria-label="打开地点图片"
				>
					{#key activeImageIndex}
						<img
							src={imageList[activeImageIndex]}
							alt={imageAlt}
							class="h-full w-full object-cover transition-transform duration-700 group-hover:scale-105"
							loading="lazy"
							referrerpolicy="no-referrer"
							in:fade={{ duration: 500 }}
							out:fade={{ duration: 500 }}
						/>
					{/key}

					{#if imageList.length > 1}
						<div
							class="absolute right-2 bottom-2 flex items-center gap-1 rounded-full bg-black/45 px-2 py-1 text-[11px] font-medium text-white backdrop-blur"
						>
							<Images size={12} />
							<span>{activeImageIndex + 1}/{imageList.length}</span>
						</div>
					{/if}
				</button>
			{/if}

			<!-- 内容区域 -->
			<div class="flex flex-1 flex-col bg-card/10 p-5 pt-7">
				<!-- 标题 -->
				<div class="mb-1 flex items-start justify-between gap-2">
					<h3 class="text-base leading-tight font-bold text-foreground">
						{place.title}
					</h3>
				</div>

				<div class="mb-3 flex items-center justify-between">
					{#if dateStr}
						<span class="text-[11px] font-semibold tracking-wider text-muted-foreground uppercase">
							{dateStr}
						</span>
					{/if}
				</div>

				{#if place.description}
					<p class="line-clamp-4 text-justify text-[13px] leading-relaxed text-muted-foreground">
						{place.description}
					</p>
				{/if}
			</div>
		</div>
	</LiquidGlass>

	<!-- 底部小三角指示器 (模拟气泡尾巴) -->
	<div
		class="absolute -bottom-2 left-1/2 z-[-1] h-4 w-4 -translate-x-1/2 rotate-45 border-r border-b border-border bg-card shadow-sm"
	></div>
</div>

{#if galleryOpen}
	<div
		class="fixed inset-0 z-modal flex items-center justify-center bg-black/80 p-4 backdrop-blur-md"
		role="dialog"
		aria-modal="true"
		tabindex="-1"
		onclick={closeGallery}
		onkeydown={handleGalleryKeydown}
		transition:fade={{ duration: 200 }}
	>
		<button
			type="button"
			class="absolute top-4 right-4 rounded-full bg-white/10 p-2 text-white transition hover:bg-white/20"
			onclick={closeGallery}
			aria-label="关闭图库"
		>
			<X size={20} />
		</button>

		{#if imageList.length > 1}
			<button
				type="button"
				class="absolute left-4 rounded-full bg-white/10 p-2 text-white transition hover:bg-white/20"
				onclick={prevImage}
				aria-label="上一张"
			>
				<ChevronLeft size={28} />
			</button>

			<button
				type="button"
				class="absolute right-4 rounded-full bg-white/10 p-2 text-white transition hover:bg-white/20"
				onclick={nextImage}
				aria-label="下一张"
			>
				<ChevronRight size={28} />
			</button>
		{/if}

		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<div
			class="flex max-h-[88vh] max-w-[92vw] flex-col items-center gap-3"
			onclick={(e) => e.stopPropagation()}
		>
			{#key activeImageIndex}
				<img
					src={imageList[activeImageIndex]}
					alt={imageAlt}
					class="max-h-[82vh] max-w-full rounded-lg object-contain shadow-2xl"
					referrerpolicy="no-referrer"
					in:fade={{ duration: 250 }}
					out:fade={{ duration: 250 }}
				/>
			{/key}

			{#if imageList.length > 1}
				<div class="rounded-full bg-white/10 px-3 py-1 text-sm text-white">
					{activeImageIndex + 1} / {imageList.length}
				</div>
			{/if}
		</div>
	</div>
{/if}
