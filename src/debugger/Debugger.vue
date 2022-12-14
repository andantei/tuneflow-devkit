<script lang="ts">
import { defineComponent, ref, computed } from 'vue';
import { TuneflowPlugin } from 'tuneflow';
import PluginClass from '../plugin/export';
import { createReadAPIs, getProxySocketClient } from './utils';

export default defineComponent({
  setup() {
    const isRunningPlugin = ref(false);
    const pluginAvailable = computed(() => PluginClass !== null && PluginClass !== undefined);
    const remoteApis = computed(() => createReadAPIs());
    const socketioClient = getProxySocketClient();
    const serializedSong = ref(null as any);
    const plugin = ref(null as any);
    const pluginName = PluginClass
      ? remoteApis.value.translateLabel(PluginClass.pluginDisplayName())
      : null;
    return {
      isRunningPlugin,
      pluginAvailable,
      socketioClient,
      serializedSong,
      remoteApis,
      plugin,
      pluginName,
    };
  },
  mounted() {
    this.socketioClient.on('set-song', async (payload, callback) => {
      if (PluginClass) {
        try {
          this.plugin = null;
          this.serializedSong = payload.serializedSong;
        } catch (e: any) {
          console.error(e);
        }
        callback({
          status: 'OK',
          pluginInfo: {
            pluginDisplayName: (PluginClass as typeof TuneflowPlugin).pluginDisplayName(),
            pluginDescription: (PluginClass as typeof TuneflowPlugin).pluginDescription(),
            providerDisplayName: (PluginClass as typeof TuneflowPlugin).providerDisplayName(),
          },
        });
      } else {
        callback({
          status: 'PLUGIN_NOT_READY',
        });
      }
    });

    this.socketioClient.on('init-plugin', async (payload, callback) => {
      if (PluginClass && this.serializedSong) {
        const song = await this.remoteApis.deserializeSong(this.serializedSong);
        this.plugin = await (PluginClass as any).create(song, this.remoteApis);

        callback({
          status: 'OK',
          paramsConfig: this.plugin.params(),
          params: this.plugin.getParamsInternal(),
        });
      } else {
        callback({
          status: 'SONG_OR_PLUGIN_NOT_READY',
        });
      }
    });

    this.socketioClient.on('run-plugin', async (payload, callback) => {
      const { params } = payload;
      if (this.plugin && this.serializedSong) {
        const song = await this.remoteApis.deserializeSong(this.serializedSong);
        this.isRunningPlugin = true;
        try {
          await this.plugin.run(song, params, this.remoteApis);
        } catch (e: any) {
          console.error(e);
        }
        this.isRunningPlugin = false;

        callback({
          status: 'OK',
          serializedSongResult: await this.remoteApis.serializeSong(song),
        });
      } else {
        callback({
          status: 'SONG_OR_PLUGIN_NOT_READY',
        });
      }
    });
  },
});
</script>

<template>
  <div :class="$style.Container">
    <h2>????????????</h2>
    <div :class="$style.Status">
      ????????????:
      {{ pluginAvailable ? pluginName : '??????????????????????????????src/plugin/exports.ts?????????????????????' }}
    </div>
    <h2>????????????TuneFlow????????????</h2>
    <div :class="$style.Instructions">
      <ul>
        <li>???src/plugin/exports.ts???PluginClass???????????????????????????</li>
        <li>???Chrome???????????????</li>
        <li>
          ??????Chrome????????????????????????????????? Ctrl+Shift+I???Windows??????
          Cmd+Opt+I???Mac????????????????????????????????????????????????(Inspect)
        </li>
        <img
          :class="$style.Image"
          :style="{ maxWidth: '400px' }"
          src="../../../public/images/toggle_devtools.png"
        />
        <li>????????????????????????????????????Sources?????????</li>
        <li>?????????????????????????????????????????? localhost:9999/src/plugin/...</li>
        <img :class="$style.Image" src="../../../public/images/switch_to_plugin_file.png" />
        <li>???????????????????????????????????????</li>
        <img :class="$style.Image" src="../../../public/images/plugin_debug.png" />
      </ul>
    </div>
  </div>
</template>

<style lang="less" module>
.Container {
  padding: 16px;
}

.Instructions {
  line-height: 20px;
}

.Image {
  max-width: 900px;
}
</style>
