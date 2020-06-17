<template>
    <div class="home">
        <mine-sweeper></mine-sweeper>
    </div>
</template>

<script>
    import MineSweeper from '@/components/MineSweeper.vue'

    export default {
        name: 'Home',
        components: {
            MineSweeper
        }
    }
</script>
<style>
    .home {
        display: flex;
        justify-content: center;
        position: absolute;
        height: 100%;
        width: 100%;
    }
</style>
