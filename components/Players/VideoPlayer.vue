<template>
  <v-container fill-height fluid class="pa-0 justify-center">
    <video
      ref="videoPlayer"
      :poster="poster"
      autoplay
      @timeupdate="onVideoProgressThrottled"
      @pause="onVideoPause"
      @play="onPlay"
      @ended="onVideoStopped"
    />
  </v-container>
</template>

<script lang="ts">
import Vue from 'vue';
import { stringify } from 'qs';
import { throttle } from 'lodash';
// eslint-disable-next-line @typescript-eslint/ban-ts-comment
// @ts-ignore
import muxjs from 'mux.js';
import { mapActions, mapGetters, mapState } from 'vuex';
import {
  ImageType,
  PlaybackInfoResponse,
  RepeatMode
} from '@jellyfin/client-axios';
import { AppState } from '~/store';
import timeUtils from '~/mixins/timeUtils';
import imageHelper from '~/mixins/imageHelper';
import { MediaSourcePreferences } from '~/store/mediaSourcePreferences';

declare global {
  interface Window {
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    muxjs: any;
  }
}

export default Vue.extend({
  mixins: [imageHelper, timeUtils],
  data() {
    return {
      playbackInfo: {} as PlaybackInfoResponse,
      source: '',
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
      player: null as any,
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
      ui: null as any
    };
  },
  computed: {
    ...mapGetters('playbackManager', ['getCurrentItem']),
    ...mapState('playbackManager', ['currentTime', 'lastProgressUpdate']),
    poster(): string | undefined {
      return this.getImageUrlForElement(ImageType.Backdrop, {
        itemId:
          this.getCurrentItem?.ParentBackdropItemId ||
          this.getCurrentItem?.SeriesId ||
          this.getCurrentItem?.Id
      });
    }
  },
  watch: {
    getCurrentItem(): void {
      this.getPlaybackUrl();
    },
    async source(newSource): Promise<void> {
      if (this.player) {
        try {
          await this.player.load(newSource, this.currentTime);
        } catch (e) {
          // No need to actually process the error here, the error handler will do this for us
        }
      }
    }
  },
  async mounted() {
    try {
      const { default: shaka } = await import(
        // eslint-disable-next-line @typescript-eslint/ban-ts-comment
        // @ts-ignore
        'shaka-player/dist/shaka-player.compiled'
      );

      this.getPlaybackUrl();

      window.muxjs = muxjs;
      shaka.polyfill.installAll();
      if (shaka.Player.isBrowserSupported()) {
        this.player = new shaka.Player(this.$refs.videoPlayer);

        // Register player events
        this.player.addEventListener('error', this.onPlayerError);
        // Subscribe to Vuex actions
        this.$store.subscribe((mutation, _state: AppState) => {
          switch (mutation.type) {
            case 'playbackManager/PAUSE_PLAYBACK':
              if (this.$refs.videoPlayer) {
                (this.$refs.videoPlayer as HTMLVideoElement).pause();
              }
              break;
            case 'playbackManager/UNPAUSE_PLAYBACK':
              if (this.$refs.videoPlayer) {
                (this.$refs.videoPlayer as HTMLVideoElement).play();
              }
              break;
            case 'playbackManager/CHANGE_CURRENT_TIME':
              if (this.$refs.videoPlayer && mutation?.payload?.time) {
                (this.$refs.videoPlayer as HTMLVideoElement).currentTime =
                  mutation?.payload?.time;
              }
              break;
            case 'playbackManager/SET_REPEAT_MODE':
              if (this.$refs.videoPlayer) {
                if (mutation?.payload?.mode === RepeatMode.RepeatOne) {
                  (this.$refs.videoPlayer as HTMLVideoElement).loop = true;
                } else {
                  (this.$refs.videoPlayer as HTMLVideoElement).loop = false;
                }
              }
          }
        });
      } else {
        this.$nuxt.error({
          message: this.$t('browserNotSupported')
        });
      }
    } catch (error) {
      this.$nuxt.error({
        statusCode: 404,
        message: error
      });
    }
  },
  beforeDestroy() {
    if (this.player) {
      window.muxjs = undefined;
      this.onVideoStopped(); // Report that the playback is stopping
      this.player.removeEventListener('error', this.onPlayerError);
      this.player.unload();
      this.player.destroy();
    }
  },
  methods: {
    ...mapActions('snackbar', ['pushSnackbarMessage']),
    ...mapActions('playbackManager', [
      'unpause',
      'pause',
      'setNextTrack',
      'setMediaSource',
      'setVideoSource',
      'setAudioSource',
      'setSubtitleSource',
      'setCurrentTime',
      'setPlaySessionId',
      'setLastProgressUpdate'
    ]),
    async getPlaybackUrl(): Promise<void> {
      if (this.getCurrentItem) {
        this.playbackInfo = (
          await this.$api.mediaInfo.getPostedPlaybackInfo({
            itemId: this.getCurrentItem?.Id || '',
            userId: this.$auth.user?.Id,
            playbackInfoDto: { DeviceProfile: this.$playbackProfile }
          })
        ).data;

        this.setPlaySessionId({ id: this.playbackInfo.PlaySessionId });

        let mediaSource;
        if (this.playbackInfo?.MediaSources) {
          mediaSource = this.playbackInfo.MediaSources[0];
          this.setMediaSource({ mediaSource });
        } else {
          throw new Error("This item can't be played.");
        }

        const streamPreferences:
          | MediaSourcePreferences
          | undefined = mediaSource.Id
          ? this.$store.state.mediaSourcePreferences.preferences[mediaSource.Id]
          : undefined;

        if (streamPreferences) {
          const {
            videoStreamIndex,
            audioStreamIndex,
            subtitleStreamIndex
          } = streamPreferences;

          if (typeof videoStreamIndex === 'number') {
            this.setVideoSource({ streamIndex: videoStreamIndex });
          }
          if (typeof audioStreamIndex === 'number') {
            this.setAudioSource({ streamIndex: audioStreamIndex });
          }
          if (typeof subtitleStreamIndex === 'number') {
            this.setSubtitleSource({ streamIndex: subtitleStreamIndex });
          }
        }

        if (mediaSource.SupportsDirectStream) {
          const directOptions: Record<
            string,
            string | boolean | undefined | null
          > = {
            Static: true,
            mediaSourceId: mediaSource.Id,
            deviceId: this.$store.state.deviceProfile.deviceId,
            api_key: this.$store.state.user.accessToken
          };

          if (mediaSource.ETag) {
            directOptions.Tag = mediaSource.ETag;
          }

          if (mediaSource.LiveStreamId) {
            directOptions.LiveStreamId = mediaSource.LiveStreamId;
          }

          directOptions.VideoStreamIndex = streamPreferences?.videoStreamIndex?.toString();
          directOptions.AudioStreamIndex = streamPreferences?.audioStreamIndex?.toString();
          directOptions.SubtitleStreamIndex = streamPreferences?.subtitleStreamIndex?.toString();

          const params = stringify(directOptions);
          this.source = `${this.$axios.defaults.baseURL}/Videos/${mediaSource.Id}/stream.${mediaSource.Container}?${params}`;
        } else if (
          mediaSource.SupportsTranscoding &&
          mediaSource.TranscodingUrl
        ) {
          const transcodeOptions: Record<
            string,
            string | null | undefined
          > = {};
          transcodeOptions.VideoStreamIndex = streamPreferences?.videoStreamIndex?.toString();
          transcodeOptions.AudioStreamIndex = streamPreferences?.audioStreamIndex?.toString();
          transcodeOptions.SubtitleStreamIndex = streamPreferences?.subtitleStreamIndex?.toString();

          const params = stringify(transcodeOptions);

          this.source = `${this.$axios.defaults.baseURL}${mediaSource.TranscodingUrl}&${params}`;
        }
      }
    },
    onPlay(_event?: Event): void {
      this.unpause();
    },
    onVideoProgressThrottled: throttle(function (_event?: Event) {
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      this.onVideoProgress(_event);
    }, 500),
    onVideoProgress(_event?: Event): void {
      if (this.$refs.videoPlayer) {
        const currentTime = (this.$refs.videoPlayer as HTMLVideoElement)
          .currentTime;

        this.setCurrentTime({ time: currentTime });
      }
    },
    onVideoPause(_event?: Event): void {
      if (this.$refs.videoPlayer) {
        const currentTime = (this.$refs.videoPlayer as HTMLVideoElement)
          .currentTime;
        this.setCurrentTime({ time: currentTime });
        this.pause();
      }
    },
    onVideoStopped(_event?: Event): void {
      if (this.$refs.videoPlayer) {
        const currentTime = (this.$refs.videoPlayer as HTMLVideoElement)
          .currentTime;
        this.setCurrentTime({ time: currentTime });
        this.setNextTrack();
      }
    },
    onPlayerError(event: ErrorEvent): void {
      this.$emit('error', event);
    }
  }
});
</script>

<style scoped>
.shaka-video-container,
video {
  max-width: 100vw;
  max-height: 100vh;
  width: 100%;
  height: 100%;
}
</style>
