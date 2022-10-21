<script setup>
import {
  ref,
  watch,
  onMounted,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
} from 'vue';

import muxjs from 'mux.js';
window.muxjs = muxjs;

import { useStore, useRoute } from '@/scripts/util.js';
import { useData, usePlayer } from '@/stores/player.js';

const player = usePlayer(),
  data = useData(),
  store = useStore();

const audio = ref(null);

function audioCanPlay() {
  player.state.status = 'pause';
  audio.value.play().catch(err => {
    console.error(err);
    player.state.status = 'play';
  });

  if (location.pathname != '/playlist') {
    useRoute(data.state.url);
  }

  document.title = `Playing: ${data.state.title} by ${data.state.artist}`;
}

async function Stream() {
  const res = player.state,
    shaka = import('shaka-player/dist/shaka-player.compiled.js');

  let url, mime;

  if (window.MediaSource !== undefined && res.streams.length > 0) {
    const { useDash } = await import('../scripts/dash.js');

    const dash = useDash(res.streams, res.duration);

    url = 'data:application/dash+xml;charset=utf-8;base64,' + btoa(dash);
    mime = 'application/dash+xml';
  } else if (res.hls) {
    url = res.hls;
    mime = 'application/x-mpegURL';
  } else if (res.streams.length > 0) {
    url = res.streams[0].url;
    mime = res.streams[0].mimeType;
  }

  if (!window.audioPlayer) {
    shaka
      .then(shaka => shaka.default)
      .then(shaka => {
        shaka.polyfill.installAll();

        if (shaka.Player.isBrowserSupported) {
          const audioPlayer = new shaka.Player(audio.value);

          const codecs = useStore().getItem('codec');

          audioPlayer.configure({
            preferredAudioCodecs: codecs ? codecs.split(':') : ['opus', 'mp4a'],
            manifest: {
              disableVideo: true,
            },
          });

          window.audioPlayer = audioPlayer;
        }
      });
  }

  const quality = useStore().getItem('quality');

  if (url) {
    window.audioPlayer
      .load(url, 0, mime)
      .then(() => {
        window.audioPlayer.configure('abr.enabled', true);

        if (quality && quality != 'auto') {
          window.audioPlayer.configure('abr.enabled', false);

          const tracks = window.audioPlayer.getVariantTracks(),
            bandwidths = tracks.map(i => i.bandwidth);

          let sel;

          if (quality == 'best') sel = Math.max(...bandwidths);
          else if (quality == 'worst') sel = Math.min(...bandwidths);

          if (sel)
            window.audioPlayer.selectVariantTrack(
              tracks[bandwidths.indexOf(sel)],
            );
        }
      })
      .catch(err => {
        console.error('Code: ' + err.code, err);
      });
  }
}

watch(
  () => player.state.play,
  () => {
    if (audio.value.paused) {
      player.state.status = 'pause';
      audio.value.play().catch(err => {
        console.error(err);
        player.state.status = 'play';
      });
    } else {
      player.state.status = 'play';
      audio.value.pause();
    }
  },
);

watch(() => player.state.streams, Stream);

watch(
  () => player.state.currentTime,
  () => {
    console.log(player.state.currentTime);
    audio.value.currentTime = player.state.currentTime;
  },
);

onMounted(() => {
  Stream();

  if ('mediaSession' in navigator) {
    navigator.mediaSession.setActionHandler('play', () => {
      player.state.status = 'pause';
      audio.value.play().catch(err => {
        alert(err);
        player.state.status = 'play';
      });
    });

    navigator.mediaSession.setActionHandler('pause', () => {
      audio.value.pause();
      player.state.status = 'play';
    });

    navigator.mediaSession.setActionHandler('previoustrack', () => {
      if (data.state.urls.length > 2) {
        const i = data.state.urls.map(s => s.url).indexOf(data.state.url);

        data.getSong(data.state.urls[i - 1].url);
      }
    });

    navigator.mediaSession.setActionHandler('nexttrack', () => {
      if (data.state.urls.length > 2) {
        const i = data.state.urls.map(s => s.url).indexOf(data.state.url);

        data.getSong(data.state.urls[i + 1].url);
      }
    });

 
  //  navigator.mediaSession.setActionHandler("previoustrack", () => AudioPlayer.previous());
	//	navigator.mediaSession.setActionHandler("nexttrack", () => AudioPlayer.next());
    


    const setPosition = () => {
	if ("mediaSession" in navigator) {
		navigator.mediaSession.setPositionState({
			duration: AudioPlayer.isWebkit ? AudioPlayer.duration / 2 : AudioPlayer.duration,
			position: AudioPlayer.currentTime,
		});

    navigator.mediaSession.setActionHandler("seekto", (session) => {
			if (session.fastSeek && "fastSeek" in AudioPlayer.player) {
				AudioPlayer.player.fastSeek(session.seekTime);
				setPosition();
				return;
			}
			AudioPlayer.seek(session.seekTime);

			setPosition();
		});
  }
  };
    
    /*
    navigator.mediaSession.setActionHandler('seekbackward', () => {
      audio.value.duration -= 10;
    });

    navigator.mediaSession.setActionHandler('seekforward', () => {
      audio.value.duration += 10;
    });
    */
  }
});
    

onBeforeUnmount(() => {
  if (window.audioPlayer) {
    window.audioPlayer.destroy();
    window.audioPlayer = undefined;
  }
});
onUnmounted(() => {
  if (window.audioPlayer) {
    window.audioPlayer.destroy();
    window.audioPlayer = undefined;
  }
});
</script>

<template>
  <audio
    id="audio"
    ref="audio"
    :volume="player.state.vol"
    @canplay="audioCanPlay"
    @timeupdate="player.setTime($event.target.currentTime)"
    @ended="data.playNext"
    autoplay></audio>
</template>
