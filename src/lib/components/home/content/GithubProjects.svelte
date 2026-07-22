<script lang="ts">
	/**
	 * Gitee 项目展示组件
	 *
	 * 通过 Gitee API 动态获取并展示最近更新的开源项目。
	 * 支持自动加载状态、骨架屏占位及响应式网格布局。
	 */
	import { onMount } from 'svelte';
	import { GitBranch, Star, GitFork, ArrowRight } from 'lucide-svelte';
	import SectionHeader from '$lib/components/home/content/common/SectionHeader.svelte';
	import ContentCard from '$lib/components/home/content/common/ContentCard.svelte';

	import Crossfade from '$lib/components/ui/effect/Crossfade.svelte';
	import Skeleton from '$lib/components/ui/feedback/Skeleton.svelte';
	import { loadJson } from '$lib/utils/network/loading';
	import { cn } from '$lib/utils/index';

	import Marquee from '$lib/components/ui/display/Marquee.svelte';

	interface GithubRepo {
		name: string;
		description: string;
		stars: number;
		forks: number;
		watchers: number;
		language: string;
		url: string;
		updatedAt: string;
	}

	function getGiteeUsername() {
		const configuredUsername = import.meta.env.VITE_GITEE_USERNAME as string | undefined;
		if (configuredUsername) return configuredUsername;

		const repoUrl = import.meta.env.VITE_REPO_URL as string | undefined;
		if (!repoUrl) return '';

		try {
			const parsedUrl = new URL(repoUrl);
			const segments = parsedUrl.pathname.split('/').filter(Boolean);
			if (segments[0] && parsedUrl.hostname.includes('gitee')) {
				return segments[0];
			}
		} catch {
			// 忽略无效 URL
		}

		return '';
	}

	/** Gitee 用户名，从环境变量或仓库地址自动解析 */
	const GITEE_USERNAME = getGiteeUsername();

	let githubData = $state<GithubRepo[]>([]);
	let loadingGithub = $state(true);

	/**
	 * 获取 Gitee 仓库数据
	 * 直接从 Gitee API 获取最近更新的项目。
	 */
	async function fetchGithubData() {
		try {
			if (!GITEE_USERNAME) {
				throw new Error('未配置 Gitee 用户名');
			}

			const data = await loadJson<any[]>(
				`https://gitee.com/api/v5/users/${GITEE_USERNAME}/repos?sort=updated&direction=desc&per_page=6`
			);

			if (Array.isArray(data) && data.length > 0) {
				githubData = data.map((repo: any) => ({
					name: repo.name,
					description: repo.description || 'No description provided.',
					stars: Number(repo.stargazers_count) || 0,
					forks: Number(repo.forks_count) || 0,
					watchers: Number(repo.watchers_count) || 0,
					language: repo.language || 'Code',
					url: repo.html_url || `https://gitee.com/${GITEE_USERNAME}/${repo.name}`,
					updatedAt: repo.updated_at || ''
				}));
				return;
			}

			throw new Error('Gitee 返回空仓库列表');
		} catch (e) {
			console.error('获取 Gitee 数据失败', e);
			githubData = [
				{
					name: 'LoadError',
					description: 'Failed to load Gitee data',
					stars: 114,
					forks: 514,
					watchers: 1919810,
					language: 'LoadError',
					url: `https://gitee.com/${GITEE_USERNAME || 'gitee'}`,
					updatedAt: new Date().toISOString()
				}
			];
		} finally {
			loadingGithub = false;
		}
	}

	onMount(() => {
		fetchGithubData();
	});
</script>

{#snippet projectCard(repo: GithubRepo | null, loading: boolean)}
	<ContentCard
		tag="div"
		class={cn(
			'group transition-all duration-300',
			loading ? 'h-[116px] border-border/50' : 'h-[116px] hover:border-purple-500/30'
		)}
		tilt={true}
		opaque={true}
	>
		<svelte:element
			this={loading ? 'div' : 'a'}
			href={repo?.url}
			target={repo?.url ? '_blank' : undefined}
			class={cn('block h-full w-full outline-none')}
		>
			<Crossfade key={loading ? 'loading' : 'loaded'} class="h-full w-full">
				{#if loading}
					<div class="flex h-full flex-col justify-between">
						<div class="flex shrink-0 items-start justify-between gap-2">
							<Skeleton class="h-5 w-2/3" />
							<Skeleton class="h-4 w-12 rounded-full" />
						</div>
						<div class="min-h-0 flex-1 space-y-1.5 py-2">
							<Skeleton class="h-3 w-full" />
							<Skeleton class="h-3 w-4/5" />
						</div>
						<div class="flex shrink-0 items-center gap-4">
							<Skeleton class="h-3 w-12" />
							<Skeleton class="h-3 w-12" />
						</div>
					</div>
				{:else if repo}
					<div class="flex h-full flex-col group-hover:no-underline">
						<!-- 顶部：项目名 + 语言 -->
						<div class="flex shrink-0 items-start justify-between gap-2">
							<h3
								class="w-0 min-w-0 flex-1 text-base font-semibold text-foreground transition-colors"
							>
								<Marquee text={repo.name} class="w-full" />
							</h3>
							<span
								class="shrink-0 rounded-full border border-border bg-secondary/20 px-2 py-0.5 text-[10px] whitespace-nowrap text-muted-foreground"
								>{repo.language || 'N/A'}</span
							>
						</div>

						<!-- 中部：描述 (自适应高度) -->
						<div class="mt-1 mb-1 min-h-0 flex-1 overflow-hidden text-sm text-muted-foreground">
							<Marquee
								text={repo.description || ''}
								direction="vertical"
								class="max-h-[43px] w-full"
								fadeSize="10%"
							/>
						</div>

						<!-- 底部：统计信息 -->
						<div class="mt-auto flex shrink-0 items-center justify-between">
							<div class="flex items-center gap-4 text-xs text-muted-foreground/70">
								<div class="flex items-center gap-1">
									<Star size={14} />
									<span>{repo.stars}</span>
								</div>
								<div class="flex items-center gap-1">
									<GitFork size={14} />
									<span>{repo.forks}</span>
								</div>
							</div>
							<div
								class="shrink-0 text-muted-foreground/50 transition-transform group-hover:translate-x-1 group-hover:text-purple-400"
							>
								<ArrowRight size={16} />
							</div>
						</div>
					</div>
				{/if}
			</Crossfade>
		</svelte:element>
	</ContentCard>
{/snippet}

<div class="pt-4">
	<SectionHeader
		icon={GitBranch}
		iconBgColor="bg-purple-500/20"
		iconColor="text-purple-400"
		titleKey="home.hero.github.title"
	/>

	<div class="grid grid-cols-1 gap-4 md:grid-cols-2">
		{#each loadingGithub ? Array(6).fill(null) : githubData as repo}
			{@render projectCard(repo, loadingGithub)}
		{/each}
	</div>
</div>
