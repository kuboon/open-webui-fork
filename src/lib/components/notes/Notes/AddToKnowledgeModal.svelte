<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { getContext, createEventDispatcher } from 'svelte';
	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import { searchKnowledgeBases, addNoteToKnowledgeById } from '$lib/apis/knowledge';

	import Modal from '$lib/components/common/Modal.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';
	import Search from '$lib/components/icons/Search.svelte';
	import Database from '$lib/components/icons/Database.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';

	export let show = false;
	export let noteId: string;

	let query = '';
	let items: any[] = [];
	let loading = false;
	let submitting = false;
	let searchDebounceTimer: ReturnType<typeof setTimeout>;

	const loadItems = async () => {
		loading = true;
		const res = await searchKnowledgeBases(localStorage.token, query || null).catch(() => null);
		if (res) {
			items = (res.items ?? []).filter((kb: any) => kb.write_access);
		} else {
			items = [];
		}
		loading = false;
	};

	const onQueryChange = (_q: string) => {
		clearTimeout(searchDebounceTimer);
		searchDebounceTimer = setTimeout(() => {
			loadItems();
		}, 300);
	};

	let prevShow = false;
	$: if (show !== prevShow) {
		prevShow = show;
		if (show) {
			query = '';
			loadItems();
		}
	}

	$: if (show) onQueryChange(query);

	const submit = async (knowledgeId: string) => {
		if (submitting) return;
		submitting = true;
		const res = await addNoteToKnowledgeById(localStorage.token, knowledgeId, noteId).catch(
			(err) => {
				toast.error(`${err}`);
				return null;
			}
		);
		submitting = false;
		if (res) {
			toast.success($i18n.t('Note added to knowledge collection'));
			dispatch('added', { knowledge_id: knowledgeId });
			show = false;
		}
	};
</script>

<Modal size="sm" bind:show>
	<div class="px-5 py-4 flex flex-col gap-3 dark:text-gray-200">
		<div class="flex justify-between items-center">
			<div class="text-lg font-semibold">{$i18n.t('Add note to knowledge')}</div>
			<button
				class="self-center dark:text-white"
				type="button"
				on:click={() => {
					show = false;
				}}
			>
				<XMark className="size-3.5" />
			</button>
		</div>

		<div class="flex items-center gap-2 px-3 py-1.5 rounded-xl bg-gray-50 dark:bg-gray-850">
			<Search className="size-4" />
			<input
				class="w-full text-sm bg-transparent outline-none"
				type="text"
				bind:value={query}
				placeholder={$i18n.t('Search knowledge collections')}
			/>
		</div>

		<div class="max-h-72 overflow-y-auto flex flex-col gap-1">
			{#if loading}
				<div class="flex justify-center py-6"><Spinner /></div>
			{:else if items.length === 0}
				<div class="text-center text-xs text-gray-500 py-6">
					{$i18n.t('No knowledge collections found')}
				</div>
			{:else}
				{#each items as item (item.id)}
					<button
						type="button"
						disabled={submitting}
						class="flex items-center gap-2 w-full text-left px-3 py-2 rounded-xl hover:bg-gray-50 dark:hover:bg-gray-850 disabled:opacity-50"
						on:click={() => submit(item.id)}
					>
						<Database className="size-4 shrink-0" />
						<div class="flex-1 min-w-0">
							<div class="text-sm font-medium line-clamp-1">{item.name}</div>
							{#if item.description}
								<div class="text-xs text-gray-500 line-clamp-1">{item.description}</div>
							{/if}
						</div>
					</button>
				{/each}
			{/if}
		</div>
	</div>
</Modal>
