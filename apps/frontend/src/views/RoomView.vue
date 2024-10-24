<template>
    <div v-if="meOccupant" class="room-layout" :class="{ 'is-stage-fullscreen':  isStageFullscreen }">
        <div class="room-container">
            <div class="room-seats">
                <div class="scroll-area">
                    <div class="seats-grid">
                        <Seat v-for="seat of seats" :key="seat.id" :seat="seat"></Seat>
                    </div>
                </div>
            </div>
            <div ref="elStage" class="room-stage">
                <div class="stage-screen">
                    <div class="fuzzy-overlay"></div>
                    <div class="vignette"></div>
                </div>
                <div class="buttons-bar">
                    <VolumeButtonSlider variant="secondary" v-model="streamVolume" />
                   
                    <BaseButtonIcon :title="isStageFullscreen ? 'Exit fullscreen' : 'Enter fullscreen'" variant="secondary" @click="toggleFullScreen">
                        <IconCompress v-if="isStageFullscreen" />
                        <IconExpand v-else />
                    </BaseButtonIcon>
                </div>
            </div>
        </div>
        <div class="room-bottom-bar">
            {{ size }} users
            <AvatarOccupant v-if="meOccupant" :occupant="meOccupant">
                <template v-slot:badges>
                    <BadgeConnectionStatus :connection-status="!!meOccupant.connectionStatus" />
                </template>
            </AvatarOccupant>

            <BaseButtonIcon :title="meOccupant.micStatus ? 'mic on' : 'mic off'" :variant="meOccupant.micStatus ? 'status-on' : 'status-off'" @click="toggleMyMic">
                <IconMicrophone v-if="meOccupant.micStatus"/>
                <IconMicrophoneSlash v-else/>
            </BaseButtonIcon>
            <BaseButtonIcon title="cam off (soon)" :disabled="true">
                <IconVideoSlash/>
            </BaseButtonIcon>
            <BaseButtonIcon v-if="isHost(meOccupant)" :title="meOccupant.screenShareStatus ? 'film on' : 'film off'" :variant="meOccupant.screenShareStatus ? 'status-on' : 'status-off'" @click="toggleMySharedScreen">
                <IconFilm v-if="meOccupant.screenShareStatus" />
                <IconFilmSlash v-else />
            </BaseButtonIcon>
            <BaseButtonIcon title="sair da sala" variant="danger" @click="handleLeaveRoom">
                <IconDoorOpen />
            </BaseButtonIcon>
        </div>
        
        <ToastContainer class="bottom-toast-container" />
    </div>
    <div v-else>loading...
    </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref, toRefs, watchEffect } from 'vue';
import { v4 as uuidV4 } from 'uuid'
import ToastContainer from '../components/ToastContainer.vue';

import BaseButtonIcon from '../components/BaseButtonIcon.vue';
import IconMicrophoneSlash from '../components/icons/IconMicrophoneSlash.vue';
import IconVideoSlash from '../components/icons/IconVideoSlash.vue';
import IconFilm from '../components/icons/IconFilm.vue';
import IconDoorOpen from '../components/icons/IconDoorOpen.vue';
import IconExpand from '../components/icons/IconExpand.vue';
import Seat from '../components/Seat.vue';
import { isHost, type IOccupant, type IUser } from '../types';
import AvatarOccupant from '../components/AvatarOccupant.vue';
import BadgeConnectionStatus from '../components/BadgeConnectionStatus.vue';
import IconCompress from '../components/icons/IconCompress.vue';

import { useFullscreen } from '@vueuse/core'
import VolumeButtonSlider from '../components/VolumeButtonSlider.vue';
import IconMicrophone from '../components/icons/IconMicrophone.vue';
import IconFilmSlash from '../components/icons/IconFilmSlash.vue';

import { useToasts } from '../composables/useToasts';

import { useRoute } from 'vue-router';
import { useRoomStore } from '../stores/useRoomStore';
import { generateRandomUser } from '../helpers/randomUser';
import { useStorage } from '@vueuse/core'


const { addToast } = useToasts()

const route = useRoute();

const { room } = useRoomStore()
const { rId: roomId, clients, seats, state, socket } = toRefs(room)

const defaultMe: IUser = {
    id: uuidV4(),
    ...generateRandomUser(),
}

const meUser = useStorage('geraflix:auth-user', defaultMe, localStorage, { mergeDefaults: true })



const connected = computed(() => state.value.connected);
const size = computed(() => clients.value.size);

const streamVolume = ref(0.4)


const elStage = ref<HTMLElement | null>(null)

const { isFullscreen: isStageFullscreen, toggle: toggleFullScreen } = useFullscreen(elStage)



const meOccupant = computed<IOccupant | undefined>(() => {
    return room.clients.get(meUser.value.id)
})

watchEffect(() => {
    if(meOccupant.value) {
        meOccupant.value.roomId = roomId.value ?? ''
        meOccupant.value.socketId = socket.value.id ?? 'puts grila'
        meOccupant.value.connectionStatus = connected.value
        addToast({ message: `Conexão ${connected.value ? 'online' : 'offline'}`})
        console.log(meOccupant.value)
    }
})

function toggleMyMic() {
    if(meOccupant.value) {
        meOccupant.value.micStatus = !meOccupant.value.micStatus
    }
}

function toggleMySharedScreen() {
    if(meOccupant.value && isHost(meOccupant.value) ) {
        meOccupant.value.screenShareStatus = !meOccupant.value.screenShareStatus
    }
}

onMounted(() => {
    room.init(meUser.value.id, ''+route.params.roomId)
    room.active = true

    console.log('onMounted', 'auth-user', meUser.value, meOccupant.value)
    if(meUser.value) {
        room.joinRoom(
            meUser.value,
            'TODO-peerId'
        )
    }
})


room.socket.on("user-connected", (user) => {
    console.log(`[socket] user ${user.socketId} joined the room:`);
    clients.value.set(user.id, user)
    const emptySeat = room.findEmptySeat()
    if(emptySeat) {
        emptySeat.occupant = user
        console.log('encontrei uma cadeira vazia ', emptySeat)
    } else {
        console.error('Parece que não tem cadeira para o usuário...')
    }
    addToast({ message: `${user.name} entrou na reunião`})
    // connectToNewUser(user.peerId, camStream.value);
    // if(screenIsSharing.value) {
    //     connectToShareScreenWithUser(user.peerId, shareScreenStream.value)
    // }
});

room.socket.on("user-disconnected", (id) => {
    console.log(`[socket] user ${id} leaved the room`);
     
    const seat = room.findSeatOfUser(id)
    const name = (seat?.occupant?.name)
    
    if(seat) {
        seat.occupant = undefined
    }
    clients.value.delete(id)
    addToast({ message: `${name} saiu da reunião`})
    // removeAllRemoteStreamsByUser(userId)
    // _closeAllConnectionsFromUser(userId)
});

// TODO ao desligar a chamada é preciso garantir que todas as conexões são fechadas, inclusive a de compartilhamento de tela, se houver
function handleLeaveRoom() {
    // removeAllRemoteStreamsByUser(userId.value)
    // _closeAllConnectionsFromUser(userId.value)
    // TODO should destroy peer?
    // destroy() 
    
    // TODO shoud reset room data when leaving
    room.leaveRoom(meUser.value.id)
    room.active = false
    addToast({ message: `${meUser.value.name} saiu da reunião`})
}

</script>

<style scoped>
.room-layout {
    display: grid;
    grid-template-rows: 1fr 62px;
    height: 100vh;
    background-color: #303030;
}

.room-container {
    display: grid;
    grid-template-columns: 1fr 2fr;
    overflow: hidden;
}

.room-bottom-bar {
    background-color: rgba(0, 0, 0, 0.5);
    height: 62px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 15px;
    color: white;
}

.room-stage {
    position: relative;
    display: grid;
    place-items: center;
}

.room-seats {
    position: relative;
    overflow: hidden;
}

.seats-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 25px;
    padding: 60px 15px;
    max-width: 385px;
    margin: auto;
    /* height: 100%; */
}

.scroll-area {
    overflow-y: auto;
    height: 100%;
    max-height: 100%;
    
    display: grid;
    place-items: center
}

.stage-screen {
    background-color: rgb(0, 0, 0);
    width: 90%;
    height: 90%;
    position: relative;
    overflow: hidden;
    border-radius: 10px;
    background-color: black;
    background-image: url('../assets/standby.png');
    background-size: contain;
    background-position: center;
    box-shadow: -11px 7px 21px -10px rgba(0,0,0,0.98);

    transform: rotateY(-2deg); 
    transition: 
        transform .3s ease-in .3s, 
        width .3s ease-in 0s, 
        height .3s ease-in 0s,
        border-radius .1s linear;
}

.stage-screen .fuzzy-overlay {
    position: absolute;
    inset: -200%;
    opacity: 25%;
    background-image: url('../assets/noise.png');
    animation: fuzzy-overlay .2s linear infinite both;
    pointer-events: none;
}

@keyframes fuzzy-overlay {
    0% {
        transform: translateX(0%) translateY(0%);
    }
    100% {
        transform: translateX(-10%) translateY(-10%);
    }
}

.stage-screen .vignette {
    position: absolute;
    inset: 0;
    box-shadow: 0 0 200px rgba(0,0,0,0.9) inset;
    pointer-events: none;
}


.room-stage {
    perspective: 90px;
    perspective-origin: center left;
}

.is-stage-fullscreen .stage-screen {
    border-radius: 0px;
    transform: rotateY(0);
    width: 100%;
    height: 100%;

    transition: 
        transform .3s ease-in 0s, 
        width .3s ease-in .3s, 
        height .3s ease-in .3s,
        border-radius 0s linear .6s;
}

.room-stage .buttons-bar {
    position: absolute;
    bottom: 0;
    right: 0;
    display: flex;
    gap: 15px;
    padding: 15px;
}

.bottom-toast-container {
    position: absolute;
    bottom: 62px;
    left: 0;
    width: 100%;
}
</style>
