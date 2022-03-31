<script lang="ts">
  import { createMachine, assign } from 'xstate';
  import { useMachine } from '@xstate/svelte';

  interface AutocompleteSearchContext {
    inputValue: string;
    suggestions: string[];
    highlightedSuggestion: string | null;
    selectedValue: string | null;
  }

  type SEARCH_EVENT = { type: 'SEARCH'; query: string };
  type HIGHLIGHT_EVENT = { type: 'HIGHLIGHT'; suggestion: string };
  type SUBMIT_EVENT = { type: 'SUBMIT'; suggestion: string };
  type CANCEL_EVENT = { type: 'CANCEL' };

  type AutocompleteSearchEvent =
    | SEARCH_EVENT
    | HIGHLIGHT_EVENT
    | SUBMIT_EVENT
    | CANCEL_EVENT;

  const SEARCH = {
    target: 'debouncing',
    cond: 'isNonEmptyQuery',
    actions: 'updateQuery',
  };

  const autocompleteSearchMachine = createMachine<
    AutocompleteSearchContext,
    AutocompleteSearchEvent
  >(
    {
      id: 'autocomplete-search',
      initial: 'idle',
      context: {
        inputValue: '',
        suggestions: [],
        highlightedSuggestion: null,
        selectedValue: null,
      },
      states: {
        idle: {
          on: { SEARCH },
          entry: ['clear'],
        },
        debouncing: {
          on: {
            SEARCH: {
              target: 'debouncing',
              actions: 'updateQuery',
            },
            CANCEL: 'idle',
          },
          after: {
            300: 'searching',
          },
        },
        searching: {
          on: {
            SEARCH: [SEARCH, 'idle'],
            CANCEL: 'idle',
          },
          invoke: {
            src: 'fetchSuggestions',
            onDone: {
              target: 'suggesting',
              actions: 'updateSuggestions',
            },
          },
          entry: ['clearSelection', 'updateQuery'],
        },
        suggesting: {
          on: {
            SEARCH: [SEARCH, 'idle'],
            HIGHLIGHT: {
              target: 'suggesting',
              actions: 'updateHighlightedSuggestion',
            },
            SUBMIT: {
              target: 'suggesting',
              actions: 'updateSelectedValue',
            },
            CANCEL: 'idle',
          },
        },
      },
    },
    {
      services: {
        fetchSuggestions: (context, _) =>
          new Promise((resolve, _) =>
            setTimeout(
              () =>
                resolve([
                  context.inputValue,
                  context.inputValue + '-1',
                  context.inputValue + '-2',
                  context.inputValue + '-3',
                ]),
              1000
            )
          ),
      },
      actions: {
        updateSelectedValue: assign((context, event) => {
          if (event.type !== 'SUBMIT') return;
          if (event.suggestion)
            return {
              selectedValue: event.suggestion,
              inputValue: event.suggestion,
            };
          if (context.highlightedSuggestion)
            return {
              selectedValue: context.highlightedSuggestion,
              inputValue: context.highlightedSuggestion,
            };
          if (context.suggestions && context.suggestions.length > 0)
            return {
              selectedValue: context.suggestions[0],
              inputValue: context.suggestions[0],
            };
          return {
            selectedValue: '',
          };
        }),
        updateSuggestions: assign((_, event) => {
          return {
            suggestions: event.data,
          };
        }),
        updateHighlightedSuggestion: assign((_, event) => {
          if (event.type !== 'HIGHLIGHT') return;
          return {
            highlightedSuggestion: event.suggestion,
          };
        }),
        updateQuery: assign((_, event) => {
          if (event.type !== 'SEARCH') return;
          return {
            inputValue: event.query,
          };
        }),
        clearSelection: assign(() => {
          return {
            highlightedSuggestion: null,
            selectedValue: null,
          };
        }),
        clear: assign(() => {
          return {
            suggestions: [],
            highlightedSuggestion: null,
            selectedValue: null,
            inputValue: '',
          };
        }),
      },
      guards: {
        isNonEmptyQuery: (_, event) => {
          if (event.type !== 'SEARCH') return;
          return event.query && event.query.length > 0;
        },
      },
    }
  );

  const { state, send } = useMachine(autocompleteSearchMachine);

  $: selectedValue = $state.context.selectedValue;
</script>

<pre>{JSON.stringify(
    { value: $state.value, context: $state.context },
    null,
    2
  )}</pre>

<form
  on:submit={(e) => {
    e.preventDefault();
    send('SUBMIT');
  }}
  on:reset={() => send('CANCEL')}
>
  <input
    value={$state.context.inputValue}
    on:input={(e) => send({ type: 'SEARCH', query: e.target.value })}
  />
  <button type="submit">submit</button>
  <button type="reset">reset</button>
</form>

{#if !$state.context.selectedValue}
  <ul>
    {#each $state.context.suggestions as suggestion}
      <button
        class="suggestion"
        type="button"
        on:click={() => send({ type: 'SUBMIT', suggestion })}
        on:mouseenter={() => send({ type: 'HIGHLIGHT', suggestion })}
        style:font-weight={$state.context.highlightedSuggestion === suggestion
          ? 'bold'
          : 'normal'}
      >
        {suggestion}
      </button>
    {/each}
  </ul>
{/if}

selectedValue: {selectedValue}
