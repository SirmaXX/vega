<template>
  <div class="editor">
    <AppBar
      :projectSync.sync="project"
      @renderVideo="renderVideo"
      @openProject="openProject"
      @downloadProject="downloadProject"
    />
    <sp-divider style="margin: 0" />
    <sp-split-view style="height: calc(100vh - 32px)">
      <sp-split-view-pane>
        <sp-split-view :vertical="true" style="height: 100%; width: 80vw">
          <sp-split-view-pane>
            <sp-split-view>
              <sp-split-view-pane style="width: 200px">
                <sp-split-view :vertical="true" style="height: 100%">
                  <sp-split-view-pane>
                    <AssetWindow
                      :assets="project.assets"
                      @changeSelectedAsset="changeSelectedAsset"
                      @addAsset="addAsset"
                    />
                  </sp-split-view-pane>
                  <sp-split-view-splitter />
                  <sp-split-view-pane>
                    <asset-inspector-window
                      :asset="selectedAsset"
                      @changeAsset="changeAsset"
                      @deleteAsset="deleteAsset"
                    />
                  </sp-split-view-pane>
                </sp-split-view>
              </sp-split-view-pane>
              <sp-split-view-splitter
                class="gripper-horizontal splitter-horizontal"
              ></sp-split-view-splitter>
              <sp-split-view-pane
                :style="`height: ${aboveTimeline}px; max-width: 80%`"
              >
                <PreviewWindow
                  ref="previewWindow"
                  :currentTime="currentTime"
                  :selectedStrip="selectedStrip"
                  :fps="project.fps"
                  :width="project.width"
                  :height="project.height"
                  @changeStripPos="changeStripPos"
                />
              </sp-split-view-pane>
            </sp-split-view>
          </sp-split-view-pane>
          <sp-split-view-splitter
            class="gripper-vertical"
            :gripper="true"
            :vertical="true"
            @change="(v) => (aboveTimeline += v)"
          ></sp-split-view-splitter>
          <sp-split-view-pane
            :style="`height: 100%; width: 100%; overflow-y: scroll`"
          >
            <TimelinePanel
              :currentTime="currentTime"
              :stripsSync.sync="project.strips"
              :selectedStrips="selectedStrips"
              :duration="project.duration"
              :isPlay="isPlay"
              :fps="project.fps"
              @addAsset="addAsset"
              @addStrip="addStrip"
              @changeCurrentTime="changeCurrentTime"
              @changeSelectedStrips="changeSelectedStrips"
              @changeStrip="
                (i, name, value) => changeStripPropery(i, name, value)
              "
              @deleteStrip="deleteStrip"
              @togglePlay="play"
            />
          </sp-split-view-pane>
        </sp-split-view>
      </sp-split-view-pane>
      <sp-split-view-splitter
        class="gripper-horizontal"
      ></sp-split-view-splitter>
      <sp-split-view-pane style="width: 100%">
        <StripInspector
          :stripSync.sync="selectedStrip"
          :assets="project.assets"
          @change="changeStrip"
        />
      </sp-split-view-pane>
    </sp-split-view>
    <Snakbar ref="snakbar" />

    <RendererWindow
      ref="rendererWindow"
      :project="project"
      :scene="scene"
      :camera="camera"
    />
  </div>
</template>

<style scoped>
.editor {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
</style>

<style>
/* Nested SplitView is not working. So override force. */
.gripper-vertical > .spectrum-SplitView-gripper {
  left: 50% !important;
}
.gripper-horizontal {
  height: auto !important;
  min-height: auto !important;
  width: 2px !important;
  min-width: 2px !important;
}
.gripper-horizontal > .spectrum-SplitView-gripper {
  /* override props */
  left: -4px !important;
  /* default horizontal */
  content: "";
  display: block;
  position: absolute;
  border-style: solid;
  border-radius: var(--spectrum-alias-border-radius-small);
  border-radius: var(
    --spectrum-dragbar-gripper-border-radius,
    var(--spectrum-alias-border-radius-small)
  );
  top: 50%;
  transform: translate(0, -50%);
  width: var(--spectrum-global-dimension-static-size-50);
  width: var(
    --spectrum-dragbar-gripper-width,
    var(--spectrum-global-dimension-static-size-50)
  );
  height: var(--spectrum-global-dimension-static-size-200);
  height: var(
    --spectrum-dragbar-gripper-height,
    var(--spectrum-global-dimension-static-size-200)
  );
  border-top-width: 4 px;
  border-top-width: var(--spectrum-dragbar-gripper-border-width-vertical, 4px);
  border-bottom-width: 4 px;
  border-bottom-width: var(
    --spectrum-dragbar-gripper-border-width-vertical,
    4px
  );
  border-left-width: 3 px;
  border-left-width: var(
    --spectrum-dragbar-gripper-border-width-horizontal,
    3px
  );
  border-right-width: 3 px;
  border-right-width: var(
    --spectrum-dragbar-gripper-border-width-horizontal,
    3px
  );
}
</style>

<script lang="ts">
import Vue from "vue";
import * as T from "three";
import { Component, Ref, Watch } from "vue-property-decorator";
import loadicons from "loadicons";
import axios from "axios";
import RendererWindow from "@/components/RendererWindow.vue";
import AppBar from "@/components/app_bar/AppBar.vue";
import PreviewWindow from "~/components/PreviewWindow.vue";
import {
  Asset,
  FontAsset,
  Strip,
  Text3DStrip,
  TextStrip,
  VEGA_VERSION,
  VideoAsset,
  VideoStrip,
  ImageStrip,
  ImageAsset,
  AudioStrip,
  AudioAsset,
} from "~/models";
import AssetWindow from "~/components/asset_window/AssetWindow.vue";
import AssetInspectorWindow from "~/components/asset_inspector/AssetInspectorWindow.vue";
import StripInspector from "~/components/strip_inspector_window/StripInspectorWindow.vue";
import TimelinePanel from "~/components/timeline_panel/TimelinePanel.vue";
import { download } from "~/plugins/download";
import { StripUtil } from "~/plugins/strip";
import { IVector3 } from "~/models/math/Vector3";
import Snakbar from "~/components/Snakbar.vue";
import { VegaError } from "~/plugins/error";
import { PlayMode, PLAY_EVERY_FRAME, SYNC_TO_AUDIO } from "~/plugins/config";
import { isProject, Project } from "~/models/Project";
import ProjectWindow from "~/components/ProjectWindow.vue";
import ControllerWindow from "~/components/ControllerWindow.vue";
import { DragAndDrop } from "~/plugins/dragAndDrop";

@Component({
  components: {
    ProjectWindow,
    RendererWindow,
    ControllerWindow,
    AppBar,
    PreviewWindow,
    AssetWindow,
    AssetInspectorWindow,
    StripInspector,
    TimelinePanel,
    Snakbar,
  },
})
export default class IndexPage extends Vue {
  @Ref() previewWindow?: PreviewWindow;
  @Ref() snakbar?: Snakbar;
  @Ref() rendererWindow?: RendererWindow;

  camera: T.OrthographicCamera | null = null;
  scene: T.Scene | null = null;
  isPlay: boolean = false;
  currentTime: number = 0;
  lastAnimationTime: number = 0;
  playRequests: number[] = [];
  selectedAsset: Asset | null = null;
  selectedStrips: Strip[] = [];
  canvas: HTMLCanvasElement | null = null;

  project: Project = new Project({
    version: VEGA_VERSION,
    name: "untitled",
    width: 1280,
    height: 720,
    fps: 60,
    duration: 10,
    assets: [],
    strips: [],
  });

  playMode: PlayMode = SYNC_TO_AUDIO;

  lastUpdate: number = 0;

  aboveTimeline: number = 400;

  get selectedStrip() {
    if (this.selectedStrips.length > 0) {
      return this.selectedStrips[0];
    }
    return null;
  }

  get selectedStripIndex() {
    return this.project.strips.findIndex((s) => s == this.selectedStrip);
  }

  async mounted() {
    DragAndDrop.init();
    loadicons("/static/svg/spectrum-css-icons.svg", () => {});
    loadicons("/static/svg/spectrum-icons.svg", () => {});
    const demo = this.$route.query.demo;

    if (demo === "true") {
      const res = await axios.get("/static/example/demo.json");
      if (isProject(res.data)) {
        this.project = new Project(res.data);
        this.openProject();
      }
    }
    this.scene = new T.Scene();
    this.camera = new T.OrthographicCamera(
      0,
      this.project.width,
      this.project.height,
      0
    );
    this.camera.position.set(0, 0, 10);
    this.canvas = this.previewWindow?.renderCanvas || null;
    await FontAsset.init();
    this.update(0);
  }

  @Watch("project")
  openProject() {
    this.project.strips.forEach((s) => {
      if (StripUtil.isThreeJsStrip(s)) {
        this.scene?.add(s.obj);
      }
    });

    this.previewWindow?.resize();
  }

  addAsset(asset: Asset) {
    this.project.assets.push(asset);
  }

  changeAsset(newAsset: Asset) {
    const i = this.project.assets.findIndex((a) => a == this.selectedAsset);
    const oldAsset = this.project.assets[i];
    this.project.assets.splice(i, 1, newAsset);
    this.project.strips.forEach((s) => {
      if (s instanceof VideoStrip && newAsset instanceof VideoAsset) {
        if (s.videoAsset == oldAsset) {
          s.updateAsset(newAsset);
        }
      } else if (s instanceof ImageStrip && newAsset instanceof ImageAsset) {
        s.updateAsset(newAsset);
      } else if (s instanceof AudioStrip && newAsset instanceof AudioAsset) {
        s.updateAsset(newAsset);
      }
    });
    this.selectedAsset = newAsset;
  }

  changeStripPropery(index: number, name: string, value: any) {
    if (index < 0 || this.project.strips.length <= index) return;
    const target = this.project.strips[index];
    switch (name) {
      case "layer":
        target.layer = value;
        break;
      case "length":
        target.length = value;
        break;
      case "start":
        target.start = value;
        break;
      case "videoOffset":
        if (target instanceof VideoStrip) {
          target.videoOffset = value;
        }
        break;
      case "text":
        if (target instanceof TextStrip || target instanceof Text3DStrip) {
          target.text = value;
        }
        break;
      case "position.x":
        if (StripUtil.isThreeJsStrip(target)) {
          target.position.x = value;
        }
        break;
      case "position.y":
        if (StripUtil.isThreeJsStrip(target)) {
          target.position.y = value;
        }
        break;
      default:
        break;
    }
  }

  // Replace the strip.
  // Also replace threejs object in scene by THREE.Object3D.uuid
  changeStrip(s: Strip) {
    if (s instanceof Text3DStrip || s instanceof VideoStrip) {
      const target = this.scene?.getObjectByProperty("uuid", s.id);
      if (target) {
        this.scene?.remove(target);
        this.scene?.add(s.obj);
      }
    } else {
      return;
    }

    const i = this.project.strips.findIndex((_s) => _s.id == s.id);
    if (this.selectedStrips.includes(this.project.strips[i])) {
      this.selectedStrips = [s];
    } else {
      this.selectedStrips = [];
    }
    this.project.strips.splice(i, 1, s);
  }

  changeStripPos(pos: IVector3) {
    if (this.selectedStrip && StripUtil.isThreeJsStrip(this.selectedStrip)) {
      this.selectedStrip.position.set(pos.x, pos.y, pos.z);
    }
  }

  changeStrips(strips: Strip[]) {
    this.project.strips = strips;
  }

  deleteStrip(strip: Strip) {
    const i = this.project.strips.findIndex((s) => s == strip);
    if (i != -1) {
      const strip = this.project.strips[i];
      if (StripUtil.isThreeJsStrip(strip)) {
        strip.obj.removeFromParent();
      }

      this.project.strips.splice(i, 1);
      this.selectedStrips = [];
    }
  }

  changeCurrentTime(time: number) {
    this.currentTime = time;
    this.change();
  }

  changeSelectedStrips(strips: Strip[]) {
    this.selectedStrips = strips;
  }

  changeSelectedAsset(asset: Asset) {
    this.selectedAsset = asset;
  }

  changePlayMode(playMode: PlayMode) {
    this.playMode = playMode;
  }

  deleteAsset() {
    const i = this.project.assets.findIndex((a) => a == this.selectedAsset);
    this.project.assets.splice(i, 1);
  }

  async update(time: number = 0) {
    if (this.rendererWindow && this.rendererWindow.isEncoding) {
      window.requestAnimationFrame(this.update);
      return;
    }
    if (time - this.lastUpdate + 0.02 <= 1000 / this.project.fps) {
      window.requestAnimationFrame(this.update);
      return;
    }
    if (this.previewWindow) this.previewWindow.start();

    let delta = time - this.lastAnimationTime;
    this.lastAnimationTime = time;

    if (this.playMode == PLAY_EVERY_FRAME) {
      delta = 1000 / this.project.fps;
    }

    if (this.isPlay) {
      this.currentTime += delta / 1000;
    }

    for (let i = 0; i < this.project.strips.length; i++) {
      const s = this.project.strips[i];
      await s.update(
        this.currentTime,
        delta,
        this.isPlay,
        this.playMode,
        this.project.fps
      );
    }

    if (this.previewWindow && this.scene && this.camera) {
      this.previewWindow.renderPreview(this.scene, this.camera);
      this.camera.top = this.project.height;
      this.camera.right = this.project.width;
      this.camera.updateProjectionMatrix();
      this.previewWindow.resize();
    }

    if (this.previewWindow) this.previewWindow.end();
    this.lastUpdate = time;
    window.requestAnimationFrame(this.update);
  }

  addStrip(ts: Strip) {
    this.project.strips.push(ts);
    if (StripUtil.isThreeJsStrip(ts)) {
      this.scene?.add(ts.obj);
    }
  }

  play() {
    this.isPlay = !this.isPlay;
    this.change();
  }

  downloadProject() {
    download(
      new Blob([JSON.stringify(this.project.toJSON())]),
      this.project.name + ".json"
    );
  }

  getAssetById(id: string) {
    return this.project.assets.find((a) => a.id == id);
  }

  change() {
    this.project.strips.forEach((s: Strip) => {
      s.update(this.currentTime, 0, false, SYNC_TO_AUDIO, this.project.fps);
    });
  }

  errorCaptured(err: Error, _vm: any, _info: any) {
    if (err instanceof VegaError) {
      this.snakbar?.open(3000, err.message);
      return false;
    }
  }

  renderVideo() {
    if (!this.rendererWindow) return;
    this.rendererWindow.open();
  }
}
</script>
