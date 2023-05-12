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
        window.alert('Fehlende Logondaten');
      }
      this.driver = neo4j.driver(this.uri, neo4j.auth.basic(this.username, this.password));
      this.connected = await this.driver.verifyAuthentication();
    }
  },
  beforeDestroy() {
    this.driver.close();
  }
}
</script>
