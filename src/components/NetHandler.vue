<script setup lang="ts">
import { onMounted, onUnmounted, Ref, ref } from 'vue';
import PotatoNet from '../lib/net/PotatoNet';
import PotatoClient, { PotatoClientProcessing } from '../lib/net/PotatoClient';
import PotatoServer, { ServerNotePayload } from '../lib/net/PotatoServer';
import { NoteEventPayload } from '../lib/SoundHandler';
import MainEventHandler from '../lib/MainEventHandler';
import SettingsUtil from "../lib/SettingsUtil.ts";

let local_mode = SettingsUtil.get('local_mode')

let props = defineProps<{
    push_payload: (payload: NoteEventPayload | ServerNotePayload) => void
}>()
let url = new URL(window.location.href)
let urlRoom = url.searchParams.get("room") || "";
let roomId = ref("");
let processingRef: Ref<PotatoClientProcessing | null> = ref(null)
// let connectId = ref("");

function panic(msg: string) {
    console.warn(msg) // This should be shown to the user somehow after the location is replaced,,
    window.location.replace(window.location.pathname);
}

async function accepted() {
    let processing = processingRef.value
    if(!processing) {
        return panic("Accepted without processing?!?!")
    }
    roomId.value = urlRoom
    window.history.replaceState(roomId.value, roomId.value, `?room=${roomId.value}`)
    processing.on("notePayload", (event) => {
        props.push_payload(event);
    })
}

let server: PotatoServer | undefined;
let client: PotatoClient | undefined;
async function init() {
    if(processingRef.value) return;
    await PotatoNet.init();
    if(urlRoom === "" || local_mode) {
        server = await PotatoNet.initServer();
        urlRoom = server.peer.id;
        processingRef.value = server.localClient;
    } else {
        client = await PotatoNet.initClient(urlRoom);
        processingRef.value = client.processing;
    }
    let processing = processingRef.value
    processing.on("connectionClosed", () => {
        panic("Connection closed!")
    })
    if(!processing.accepted) {
        setInterval(() => {
            if(!processing.accepted) {
                panic("Server did not respond!");
            }
        }, 10000)
        processing.once("accepted", () => accepted())
    } else {
        accepted()
    }
}

// When a key is pressed by the user
function process_payload(payload: NoteEventPayload) {
    if(processingRef.value == null) return;
    processingRef.value.sendNotePayload(payload);
}

MainEventHandler.on("userNotePayload", process_payload)

async function unmounted() {
    if(server) {
        server.peer.destroy();
    }
    if(client) {
        client.peer.destroy();
    }
}

function copyRoomLink() {
    if(!processingRef) return;
    navigator.clipboard.writeText(window.location.href)
}

onMounted(init)
onUnmounted(unmounted)
</script>

<template>
    Room: <b><button id="copyRoomLink" class="btn btn-link" @click="copyRoomLink">{{ roomId || "Connecting..." }}</button></b>
    <div class="flex" v-if="processingRef !== null">
        <div v-for="user in processingRef.connected">
            <img :src="user.icon" width="64" height="64">
            <span :style="{color: user.color}">{{ user.display_name }}</span>
        </div>
    </div>
</template>

<style scoped>
    #copyRoomLink {
        padding: 0;
    }
</style>