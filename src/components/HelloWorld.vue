<template>
  <section class="todoapp">
    <header class="header">
      <h1>todos</h1>
      <input
        v-model="state.newTodo"
        class="new-todo"
        autofocus
        autocomplete="off"
        placeholder="What needs to be done?"
        @keyup="onNewTodoKeyup"
      >
    </header>
    <section
      v-if="state.todos.length"
      class="main"
    >
      <input
        id="toggle-all"
        v-model="state.allDone"
        class="toggle-all"
        type="checkbox"
      >
      <label for="toggle-all">Mark all as complete</label>
      <ul class="todo-list">
        <li
          v-for="todo in state.filteredTodos"
          :key="todo.id"
          class="todo"
          :class="{ completed: todo.completed, editing: todo == state.editedTodo }"
        >
          <div class="view">
            <input
              v-model="todo.completed"
              class="toggle"
              type="checkbox"
            >
            <label @dblclick="editTodo(todo)">{{ todo.title }}</label>
            <button
              class="destroy"
              @click="removeTodo(todo)"
            />
          </div>
          <input
            v-model="todo.title"
            class="edit"
            type="text"
            @blur="doneEdit(todo)"
            @keyup="onEditKeyup($event,todo)"
          >
        </li>
      </ul>
    </section>
    <footer
      v-if="state.todos.length"
      class="footer"
    >
      <span class="todo-count">
        <strong>{{ state.remaining }}</strong>
        <span>{{ state.remainingText }}</span>
      </span>
      <ul class="filters">
        <li>
          <a
            href="#/all"
            :class="{ selected: state.visibility == 'all' }"
          >All</a>
        </li>
        <li>
          <a
            href="#/active"
            :class="{ selected: state.visibility == 'active' }"
          >Active</a>
        </li>
        <li>
          <a
            href="#/completed"
            :class="{ selected: state.visibility == 'completed' }"
          >Completed</a>
        </li>
      </ul>
      <button
        v-if="state.todos.length > remaining"
        class="clear-completed"
        @click="removeCompleted"
      >
        Clear completed
      </button>
    </footer>
  </section>
</template>
<script>

import { onMounted, onUnmounted, reactive, computed, watchEffect } from 'vue';
import { filters, todoStorage, pluralize } from './util';


export default {

  setup() {
    const state = reactive({
      todos: todoStorage.fetch(),
      editedTodo: null,
      newTodo: '',
      beforeEditCache: '',
      visibility: 'all',
      remaining: computed(() => filters.active(state.todos).length),
      remainingText: computed(() => ` ${pluralize(state.remaining)} left`),
      filteredTodos: computed(() => filters[state.visibility](state.todos)),
      allDone: computed({
        get() {
          return state.remaining === 0;
        },
        set(value) {
          state.todos.forEach((todo) => {
            todo.completed = value;
          });
        }
      })
    });


    watchEffect(() => {
      todoStorage.save(state.todos);
    }, {
      deep: true
    });

    onMounted(() => {
      window.addEventListener('hashchange', onHashChange);
      onHashChange();
    });

    onUnmounted(() => {
      window.removeEventListener('hashchange', onHashChange);
    });

    function onHashChange() {
      const visibility = window.location.hash.replace(/#\/?/, '');
      if (filters[visibility]) {
        state.visibility = visibility;
      } else {
        window.location.hash = '';
        state.visibility = 'all';
      }
    }

    function onNewTodoKeyup(event) {
      if (event.code === 'Enter') {
        addTodo();
      }
    }

    function onEditKeyup(event, todo) {
      if (event.code === 'Enter') {
        doneEdit(todo);
      } else if (event.code === 'Escape') {
        cancelEdit(todo);
      }
    }

    function addTodo() {
      const value = state.newTodo && state.newTodo.trim();
      if (!value) {
        return;
      }
      state.todos.push({
        id: todoStorage.uid++,
        title: value,
        completed: false
      });
      state.newTodo = '';
    }

    function removeTodo(todo) {
      state.todos.splice(state.todos.indexOf(todo), 1);
    }

    function editTodo(todo) {
      state.beforeEditCache = todo.title;
      state.editedTodo = todo;
    }

    function doneEdit(todo) {
      if (!state.editedTodo) {
        return;
      }
      state.editedTodo = null;
      todo.title = todo.title.trim();
      if (!todo.title) {
        removeTodo(todo);
      }
    }

    function cancelEdit(todo) {
      state.editedTodo = null;
      todo.title = state.beforeEditCache;
    }

    function removeCompleted() {
      state.todos = filters.active(state.todos);
    }

    return {
      state,
      onNewTodoKeyup,
      onEditKeyup,
      removeTodo,
      editTodo,
      doneEdit,
      removeCompleted
    };
  }
};
</script>
