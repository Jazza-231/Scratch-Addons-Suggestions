<script lang="ts">
	import Load from "$lib/icons/Load.svelte";
	import { fade, fly } from "svelte/transition";
	import Suggestion from "./Suggestion.svelte";
	import { enhance } from "$app/forms";
	import type { Suggestion as SuggestionType, User } from "$lib/server/db/schema";
	import Select from "svelte-select";
	import { browser } from "$app/environment";
	import type { ActionResult } from "@sveltejs/kit";

	type returnedSuggestion = {
		suggestion: SuggestionType;
		user: Omit<User, "oauthProvider" | "oauthId"> | null;
	};

	const { data } = $props();
	const { getImages, session, message } = $derived(data);
	// prettier-ignore
	const sortItems = [{ value: "trending", label: "Trending" }, { value: "newest", label: "Newest" },
	{ value: "oldest", label: "Oldest" }, { value: "most", label: "Most upvoted" }, { value: "least", label: "Least upvoted" }];

	// prettier-ignore
	const filterItems = [{value: "all", label: "All"}, { value: "pending", label: "Pending" }, { value: "good", label: "Good Idea" },
	{ value: "implemented", label: "Implemented" }, { value: "in-dev", label: "In Development" }, { value: "incompatible", label: "Incompatible" },
	{ value: "impractical", label: "Impractical" }, { value: "rejected", label: "Rejected" }, { value: "impossible", label: "Impossible" }];

	let getSuggestions = $state(data.getSuggestions);
	let count = $state(data.count);
	let numOfPages = $state(0);

	$effect(() => {
		numOfPages = Math.ceil((count || 10) / 10);
	});

	let page = $state(1);
	let sort = $state(sortItems[0]);
	let filter = $state(filterItems[0]);
	let search = $state("");
	let navForm = $state() as HTMLFormElement;
	let filterForm = $state() as HTMLFormElement;
	let sortForm = $state() as HTMLFormElement;
	let loading = $state(false);
	let previousFormContent = $state("");

	let observer = $state() as IntersectionObserver;

	if (browser) {
		observer = new IntersectionObserver(
			([e]) => e.target.classList.toggle("is-pinned", e.intersectionRatio < 1),
			{ threshold: [1] },
		);
	}

	const useEnhanceCallback = (result: ActionResult) => {
		if (result.type === "success") {
			const data = result.data as {
				suggestions: returnedSuggestion[];
				page: number;
				sort: string;
				count: number;
			};

			if (data) {
				getSuggestions = (async () => {
					return data.suggestions;
				})();

				count = data.count;
				page = data.page;
			}
		}

		loading = false;
	};
</script>

{#snippet pageInput()}
	<input type="hidden" name="page" bind:value={page} />
{/snippet}

{#snippet sortInput()}
	<input
		type="hidden"
		name="sort"
		bind:value={() => JSON.stringify(sort), (v) => (sort = JSON.parse(v))}
	/>
{/snippet}

{#snippet filterInput()}
	<input
		type="hidden"
		name="filter"
		bind:value={() => JSON.stringify(filter), (v) => (filter = JSON.parse(v))}
	/>
{/snippet}

{#snippet searchInput()}
	<input type="hidden" name="search" bind:value={search} />
{/snippet}

{#snippet navFilterSearch()}
	<div class="filter" in:fly|global={{ duration: 400, y: 100 }} out:fade|global={{ duration: 200 }}>
		<form
			bind:this={filterForm}
			method="POST"
			action="?/filter"
			use:enhance={() => {
				loading = true;
				return async ({ result }) => {
					useEnhanceCallback(result);
				};
			}}
		>
			<div class="search">
				<input
					name="search"
					type="text"
					placeholder="Search for..."
					onblur={() => {
						const currentFormContent = filterForm.querySelector("input")!.value;
						if (currentFormContent !== previousFormContent) {
							filterForm.requestSubmit();
							previousFormContent = currentFormContent;
						}
					}}
					disabled={loading}
					bind:value={search}
				/>
				<button type="submit" formaction="?/search" hidden>Search</button>
			</div>

			<div class="select">
				{@render pageInput()}
				{@render sortInput()}
				<Select
					name="filter"
					items={filterItems}
					clearable={false}
					bind:value={filter}
					on:change={() => {
						filterForm.requestSubmit();
					}}
					showChevron={!loading}
					disabled={loading}
					bind:loading
				/>
			</div>
		</form>
	</div>
	<div
		class="allNav"
		use:observer.observe
		in:fly|global={{ duration: 400, y: 100 }}
		out:fade|global={{ duration: 200 }}
	>
		<div class="bottom">
			<div class="sort">
				<form
					method="POST"
					action="?/sort"
					bind:this={sortForm}
					use:enhance={() => {
						loading = true;
						return async ({ result }) => {
							useEnhanceCallback(result);
						};
					}}
				>
					{@render pageInput()}
					{@render filterInput()}
					{@render searchInput()}
					<Select
						name="sort"
						items={sortItems}
						required
						class="sort-select"
						on:change={() => {
							sortForm.requestSubmit();
						}}
						searchable
						bind:value={sort}
						clearable={false}
						showChevron={!loading}
						disabled={loading}
						bind:loading
					/>
				</form>
			</div>

			<div class="navButtons">
				<form
					method="POST"
					bind:this={navForm}
					use:enhance={() => {
						loading = true;
						return async ({ result }) => {
							useEnhanceCallback(result);
						};
					}}
				>
					{@render pageInput()}
					{@render sortInput()}
					{@render filterInput()}
					{@render searchInput()}

					<button type="submit" formaction="?/prev" disabled={page === 1}>Prev</button>
					<button type="submit" formaction="?/next" disabled={page >= numOfPages}>Next</button>
				</form>
			</div>
		</div>
	</div>
{/snippet}

{#if message}
	<div class="error">
		{message}
	</div>
{:else}
	<div class="suggestions-container">
		{#await getSuggestions}
			<div class="loading" in:fade|global={{ duration: 200 }} out:fade|global={{ duration: 200 }}>
				<Load />
			</div>
		{:then suggestions}
			{#if suggestions && getImages}
				{@render navFilterSearch()}

				<div class="suggestions">
					{#each suggestions as suggestion, index}
						<Suggestion {suggestion} {index} length={suggestions.length} {getImages} {session} />
					{/each}
				</div>
			{:else}
				<h2>No suggestions found</h2>
			{/if}
		{:catch error}
			<span class="error">An error occurred: {error.message}</span>
		{/await}
	</div>
{/if}

<style>
	.error {
		font-size: 2rem;
		color: red;
		text-align: center;
		margin-top: 2rem;
		width: 100%;
		display: block;
	}
	.suggestions-container {
		margin-block-start: 2rem;
		display: flex;

		flex-direction: column;
		align-items: center;
		margin-block-end: 2rem;

		.search {
			width: -webkit-fill-available;
			margin-right: 1rem;

			input {
				padding: 0.65rem;
				border-radius: 0.5rem;
				border: color-mix(in srgb, var(--brand) 20%, transparent) 2px solid;
				background-color: var(--background);
				color: var(--text);
				font-size: 1rem;
				width: 95%;
				transition:
					background-color var(--transition-short),
					border var(--transition-short),
					color var(--transition-short);

				&:disabled {
					border-color: var(--surface2);
				}

				&:hover {
					border: color-mix(in srgb, var(--brand) 40%, transparent) 2px solid;
				}

				&:focus {
					border: var(--brand) 2px solid;
					outline: none;
				}
			}
		}

		.filter {
			width: calc(60% + 2rem);

			@media (width <= 768px) {
				width: calc(90% + 2rem);
			}

			form {
				display: flex;
				justify-content: space-between;

				.select {
					width: 30%;

					@media (width <= 768px) {
						width: 50%;
					}
				}
			}
		}

		.allNav {
			display: flex;
			flex-direction: column;
			width: calc(60% + 2rem);
			position: sticky;
			z-index: 2;
			top: -1px;
			padding: 0.5rem 1rem 0 1rem;
			background-color: var(--background);
			margin-block-end: 2rem;
			transition:
				padding var(--transition-short),
				background-color var(--transition-short);

			@media (width <= 768px) {
				width: calc(100% - 2rem);
			}

			:global(&.is-pinned) {
				padding: 1rem;
			}

			& > div {
				display: flex;
				justify-content: space-between;
				align-items: center;
				width: 100%;

				@media (width <= 768px) {
					gap: 1rem;
				}

				.sort {
					:global {
						.svelte-select {
							width: 12rem;
						}
					}
				}

				.navButtons {
					display: flex;
					align-items: center;

					form {
						display: flex;
						justify-content: center;
						gap: 1rem;

						button {
							transition:
								opacity var(--transition-short),
								transform var(--transition-short);
							&:disabled {
								opacity: 0.5;
								transform: scale(1);
							}
						}
					}
				}
			}
		}

		.loading {
			display: flex;
			justify-content: center;
			height: 0px;

			/* Note: :global {svg {}} does not work in this case, it must be :global(svg) {} */
			:global(svg) {
				height: 3rem;
				width: 3rem;
				animation: spinner 1s linear infinite;

				:global(path) {
					fill: var(--brand);
				}
			}
		}

		.suggestions {
			display: flex;
			justify-content: center;
			flex-wrap: wrap;
			gap: 1rem;
			width: 100%;
		}
	}

	@keyframes spinner {
		0% {
			transform: rotate(360deg);
		}
		100% {
			transform: rotate(0deg);
		}
	}
</style>
