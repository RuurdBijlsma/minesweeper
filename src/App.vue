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
    *{
        user-select: none;
    }
    html, body {
        margin: 0;
        padding: 0;
        height:100%;
    }

    .app {
        height:100%;
    }
</style>
