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

<template>
  <div>
    Data
    <div v-for="todo in data" :key="todo.id">
      <span @click="mutation.mutate(todo.id)" class="todo">{{ todo }}</span>
    </div>
  </div>
</template>

<style scoped>
.todo {
  cursor: pointer;
}
</style>
