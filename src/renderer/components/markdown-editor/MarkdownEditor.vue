<template>
  <div class="root">
    <aside class="side-bar">
      <div class="toolbar" id="test">
        <button @click="addNote" :title="noteManager.notes.length + ' note(s) already'"><i class="fa fa-plus" aria-hidden="true"></i> Add note</button>
      </div>
      <div class="notes">
        <div class="note" v-for="note of sortedNotes" :class="{selected: note === selectedNote}" @click="selectNote(note)" @contextmenu="openNoteContextMenu(note)">{{note.title}}</div>
      </div>
    </aside>
    <!-- vmd ==> vue markdown -->
    <div class="vmd" ref="vmd">
      <div class="vmd-header" ref="vmdHeader">
        <div class="vmd-btn-group">
          <button type="button" class="vmd-btn vmd-btn-default" @click="addBold" title="Bold (Ctrl + B)" :disabled="disabled"><i class="fa fa-bold" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addItalic" title="Italic (Ctrl + I)" :disabled="disabled"><i class="fa fa-italic" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addHeading(markdown.h1)" title="Head 1 (Ctrl + 1)" :disabled="disabled"><i class="fa heading-bold" aria-hidden="true">H1</i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addHeading(markdown.h2)" title="Head 2 (Ctrl + 2)" :disabled="disabled"><i class="fa heading-bold" aria-hidden="true">H2</i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addHeading(markdown.h3)" title="Head 3 (Ctrl + 3)" :disabled="disabled"><i class="fa heading-bold" aria-hidden="true">H3</i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addStrikethrough" title="Strikethrough (Ctrl + D)" :disabled="disabled"><i class="fa fa-strikethrough" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addHR" title="Horizontal rule (Ctrl + R)" :disabled="disabled"><i class="fa fa-minus" aria-hidden="true"></i></button>
          
        </div>
        <div class="vmd-btn-group">
          <button type="button" class="vmd-btn vmd-btn-default" @click="addUl" title="Unordered list (Ctrl + U)" :disabled="disabled"><i class="fa fa-list-ul" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addOl" title="Ordered list (Ctrl + O)" :disabled="disabled"><i class="fa fa-list-ol" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addTable" title="Table (Ctrl + T)" :disabled="disabled"><i class="fa fa-table" aria-hidden="true"></i></button>
        </div>
        <div class="vmd-btn-group">
          <button type="button" class="vmd-btn vmd-btn-default" @click="addLink" title="Hyperlink (Ctrl + L)" :disabled="disabled"><i class="fa fa-link" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addImage" title="Image (Ctrl + G)" :disabled="disabled"><i class="fa fa-picture-o" aria-hidden="true"></i></button>
        </div>
        <div class="vmd-btn-group">
          <button type="button" class="vmd-btn vmd-btn-default" @click="addCode" title="Code (Ctrl + K)" :disabled="disabled"><i class="fa fa-code" aria-hidden="true"></i></button>
          <button type="button" class="vmd-btn vmd-btn-default" @click="addQuote" title="Quote (Ctrl + Q)" :disabled="disabled"><i class="fa fa-quote-left" aria-hidden="true"></i></button>
        </div>
        <div class="vmd-btn-group">
          <button type="button" title="Preview" class="vmd-btn vmd-btn-default" @click="preview"><i :class="previewClass" aria-hidden="true"></i></button>
          <button type="button" title="HTML" class="vmd-btn vmd-btn-default" @click="sanitizeHtml">HTML</button>
        </div>
        <div class="vmd-btn-group">
          <button type="button" title="Save to GitHub" class="vmd-btn vmd-btn-default" @click="saveGitHub" :disabled="disabled"><i class="fa fa-github" aria-hidden="true"></i></button>
          <button type="button" title="Export to file" class="vmd-btn vmd-btn-default" @click="exportFile" :disabled="disabled"><i class="fa fa-floppy-o" aria-hidden="true"></i></button>
        </div>
      </div>
      <div class="vmd-body" ref="vmdBody">
        <template v-if="selectedNote">
          <CodeMirrorEditor v-model="selectedNote.content" :options="cmOption" :readonly="disabled" @scroll="vmdSyncScrolling(cm)" @focus="vmdActive" @blur="vmdInactive" @contextmenu="onCMContextmenu" class="vmd-editor CodeMirror" style="overflow-y: hidden; padding: 0px;" ref="vmdEditor"></CodeMirrorEditor>
          <div class="vmd-preview markdown-body" ref="vmdPreview" v-show="isPreview" v-html="compiledMarkdown" @contextmenu="copyHTMLContextMenu()"></div>
        </template>
      </div>
      <div class="vmd-footer" ref="vmdFooter">
        <a type="button" class="vmd-btn vmd-btn-default vmd-btn-borderless">Markdown</a>
        <a type="button" class="txt">Words: {{ wordsCount }}</a>
        <a type="button" class="txt">Characters: {{ charactersCount }}</a>
        <a type="button" class="message" @click="updateMessage()">{{ message }}</a>
      </div>
    </div>
  </div>

</template>

<script>
  // 引入依赖库
  import marked from 'marked'
  import hljs from 'highlight.js'

  import './styles/markdown.css'
  import './styles/font-awesome-4.7.0/css/font-awesome.min.css'

  import MEditor from './lib/meditor'
  import DataProvider from './lib/data-provider'
  import ElectronUtil from './lib/electron-util'
  import GitHub from './lib/github'
  import Note from './lib/note'
  import NoteManager from './lib/note-manager'
  import CodeMirrorEditor from './CodeMirrorEditor'

  const electron = require('electron');
  const remote = electron.remote;
  const Menu = remote.Menu;
  const MenuItem = remote.MenuItem;
  const fs = require('fs');
  console.log('start');
  const dataProvider = new DataProvider();
  const gh = new GitHub();


  // 配置marked环境
  marked.setOptions({
    highlight: function (code) {
      return hljs ? hljs.highlightAuto(code).value : code
    }
  });
  /**
   * 定义__debounce函数
   *
   * @param fn 最终将执行的方法
   * @param delay 延时
   */
  function __debounce(fn, delay) {
    let timer = null;
    return function () {
      let context = this;
      let args = arguments;
      // 清除 timer
      clearTimeout(timer);
      timer = setTimeout(function () {
        fn.apply(context, args);
      }, delay);
    }
  }

  export default {
    name: 'MarkdownEditor',
    components:{CodeMirrorEditor},
    props: {
      value: {
        type: String,
        default: null
      }
    },
    data() {
      return {
        vmd: null,
        vmdBody: null,
        vmdHeader: null,
        vmdFooter: null,
        vmdPreview: null,
        isPreview: true,
        isSanitize: true,
        selectedId: dataProvider.loadSelectedNoteId(),
        noteContextMenu: null,
        cmContextmenu: null,
        htmlContextMenu: null,
        contextMenuOpNote: null,
        addedLink: false,
        messageTxt: "",
        me: null,
        markdown: MEditor.Markdown,
        noteManager: new NoteManager(dataProvider),
        cmOption: {
          tabSize: 4,
          theme: 'default',
        },
      }
    },
    created(){
      let that = this;
      this.noteContextMenu = new Menu();
      this.noteContextMenu.append(new MenuItem({
        label: "New note with title", click: function(){
          let inputDef = {
            title:'Enter the title', 
            inputs:[{
              msg: "New note title", 
              val:'', 
              required: true, 
              name: "title"
            }
            ]
          }

          let ret = ElectronUtil.inputPrompt(inputDef);
          if(ret != null){
            that.addNoteWithTitle(ret.title);
          }
        }
      }));
      this.noteContextMenu.append(new MenuItem({ label: 'Rename', click: function(){
          let ret = ElectronUtil.inputPrompt({
            title:'Rename', 
            inputs:[ 
            {
              msg: "New note title", 
              val:that.contextMenuOpNote.title, 
              required: true, 
              name: "title"
            }
            ]
          })
          if(ret && ret.title != that.contextMenuOpNote.title){
            if(that.contextMenuOpNote.github){
              let renameNote = that.contextMenuOpNote;
              ElectronUtil.showProgressDialog();
              that.renameGHNote(that.contextMenuOpNote, ret.title).then(function(r){
                  renameNote.title = ret.title;
                  ElectronUtil.updateProgress(`${renameNote.title} has been renamed to ${ret.title}`);
                  ElectronUtil.finishProgress();
              }).catch(function(err){
                // if(ElectronUtil.confirm("Confirm", `Fail to rename on GitHub:,status: ${err.statusCode}, message: ${err.message}. Continue rename locally?`)){
                //   renameNote.title = ret.title;
                // }
                ElectronUtil.updateProgress(`Fail to rename on GitHub:,status: ${err.statusCode}, message: ${err.message}`);
                ElectronUtil.finishProgress();
              });
            }else{
              that.contextMenuOpNote.title = ret.title;
            }
            
          }
          that.contextMenuOpNote = null;
        }}))
      this.noteContextMenu.append(new MenuItem({ type: 'separator' }))
      this.noteContextMenu.append(new MenuItem({ label: 'Delete', click: function(){
          if(ElectronUtil.confirm("Confirm", `Are you sure to delete ${that.contextMenuOpNote.title} ?`)){
            that.removeNote(that.noteManager.removeNote(that.contextMenuOpNote));
          }
          that.contextMenuOpNote = null;
        }}));

      this.htmlContextMenu = new Menu();
      this.htmlContextMenu.append(new MenuItem({ label: "Copy HTML", click: function(){
        let clipboard = electron.clipboard;
        let html = that.compiledMarkdown;
        clipboard.writeHTML(html);
      }}));

      this.cmContextMenu = new Menu();
      this.cmContextMenu.append(new MenuItem({
        label: 'Undo',
        accelerator: "Ctrl+Z",
        id: 'undo',
        click: () => this.cm.undo(),
      }));

      this.cmContextMenu.append(new MenuItem({
        label: "Redo",
        accelerator: "Ctrl+Y",
        id: "redo",
        click: () => this.cm.redo(),
      }));

      this.cmContextMenu.append(new MenuItem({
        type: 'separator'
      }));

      this.cmContextMenu.append(new MenuItem({
        label: 'Cut',
        accelerator: "Ctrl+X",
        id: "cut",
        click: () => this.me.cutSelection(),
      }));

      this.cmContextMenu.append(new MenuItem({
        label: "Copy",
        accelerator: "Ctrl+C",
        id: "copy",
        click: () => this.me.copySelection(),
      }));

      this.cmContextMenu.append(new MenuItem({
        label: "Paste",
        accelerator: "Ctrl+V",
        id: "paste",
        click: () => this.me.paste(),
      }));

      this.cmContextMenu.append(new MenuItem({
        label: "Delete",
        accelerator: "Delete",
        id: "delete",
        click: () => this.me.deleteSelection(),
      }));

      this.cmContextMenu.append(new MenuItem({
        type: 'separator'
      }));

      this.cmContextMenu.append(new MenuItem({
        label: "Select All",
        accelerator: "Ctrl+A",
        click: () => this.cm.execCommand('selectAll'),
      }));
    },
    updated: function () {
      this.$nextTick(function () {
        // Code that will run only after the
        // entire view has been re-rendered
        if(this.addedLink){
          this.addedLink = false;
          ElectronUtil.hackLinks();
        };
      })
    },
    computed: {
      /**
       * 编译成markdown文档
       */
      compiledMarkdown() {
        return marked(this.selectedNote.content, {sanitize: this.isSanitize})
      },
      vmdEditorStyle() {
        return this.isPreview ? null : {
          width: '100%'
        }
      },
      disabled(){
        return this.selectedId == null || this.selectedNote.id == null || this.noteManager.notes.length == 0;
      },
      previewClass() {
        return this.isPreview ? 'fa fa-eye-slash' : 'fa fa-eye'
      },
      selectedNote () {
        // We return the matching note with selectedId
        return this.noteManager.findNoteById(this.selectedId, Note.DefaultNote);
      },
      sortedNotes () {
        return this.noteManager.notes.slice().sort((a, b) => a.created - b.created)
      },
      linesCount () {
        if (this.selectedNote) {
          // Count the number of new line characters
          return this.selectedNote.content.split(/\r\n|\r|\n/).length
        }
      },

      wordsCount () {
        if (this.selectedNote) {
          var s = this.selectedNote.content
          // Turn new line cahracters into white-spaces
          s = s.replace(/\n/g, ' ')
          // Exclude start and end white-spaces
          s = s.replace(/(^\s*)|(\s*$)/gi, '')
          // Turn 2 or more duplicate white-spaces into 1
          s = s.replace(/[ ]{2,}/gi, ' ')
          // Return the number of spaces
          return s.split(' ').length
        }
      },

      charactersCount () {
        if (this.selectedNote) {
          return this.selectedNote.content.split('').length
        }
      },
      message (){
        return this.messageTxt;
      },
    },
    mounted() {
      // 缓存DOM组件
      this.__saveDom();
      // 重置组件大小
      this.__resize();
      // 添加滚动监听事件
      window.addEventListener('resize', this.vmdResize, false);
      // this.cm.addEventListener('scroll', this.vmdSyncScrolling, false);
      // this.vmdPreview.addEventListener('scroll', this.vmdSyncScrolling, false);
      // 自动获取焦点
      this.cm.focus();
      ElectronUtil.hackLinks();
    },
    beforeDestroy() {
      // 移除滚动监听事件
      dataProvider.saveNotes(this.noteManager.rawNotes)
      window.removeEventListener('resize', this.vmdResize, false);
      // this.vmdEditor.removeEventListener('scroll', this.vmdSyncScrolling, false);
      // this.vmdPreview.removeEventListener('scroll', this.vmdSyncScrolling, false);

      // 移除DOM组件
      this.__removeDom();
    },
    methods: {
      onCMContextmenu(e){
        let readOnly = e.isReadOnly();
        const findMenu = (menu, id) => menu.items.find(item => item.id && item.id == id);
        const hasSel = (e.doc.getSelection().length > 0);
        findMenu(this.cmContextMenu, 'redo').enabled = !readOnly && e.doc.historySize().redo > 0;
        findMenu(this.cmContextMenu, 'undo').enabled = !readOnly && e.doc.historySize().undo > 0;
        findMenu(this.cmContextMenu, 'copy').enabled = hasSel;
        findMenu(this.cmContextMenu, 'paste').enabled = !readOnly && this.me.canPaste();
        findMenu(this.cmContextMenu, 'cut').enabled = !readOnly && hasSel;
        findMenu(this.cmContextMenu, 'delete').enabled = !readOnly && hasSel;
        this.cmContextMenu.popup(remote.getCurrentWindow());
      },
      checkGHToken(){
        return new Promise(function(resolve, reject){
          let ghToken = dataProvider.loadGitHubToken();
          if(!ghToken.user || !ghToken.token){
            let inputDef = {
              title:'Enter the GitHub user and access token', 
              inputs:[{
                msg: "GitHub user",
                val:'',
                required: true,
                name: "user"
              },
              {
                msg: "GitHub token",
                val:'',
                required: true,
                name: "token"
              }
              ]
            }
            let ret = ElectronUtil.inputPrompt(inputDef);
            if(ret && ret.token.trim() && ret.user.trim()){
              ghToken = {"user": ret.user, "token": ret.token};
              dataProvider.saveGitHubToken(ghToken);
              resolve(ghToken);
            } else{
              reject("user canceled");
            }
          }else{
            resolve(ghToken);
          }
        });
      },
      renameGHNote(oldN, newTitle){
        let that = this;
        return new Promise(function(resolve, reject){
          let newN = new Note(JSON.parse(JSON.stringify(oldN.data)));
          newN.title = newTitle;
          that.checkGHToken().then(function(ghToken){

            let oldFileName = oldN.ghfilename;
            let newFileName = newN.ghfilename;

            if(oldFileName && newFileName){
              gh.renameFile(ghToken, '_posts', oldFileName, '_posts', newFileName, newN.ghcontent).then(function(r){
                resolve(r);
              }).catch(function(err){
                reject(err);
              });
            } else{
              ElectronUtil.errorAlert('Note Format Error', `Can't parse file name for note: ${note.title}`);
            }
            
          }).catch(function(err){});
        });
      },
      addNote(){
        this.selectNote(this.noteManager.addNote());
      },
      updateMessage(message){
        this.messageTxt = message? message : "";
      },
      addNoteWithTitle(title){
        this.selectNote(this.noteManager.addNote(title));
      },
      vmdActive() {
        this.$refs.vmd.classList.add('active')
      },
      vmdInactive() {
        this.$refs.vmd.classList.remove('active')
      },
      sanitizeHtml() {
        this.isSanitize = !this.isSanitize;
      },
      saveGitHub() {
        let that = this;

        this.checkGHToken().then(function(ghToken){
          let selNote = that.selectedNote;
          let note = new Note(selNote.data);
          let ghfilename = note.ghfilename;
          
          if(ghfilename){
            ElectronUtil.showProgressDialog();
            gh.saveFile(ghToken, '_posts', ghfilename, note.ghcontent, function(sta){
              ElectronUtil.updateProgress(sta);
            }).then(function(result){
              ElectronUtil.updateProgress(`File ${ghfilename} saved. New sha is ${result.commit.sha}`);
              selNote.github = true;
              ElectronUtil.finishProgress();
            }).catch(function(err){
              ElectronUtil.updateProgress(`GitHub Error ${err.statusCode}: ${err.message}`);
              ElectronUtil.finishProgress();
            });

          }else{
            ElectronUtil.errorAlert('Note Format Error', `Can't parse file name for note: ${selNote.title}`);
          }
        }).catch(function(err){});
      },
      exportFile(){
        var dialog = remote.dialog;
        let that = this;

        dialog.showSaveDialog(remote.getCurrentWindow(), {
            title: 'Save ' + this.selectedNote.title,
            defaultPath: '~/' + this.selectedNote.title + ".markdown"
          }, function(result){
            if(result){
              that.updateMessage("Saving file to " + result);
              fs.writeFile(result, that.selectedNote.content, function(err) {
                  if(err) {
                    that.updateMessage();
                    ElectronUtil.errorAlert('File Save Error', err)
                  } else {
                    that.updateMessage("File saved " + result);
                    __debounce(function(e){
                      that.updateMessage();
                    }, 10000);
                  }
              }); 
            }
            
        });
      },
      /**
       * 同步滚动
       */
      vmdSyncScrolling(e) {
        this.vmdPreview.scrollTop = this.vmdPreview.scrollHeight* e.getScrollInfo().top/e.getScrollInfo().height;
      },
      vmdResize: __debounce(function (e) {
        this.__resize()
      }, 100),
      preview() {
        this.isPreview = !this.isPreview
      },
      selectNote (note) {
        if(this.selectedNote){
          this.selectedNote.history = this.cm.getHistory();
        }
        this.selectedId = note.id
        this.cm.clearHistory();

        this.cm.setValue(note.content);

        if(note.history){
            this.cm.setHistory(note.history);
        }
        this.cm.focus()
      },
      removeNote(note){
        if(note){
          if(note.id === this.selectedId){
            if(this.noteManager.notes.length > 0){
              this.selectNote(this.noteManager.notes[0]);
            } else{
              this.selectedId = null;
            }
          }
        }
      },
      openNoteContextMenu(note){
        this.contextMenuOpNote = note;
        this.noteContextMenu.popup(remote.getCurrentWindow());
      },
      copyHTMLContextMenu(){
        this.htmlContextMenu.popup(remote.getCurrentWindow());
      },
      addBold(){
        this.me.addBold();
      },
      addItalic() {
        this.me.addItalic();
      },
      addStrikethrough() {
        this.me.addStrikethrough();
      },
      addHR(){
        this.me.addHorizonRule();
      },
      addHeading(heading) {
        this.me.addHeading(heading);
      },
      addQuote() {
        this.me.addQuote();
      },
      addCode() {
        this.me.addCode();
      },
      addLink() {
        let sel = this.me.getSelection();
        let link;

        let ret = ElectronUtil.inputPrompt({
            title:'Hyperlink', 
            inputs:[ 
            {
              msg: "Input URL", 
              val: 'http://', 
              required: true, 
              name: "link"
            },
            {
              msg: "Title (Optional)",
              val: sel.text,
              required: false,
              name: "title"
            }
            ]
          })

        link = ret == null? null : ret.link;
        let title = ret != null && ret.title ? ret.title : link;
        let urlRegex = new RegExp('^((http|https)://|(mailto:)|(//))[a-z0-9]', 'i');
        
    
        if (link !== null && link !== '' && link !== 'http://' && urlRegex.test(link)) {
          let div = document.createElement('div');
          div.appendChild(document.createTextNode(link));
          let sanitizedLink = div.innerHTML;

          // 替换选择内容并将光标设置到chunk内容前
          let replaceContent = '[' + title + '](' + sanitizedLink + ')';
          this.me.addNormalText(replaceContent);

          this.addedLink = true;
        }else{
          this.cm.focus();
        }
      },
      addImage() {
        let sel = this.me.getSelection();
        let link;
        let ret = ElectronUtil.inputPrompt({
            title:'Image', 
            inputs:[ 
            {
              msg: "Input Image URL", 
              val: 'http://', 
              required: true, 
              name: "link"
            },
            {
              msg: "Alternate (Optional)",
              val: sel.text,
              required: false,
              name: "alternate"
            }
            ]
          })
        link = ret == null ? null : ret.link;
        let alternate = ret != null &&ret.alternate ? ret.alternate : link;

        let urlRegex = new RegExp('^((http|https)://|(//))[a-z0-9]', 'i');
        if (link !== null && link !== '' && link !== 'http://' && urlRegex.test(link)) {
          let div = document.createElement('div');
          div.appendChild(document.createTextNode(link));
          let sanitizedLink = div.innerHTML;

          // 替换选择内容并将光标设置到chunk内容前
          let replaceText = '![' + alternate + '](' + sanitizedLink + ' "' + alternate + '")';
          this.me.addNormalText(replaceText);
        }else{
          this.cm.focus();
        }
      },
      addTable() {
        this.me.addTable();
      },
      addUl() {
        this.me.addUl();
      },
      addOl() {
        this.me.addOl();
      },
      fullscreen() {

      },
      __saveDom() {
        this.vmd = this.$refs.vmd;
        this.vmdBody = this.$refs.vmdBody;
        this.vmdHeader = this.$refs.vmdHeader;
        this.vmdFooter = this.$refs.vmdFooter;
        this.vmdPreview = this.$refs.vmdPreview;
        this.cm = this.$refs.vmdEditor.cm;
        this.me = new MEditor(this.cm);
        let that = this;
        let extraKeys = {
            Tab: (cm) => that.me.addTab(),
            Enter: (cm) => that.me.addEnter(),
            "Ctrl-I": (cm) => that.me.addItalic(),
            "Ctrl-B": (cm) => that.me.addBold(),
            "Ctrl-1": (cm) => that.me.addHeading(MEditor.Markdown.h1),
            "Ctrl-2": (cm) => that.me.addHeading(MEditor.Markdown.h2),
            "Ctrl-3": (cm) => that.me.addHeading(MEditor.Markdown.h3),
            "Ctrl-D": (cm) => that.me.addStrikethrough(),
            "Ctrl-R": (cm) => that.me.addHorizonRule(),
            "Ctrl-U": (cm) => that.me.addUl(),
            "Ctrl-O": (cm) => that.me.addOl(),
            "Ctrl-T": (cm) => that.me.addTable(),
            "Ctrl-L": (cm) => that.me.addLink(),
            "Ctrl-G": (cm) => that.me.addImage(),
            "Ctrl-K": (cm) => that.me.addCode(),
            "Ctrl-Q": (cm) => that.me.addQuote(),
        };
        this.cm.setOption("extraKeys", extraKeys);
        this.cm.setOption("readOnly", this.disabled);
      },
      __removeDom() {
        this.vmd = null;
        this.vmdBody = null;
        this.vmdHeader = null;
        this.vmdFooter = null;
        this.vmdPreview = null
      },
      __resize() {
        let vmdHeaderOffset = this.vmdHeader ? this.vmdHeader.offsetHeight : 0,
        vmdFooterOffset = this.vmdFooter ? this.vmdFooter.offsetHeight : 0;
        this.vmdBody.style.height = (this.vmd.offsetHeight - vmdHeaderOffset - vmdFooterOffset) + 'px';

      },
    },  // Change watchers
    watch: {
      // When our notes change, we save them
      noteManager: {
        // The method name
        handler: function(){
          dataProvider.saveNotes(this.noteManager.rawNotes);
        },
        // We need this to watch each note's properties inside the array
        deep: true,
      },
      // Let's save the selection too
      selectedId (val, oldVal) {
        dataProvider.saveSelectedNoteId(val);
      },
    },
  }
</script>

<style>

  *:focus {
    outline: none;
  }

  *, :after, :before {
    -webkit-box-sizing: border-box;
    -moz-box-sizing: border-box;
    box-sizing: border-box;
  }
  .material-icons {
    font-size: 24px;
    line-height: 1;
    vertical-align: middle;
    margin: -3px;
    padding-bottom: 1px;
  }

  button {
    border-radius: 3px;
    border: none;
    display: inline-block;
    padding: 4px 6px;
    cursor: pointer;
  }

  button:enabled:hover {
    background: #63c89b;
  }

  button:disabled {
    background: #dddddd;
  }

  button:disabled:hover {
    background: #dddddd;
  }

  .CodeMirror { height: inherit !important; }

  .toolbar {
    padding: 4px;
    box-sizing: border-box;
  }

  .heading-bold {
    display: inline-block;
    font-weight: bold;
  }

  .txt {
    display: inline-block;
    padding: 6px 12px;
    margin-bottom: 0;
    font-size: 14px;
    font-weight: 400;
    line-height: 1.42857143;
    text-align: center;
    white-space: nowrap;
    vertical-align: middle;
    -ms-touch-action: manipulation;
    touch-action: manipulation;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    background-image: none;
    border: 1px solid transparent;
    border-radius: 4px;
  }

  .message {
    display: inline-block;
    padding: 6px 12px;
    float: right;
    margin-bottom: 0;
    font-size: 11px;
    font-weight: 400;
    line-height: 1.42857143;
    text-align: center;
    white-space: nowrap;
    vertical-align: middle;
    -ms-touch-action: manipulation;
    touch-action: manipulation;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    background-image: none;
    border: 1px solid transparent;
    border-radius: 4px;
  }

  .root {
    display: flex;
    width: 100%;
    height: 100%;
  }

  .side-bar {
    background: #f8f8f8;
    width: 20%;
    box-sizing: border-box;
    margin-left: 2px;
  }
  .note {
    padding: 10px;
    cursor: pointer;
    font-size: 12px;
  }

  .note:hover {
    background: #ade2ca;
  }
  .notes {
    flex: auto 1 1;
    overflow: auto;
  }
  .note.selected {
    background: #40b883;
    color: white;
  }
  .vmd {
    position: relative;
    width: 80%;
    height: 99%;
    overflow: hidden;
    border: thin solid #ddd;
    text-align: left;
    -webkit-border-radius: 5px;
    -moz-border-radius: 5px;
    border-radius: 5px;
    -webkit-transition: all .3s linear;
    -moz-transition: all .3s linear;
    -ms-transition: all .3s linear;
    -o-transition: all .3s linear;
    transition: all .3s linear;
    margin: 2px;
  }

  .vmd.active {
    border-color: #4395ff;
  }

  .vmd .vmd-header, .vmd .vmd-footer {
    display: block;
    padding: 6px;
    border-bottom: thin solid #ddd;
    background: #f5f5f5;
  }

  .vmd .vmd-header {
    border-bottom: thin solid #ddd;
  }

  .vmd .vmd-footer {
    border-top: thin solid #ddd;
  }

  .vmd .vmd-body {
    height: inherit;
  }

  .vmd-body .vmd-editor, .vmd-body .vmd-preview {
    display: block;
    padding: .8rem;
    height: inherit;
    width: 50%;
    min-height: 100px;
    float: left;
    overflow: auto;
    box-sizing: border-box;
    border-left: solid 4px #f8f8f8;
  }

  .vmd-body .vmd-editor {
    color: #3d4043;
    font-size: 1rem;
    line-height: 1.2rem;
    border: 0;
    resize: none;
    /*background: #00d1b2;*/
  }

  .vmd-body .vmd-preview {
    background: #fff;
  }

  .vmd-btn {
    display: inline-block;
    padding: 6px 12px;
    margin-bottom: 0;
    font-size: 14px;
    font-weight: 400;
    line-height: 1.42857143;
    text-align: center;
    white-space: nowrap;
    vertical-align: middle;
    -ms-touch-action: manipulation;
    touch-action: manipulation;
    cursor: pointer;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    background-image: none;
    border: 1px solid transparent;
    border-radius: 4px;
  }

  .vmd-btn:hover {
    color: #333;
    text-decoration: none;
  }

  .vmd-btn:active,
  .vmd-btn.active {
    background-image: none;
    outline: 0;
    -webkit-box-shadow: inset 0 3px 5px rgba(0, 0, 0, .125);
    box-shadow: inset 0 3px 5px rgba(0, 0, 0, .125);
  }

  .vmd-btn:focus {
    outline: none;
  }

  .vmd-btn-default {
    color: #333;
    background-color: #fff;
    border-color: #ccc;
  }

  .vmd-btn-default:hover {
    color: #333;
    background-color: #e6e6e6;
    border-color: #adadad;
  }

  .vmd-btn-default:active,
  .vmd-btn-default.active {
    color: #333;
    background-color: #e6e6e6;
    border-color: #adadad;
  }

  .vmd-btn-borderless {
    padding-top: 7px;
    padding-bottom: 7px;
    border: 0;
  }

  .vmd-btn-borderless, .vmd-btn-borderless:hover, .vmd-btn-borderless:active, .vmd-btn-borderless.active {
    box-shadow: none;
    background-color: transparent;
  }

  .vmd-btn-default:hover.vmd-btn-borderless {
    opacity: .5;
  }

  .vmd-btn-default:active.vmd-btn-borderless, .vmd-btn-default.active.vmd-btn-borderless {
    opacity: .7;
  }

  .vmd-btn-group {
    position: relative;
    display: inline-block;
    vertical-align: middle;
  }

  .vmd-btn-group > .vmd-btn {
    position: relative;
    float: left;
  }

  .vmd-btn-group > .vmd-btn:hover,
  .vmd-btn-group > .vmd-btn:focus,
  .vmd-btn-group > .vmd-btn:active,
  .vmd-btn-group > .vmd-btn.active {
    z-index: 2;
  }

  .vmd-btn-group .vmd-btn + .vmd-btn,
  .vmd-btn-group .vmd-btn + .vmd-btn-group,
  .vmd-btn-group .vmd-btn-group + .vmd-btn,
  .vmd-btn-group .vmd-btn-group + .vmd-btn-group {
    margin-left: -1px;
  }

  .vmd-btn-group > .vmd-btn:not(:first-child):not(:last-child):not(.dropdown-toggle) {
    border-radius: 0;
  }

  .vmd-btn-group > .vmd-btn:first-child {
    margin-left: 0;
  }

  .vmd-btn-group > .vmd-btn:first-child:not(:last-child):not(.dropdown-toggle) {
    border-bottom-right-radius: 0;
    border-top-right-radius: 0;
  }

  .vmd-btn-group > .vmd-btn:last-child:not(:first-child) {
    border-bottom-left-radius: 0;
    border-top-left-radius: 0;
  }

  .vmd-btn-group > .vmd-btn-group {
    float: left;
  }

  .vmd-btn-group > .vmd-btn-group:not(:first-child):not(:last-child) > .vmd-btn {
    border-radius: 0;
  }

  .vmd-btn-group > .vmd-btn-group:first-child:not(:last-child) > .vmd-btn:last-child {
    border-bottom-right-radius: 0;
    border-top-right-radius: 0;
  }

  .vmd-btn-group > .vmd-btn-group:last-child:not(:first-child) > .vmd-btn:first-child {
    border-bottom-left-radius: 0;
    border-top-left-radius: 0;
  }

  .vmd-body:before, .vmd-body:after,
  .vmd-btn-group:before, .vmd-btn-group:after {
    display: table;
    content: '';
  }

  .vmd-body:after,
  .vmd-btn-group:after {
    clear: both;
  }
</style>
