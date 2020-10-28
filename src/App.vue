<template>
  <div id="app">
    <div v-if="signedIn">
      <h1>Todo App</h1>
      <div>Hello, {{user.username}}</div>
      <input type="text" v-model="name" placeholder="Todo name">
      <input type="text" v-model="description" placeholder="Todo description">
      <button v-on:click="createTodo">Create Todo</button>
      <div v-for="item in todos" :key="item.id">
        <h3>{{ item.name }}</h3>
        <p>{{ item.description }}</p>
      </div>
      <!-- <button v-on:click="signOut">Sign out</button> -->
      <amplify-sign-out></amplify-sign-out>
    </div>
    <div v-else>
      <amplify-authenticator />
    </div>    
  </div>
</template>

<script>
import { API, Auth } from 'aws-amplify';
import { createTodo } from './graphql/mutations';
import { listTodos } from './graphql/queries';
import { onCreateTodo } from './graphql/subscriptions';
import { AmplifyEventBus } from 'aws-amplify-vue'

export default {
  name: 'app',
  data() {
    return {
      signedIn: false,
      user: '',
      name: '',
      description: '',
      todos: []
    }
  },
  async beforeCreate() {
    try {
      let cognitoUser = await Auth.currentAuthenticatedUser()
      this.signedIn = true
      this.user = cognitoUser
    } catch (err) {
      console.log('not auth')
      this.signedIn = false
    }
    AmplifyEventBus.$on('authState', async  info => {
      if (info === 'signedIn') {
        this.todos = []
        let cognitoUser = await Auth.currentAuthenticatedUser()
        this.signedIn = true
        this.user = cognitoUser
        const username = this.user.username
        const todos = await API.graphql({
          query: listTodos,
          variables: {filter: { username: { eq: username } } }, // filter samples=>https://docs.amplify.aws/cli/graphql-transformer/examples#blog-queries
        });
        this.todos = todos.data.listTodos.items;

      } else {
        this.signedIn = false
      }
    });    
  },
  created() {
    this.getTodos();
    this.subscribe();
  },
  mounted() {
    console.log("mounted")
  },
  // async updated() {
  //   const username = this.user.username
  //   const todos = await API.graphql({
  //     query: listTodos,
  //     variables: {filter: { username: { eq: username } } }, // filter samples=>https://docs.amplify.aws/cli/graphql-transformer/examples#blog-queries
  //   });
  //   this.todos = todos.data.listTodos.items;
  // },
  methods: {
    signOut() {
      console.log("sign out")
      this.todos = []
      this.signedIn = false
      Auth.signOut()
      this.beforeCreate()
    },
    async createTodo() {
      const { user, name, description } = this;
      const username = user.username
      if (!name || !description || !user.username) return;
      const todo = { username, name, description };
      await API.graphql({
        query: createTodo,
        variables: {input: todo},
      });
      this.name = '';
      this.description = '';
    },
    async getTodos() {
      const user =  await Auth.currentAuthenticatedUser()
      const username = user.username
      const todos = await API.graphql({
        query: listTodos,
        variables: {filter: { username: { eq: username } } }, // filter samples=>https://docs.amplify.aws/cli/graphql-transformer/examples#blog-queries
      });
      this.todos = todos.data.listTodos.items;
    },
    subscribe() {
      API.graphql({ query: onCreateTodo })
        .subscribe({
          next: (eventData) => {
            let todo = eventData.value.data.onCreateTodo;
            if (this.todos.some(item => item.name === todo.name)) return; // remove duplications
            this.todos = [...this.todos, todo];
          }
        });
    }    
  }
};
</script>