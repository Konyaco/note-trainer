<script setup>
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup
import Stave from "./components/Stave.vue";
import {darkTheme} from 'naive-ui';

</script>

<template>
    <n-config-provider style="width: 100%; height: 100%" :theme="darkTheme">
        <n-layout style="width: 100%; height: 100%">
            <Stave></Stave>
        </n-layout>
    </n-config-provider>
</template>

<style scoped>

</style>
