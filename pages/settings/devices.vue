<template>
  <settings-page page-title="settings.devices.devices">
    <template #actions>
      <v-btn
        v-if="devices"
        color="error"
        class="ml-a"
        @click="deleteAllDevices"
      >
        {{ $t('settings.devices.deleteAll') }}
      </v-btn>
      <v-btn
        ref="noreferrer noopener"
        href="https://jellyfin.org/docs/general/server/devices.html"
        target="_blank"
      >
        {{ $t('settings.help') }}
      </v-btn>
    </template>
    <template #content>
      <v-col>
        <v-data-table
          :headers="headers"
          :items="devices"
          @click:row="setSelectedDevice"
        >
          <!-- eslint-disable-next-line vue/valid-v-slot -->
          <template #item.DateLastActivity="{ item }">
            <p class="text-capitalize-first-letter mb-0">
              {{
                $dateFns.formatRelative(
                  $dateFns.parseJSON(item.DateLastActivity),
                  new Date()
                )
              }}
            </p>
          </template>
        </v-data-table>
      </v-col>
      <v-dialog v-model="deviceInfoDialog" width="fit-content">
        <selected-device-info
          v-if="selectedDevice.Name"
          :selected-device="selectedDevice"
          :is-dialog="true"
          @close-dialog="closeDialog"
          @delete-selected="deleteSelectedDevice"
        />
      </v-dialog>
    </template>
  </settings-page>
</template>

<script lang="ts">
import Vue from 'vue';
import { mapActions } from 'vuex';
import { DeviceInfo } from '@jellyfin/client-axios';

export default Vue.extend({
  async asyncData({ $api }) {
    const devices = (await $api.devices.getDevices()).data.Items;

    return { devices };
  },
  data() {
    return {
      devices: [] as DeviceInfo[],
      selectedDevice: {} as DeviceInfo,
      deviceInfoDialog: false
    };
  },
  computed: {
    headers(): any[] {
      return [
        {
          text: this.$t('settings.devices.userName'),
          value: 'LastUserName'
        },
        { text: this.$t('settings.devices.deviceName'), value: 'Name' },
        { text: this.$t('settings.devices.appName'), value: 'AppName' },
        { text: this.$t('settings.devices.appVersion'), value: 'AppVersion' },
        {
          text: this.$t('settings.devices.lastActive'),
          value: 'DateLastActivity'
        }
      ];
    }
  },
  methods: {
    ...mapActions('snackbar', ['pushSnackbarMessage']),
    async deleteDevice(item: DeviceInfo): Promise<void> {
      try {
        await this.$api.devices.deleteDevice({
          id: item.Id || ''
        });

        this.pushSnackbarMessage({
          message: this.$t('settings.devices.deleteDeviceSuccess'),
          color: 'success'
        });

        this.devices = (await this.$api.devices.getDevices()).data.Items || [];
      } catch (error) {
        this.pushSnackbarMessage({
          message: this.$t('settings.devices.deleteDeviceError'),
          color: 'error'
        });

        // eslint-disable-next-line no-console
        console.error(error);
      }
    },
    async deleteAllDevices(): Promise<void> {
      try {
        await this.devices?.forEach(async (device) => {
          if (this.$store.state.deviceProfile.deviceId === device.Id) {
            return;
          }
          await this.$api.devices.deleteDevice({ id: device.Id || '' });
        });

        this.pushSnackbarMessage({
          message: this.$t('settings.devices.deleteAllDevicesSuccess'),
          color: 'success'
        });

        this.devices = (await this.$api.devices.getDevices()).data.Items || [];
      } catch (error) {
        this.pushSnackbarMessage({
          message: this.$t('settings.devices.deleteAllDevicesError'),
          color: 'error'
        });

        // eslint-disable-next-line no-console
        console.error(error);
      }
    },
    setSelectedDevice(device: DeviceInfo): void {
      this.selectedDevice = device;
      this.deviceInfoDialog = true;
    },
    closeDialog(): void {
      this.deviceInfoDialog = false;
      this.selectedDevice = {};
    },
    async deleteSelectedDevice(): Promise<void> {
      try {
        await this.$api.devices.deleteDevice({
          id: this.selectedDevice.Id || ''
        });

        this.pushSnackbarMessage({
          message: this.$t('settings.devices.deleteDeviceSuccess'),
          color: 'success'
        });

        this.selectedDevice = {};

        this.devices = (await this.$api.devices.getDevices()).data.Items || [];
      } catch (error) {
        this.pushSnackbarMessage({
          message: this.$t('deleteDeviceError'),
          color: 'error'
        });

        // eslint-disable-next-line no-console
        console.error(error);
      }
      this.deviceInfoDialog = false;
    },
    getDeviceIcon({ AppName, Name }: DeviceInfo): string {
      switch (AppName?.toLowerCase()) {
        case 'samsung smart tv':
          return 'mdi-television';
        case 'xbox one':
          return 'microsoft-xbox';
        case 'sony ps4':
          return 'mdi-sony-playstation';
        case 'kodi':
          return 'mdi-kodi';
        case 'jellyfin android':
        case 'android tv':
          return 'mdi-android';
        case 'jellyfin web':
        case 'jellyfin web (vue)':
          switch (Name?.toLowerCase()) {
            case 'opera':
            case 'opera tv':
            case 'opera android':
              return 'mdi-opera';
            case 'chrome':
            case 'chrome android':
              return 'mdi-google-chrome';
            case 'firefox':
            case 'firefox android':
              return 'mdi-firefox';
            case 'safari':
            case 'safari ipad':
            case 'safari iphone':
              return 'mdi-apple-safari';
            case 'edge chromium':
            case 'edge chromium android':
            case 'edge chromium ipad':
            case 'edge chromium iphone':
              return 'mdi-microsoft-edge';
            case 'edge':
              return 'mdi-microsoft-edge-legacy';
            default:
              return 'mdi-language-html5';
          }
        default:
          return 'mdi-devices';
      }
    }
  }
});
</script>
