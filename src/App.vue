<template>
    <v-app class="app">
        <router-view></router-view>
    </v-app>
</template>

<script>
    export default {
        name: 'App',

        components: {},

        data: () => ({
            //
        }),
    };
</script>
<style>
    @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

    * {
        font-family: 'Press Start 2P', cursive;;
        user-select: none;
    }

    html, body {
        margin: 0;
        padding: 0;
        height: 100%;
    }

    .app {
        height: 100%;
    }
</style>
