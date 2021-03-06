<template>
  <div ref="controller"
    :data-component-name="$options.name"
    class="the-video-controller"
    :style="{ cursor: cursorStyle }"
    @mousemove="handleMousemove"
    @mouseenter="handleMouseenter"
    @mouseleave="handleMouseleave"
    @mousedown.right="handleMousedownRight"
    @mousedown.left="handleMousedownLeft"
    @mouseup.left="handleMouseupLeft"
    @dblclick="handleDblclick">
    <titlebar currentView="Playingview" v-hidden="displayState['titlebar']" ></titlebar>
    <notification-bubble/>
    <recent-playlist class="recent-playlist"
    :displayState="displayState['recent-playlist']"
    :mousemove="eventInfo.get('mousemove')"
    v-bind.sync="widgetsStatus['recent-playlist']"
    @update:playlistcontrol-showattached="updatePlaylistShowAttached"/>
    <div class="masking" v-hidden="displayState['the-progress-bar']"></div>
    <play-button :paused="paused" />
    <volume-indicator v-hidden="displayState['volume-indicator']"/>
    <div class="control-buttons">
      <subtitle-control class="button subtitle" v-hidden="displayState['subtitle-control']" v-bind.sync="widgetsStatus['subtitle-control']" />
      <playlist-control class="button playlist" v-hidden="displayState['playlist-control']" v-bind.sync="widgetsStatus['playlist-control']"/>
      <advance-control class="button advance" v-hidden="displayState['advance-control']" v-bind.sync="widgetsStatus['advance-control']"/>
    </div>
    <the-time-codes v-hidden="displayState['the-progress-bar']" />
    <the-progress-bar v-hidden="displayState['the-progress-bar']" v-bind.sync="widgetsStatus['the-progress-bar']"/>
  </div>
</template>
<script>
import _ from 'lodash';
import { mapGetters } from 'vuex';
import TimerManager from '@/helpers/timerManager.js';
import Titlebar from '../Titlebar.vue';
import PlayButton from './PlayButton.vue';
import VolumeIndicator from './VolumeIndicator.vue';
import AdvanceControl from './AdvanceControl.vue';
import SubtitleControl from './SubtitleControl.vue';
import PlaylistControl from './PlaylistControl.vue';
import TheTimeCodes from './TheTimeCodes.vue';
import TheProgressBar from './TheProgressBar';
import NotificationBubble from '../NotificationBubble.vue';
import RecentPlaylist from './RecentPlaylist.vue';
import SpeedLabel from './RateLabel.vue';

export default {
  name: 'the-video-controller',
  components: {
    titlebar: Titlebar,
    'play-button': PlayButton,
    'volume-indicator': VolumeIndicator,
    'subtitle-control': SubtitleControl,
    'advance-control': AdvanceControl,
    'playlist-control': PlaylistControl,
    'the-time-codes': TheTimeCodes,
    'the-progress-bar': TheProgressBar,
    'notification-bubble': NotificationBubble,
    'recent-playlist': RecentPlaylist,
    SpeedLabel,
  },
  directives: {
    hidden: {
      update(el, binding) {
        const { oldValue, value } = binding;
        if (oldValue !== value) {
          if (value) {
            el.classList.add('fade-in');
            el.classList.remove('fade-out');
          } else {
            el.classList.add('fade-out');
            el.classList.remove('fade-in');
          }
        }
      },
    },
  },
  data() {
    return {
      start: null,
      UIElements: [],
      currentWidget: this.$options.name,
      mouseStopMoving: false,
      mousestopDelay: 3000,
      mouseLeftWindow: false,
      mouseleftDelay: 1000,
      hideVolume: false,
      muteDelay: 3000,
      hideVolumeDelay: 1000,
      hideProgressBar: false,
      popupShow: false,
      clicksTimer: 0,
      clicksDelay: 200,
      dragDelay: 200,
      displayState: {},
      widgetsStatus: {},
      currentSelectedWidget: 'the-video-controller',
      preventSingleClick: false,
      lastAttachedShowing: false,
      isDragging: false,
      focusedTimestamp: 0,
      focusDelay: 500,
      listenedWidget: 'the-video-controller',
      progressBarHovering: false,
    };
  },
  computed: {
    ...mapGetters(['muted', 'paused']),
    showAllWidgets() {
      return (!this.mouseStopMoving && !this.mouseLeftWindow) ||
        (!this.mouseLeftWindow && this.onOtherWidget);
    },
    onOtherWidget() {
      return this.currentWidget !== this.$options.name;
    },
    cursorStyle() {
      return this.showAllWidgets || !this.isFocused ? 'default' : 'none';
    },
    isFocused() {
      return this.$store.state.Window.isFocused;
    },
  },
  watch: {
    isFocused(newValue) {
      if (newValue) {
        this.focusedTimestamp = Date.now();
      }
    },
    progressBarHovering(newValue) {
      if (!newValue) {
        this.timerManager.updateTimer('sleepingProgressBar', this.mousestopDelay);
        // Prevent all widgets display before the-progress-bar
        if (this.showAllWidgets) {
          this.timerManager.updateTimer('mouseStopMoving', this.mousestopDelay);
        }
        this.hideProgressBar = false;
      }
    },
  },
  created() {
    this.eventInfo = new Map([
      ['mousemove', {}],
      ['mousedown', {}],
      ['mouseup', {}],
      ['mouseenter', {}],
      ['wheel', {}],
      ['keydown', {
        ArrowUp: false,
        ArrowDown: false,
        ArrowLeft: false,
        ArrowRight: false,
        Space: false,
        KeyM: false,
        BracketLeft: false,
        BracketRight: false,
      }],
    ]);
    // Use Object due to vue's lack support of reactive Map
    this.timerState = {};
    // Use Map constructor to shallow-copy eventInfo
    this.lastEventInfo = new Map(this.eventInfo);
    this.timerManager = new TimerManager();
    this.timerManager.addTimer('mouseStopMoving', this.mousestopDelay);
    this.timerManager.addTimer('sleepingVolumeButton', this.mousestopDelay);
    this.timerManager.addTimer('sleepingProgressBar', this.mousestopDelay);
    this.timerManager.addTimer('hoveringProgressBar', this.mousestopDelay);
  },
  mounted() {
    this.UIElements = this.getAllUIComponents(this.$refs.controller);
    this.UIElements.forEach((value) => {
      this.timerState[value.name] = true;
      this.displayState[value.name] = true;
      this.widgetsStatus[value.name] = {
        selected: false,
        showAttached: false,
        mousedownOnOther: false,
        mouseupOnOther: false,
        hovering: false,
      };
    });

    document.addEventListener('keydown', this.handleKeydown);
    document.addEventListener('keyup', this.handleKeyup);
    document.addEventListener('wheel', this.handleWheel);
    requestAnimationFrame(this.UIManager);
    this.$bus.$on('currentWidget', (widget) => {
      this.listenedWidget = widget;
      this.timerManager.updateTimer('mouseStopMoving', this.mousestopDelay, false);
    });
  },
  methods: {
    updatePlaylistShowAttached(event) {
      this.widgetsStatus['playlist-control'].showAttached = event;
      this.$electron.ipcRenderer.send('callCurrentWindowMethod', 'setMinimumSize', [320, 180]);
    },
    // UIManagers
    UIManager(timestamp) {
      if (!this.start) {
        this.start = timestamp;
      }

      // Use Map constructor to shallow-copy eventInfo
      const lastEventInfo = new Map(this.inputProcess(this.eventInfo, this.lastEventInfo));
      this.UITimerManager(timestamp - this.start);
      // this.UILayerManager();
      this.UIDisplayManager();
      this.UIStateManager();
      this.lastEventInfo = lastEventInfo;

      this.start = timestamp;
      requestAnimationFrame(this.UIManager);
    },
    inputProcess(currentEventInfo, lastEventInfo) { // eslint-disable-line
      // mousemove timer
      const currentChanged = currentEventInfo.get('mousemove').target !== lastEventInfo.get('mousemove').target;
      if (currentChanged) {
        this.listenedWidget = this.getComponentName(currentEventInfo.get('mousemove').target);
      }
      this.currentWidget = this.listenedWidget;
      this.mouseStopMoving = _.isEqual(currentEventInfo.get('mousemove').position, lastEventInfo.get('mousemove').position);
      if (!this.mouseStopMoving) { this.timerManager.updateTimer('mouseStopMoving', this.mousestopDelay, false); }
      // mouseenter timer
      const { mouseLeavingWindow } = currentEventInfo.get('mouseenter');
      const changed = mouseLeavingWindow !== lastEventInfo.get('mouseenter').mouseLeavingWindow;
      if (this.andify(mouseLeavingWindow, changed)) {
        this.timerManager.addTimer('mouseLeavingWindow', this.mouseleftDelay);
      } else if (this.andify(!mouseLeavingWindow, changed)) {
        this.timerManager.removeTimer('mouseLeavingWindow');
        this.mouseLeftWindow = false;
      }
      // hideVolume timer
      const volumeKeydown = this.orify(currentEventInfo.get('keydown').ArrowUp, currentEventInfo.get('keydown').ArrowDown, currentEventInfo.get('keydown').KeyM); // eslint-disable-line
      const mouseScrolling = currentEventInfo.get('wheel').time !== lastEventInfo.get('wheel').time;
      const lastWidget = this.getComponentName(lastEventInfo.get('mousemove').target);
      const mouseWakingUpVolume = this.enterWidgets(lastWidget, this.currentWidget, 'volume-indicator');
      const mouseLeavingVolume = this.leaveWidgets(lastWidget, this.currentWidget, 'volume-indicator');
      const mouseMovingInVolume = this.andify(!this.mouseStopMoving, this.inWidgets(lastWidget, this.currentWidget, 'volume-indicator'));
      const wakingupVolume = this.orify(volumeKeydown, mouseScrolling, this.andify(!this.muted, this.orify(mouseWakingUpVolume, mouseLeavingVolume, mouseMovingInVolume))); // eslint-disable-line
      if (wakingupVolume) {
        this.timerManager.updateTimer('sleepingVolumeButton', this.orify(mouseWakingUpVolume, mouseMovingInVolume) ? this.muteDelay : this.hideVolumeDelay);
        // Prevent all widgets display before volume-control
        if (this.andify(this.showAllWidgets, mouseMovingInVolume)) {
          this.timerManager.updateTimer('mouseStopMoving', this.mousestopDelay);
        }
        this.hideVolume = false;
      }
      // hideProgressBar timer
      const progressKeydown = this.orify(currentEventInfo.get('keydown').ArrowLeft, currentEventInfo.get('keydown').ArrowRight, currentEventInfo.get('keydown').BracketLeft, currentEventInfo.get('keydown').BracketRight);
      if (progressKeydown || this.showAllWidgets) {
        this.timerManager.updateTimer('sleepingProgressBar', this.mousestopDelay);
      }
      if (this.currentWidget === 'the-progress-bar') {
        this.timerManager.updateTimer('hoveringProgressBar', this.mousestopDelay);
        this.widgetsStatus['the-progress-bar'].hovering = this.progressBarHovering = true;
      }
      if (this.currentWidget === 'the-video-controller' &&
        this.getComponentName(lastEventInfo.get('mousemove').target) === 'the-progress-bar') {
        this.timerManager.updateTimer('hoveringProgressBar', 0);
      }
      // mouseup status
      if (lastEventInfo.get('mouseup').leftMouseup !== currentEventInfo.get('mouseup').leftMouseup) {
        this.currentSelectedWidget = this.getComponentName(currentEventInfo.get('mouseup').target);
      }

      Object.keys(this.timerState).forEach((uiName) => {
        this.timerState[uiName] = this.showAllWidgets;
      });
      this.timerState['volume-indicator'] = !this.hideVolume;
      this.timerState['the-progress-bar'] = this.progressBarHovering || !this.hideProgressBar;
      return currentEventInfo;
    },
    UITimerManager(frameTime) {
      this.timerManager.tickTimer('mouseStopMoving', frameTime);
      this.timerManager.tickTimer('mouseLeavingWindow', frameTime);
      this.timerManager.tickTimer('sleepingVolumeButton', frameTime);
      this.timerManager.tickTimer('hoveringProgressBar', frameTime);
      this.timerManager.tickTimer('sleepingProgressBar', frameTime);

      const timeoutTimers = this.timerManager.timeoutTimers();
      this.mouseStopMoving = timeoutTimers.includes('mouseStopMoving');
      this.mouseLeftWindow = timeoutTimers.includes('mouseLeavingWindow');
      this.hideVolume = timeoutTimers.includes('sleepingVolumeButton');
      this.widgetsStatus['the-progress-bar'].hovering = this.progressBarHovering = !timeoutTimers.includes('hoveringProgressBar');
      this.hideProgressBar = timeoutTimers.includes('sleepingProgressBar');

      this.timerState['volume-indicator'] = !this.hideVolume;
      this.timerState['the-progress-bar'] = this.progressBarHovering || !this.hideProgressBar;
    },
    // UILayerManager() {
    // },
    UIDisplayManager() {
      const tempObject = {};
      Object.keys(this.displayState).forEach((index) => {
        tempObject[index] = (this.showAllWidgets ||
          (!this.showAllWidgets && this.timerState[index])) &&
          !this.widgetsStatus['playlist-control'].showAttached;
      });
      tempObject['recent-playlist'] = this.widgetsStatus['playlist-control'].showAttached;
      tempObject['volume-indicator'] = !this.mute ? this.timerState['volume-indicator'] : tempObject['volume-indicator'];
      this.displayState = tempObject;
    },
    UIStateManager() {
      const currentMousedownWidget = this.getComponentName(this.eventInfo.get('mousedown').target);
      const lastMousedownWidget = this.getComponentName(this.lastEventInfo.get('mousedown').target);
      const mousedownChanged = currentMousedownWidget !== lastMousedownWidget;
      const currentMouseupWidget = this.getComponentName(this.eventInfo.get('mouseup').target);
      const lastMouseupWidget = this.getComponentName(this.lastEventInfo.get('mouseup').target);
      const mouseupChanged = currentMouseupWidget !== lastMouseupWidget;
      Object.keys(this.widgetsStatus).forEach((name) => {
        this.widgetsStatus[name].selected = this.currentSelectedWidget === name;
        if (mousedownChanged) {
          this.widgetsStatus[name].mousedownOnOther = currentMousedownWidget !== name;
          if (name === 'recent-playlist') {
            this.widgetsStatus[name].mousedownOnOther = currentMousedownWidget !== name
              && currentMousedownWidget !== 'playlist-control';
          }
        }
        if (mouseupChanged) {
          this.widgetsStatus[name].mouseupOnOther = currentMouseupWidget !== name;
          if (name === 'recent-playlist') {
            this.widgetsStatus[name].mouseupOnOther = currentMouseupWidget !== name
              && currentMousedownWidget !== 'playlist-control';
          }
        }
        if (!this.showAllWidgets) {
          if (name !== 'playlist-control') {
            this.widgetsStatus[name].showAttached = false;
          }
        }
      });
    },
    // Event listeners
    handleMousemove(event) {
      this.eventInfo.set('mousemove', {
        target: event.target,
        position: [event.clientX, event.clientY],
      });
      if (this.eventInfo.get('mousedown').leftMousedown) {
        this.isDragging = true;
      }
    },
    handleMouseenter() {
      this.eventInfo.set('mouseenter', { mouseLeavingWindow: false });
    },
    handleMouseleave() {
      this.eventInfo.set('mousemove', { target: null });
      this.eventInfo.set('mouseenter', { mouseLeavingWindow: true });
    },
    handleMousedownRight() {
      this.eventInfo.set('mousedown', Object.assign(
        {},
        this.eventInfo.get('mousedown'),
        { rightMousedown: true },
      ));
      this.eventInfo.set('mouseup', Object.assign(
        {},
        this.eventInfo.get('mouseup'),
        { rightMouseup: false },
      ));
      if (process.platform !== 'darwin') {
        const menu = this.$electron.remote.Menu.getApplicationMenu();
        menu.popup(this.$electron.remote.getCurrentWindow());
        this.popupShow = true;
      }
    },
    handleMousedownLeft(event) {
      if (!this.isValidClick()) { return; }
      this.eventInfo.set('mousedown', Object.assign(
        {},
        this.eventInfo.get('mousedown'),
        { leftMousedown: true },
        { target: event.target },
      ));
      this.eventInfo.set('mouseup', Object.assign(
        {},
        this.eventInfo.get('mouseup'),
        { leftMouseup: false },
      ));
      if (process.platform !== 'darwin') {
        const menu = this.$electron.remote.Menu.getApplicationMenu();
        if (this.popupShow === true) {
          menu.closePopup();
          this.popupShow = false;
        }
      }
    },
    handleMouseupLeft(event) {
      if (!this.isValidClick()) { return; }
      this.eventInfo.set('mousedown', Object.assign(
        {},
        this.eventInfo.get('mousedown'),
        { leftMousedown: false },
      ));
      this.eventInfo.set('mouseup', Object.assign(
        {},
        this.eventInfo.get('mousedown'),
        { leftMouseup: true, target: event.target },
      ));
      this.clicksTimer = setTimeout(() => {
        const attachedShowing = this.lastAttachedShowing;
        if (
          this.getComponentName(this.eventInfo.get('mousedown').target) === 'the-video-controller' &&
          this.currentSelectedWidget === 'the-video-controller' && !this.preventSingleClick && !attachedShowing && !this.isDragging) {
          this.togglePlayback();
        }
        this.preventSingleClick = false;
        this.lastAttachedShowing = this.widgetsStatus['subtitle-control'].showAttached || this.widgetsStatus['advance-control'].showAttached || this.widgetsStatus['playlist-control'].showAttached;
        this.isDragging = false;
      }, this.clicksDelay);
    },
    handleDblclick() {
      clearTimeout(this.clicksTimer); // cancel the time out
      this.preventSingleClick = true;
      if (this.currentSelectedWidget === 'the-video-controller') {
        this.toggleFullScreenState();
      }
    },
    handleKeydown(event) {
      this.eventInfo.set('keydown', Object.assign(
        {},
        this.eventInfo.get('keydown'),
        { [event.code]: true },
      ));
    },
    handleKeyup(event) {
      this.eventInfo.set('keydown', Object.assign(
        {},
        this.eventInfo.get('keydown'),
        { [event.code]: false },
      ));
    },
    handleWheel(event) {
      let isAdvanceColumeItem;
      const nodeList = document.querySelector('.advance-column-items').childNodes;
      for (let i = 0; i < nodeList.length; i += 1) {
        isAdvanceColumeItem = nodeList[i].contains(event.target);
      }
      if (!isAdvanceColumeItem) {
        this.eventInfo.set('wheel', { time: event.timeStamp });
      }
    },
    // Helper functions
    getAllUIComponents(rootElement) {
      const { children } = rootElement;
      const names = [];
      for (let i = 0; i < children.length; i += 1) {
        this.processSingleElement(children[i]).forEach((componentName) => {
          names.push(componentName);
        });
      }
      return names;
    },
    isChildComponent(element) {
      let componentName = null;
      this.$children.forEach((childComponenet) => {
        if (childComponenet.$el === element) {
          componentName = childComponenet.$options.name;
        }
      });
      return componentName;
    },
    processSingleElement(element) {
      const names = [];
      const name = this.isChildComponent(element);
      if (name) {
        names.push({
          name,
          element,
        });
      } else {
        const { children } = element;
        for (let i = 0; i < children.length; i += 1) {
          names.push(this.processSingleElement(children[i])[0]);
        }
      }
      return names;
    },
    getComponentName(element) {
      let componentName = this.$options.name;
      if (element instanceof HTMLElement || element instanceof SVGElement) {
        /* eslint-disable consistent-return */
        this.UIElements.forEach((UIElement) => {
          if (UIElement.element.contains(element)) {
            componentName = UIElement.name;
            return componentName;
          }
        });
      }
      return componentName;
    },
    isValidClick() {
      return Date.now() - this.focusedTimestamp > this.focusDelay;
    },
    toggleFullScreenState() {
      this.$bus.$emit('toggle-fullscreen');
    },
    togglePlayback() {
      this.$bus.$emit('toggle-playback');
    },
    orify(...args) {
      return args.some(arg => arg == true); // eslint-disable-line
    },
    andify(...args) {
      return args.every(arg => arg == true); // eslint-disable-line
    },
    enterWidgets(last, current, ...widgets) {
      return this.andify(
        widgets.some(widget => current === widget),
        widgets.every(widget => last !== widget),
      );
    },
    leaveWidgets(last, current, ...widgets) {
      return this.andify(
        widgets.every(widget => current !== widget),
        widgets.some(widget => last === widget),
      );
    },
    inWidgets(last, current, ...widgets) {
      return this.andify(
        widgets.some(widget => current === widget),
        widgets.some(widget => last === widget),
      );
    },
  },
};
</script>
<style lang="scss">
.the-video-controller {
  position: relative;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 4px;
  opacity: 1;
  transition: opacity 400ms;
  z-index: auto;
}
.masking {
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
  height: 50%;
  opacity: 0.3;
  z-index: 1;
  background-image: linear-gradient(
    -180deg,
    rgba(0, 0, 0, 0) 0%,
    rgba(0, 0, 0, 0.19) 62%,
    rgba(0, 0, 0, 0.29) 100%
  );
}
.recent-playlist {
  position: absolute;
  bottom: 0;
  width: 100%;
  z-index: 1000; // in front of all widgets
}

.translate-enter-active, .translate-leave-active {
  transition: opacity 300ms cubic-bezier(0.2, 0.3, 0.01, 1), transform 300ms cubic-bezier(0.2, 0.3, 0.01, 1);
}
.translate-enter, .translate-leave-to {
  opacity: 0;
  transform: translateY(100px);
}
.control-buttons {
  display: flex;
  justify-content: space-between;
  position: fixed;
  z-index: 10;
  .button {
    -webkit-app-region: no-drag;
    cursor: pointer;
    position: relative;
  }
  .subtitle {
    @media screen and (min-width: 513px) and (max-width: 854px) {
      margin-right: 17.6px;
    }
    @media screen and (min-width: 855px) and (max-width: 1920px) {
      margin-right: 25.6px;
    }
    @media screen and (min-width: 1921px) {
      margin-right: 40px;
    }
  }
  .playlist {
    @media screen and (min-width: 513px) and (max-width: 854px) {
      margin-right: 17.6px;
    }
    @media screen and (min-width: 855px) and (max-width: 1920px) {
      margin-right: 25.6px;
    }
    @media screen and (min-width: 1921px) {
      margin-right: 40px;
    }
  }
  img {
    width: 100%;
    height: 100%;
  }
}
@media screen and (max-width: 512px) {
  .control-buttons {
    display: none;
  }
}
@media screen and (min-width: 513px) and (max-width: 854px) {
  .control-buttons {
    width: 115px;
    height: 22px;
    right: 25px;
    bottom: 20px;
    .button {
      width: 26.4px;
      height: 22px;
    }
  }
}
@media screen and (min-width: 855px) and (max-width: 1920px) {
  .control-buttons {
    width: 167px;
    height: 32px;
    right: 30px;
    bottom: 24px;
    .button {
      width: 38.4px;
      height: 32px;
    }
  }
}
@media screen and (min-width: 1921px) {
  .control-buttons {
    width: 260px;
    height: 50px;
    right: 45px;
    bottom: 32px;
    .button {
      width: 60px;
      height: 50px;
    }
  }
}
.fade-in {
  visibility: visible;
  opacity: 1;
  transition: opacity 100ms ease-in;
}
.fade-out {
  visibility: hidden;
  opacity: 0;
  transition: visibility 0s 300ms, opacity 300ms ease-out;
}
</style>
