<template>
  <ion-card>
    <ion-card-header>
      <ion-card-title>
        {{ $t('Timezone') }}
      </ion-card-title>
    </ion-card-header>
    <ion-card-content>
      {{ $t('The timezone you select is used to ensure automations you schedule are always accurate to the time you select.') }}
    </ion-card-content>
    <ion-item v-if="showBrowserTimeZone">
      <ion-label>
        <p class="overline">{{ $t("Browser TimeZone") }}</p>
        {{ browserTimeZone.id }}
        <p v-if="showDateTime">{{ getCurrentTime(browserTimeZone.id, dateTimeFormat) }}</p>
      </ion-label>
    </ion-item>
    <ion-item lines="none">
      <ion-label>
        <p class="overline">{{ $t("Selected TimeZone") }}</p>
        {{ currentTimeZoneId }}
        <p v-if="showDateTime">{{ getCurrentTime(currentTimeZoneId, dateTimeFormat) }}</p>
      </ion-label>
      <ion-button id="time-zone-modal" slot="end" fill="outline" color="dark">{{ $t("Change") }}</ion-button>
    </ion-item>
  </ion-card>
  <!-- Using inline modal(as recommended by ionic), also using it inline as the component inside modal is not getting mounted when using modalController -->
  <ion-modal ref="timeZoneModal" trigger="time-zone-modal" @didPresent="search()" @didDismiss="clearSearch()">
    <ion-header>
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-button @click="closeModal">
            <ion-icon :icon="closeOutline" />
          </ion-button>
        </ion-buttons>
        <ion-title>{{ $t("Select time zone") }}</ion-title>
      </ion-toolbar>
      <ion-toolbar>
        <ion-searchbar @ionFocus="selectSearchBarText($event)" :placeholder="$t('Search time zones')"  v-model="queryString" @keyup.enter="queryString = $event.target.value; findTimeZone()" @keydown="preventSpecialCharacters($event)" />
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <div>
        <ion-radio-group value="rd" v-model="timeZoneId">
          <ion-list v-if="showBrowserTimeZone">
            <ion-list-header>{{ $t("Browser time zone") }}</ion-list-header>
            <ion-item>
              <ion-radio label-placement="end" justify="start" :value="browserTimeZone.id">
                <ion-label>
                  {{ browserTimeZone.label }} ({{ browserTimeZone.id }})
                  <p v-if="showDateTime">{{ getCurrentTime(browserTimeZone.id, dateTimeFormat) }}</p>
                </ion-label>
              </ion-radio>
            </ion-item>
          </ion-list>
          <ion-list>
            <ion-list-header v-if="showBrowserTimeZone">{{ $t("Select a different time zone") }}</ion-list-header>
            <!-- Loading state -->
            <div class="empty-state" v-if="isLoading">
              <ion-item lines="none">
                <ion-spinner color="secondary" name="crescent" slot="start" />
                {{ $t("Fetching time zones") }}
              </ion-item>
            </div>
            <!-- Empty state -->
            <div class="empty-state" v-else-if="filteredTimeZones.length === 0">
              <p>{{ $t("No time zone found") }}</p>
            </div>
            <div v-else>
              <ion-item :key="timeZone.id" v-for="timeZone in filteredTimeZones">
                <ion-radio label-placement="end" justify="start" :value="timeZone.id">
                  <ion-label>
                    {{ timeZone.label }} ({{ timeZone.id }})
                    <p v-if="showDateTime">{{ getCurrentTime(timeZone.id, dateTimeFormat) }}</p>
                  </ion-label>
                </ion-radio>
              </ion-item>
            </div>
          </ion-list>
        </ion-radio-group>
      </div>

      <ion-fab vertical="bottom" horizontal="end" slot="fixed">
        <ion-fab-button :disabled="!currentTimeZoneId" @click="setUserTimeZone">
          <ion-icon :icon="saveOutline" />
        </ion-fab-button>
      </ion-fab>
    </ion-content>
  </ion-modal>
</template>

<script setup lang="ts">
import {
  IonButton,
  IonButtons,
  IonCard, 
  IonCardHeader,
  IonCardTitle,
  IonCardContent,
  IonContent,
  IonFab,
  IonFabButton,
  IonHeader,
  IonIcon,
  IonItem,
  IonLabel,
  IonList,
  IonListHeader,
  IonModal,
  IonRadio,
  IonRadioGroup,
  IonSearchbar,
  IonSpinner,
  IonTitle,
  IonToolbar
} from '@ionic/vue';
import { closeOutline, saveOutline } from "ionicons/icons";
import { appContext, useUserStore } from '../index';
import { computed, onBeforeMount, ref } from "vue";
import { getCurrentTime } from '../utils'

const appState = appContext.config.globalProperties.$store;
const userStore = useUserStore();

const userProfile: any = computed(() => appState.getters['user/getUserProfile'])
const timeZones = computed(() => userStore.getTimeZones)
const currentTimeZoneId = computed(() => userStore.getCurrentTimeZone)

const isLoading = ref(true);
const timeZoneModal = ref();
const queryString = ref('');
const filteredTimeZones = ref([])
const timeZoneId = ref('')
// Fetching timeZone of the browser
const browserTimeZone = ref({
  label: '',
  id: Intl.DateTimeFormat().resolvedOptions().timeZone
})

const emit = defineEmits(["timeZoneUpdated"])

const props = defineProps({
  showBrowserTimeZone: {
    type: Boolean,
    default: true
  },
  showDateTime: {
    type: Boolean,
    default: true
  },
  dateTimeFormat: {
    type: String,
    default: 't ZZZZ'
  }
})

const closeModal = () => {
  timeZoneModal.value.$el.dismiss(null, 'cancel');
}

onBeforeMount(async () => {
  isLoading.value = true;
  await userStore.getAvailableTimeZones();

  if(userProfile.value && userProfile.value.userTimeZone) {
    userStore.currentTimeZoneId = userProfile.value.userTimeZone
    timeZoneId.value = userProfile.value.userTimeZone
  }

  if(props.showBrowserTimeZone) {
    browserTimeZone.value.label = timeZones.value.find((timeZone: any) => timeZone.id.toLowerCase().match(browserTimeZone.value.id.toLowerCase()))?.label
  }

  findTimeZone();
  isLoading.value = false;
})

async function setUserTimeZone() {
  await userStore.setUserTimeZone(timeZoneId.value).then((tzId) => {
    emit('timeZoneUpdated', tzId);
  }).catch(err => err)
  closeModal();
}

function findTimeZone() {
  const searchedString = queryString.value.toLowerCase();
  filteredTimeZones.value = timeZones.value.filter((timeZone: any) => timeZone.id.toLowerCase().match(searchedString) || timeZone.label.toLowerCase().match(searchedString));

  if(props.showBrowserTimeZone) {
    filteredTimeZones.value = filteredTimeZones.value.filter((timeZone: any) => !timeZone.id.toLowerCase().match(browserTimeZone.value.id.toLowerCase()));
  }
}

async function selectSearchBarText(event: any) {
  const element = await event.target.getInputElement()
  element.select();
}

function preventSpecialCharacters($event: any) {
  // Searching special characters fails the API, hence, they must be omitted
  if(/[`!@#$%^&*()_+\-=\\|,.<>?~]/.test($event.key)) $event.preventDefault();
}

function search() {
  isLoading.value = true;
  findTimeZone();
  isLoading.value = false;
}

// clearing the data explicitely as the modal is mounted due to the component being mounted always
function clearSearch() {
  queryString.value = ''
  filteredTimeZones.value = []
  isLoading.value = true
}
</script>
