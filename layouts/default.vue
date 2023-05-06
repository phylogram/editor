<template>
  <v-app dark>
    <v-navigation-drawer
      v-model="drawer"
      fixed
      app
    >
     TODO
    </v-navigation-drawer>
    <v-app-bar
      fixed
      app
    >
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" />

    </v-app-bar>
    <v-main>
      <v-container>
        <Nuxt />
      </v-container>
    </v-main>
  </v-app>
</template>

<script>
export default {
  name: 'DefaultLayout',
  data () {
    return {
      drawer: false,
      fixed: false
    }
  }
}
</script>
