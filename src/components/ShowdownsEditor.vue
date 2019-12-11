<!--
 * Copyright (c) 2019-present, Jhuix (Hui Jin) <jhuix0117@gmail.com>. All rights reserved.
 * Use of this source code is governed by a MIT license that can be found in the LICENSE file.
 * Description: markdown showdowns editor (MDSE)
-->

<template>
  <div class="mde-workspace-container">
    <div class="mde-toolbar" v-if="hasToolbar">
      <div class="mde-toolbar-actions nav">
        <mde-menubutton
          ref="rootmenu"
          v-bind:item="rootSet.item"
          v-bind:items="rootSet.menuItems"
          v-on:menuclick="handleClick"
        ></mde-menubutton>
        <mde-buttons v-bind:items="toolSet.editItems" v-on:click="handleClick"></mde-buttons>
      </div>
      <div class="mde-toolbar-actions edit" ref="edittool">
        <mde-menubutton
          ref="titlemenutool"
          v-bind:item="menuSet.heading.item"
          v-bind:items="menuSet.heading.menuItems"
          v-if="showTitleMenu"
          v-on:menuclick="handleClick"
        ></mde-menubutton>
        <mde-buttons ref="titletool" v-bind:items="toolSet.headingItems" v-else v-on:click="handleClick"></mde-buttons>
        <mde-menubutton
          ref="fontmenutool"
          v-bind:item="menuSet.font.item"
          v-bind:items="menuSet.font.menuItems"
          v-if="showFontMenu"
          v-on:menuclick="handleClick"
        ></mde-menubutton>
        <mde-buttons ref="fonttool" v-bind:items="toolSet.fontItems" v-else v-on:click="handleClick"></mde-buttons>
        <mde-menubutton
          ref="alignmenutool"
          v-bind:item="menuSet.align.item"
          v-bind:items="menuSet.align.menuItems"
          v-if="showAlignMenu"
          v-on:menuclick="handleClick"
        ></mde-menubutton>
        <mde-buttons ref="aligntool" v-bind:items="toolSet.alignItems" v-else v-on:click="handleClick"></mde-buttons>
        <mde-menubutton
          ref="listmenutool"
          v-bind:item="menuSet.list.item"
          v-bind:items="menuSet.list.menuItems"
          v-if="showListMenu"
          v-on:menuclick="handleClick"
        ></mde-menubutton>
        <mde-buttons ref="listtool" v-bind:items="toolSet.listItems" v-else v-on:click="handleClick"></mde-buttons>
        <mde-buttons ref="othertool" v-bind:items="toolSet.otherItems" v-on:click="handleClick"></mde-buttons>
      </div>
    </div>
    <div class="mde-editor-container">
      <mde-splitpane defaultSize="50%" pane1ClassName="raw-editor" pane2ClassName="raw-previewer" split="vertical">
        <template slot="pane1">
          <div class="mde-scroll-container">
            <mdse-editor
              :markdown="mdDoc"
              :options="mdeOptions"
              @input="onMdChange"
              @scroll="onMdScroll"
              class="mdse-editor"
              ref="mdseEditor"
            ></mdse-editor>
          </div>
        </template>
        <template slot="pane2">
          <div class="mde-scroll-container">
            <mdse-showdowns
              :extensions="previewExtensions"
              :markdown="mdDoc"
              :showdownOptions="mdpOptions"
              class="mdse-preview"
              ref="mdsePreviewer"
            ></mdse-showdowns>
          </div>
        </template>
      </mde-splitpane>
    </div>
  </div>
</template>

<script type="text/javascript">
import debounce from 'lodash/debounce';
import ToolbarSet from '@/scripts/toolbar.js';
import SplitPane from '@/components/mde-ui/mde-splitpane.vue';
import Buttons from '@/components/mde-ui/mde-buttons.vue';
import MenuButton from '@/components/mde-ui/mde-menubutton.vue';
import Editor from '@/components/Editor.vue';
import Previewer from '@/components/Showdowns.vue';
import i18n from '@/scripts/i18n.js';

export default {
  name: 'mdse-showdowns-editor',
  props: {
    markdown: String,
    hasToolbar: {
      type: Boolean,
      default: true
    },
    editOptions: {
      type: Object,
      default: () => ({})
    },
    previewOptions: {
      type: Object,
      default: () => ({})
    },
    previewExtensions: {
      type: Object,
      required: false,
      default: () => ({})
    }
  },
  components: {
    [Editor.name]: Editor,
    [Previewer.name]: Previewer,
    [SplitPane.name]: SplitPane,
    [Buttons.name]: Buttons,
    [MenuButton.name]: MenuButton
  },
  data() {
    return {
      mdDoc: this.markdown,
      mdeOptions: this.editOptions,
      mdpOptions: this.previewOptions,
      editor: null,
      previewer: null,
      rootmenu: null,
      edittool: null,
      othertool: null,
      rootSet: ToolbarSet.getRootSet(),
      toolSet: ToolbarSet.getToolSet(),
      menuSet: ToolbarSet.getMenuSet(),
      screenWidth: document.documentElement.clientWidth,
      lastToolOffset: 0,
      titleToolWidth: 0,
      fontToolWidth: 0,
      alignToolWidth: 0,
      listToolWidth: 0,
      showTitleMenu: false,
      showFontMenu: false,
      showAlignMenu: false,
      showListMenu: false,
      activeLangItem: null
    };
  },
  created() {},
  watch: {
    markdown(val) {
      this.mdDoc = val;
    },
    editOptions: {
      deep: true,
      handler(options) {
        this.mdeOptions = Object.assign({}, this.mdeOptions, options);
      }
    },
    previewOptions: {
      deep: true,
      handler(options) {
        this.mdpOptions = Object.assign({}, this.mdpOptions, options);
      }
    },
    disableUndo(n) {
      if (this.toolSet.editItems && this.toolSet.editItems.length > 0) this.toolSet.editItems[0].disabled = n;
    },
    disableRedo(n) {
      if (this.toolSet.editItems && this.toolSet.editItems.length > 1) this.toolSet.editItems[1].disabled = n;
    },
    showMenuChange() {
      this.$nextTick(() => {
        this.getLastToolOffset();
        this.controlChangeMenu = false;
      });
    },
    lastOffsetChange() {
      debounce(function(that) {
        if (!that.controlChangeMenu) {
          const srceenWidth = that.screenWidth;
          const lastToolOffset = that.lastToolOffset;
          //Switch menus and toolbars
          if (lastToolOffset > srceenWidth) {
            if (!that.showTitleMenu) {
              that.controlChangeMenu = true;
              that.showTitleMenu = true;
            } else if (!that.showFontMenu) {
              that.controlChangeMenu = true;
              that.showFontMenu = true;
            } else if (!that.showAlignMenu) {
              that.controlChangeMenu = true;
              that.showAlignMenu = true;
            } else if (!that.showListMenu) {
              that.controlChangeMenu = true;
              that.showListMenu = true;
            }
          } else if (lastToolOffset < srceenWidth && that.edittool) {
            const editToolOffset = that.edittool.offsetLeft + that.edittool.offsetWidth;
            if (editToolOffset <= srceenWidth) {
              let offset = srceenWidth - editToolOffset;
              if (that.showListMenu) {
                offset += that.$refs.listmenutool.$el.clientWidth;
                if (offset > that.listToolWidth) that.showListMenu = false;
              } else if (that.showAlignMenu) {
                offset += that.$refs.alignmenutool.$el.clientWidth;
                if (offset > that.alignToolWidth) that.showAlignMenu = false;
              } else if (that.showFontMenu) {
                offset += that.$refs.fontmenutool.$el.clientWidth;
                if (offset > that.fontToolWidth) that.showFontMenu = false;
              } else if (that.showTitleMenu) {
                offset += that.$refs.titlemenutool.$el.clientWidth;
                if (offset > that.titleToolWidth) that.showTitleMenu = false;
              }
            }
          }
        }
      }, 300)(this);
    }
  },
  computed: {
    disableUndo() {
      return !this.editor || !this.editor.hasUndo();
    },
    disableRedo() {
      return !this.editor || !this.editor.hasRedo();
    },
    showMenuChange() {
      return [this.showTitleMenu, this.showFontMenu, this.showAlignMenu, this.showListMenu];
    },
    lastOffsetChange() {
      return [this.screenWidth, this.lastToolOffset];
    },
    toolLeft() {
      return this.othertool ? this.othertool.getOffsetLeft() : 0;
    }
  },
  methods: {
    insertMarkdownContent(type) {
      return this.editor.insertMarkdownContent(type);
    },
    getLastToolOffset() {
      if (this.othertool) this.lastToolOffset = this.othertool.$el.offsetLeft + this.othertool.$el.offsetWidth;
      return this.lastToolOffset;
    },
    getEditor() {
      return this.editor;
    },
    getPreviewor() {
      return this.previewer;
    },
    getPreviewHtml() {
      let csslinks = [];
      const cssTypes = this.previewer.cssTypes;
      if (cssTypes.hasKatex) {
        csslinks.push(cssTypes.css.katex);
      }
      if (cssTypes.hasSequence) {
        csslinks.push(cssTypes.css.sequence);
      }
      if (cssTypes.hasRailroad) {
        csslinks.push(cssTypes.css.railroad);
      }
      let cssstyles = [];
      const vegaStyle = document.getElementById('vega-embed-style');
      if (vegaStyle && vegaStyle.tagName.toLowerCase() === 'style') {
        let styleHTML = vegaStyle.outerHTML;
        styleHTML = styleHTML.replace(/\<br[\/]?\>/g, '');
        cssstyles.push(styleHTML);
      }
      return {
        html: this.previewer ? this.previewer.$el.innerHTML : '',
        cssLinks: csslinks,
        cssStyles: cssstyles
      };
    },
    addRootMenu(item) {
      if (this.rootSet) this.rootSet.menuItems.push(item);
    },
    handleClick(e, item) {
      if (item.type) {
        if (!this.insertMarkdownContent(item.type)) {
          switch (item.type) {
            case 'zh-cn':
              i18n.lang = item.type;
              item.disabled = true;
              if (this.activeLangItem) {
                this.activeLangItem.disabled = false;
              }
              this.activeLangItem = item;
              break;
            case 'en':
              i18n.lang = item.type;
              item.disabled = true;
              if (this.activeLangItem) {
                this.activeLangItem.disabled = false;
              }
              this.activeLangItem = item;
              break;
            default:
              this.$emit('toolclick', item.type);
              break;
          }
        }
      }
    },
    onMdChange(doc) {
      this.mdDoc = doc;
    },
    onMdScroll(cm) {
      if (this.previewer) {
        try {
          // scrolling with the same percentage
          const scrollInfo = cm.getScrollInfo();
          let previewElement = this.previewer.$el;
          previewElement.scrollTop = (previewElement.scrollHeight * scrollInfo.top) / scrollInfo.height;
        } catch (e) {
          console.log('scroll current object to failed:', cm);
        }
      }
    }
  },
  mounted() {
    this.editor = this.$refs.mdseEditor;
    this.previewer = this.$refs.mdsePreviewer;
    this.rootmenu = this.$refs.rootmenu;
    if (this.rootSet) {
      this.rootSet.menuItems.find(
        function(item) {
          if (item.type === i18n.locale) {
            item.disable = true;
            this.activeLangItem = item;
            return true;
          }
          return false;
        }.bind(this)
      );
    }
    if (this.hasToolbar) {
      this.edittool = this.$refs.edittool;
      this.titletool = this.$refs.titletool;
      if (this.titletool) this.titleToolWidth = this.titletool.$el.clientWidth;
      this.fonttool = this.$refs.fonttool;
      if (this.fonttool) this.fontToolWidth = this.fonttool.$el.clientWidth;
      this.aligntool = this.$refs.aligntool;
      if (this.aligntool) this.alignToolWidth = this.aligntool.$el.clientWidth;
      this.listtool = this.$refs.listtool;
      if (this.listtool) this.listToolWidth = this.listtool.$el.clientWidth;
      this.othertool = this.$refs.othertool;
      this.getLastToolOffset();

      let that = this;
      window.addEventListener('resize', function() {
        that.screenWidth = document.documentElement.clientWidth;
        that.getLastToolOffset();
      });
    }
  }
};
</script>

<style lang="stylus">
.mde-workspace-container {
  position: relative;
  flex: 1;
  outline: none;
  min-width: 885px;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  flex-wrap: nowrap;

  .mde-toolbar {
    position: relative;
    border-bottom: 1px solid #ddd;
    display: inline-flex;
    flex-direction: row;
    flex-wrap: nowrap;
    z-index: 201;

    .mde-toolbar-actions {
      padding: 4px 5px;
      min-height: 44px;
      display: inline-flex;
      flex-direction: row;
      flex-wrap: nowrap;

      &.nav {
        min-width: 190px;

        .mde-ui.buttons {
          margin: 0 1em 0 0;
        }
      }
    }
  }

  .mde-editor-container {
    position: relative;
    flex: 1;

    .raw-editor {
      min-width: 442px;
    }

    .mde-scroll-container {
      overflow: hidden;
      width: 100%;
      height: 100%;

      .mdse-preview {
        width: auto;
        height: 100%;
        overflow-y: auto;
      }
    }

    ::-webkit-scrollbar {
      -webkit-appearance: none;
      width: 10px;
      height: 10px;
    }

    ::-webkit-scrollbar-track {
      background: rgb(241, 241, 241);
      border-radius: 0;
    }

    ::-webkit-scrollbar-thumb {
      cursor: pointer;
      border-radius: 5px;
      background: rgba(0, 0, 0, 0.25);
      transition: color 0.2s ease;
    }

    ::-webkit-scrollbar-thumb:window-inactive {
      background: rgba(0, 0, 0, 0.15);
    }

    ::-webkit-scrollbar-thumb:hover {
      background: rgba(128, 135, 139, 0.8);
    }
  }
}
</style>