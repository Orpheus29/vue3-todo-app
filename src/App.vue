<template>
  <div class="todoapp">
    <h1 class="todoapp__title">todos</h1>

    <div class="todoapp__content">
      <header class="todoapp__header">
        <button
          class="todoapp__toggle-all"
          :class="{ 'active': activeTodos.length === 0 }"
          :style="{ visibility: todos.length ? 'visible' : 'hidden' }"
          @click="toggleAll"
        ></button>

        <form @submit.prevent="handleSubmit">
          <input
            type="text"
            class="todoapp__new-todo"
            placeholder="What needs to be done?"
            v-model="title"
          />
        </form>
      </header>

      <TransitionGroup
        name="list"
        tag="section"
        class="todoapp__main"
      >
        <TodoItem
          v-for="todo of visibleTodos"
          :key="todo.id"
          :todo="todo"
          :loadingTodos="loadingTodos"
          @update="handleUpdate"
          @delete="handleDelete(todo.id)"
        />
      </TransitionGroup>

      <footer class="todoapp__footer">
        <span class="todoapp__active-count">
          {{ activeTodos.length }} item{{ activeTodos.length !== 1 ? 's' : '' }} left
        </span>

        <StatusFilter v-model="status" />

        <button
          class="todoapp__clear-completed"
          :class="{ 'visible': completedTodos.length > 0 }"
          @click="clearAllCompleted"
        >
          Clear completed
        </button>
      </footer>
    </div>

    <Message
      class="is-warning"
      ref="errorMessage"
    >
      <template #default="{ text }">
        <p>{{ text }}</p>
        <!-- <button @click="$refs.errorMessage.hide()">x</button> -->
      </template>

      <template #header>
        <p>Server Error</p>
      </template>
    </Message>
  </div>
</template>

<script>
import { TodoItem, StatusFilter, Message } from '@/components';
import { getTodos, createTodo, updateTodo, deleteTodo } from './api/todos';

export default {
  components: {
    StatusFilter,
    TodoItem,
    Message,
  },
  data() {
    // FOR SAVING IN LOCALSTORAGE
    // let todos = [];
    // const jsonData = localStorage.getItem('todos') || '[]';

    // try {
    //   todos = JSON.parse(jsonData);
    // } catch (e) {}

    return {
      todos: [],
      title: '',
      status: 'all',
      errorMessage: '',
      loadingTodos: [],
    }
  },
  computed: {
    activeTodos() {
      return this.todos.filter(todo => !todo.completed);
    },
    completedTodos() {
      return this.todos.filter(todo => todo.completed);
    },
    visibleTodos() {
      switch (this.status) {
        case 'active':
          return this.activeTodos;
        case 'completed':
          return this.completedTodos;
        default:
          return this.todos;
      }
    },
  },
  mounted() {
    this.loadTodos();
  },
  watch: {
    // FOR SAVING IN LOCALSTORAGE
    // todos: {
    //   deep: true,
    //   handler() {
    //     localStorage.setItem('todos', JSON.stringify(this.todos));
    //   },
    // }
  },
  methods: {
    loadTodos() {
      getTodos()
        .then(({ data }) => this.todos = data)
        .catch(() => {
          this.$refs.errorMessage.show('Unable to load todos');
        });
    },
    handleSubmit() {
      if (this.title) {
        createTodo(this.title)
          .then(({ data }) => {
            this.todos = [...this.todos, data];
            this.title = '';
        });
      }
    },
    handleUpdate({ id, title, completed }) {
      this.loadingTodos.push(id);
      updateTodo({ id, title, completed })
        .then(({ data }) => {
          this.todos = this.todos.map(
            todo => todo.id !== id
            ? todo
            : data,
          )
        })
        .finally(() => {
          setTimeout(() => this.loadingTodos = [], 300);
        });
    },
    handleDelete(todoId) {
      this.loadingTodos.push(todoId);
      deleteTodo(todoId)
        .then(() => {
          this.todos = this.todos.filter(({ id }) => id !== todoId);
        })
        .finally(() => {
          setTimeout(() => this.loadingTodos = [], 300);
        });
    },
    toggleAll() {
      const completedValue = this.activeTodos.length !== 0;
      this.array = completedValue
        ? [...this.activeTodos]
        : [...this.completedTodos],
      
      Promise.all(
        this.array.map(todo => {
          this.loadingTodos.push(todo.id)
          updateTodo({
            id: todo.id,
            title: todo.title,
            completed: completedValue,
          })
          .then(updatedTodo => {
            todo.completed = updatedTodo.data.completed;
          })
          .catch(error => {
            console.error('Error toggling all todos:', error);
          })
          .finally(() => {
            setTimeout(() => this.loadingTodos = [], 300);
          });
        })
      )
    },
    clearAllCompleted() {
      Promise.all(
        this.completedTodos.map(({ id }) => {
          this.loadingTodos.push(id);
          deleteTodo(id);
        })
      )
        .catch(error => {
            console.error('Error deleting todos:', error);
          })
        .finally(() => {
          setTimeout(() => {
            this.loadingTodos = [];
            this.loadTodos();
          }, 300);
        })
    },
  },
}
</script>

<style>
.list-enter-active,
.list-leave-active {
  max-height: 60px;
  transition: all 0.5s ease;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  max-height: 0;
  transform: scaleY(0);
}
</style>
