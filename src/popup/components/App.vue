<template>
  <div id="puppeteer-recorder" class="recorder">
    <div class="header">
      <a href="#" @click="goHome">
        Cypress recorder
        <span class="text-muted">
          <small>{{version}}</small>
        </span>
      </a>
      <div class="left">
        <div class="recording-badge" v-show="isRecording">
          <span class="red-dot"></span>
          {{recordingBadgeText}}
        </div>
        <a href="#" @click="toggleShowHelp" class="header-button">
          <img src="/images/help.svg" alt="help" width="18px" />
        </a>
        <a href="#" @click="openOptions" class="header-button">
          <img src="/images/settings.svg" alt="settings" width="18px" />
        </a>
      </div>
    </div>
    <div class="main">
      <div class="tabs" v-show="!showHelp">
        <RecordingTab
          :code="code"
          :is-recording="isRecording"
          :live-events="liveEvents"
          v-show="!showResultsTab"
        />
        <div class="recording-footer" v-show="!showResultsTab">
          <button
            class="btn btn-sm btn-primary btn-outline-primary"
            @click="togglePause"
            v-show="isRecording"
          >{{pauseButtonText}}</button>
          <input
            :value="title"
            @input="updateTitle"
            required="required"
            class="testcase-title"
            placeholder="Test Title"
            v-show="!isRecording"
          />
          <input
            :value="titleDesc"
            @input="updateTitleDesc"
            required="required"
            class="testcase-title"
            placeholder="Test Description"
            v-show="!isRecording"
          />
          <button
            class="btn btn-sm"
            @click="toggleRecord"
            :class="isRecording ? 'btn-danger' : 'btn-primary'"
          >{{recordButtonText}}</button>

          <a href="#" @click="showResultsTab = true" v-show="code">view code</a>
        </div>
        <ResultsTab
          :code="code"
          :copy-link-text="copyLinkText"
          :restart="restart"
          :set-copying="setCopying"
          @update="updateCode"
          v-if="showResultsTab"
        />
        <div class="results-footer" v-show="showResultsTab">
          <button class="btn btn-sm btn-primary" @click="restart" v-show="code">Restart</button>
          <a href="#" v-clipboard:copy="code" @click="setCopying" v-show="code">{{copyLinkText}}</a>
          <button class="btn btn-sm btn-primary" @click="saveToServer" v-show="code">Save to Server</button>
        </div>
      </div>
      <HelpTab v-show="showHelp"></HelpTab>
    </div>
  </div>
</template>

<script>
import { version } from "../../../package.json";
import CodeGenerator from "../../code-generator/CodeGenerator";
import RecordingTab from "./RecordingTab.vue";
import ResultsTab from "./ResultsTab.vue";
import HelpTab from "./HelpTab.vue";

export default {
  name: "App",
  components: { ResultsTab, RecordingTab, HelpTab },
  data() {
    return {
      isLoading: false,
      msg: "",
      title: "",
      titleDesc: "",
      code: "",
      showResultsTab: false,
      showHelp: false,
      liveEvents: [],
      recording: [],
      isRecording: false,
      isPaused: false,
      isCopying: false,
      bus: null,
      version,
      errors: []
    };
  },
  mounted() {
    this.loadState(() => {
      if (this.isRecording) {
        console.debug("opened in recording state, fetching recording events");
        this.$chrome.storage.local.get(
          ["recording", "code"],
          ({ recording }) => {
            console.debug("loaded recording", recording);
            this.liveEvents = recording;
          }
        );
      }

      if (!this.isRecording && this.code) {
        this.showResultsTab = true;
      }
    });
    this.bus = this.$chrome.extension.connect({ name: "recordControls" });
  },
  methods: {
    updateTitle(e) {
      this.title = e.target.value;
      this.storeState();
    },
    updateTitleDesc(e) {
      this.titleDesc = e.target.value;
      this.storeState();
    },
    updateCode(code) {
      this.code = code;
      this.storeState();
    },
    toggleRecord() {
      if (this.isRecording) {
        this.stop();
      } else {
        this.start();
      }
      this.isRecording = !this.isRecording;
      this.storeState();
    },
    togglePause() {
      if (this.isPaused) {
        this.bus.postMessage({ action: "unpause" });
        this.isPaused = false;
      } else {
        this.bus.postMessage({ action: "pause" });
        this.isPaused = true;
      }
      this.storeState();
    },
    checkForm(e) {
      this.errors = [];
      if (!this.title) {
        alert("Test Title required.");
        this.errors.push("Test Title required.");
      }
      if (!this.titleDesc) {
        alert("Title Description required.");
        this.errors.push("Test Description required.");
      }
      if (!this.errors.length) {
        return true;
      }
      e.preventDefault();
    },
    start() {
      if (this.checkForm()) this.cleanUp({ clearTitle: false });
      console.debug("start recorder, popup app");
      this.bus.postMessage({ action: "start" });
    },
    stop() {
      console.debug("stop recorder");
      this.bus.postMessage({ action: "stop" });
      window.localStorage.setItem("messageStarted", 0);

      this.$chrome.storage.local.get(
        ["recording", "options"],
        ({ recording, options }) => {
          console.debug("loaded recording", recording);
          console.debug("loaded options", options);

          this.recording = recording;
          const codeOptions = options ? options.code : {};

          const codeGen = new CodeGenerator({
            ...codeOptions,
            testTitle: this.title || "test_name_default",
            titleDesc: this.titleDesc || "what_it_does"
          });
          this.code = codeGen.generate(this.recording);
          this.showResultsTab = true;
          this.storeState();

          this.$chrome.storage.local.set({ firstRun: 0 });
        }
      );
    },
    restart() {
      console.log("restart");
      this.cleanUp({});
      this.bus.postMessage({ action: "cleanUp" });
    },
    saveToServer() {
      this.$chrome.storage.local.get("options", ({ options }) => {
        let opt = options;
        let serverUrl = options && options.code && options.code.serverUrl;
        if (serverUrl) {
          let body = JSON.stringify({
            title: this.title,
            code: this.code
          });
          fetch(serverUrl, {
            method: "POST",
            headers: {
              "Content-type": "application/json",
              Accept: "application/json",
              "Accept-Charset": "utf-8"
            },
            body
          })
            .then(() => {
              alert("Data Saved Successfully");
              this.restart();
            })
            .catch(e => {
              alert("Unuccessfully, Please Try Again");
            });
        }
      });
    },
    cleanUp({ clearTitle = true }) {
      if (clearTitle) {
        this.title = "";
        this.titleDesc = "";
      }
      this.recording = this.liveEvents = [];
      this.code = "";
      this.showResultsTab = this.isRecording = this.isPaused = false;
      this.storeState();
    },
    openOptions() {
      if (this.$chrome.runtime.openOptionsPage) {
        this.$chrome.runtime.openOptionsPage();
      }
    },
    loadState(cb) {
      this.$chrome.storage.local.get(
        ["controls", "code", "title", "titleDesc"],
        ({ controls, code, title, titleDesc }) => {
          if (controls) {
            this.isRecording = controls.isRecording;
            this.isPaused = controls._isPaused;
          }

          if (code) {
            this.code = code;
          }
          if (title) {
            this.title = title;
          }
          if (titleDesc) {
            this.titleDesc = titleDesc;
          }
          cb();
        }
      );
    },
    storeState() {
      this.$chrome.storage.local.set({
        code: this.code,
        title: this.title,
        titleDesc: this.titleDesc,
        controls: {
          isRecording: this.isRecording,
          isPaused: this.isPaused
        }
      });
    },
    setCopying() {
      this.isCopying = true;
      setTimeout(() => {
        this.isCopying = false;
      }, 1500);
    },
    goHome() {
      this.showResultsTab = false;
      this.showHelp = false;
    },
    toggleShowHelp() {
      this.showHelp = !this.showHelp;
    }
  },
  computed: {
    recordingBadgeText() {
      return this.isPaused ? "paused" : "recording";
    },
    recordButtonText() {
      return this.isRecording ? "Stop" : "Record";
    },
    pauseButtonText() {
      return this.isPaused ? "Resume" : "Pause";
    },
    copyLinkText() {
      return this.isCopying ? "copied!" : "copy to clipboard";
    }
  }
};
</script>

<style lang="scss" scoped>
@import "~styles/_animations.scss";
@import "~styles/_variables.scss";
@import "~styles/_mixins.scss";

.recorder {
  font-size: 14px;

  .header {
    @include header();

    a {
      color: $gray-dark;
    }

    .left {
      margin-left: auto;
      display: flex;
      justify-content: flex-start;
      align-items: center;

      .recording-badge {
        color: $brand-danger;
        .red-dot {
          height: 9px;
          width: 9px;
          background-color: $brand-danger;
          border-radius: 50%;
          display: inline-block;
          margin-right: 0.4rem;
          vertical-align: middle;
          position: relative;
        }
      }

      .header-button {
        margin-left: $spacer;
        img {
          vertical-align: middle;
        }
      }
    }
  }

  .recording-footer {
    @include footer();
  }
  .results-footer {
    @include footer();
  }
}
</style>
