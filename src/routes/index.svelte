<script lang="ts">
	import { createMachine, assign } from 'xstate';
	import { useMachine } from '@xstate/svelte';

	interface AutocompleteSearchContext {
		suggestions: string[];
		highlightedSuggestion: string;
		selected: string;
	}

	type SEARCH_EVENT = { type: 'SEARCH'; query: string };
	type HIGHLIGHT_EVENT = { type: 'HIGHLIGHT'; suggestion: string };
	type SUBMIT_EVENT = { type: 'SUBMIT'; suggestion: string };
	type CANCEL_EVENT = { type: 'CANCEL' };

	type AutocompleteSearchEvent = SEARCH_EVENT | HIGHLIGHT_EVENT | SUBMIT_EVENT | CANCEL_EVENT;

	const autocompleteSearchMachine = createMachine<
		AutocompleteSearchContext,
		AutocompleteSearchEvent
	>(
		{
			id: 'autocomplete-search',
			initial: 'idle',
			context: {
				suggestions: [],
				highlightedSuggestion: '',
				selected: ''
			},
			states: {
				idle: {
					on: {
						SEARCH: {
							target: 'search',
							cond: 'isNonEmptyQuery'
						}
					},
					entry: ['clearContext']
				},
				search: {
					on: {
						SEARCH: [
							{
								target: 'search',
								cond: 'isNonEmptyQuery'
							},
							{
								target: 'idle'
							}
						],
						CANCEL: {
							target: 'idle'
						}
					},
					invoke: {
						src: 'fetchSuggestions',
						onDone: {
							target: 'display_suggestions',
							actions: 'assignSuggestionsToContext'
						}
					},
					entry: ['clearSelection']
				},
				display_suggestions: {
					on: {
						SEARCH: [
							{
								target: 'search',
								cond: 'isNonEmptyQuery'
							},
							{
								target: 'idle'
							}
						],
						HIGHLIGHT: {
							target: 'display_suggestions',
							actions: 'assignHighlightedSuggestionToContext'
						},
						SUBMIT: {
							target: 'display_suggestions',
							actions: 'assignResultToContext'
						},
						CANCEL: {
							target: 'idle'
						}
					}
				}
			}
		},
		{
			services: {
				fetchSuggestions: (_, event: SEARCH_EVENT) =>
					new Promise((resolve, _) =>
						setTimeout(
							() =>
								resolve([event.query, event.query + '-1', event.query + '-2', event.query + '-3']),
							1000
						)
					)
			},
			actions: {
				assignResultToContext: assign((context, event) => {
					if (event.type !== 'SUBMIT') return;
					if (event.suggestion) return { selected: event.suggestion };
					if (context.highlightedSuggestion) return { selected: context.highlightedSuggestion };
					if (context.suggestions && context.suggestions.length > 0)
						return { selected: context.suggestions[0] };
					return {
						selected: ''
					};
				}),
				assignSuggestionsToContext: assign((_, event) => {
					return {
						suggestions: event.data
					};
				}),
				assignHighlightedSuggestionToContext: assign((_, event) => {
					if (event.type !== 'HIGHLIGHT') return;
					return {
						highlightedSuggestion: event.suggestion
					};
				}),
				clearSelection: assign(() => {
					return {
						highlightedSuggestion: '',
						selected: ''
					};
				}),
				clearContext: assign(() => {
					return {
						suggestions: [],
						highlightedSuggestion: '',
						selected: ''
					};
				})
			},
			guards: {
				isNonEmptyQuery: (_, event) => event.query && event.query.length > 0
			}
		}
	);

	const { state, send } = useMachine(autocompleteSearchMachine);

	$: result = $state.context.selected;
</script>

<pre>{JSON.stringify({ value: $state.value, context: $state.context }, null, 2)}</pre>

<form
	on:submit={(e) => {
		e.preventDefault();
		send('SUBMIT');
	}}
	on:reset={() => send('CANCEL')}
>
	<input on:input={(e) => send({ type: 'SEARCH', query: e.target.value })} />
	<button type="submit">submit</button>
	<button type="reset">reset</button>
</form>

<ul>
	{#each $state.context.suggestions as suggestion}
		<button
			class="suggestion"
			type="button"
			on:click={() => send({ type: 'SUBMIT', suggestion })}
			on:mouseenter={() => send({ type: 'HIGHLIGHT', suggestion })}
			style:font-weight={$state.context.highlightedSuggestion === suggestion ? 'bold' : 'normal'}
		>
			{suggestion}
		</button>
	{/each}

	result: {result}
</ul>

<style>
	.suggestion {
		border: none;
		background: none;
	}
</style>
