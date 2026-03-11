<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'
import { createDatabase } from 'storion'
import type { Database, StorionChangeEvent } from 'storion'

type Todo = {
  id: number
  title: string
  completed: boolean
}

type LogEntry = {
  message: string
  response: StorionChangeEvent
  time: string
}

const TABLE_NAME = 'todos'

const db = ref<Database | null>(null)
const todos = ref<Todo[]>([])
const newTitle = ref('')
const editingId = ref<number | null>(null)
const editTitle = ref('')
const isReady = ref(false)
const error = ref<string | null>(null)
const logEntries = ref<LogEntry[]>([])

let unsubscribe: (() => void) | null = null

const completedCount = computed(
  () => todos.value.filter((t) => t.completed).length,
)
const activeCount = computed(
  () => todos.value.filter((t) => !t.completed).length,
)

async function initDatabase(): Promise<Database> {
  const database = await createDatabase({
    name: 'vue-storion-todos',
    storage: 'localStorage',
  })

  const tables = await database.listTables()
  if (!tables.includes(TABLE_NAME)) {
    await database.createTable(TABLE_NAME, [
      { name: 'id', type: 'int' },
      { name: 'title', type: 'string' },
      { name: 'completed', type: 'boolean' },
    ])
  }

  return database
}

async function refreshTodos(localDb: Database): Promise<void> {
  const { rows } = await localDb.query(TABLE_NAME, {
    orderBy: [{ field: 'id', direction: 'asc' }],
  })
  todos.value = (rows as unknown as Todo[]).slice()
}

onMounted(async () => {
  try {
    const database = await initDatabase()
    db.value = database
    await refreshTodos(database)

    // Subscribe so both components (todo view and change log)
    // react to the same underlying todos table changes.
    unsubscribe = database.subscribe('todos', async (event) => {
      logEntries.value = [
        ...logEntries.value,
        {
          message: `todos: ${event.type}${
            event.rowId != null ? ` (id=${event.rowId})` : ''
          }`,
          response: event,
          time: new Date().toLocaleTimeString(),
        },
      ]
      await refreshTodos(database)
    })
  } catch (e) {
    console.error('Storion init failed', e)
    error.value = 'Failed to initialize database.'
  } finally {
    isReady.value = true
  }
})

onUnmounted(() => {
  unsubscribe?.()
  unsubscribe = null
})

async function addTodo() {
  if (!db.value) return
  const title = newTitle.value.trim()
  if (!title) return
  await db.value.insert(TABLE_NAME, { title, completed: false })
  newTitle.value = ''
  // refresh is triggered via subscription
}

function startEdit(todo: Todo) {
  editingId.value = todo.id
  editTitle.value = todo.title
}

function cancelEdit() {
  editingId.value = null
  editTitle.value = ''
}

async function saveEdit() {
  if (!db.value || editingId.value == null) return
  const title = editTitle.value.trim()
  if (title) {
    await db.value.update(TABLE_NAME, editingId.value, { title })
    // refresh is triggered via subscription
  }
  cancelEdit()
}

async function toggleCompleted(todo: Todo) {
  if (!db.value) return
  await db.value.update(TABLE_NAME, todo.id, { completed: !todo.completed })
  // refresh is triggered via subscription
}

async function remove(todo: Todo) {
  if (!db.value) return
  await db.value.delete(TABLE_NAME, todo.id)
  // refresh is triggered via subscription
}
</script>

<template>
  <div class="app-root">
    <h1>Vue Storion TODOs</h1>

    <p v-if="!isReady">Initializing database...</p>
    <p v-else-if="!db">Database not available.</p>

    <div v-if="error" class="error">
      {{ error }}
    </div>

    <div v-if="db" class="side-by-side">
      <section class="todo-view">
        <div class="component-box">
          <div class="todo-input">
            <input
              v-model="newTitle"
              type="text"
              placeholder="What needs to be done?"
              @keydown.enter.prevent="addTodo"
            />
            <button @click="addTodo">Add</button>
          </div>

          <ul class="todo-list">
            <li
              v-for="todo in todos"
              :key="todo.id"
              :class="{ completed: todo.completed }"
            >
              <input
                type="checkbox"
                :checked="todo.completed"
                @change="toggleCompleted(todo)"
              />

              <input
                v-if="editingId === todo.id"
                v-model="editTitle"
                class="edit-input"
                type="text"
                @keydown.enter.prevent="saveEdit"
                @keydown.esc.prevent="cancelEdit"
              />
              <span v-else class="title">
                {{ todo.title }}
              </span>

              <div class="actions">
                <template v-if="editingId === todo.id">
                  <button @click="saveEdit">Save</button>
                  <button @click="cancelEdit">Cancel</button>
                </template>
                <template v-else>
                  <button @click="startEdit(todo)">Edit</button>
                </template>
                <button class="danger" @click="remove(todo)">Delete</button>
              </div>
            </li>
          </ul>

          <footer class="footer">
            <span>{{ activeCount }} active</span>
            <span>{{ completedCount }} completed</span>
          </footer>

          <div class="component-label">Component 1</div>
        </div>
      </section>

      <aside class="log-view">
        <div class="component-box">
          <h2 class="log-title">Todos table changes</h2>
          <p class="log-hint">
            Events from <code>db.subscribe('todos', …)</code> — each change is
            logged below.
          </p>
          <ul class="log-list">
            <li
              v-for="entry in logEntries"
              :key="entry.time + entry.message"
              class="log-item"
            >
              <span class="log-time">{{ entry.time }}</span>
              <span class="log-message">{{ entry.message }}</span>
              <pre class="log-response">{{
                JSON.stringify(entry.response, null, 2)
              }}</pre>
            </li>
            <li v-if="logEntries.length === 0" class="log-empty">
              No events yet. Add, edit, or delete a todo to see events here.
            </li>
          </ul>

          <div class="component-label">Component 2</div>
        </div>
      </aside>
    </div>
  </div>
</template>

<style scoped>
.app-root {
  max-width: 960px;
  margin: 3rem auto;
  padding: 2rem;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 18px 45px rgba(15, 23, 42, 0.15);
  text-align: left;
}

.app-root h1 {
  margin-top: 0;
  margin-bottom: 1.5rem;
  font-size: 2rem;
  letter-spacing: -0.03em;
}

.error {
  margin-bottom: 1rem;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
  font-size: 0.9rem;
}

.side-by-side {
  display: flex;
  gap: 1.5rem;
  align-items: flex-start;
}

.todo-view {
  flex: 0 1 380px;
  min-width: 0;
}

.log-view {
  flex: 1 1 320px;
  min-width: 220px;
}

.component-box {
  border: 2px dotted #b0b0b0;
  border-radius: 10px;
  padding: 1rem;
  background: #f9fafb;
}

.component-label {
  margin-top: 0.4rem;
  font-size: 0.8rem;
  text-align: center;
  color: #666;
  font-style: italic;
}

.todo-input {
  display: flex;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.todo-input input[type='text'] {
  flex: 1;
  padding: 0.7rem 0.9rem;
  border-radius: 999px;
  border: 1px solid #e2e8f0;
  font-size: 0.95rem;
}

.todo-input input[type='text']:focus {
  outline: none;
  border-color: #22c55e;
  box-shadow: 0 0 0 2px rgba(34, 197, 94, 0.18);
}

.todo-input button {
  border-radius: 999px;
  padding-inline: 1.3rem;
  background: #22c55e;
  color: #ffffff;
  border: none;
  font-size: 0.95rem;
}

.todo-input button:hover {
  background: #16a34a;
}

.todo-list {
  list-style: none;
  padding: 0;
  margin: 0 0 1.25rem;
}

.todo-list li {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.6rem 0.3rem;
  border-bottom: 1px solid #f1f5f9;
}

.todo-list li:last-of-type {
  border-bottom: none;
}

.todo-list input[type='checkbox'] {
  width: 1rem;
  height: 1rem;
}

.title {
  flex: 1;
  font-size: 0.95rem;
}

.completed .title {
  text-decoration: line-through;
  color: #9ca3af;
}

.edit-input {
  flex: 1;
  padding: 0.4rem 0.6rem;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
  font-size: 0.95rem;
}

.actions {
  display: flex;
  gap: 0.4rem;
}

.actions button {
  padding: 0.3rem 0.7rem;
  font-size: 0.8rem;
  border-radius: 999px;
  border: 1px solid #e2e8f0;
  background: #f8fafc;
}

.actions button:hover {
  background: #e5e7eb;
}

.actions .danger {
  border-color: #fecaca;
  color: #b91c1c;
  background: #fef2f2;
}

.actions .danger:hover {
  background: #fee2e2;
}

.footer {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: #6b7280;
  margin-top: 0.5rem;
}

.log-title {
  margin: 0 0 0.25rem;
  font-size: 1rem;
}

.log-hint {
  margin: 0 0 0.75rem;
  font-size: 0.8rem;
  color: #6b7280;
}

.log-list {
  list-style: none;
  margin: 0;
  padding: 0;
}

.log-item {
  padding: 0.5rem 0;
  border-bottom: 1px solid #e5e7eb;
  font-size: 0.85rem;
}

.log-item:last-child {
  border-bottom: none;
}

.log-time {
  display: block;
  color: #9ca3af;
  font-size: 0.75rem;
  margin-bottom: 0.25rem;
}

.log-message {
  font-weight: 500;
  color: #22c55e;
}

.log-response {
  margin: 0.25rem 0 0;
  padding: 0.5rem;
  background: #ffffff;
  border-radius: 4px;
  border: 1px solid #e5e7eb;
  font-size: 0.75rem;
  white-space: pre-wrap;
  word-break: break-all;
}

.log-empty {
  padding: 1rem 0;
  color: #6b7280;
  font-style: italic;
}
</style>
