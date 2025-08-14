# TanStack Query `setQueriesData` with `BigInt` in `queryKey` Error

This repository demonstrates an issue with `@tanstack/query`'s `setQueriesData` method when using a `BigInt` in the `queryKey` along with a custom `queryKeyHashFn`.

## The Problem

When `setQueriesData` is called with filters (e.g., `{ queryKey: ['todos'], exact: false }`), it attempts to match and update existing queries. However, it appears that `setQueriesData` does not utilize the `queryKeyHashFn` provided to the original `useQuery` hook during this matching and update process.

If a `queryKey` contains a data type that requires custom serialization, like `BigInt`, this results in a serialization error because the default `JSON.stringify` is used internally by `setQueriesData`, which does not know how to handle `BigInt`.

In this example, we have a query with `queryKey: ['todos', BigInt(1)]` and a `queryKeyHashFn` to handle the `BigInt`. When a mutation succeeds, we call `setQueriesData` to update all queries that start with `['todos']`. This triggers the error.

## Steps to Reproduce

1.  Clone the repository.
2.  Install dependencies with `npm install`.
3.  Run the development server with `npm run dev`.
4.  Open the application in your browser.
5.  Open the developer console.
6.  Click on one of the todo items.

## Behavior

-   **Expected:** The query data is updated, and the UI reflects the change without any errors in the console.
-   **Actual:** An error is thrown and caught in the `onSuccess` handler of the mutation: `TypeError: Do not know how to serialize a BigInt`. The UI is not updated via the query cache.

## Relevant Code (`src/App.vue`)

```vue
<script setup>
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query';
import { ref, watchEffect } from 'vue';

const queryClient = useQueryClient();

const todos = ref([
  { id: 1, name: 'todo 1' },
  { id: 2, name: 'todo 2' },
  { id: 3, name: 'todo 3' },
]);

const jsonStringifyBigintCompatible = (obj) =>
  JSON.stringify(obj, (key, value) =>
    typeof value === 'bigint' ? value.toString() : value
  );

const { data } = useQuery({
  queryKey: ['todos', BigInt(1)],
  queryKeyHashFn: jsonStringifyBigintCompatible,
  queryFn: () => todos.value,
});

const mutation = useMutation({
  mutationFn: (id) =>
    (todos.value = todos.value.map((todo) => {
      if (todo.id === id) return { id, name: todo.name + 'edited' };
      else return todo;
    })),
  onSuccess: (_, id) => {
    try {
      // This is where the error occurs
      queryClient.setQueriesData(
        { queryKey: ['todos'], exact: false },
        (oldData) => {
          if (!oldData) return;

          return oldData.map((todo) => {
            if (todo.id === id) return { id, name: todo.name + 'edited' };
            else return todo;
          });
        }
      );
    } catch (e) {
      // Error: JSON.stringify - Do not know how to serialize a BigInt
      console.error(e);
    }
  },
});
</script>
```
