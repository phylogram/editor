<template>
  <v-app dark>
    <v-navigation-drawer
      v-model="drawer"
      fixed
      app
    >
      <v-container>
        <v-text-field v-model="username" label="user"></v-text-field>
        <v-text-field v-model="uri" label="uri"></v-text-field>
        <v-text-field v-model="password" label="password" type="password"></v-text-field>
        <button @click="login">Login</button>
      </v-container>
    </v-navigation-drawer>
    <v-app-bar
      fixed
      app
    >
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" :color="statusColor"/>

    </v-app-bar>
    <v-main>
      <v-container>
        <editor-playground v-if="connected" :driver="driver" />
      </v-container>
    </v-main>
  </v-app>
</template>

<script>
import * as neo4j from "neo4j-driver";

const USERNAME_KEY = "USERNAME_KEY";

const PASSWORD_KEY = "PASSWORD_KEY";

const URI_KEY = "URI_KEY";


export default {
  name: 'DefaultLayout',
  data() {
    return {
      drawer: false,
      username: null,
      password: null,
      uri: null,
      driver: null,
      connected: null,
    };
  },
  computed: {
    statusColor() {
      if (this.connected === true) {
        return 'green';
      }
      if (this.connected === false) {
        return 'red';
      }
      if (this.connected === null) {
        return 'yellow';
      }
      throw new Error('Check connection type!');
    }
  },
  methods: {
    async login() {
      if (!(this.uri && this.username && this.password)) {
        throw new Error('Missing login data data');
      }
      this.driver = neo4j.driver(this.uri, neo4j.auth.basic(this.username, this.password));
      this.connected = await this.driver.verifyAuthentication();
    },
    async tryToLoadLoginData() {
        this.uri = window.localStorage.getItem(URI_KEY);
        this.username = window.localStorage.getItem(USERNAME_KEY);
        this.password = window.localStorage.getItem(PASSWORD_KEY);
    },
    cleanUp() {
      this.driver.close();
      console.debug('Closed driver');
    },
  },
  watch: {
    uri() {
      window.localStorage.setItem(URI_KEY, this.uri);
    },
    username() {
      window.localStorage.setItem(USERNAME_KEY, this.username);
    },
    password() {
      window.localStorage.setItem(PASSWORD_KEY, this.password);
    },
  },
  beforeDestroy() {
    this.cleanUp();
  },

  mounted() {
    this.tryToLoadLoginData().then(this.login);
    window.onbeforeunload = () => this.cleanUp();
  }
}
</script>
