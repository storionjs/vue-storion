## Vue 3 + Storion

Integrate the `@storion/storion` client‑side database with a Vue 3 application.

---

### Installation

```bash
npm install @storion/storion
```

or

```bash
yarn add @storion/storion
```

---

### Basic usage in Vue 3 (Composition API)

Create a composable that initializes a Storion database and exposes it to components.

```ts
// useStorionDb.ts
import { ref, onMounted } from 'vue';
import { createDatabase } from '@storion/storion';

export function useStorionDb() {
  const db = ref<any | null>(null);

  onMounted(async () => {
    db.value = await createDatabase({
      name: 'vue-demo',
      storage: 'localStorage',
    });
  });

  return { db };
}
```

Use the composable inside a component to create a table and load data:

```vue
<script setup lang="ts">
import { computed } from 'vue';
import { useStorionDb } from './useStorionDb';

type Todo = {
  id: number;
  title: string;
  done: boolean;
};

const { db } = useStorionDb();
const todos = computed<Todo[]>(() => (db.value?._todos as Todo[]) ?? []);

async function init() {
  if (!db.value) return;

  await db.value.createTable('todos', [
    { name: 'id', type: 'int' },
    { name: 'title', type: 'string' },
    { name: 'done', type: 'boolean' },
  ]);

  const rows = await db.value.fetch('todos');
  (db.value as any)._todos = rows;
}

async function addTodo() {
  if (!db.value) return;
  await db.value.insert('todos', { title: 'New todo', done: false });
  const rows = await db.value.fetch('todos');
  (db.value as any)._todos = rows;
}

init();
</script>

<template>
  <section>
    <h2>Todos</h2>
    <button @click="addTodo">Add todo</button>
    <ul>
      <li v-for="t in todos" :key="t.id">
        {{ t.title }} {{ t.done ? '✅' : '⬜️' }}
      </li>
    </ul>
  </section>
</template>
```

This example keeps rows in a temporary `_todos` property on the database instance for simplicity; in a real app you would likely store them in Vue state (`ref` / `reactive`) instead.

---

### Vue + Storion patterns

- **App‑level plugin**: create the `Database` in a plugin and expose it via `app.config.globalProperties` or provide/inject.
- **Reactivity**: bridge `db.subscribe` events into Vue reactivity by updating `ref`/`reactive` state whenever a change event arrives.
- **Config‑driven schema**: use `createDatabase({ name, storage, config })` with a JSON config to define tables and columns.

For full API details, see the main Storion docs.