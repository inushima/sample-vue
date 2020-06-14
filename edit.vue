<template>
  <v-layout id="edit-page" :class="$style.edit_page">
    <image-edit
      :image-file-cache="imageFileCache"
      @processed="uploadEyecatch"
    />
    <v-layout column :class="$style.main">
      <v-layout
        :class="[
          $style.eyecatch,
          { [$style.eyecatch_selected]: this.headerImagePath },
        ]"
      >
        <div v-if="this.headerImagePath" :class="$style.eyecatch_container">
          <img :src="this.headerImagePath" :class="$style.selected" />
          <div :class="$style.close_button" @click="deleteEyecatch"></div>
        </div>
        <div
          :class="[
            isEyecatchDragover ? $style.unselected_dragging : $style.unselected,
          ]"
          @click="onEyecatchUpload"
          @drop.prevent="onEyecatchDrop($event)"
          @dragover.prevent="onEyecatchDragover($event)"
          @dragleave.prevent="isEyecatchDragover = false"
          v-else
        >
          <v-layout
            align-center
            justify-center
            column
            :class="$style.eyecatch_wrapper"
          >
            <div :class="$style.line1">
              <img
                src="~/assets/icon/icon_edit_add.png"
                :class="$style.edit_add_icon"
              />
              <div :class="$style.line1_text">メイン画像を設定する</div>
            </div>
            <div :class="$style.attention">（推奨サイズ: 1280x670）</div>
          </v-layout>
          <input
            type="file"
            accept="image/jpeg, image/png"
            v-show="false"
            ref="headerImageForm"
            @change="eyecatchUpload"
          />
        </div>
      </v-layout>
      <div
        :class="$style.editor"
        class="editor"
        @dragover.prevent="onImageDragoverBody($event)"
        @dragleave.prevent="isImageDragoverBody = false"
      >
        <!-- <div
          id="title"
          :class="$style.title"
          @keypress.enter.prevent
          contenteditable="true"
          v-text="title"
          ref="titleForm"
          :disabled="!editable"
        ></div> -->
        <v-text-field
          single-line
          v-model="title"
          @keypress.enter.prevent
          color="#212121"
          placeholder="記事のタイトルを入力してください"
          :class="$style.title"
          :rules="titleRule"
        />
        <client-only placeholder="読み込み中...">
          <vue-editor
            id="editor"
            v-model="body"
            :editorToolbar="this.editorToolbar"
            :editorOptions="this.editorOptions"
            :disabled="!editable"
            :class="{ [$style.image_dragging_on_editor]: isImageDragoverBody }"
            placeholder="記事の内容を入力してください"
            useCustomImageHandler
            @image-added="handleImageAdded"
            @text-change="onBodyChange"
          />
          <div :class="$style.editor_alert">
            ※ URLだけ書かれた行は公開時にカード表示されます
          </div>
        </client-only>
      </div>
    </v-layout>
  </v-layout>
</template>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator'
import ImageEdit from '@/components/pages/ImageEdit.vue'
import HyperLink from '@/components/pages/HyperLink.vue'
import {
  payArticleLine,
  serializeBody,
  sanitizeDefaultOptions,
} from '@/factory/article'
import { dataURLToBlob } from 'blob-util'
import hljs from 'highlight.js'
import 'highlight.js/styles/monokai.css'

interface HTMLElementEvent extends Event {
  target: HTMLElement
}

interface FileInputTarget extends EventTarget {
  files: File[]
}

interface FileInputEvent extends Event {
  target: FileInputTarget
}

interface KeyPressEvent extends HTMLElementEvent {
  keyCode: number
  shiftKey: boolean
}

interface RouterParams {
  params: { [key: string]: any }
}

interface Selected {
  lineNumber: number
  begin: number
  end: number
  isSelected: boolean
  linkSelection: Node | null
}

@Component({
  middleware: ['authenticated'],
  components: {
    ImageEdit,
    HyperLink,
  },
  layout: 'editLayout',
})
export default class Edit extends Vue {
  public $refs!: {
    paragraph: HTMLElement[]
    headerImageForm: HTMLInputElement
    itemImageForm: HTMLInputElement
    linkInput: HTMLInputElement[]
  }
  private intervalId: number | NodeJS.Timeout = -1

  // リンク挿入時のURLのキャッシュデータ
  hyperLink = ''

  // 文字編集が行われるたびにカウントを増やす
  edittingCount = 0

  // 記事本文の編集可否フラグ
  editable = false

  // ヘッダー画像ドラッグ状態
  isEyecatchDragover = false

  // 本文画像ドラッグ状態
  isImageDragoverBody = false

  // store の情報をなんども変更しないためのcache
  publishable = false

  showInlineLinkInput = -1
  inlineHyperLink = ''

  // Quill Editor に関するもの
  title = ''
  body = ''
  editorToolbar = [
    // [{ header: [false, 2, 3] }],
    // [{ header: 1 }, { header: 2 }, { header: 3 }],
    [{ header: 2 }, { header: 3 }],
    ['tweet'],
    ['bold'],
    // [{ list: 'ordered' }, { list: 'bullet' }, { list: 'check' }],
    ['blockquote', 'code-block'],
    ['link', 'image', 'video'],
    [{ align: '' }, { align: 'center' }, { align: 'right' }],
  ]
  editorOptions = {
    theme: 'snow',
    modules: {
      syntax: {
        highlight: (text: any): string => {
          return hljs.highlightAuto(text).value
        },
      },
      imageDropAndPaste: { handler: this.quillImageDropHandler },
    },
    handlers: {
      tweet: function (value) {
        if (value) {
          const href = prompt('Enter the URL')
        } else {
          const href = prompt('Enter the URL')
        }
      },
    },
  }

  get titleRule(): string[] {
    const rules = []
    if (!this.title) {
      const rule = '記事のタイトルを入力してください'
      rules.push(rule)
    } else if (this.title.length > 100) {
      const rule = `記事のタイトルは100文字までです`
      rules.push(rule)
    }

    return rules
  }

  validate({ params }: RouterParams): boolean {
    return /^\d+$/.test(params.id)
  }

  get headerImagePath(): string {
    return this.$store.getters['editArticle/eyecacheImagePath']
  }

  get payArticleLineTag(): string {
    return payArticleLine
  }

  get imageFileCache(): File {
    return this.$store.getters['image/imageFileCache']
  }

  async created(): Promise<void> {
    /*
    記事本文の内容を初期読み込み時にセット
    */
    await this.$store.dispatch('editArticle/loadDraft', this.$route.params.id)
    this.body = serializeBody(this.$store.getters['editArticle/body'])
    this.title = this.$store.getters['editArticle/title'] || ''

    /*
    edit内の動作を layout/editableにも共有
    */
    this.$nuxt.$on('edit/saveDraft', this.saveDraft)
    this.$nuxt.$on('edit/saveDraftQuietly', this.saveDraftQuietly)
    /*
    公開可否を判定
    */
    this.publishable =
      this.title.length > 0 && this.title.length <= 100 && this.body.length > 0
    this.$store.dispatch('editArticle/setPublishable', this.publishable)

    /*
    本文編集の動作を hyperlinkにも共有
    */
    // this.$nuxt.$on('edit/detatchLinkComponent', this.detatchLinkComponent)

    this.editable = true
  }

  mounted(): void {
    if (process.client) {
      document
        .getElementById('title')
        ?.addEventListener('input', this.onTitleChange, false)

      this.$nextTick(() => {
        // quillの定義上書き
        const videoButton = document.querySelector('button.ql-video')
        videoButton?.addEventListener('click', () => {
          const videoInput = document.querySelector('input[data-video]')
          videoInput?.setAttribute('placeHolder', 'YouTube URL')
        })
      })
    }
  }

  beforeDestroy(): void {
    this.$store.dispatch('editArticle/setPublishable', false)
    this.$nuxt.$off('edit/saveDraft')
    this.$nuxt.$off('edit/saveDraftQuietly')
    this.$nuxt.$off('editView/resetFormImage')
    this.$nuxt.$off('imageEdit/completeLoading')
  }

  async saveDraftIfTextChanged(beforeEdittingCount: number): Promise<void> {
    if (beforeEdittingCount === this.edittingCount) {
      await this.saveDraftQuietly()
    }
  }

  onTitleChange(): void {
    this.onTextChange()
  }

  onBodyChange(delta: any, oldDelta: any, source: any): void {
    this.onTextChange()
  }

  onTextChange(): void {
    const sanitizedBody = (this as any).$sanitize(
      this.body,
      sanitizeDefaultOptions
    )
    if (this.body !== sanitizedBody) {
      this.body = sanitizedBody
    }

    // 公開可否判定
    const publishable =
      this.title.length > 0 && this.title.length <= 100 && this.body.length > 0

    if (!this.publishable === publishable) {
      this.publishable = publishable
      this.$store.dispatch('editArticle/setPublishable', publishable)
    }

    this.edittingCount += 1
    const thisEdittingCount = this.edittingCount

    // 初回ロード時は自動保存しないようにする
    if (this.edittingCount > 1) {
      setTimeout(this.saveDraftIfTextChanged, 5000, thisEdittingCount)
    }
  }

  getCurrentBody(): string[] {
    if (process.client) {
      const wrapperHtml = document.createElement('div')
      wrapperHtml.innerHTML = this.body
      const body = Array.from(wrapperHtml.children).map((x) => {
        return x.outerHTML
      })

      return body
    } else {
      return ['']
    }
  }

  onEyecatchUpload(): void {
    this.$refs.headerImageForm.click()
  }

  async eyecatchUpload(evt: FileInputEvent): Promise<void> {
    const file = evt.target.files[0]
    await this.processEyecatchUpload(file)
  }

  async onEyecatchDrop(evt: DragEvent): Promise<void> {
    if (evt.dataTransfer == null) {
      return
    }
    this.isEyecatchDragover = false
    const files = evt.dataTransfer.files
    await this.processEyecatchUpload(files[0])
  }

  private async processEyecatchUpload(file: File): Promise<void> {
    await this.$store.dispatch('image/upload', file)
    await this.$store.dispatch('modal/show')

    // TODO: 全体の Listen は不具合のもとなので後日方法を変更する
    this.$nuxt.$on('editView/resetFormImage', this.resetFormImage)
  }

  onEyecatchDragover(evt: DragEvent): void {
    this.isEyecatchDragover = this.handleImageDropover(evt)
  }

  onImageDragoverBody(evt: DragEvent): void {
    this.isImageDragoverBody = this.handleImageDropover(evt)
  }

  private handleImageDropover(evt: DragEvent): boolean {
    if (evt.dataTransfer == null) {
      return false
    }

    const items = evt.dataTransfer.items
    const type = items[0].type

    return type === 'image/jpeg' || type === 'image/png' || type === 'image/gif'
  }

  resetFormImage(): void {
    this.$refs.headerImageForm.value = ''
    this.$nuxt.$off('editView/resetFormImage')
  }

  deleteEyecatch(): void {
    this.$store.dispatch('editArticle/deleteEyecache')
  }

  async uploadEyecatch(file: File): Promise<void> {
    await this.$store.dispatch('image/upload', file)
    const url = await this.$store.dispatch('image/saveHeader')
    await this.$store.dispatch('editArticle/setEyecatch', url)
    await this.$store.dispatch('modal/hide')
    await this.$nuxt.$emit('imageEdit/completeLoading')
    this.$nuxt.$off('imageEdit/completeLoading')
  }

  async handleImageAdded(
    file: File,
    Editor: any,
    cursorLocation: any,
    resetUploader: any
  ): Promise<void> {
    this.editable = false
    const formData = new FormData()
    formData.append('image', file)

    await this.$store
      .dispatch('image/upload', file)
      .then(async () => {
        const url = await this.$store.dispatch('image/postItem')
        Editor.insertEmbed(cursorLocation, 'image', url)
      })
      .finally(() => {
        this.editable = true
      })
  }

  async quillImageDropHandler(imageDataUrl: any, type: any) {
    if (process.client) {
      // eslint-disable-next-line @typescript-eslint/no-var-requires
      const Quill = require('quill')

      this.editable = false
      if (
        type !== 'image/png' &&
        type !== 'image/jpeg' &&
        type !== 'image/gif'
      ) {
        this.editable = true
        return
      }

      const blob = dataURLToBlob(imageDataUrl)
      const formData = new FormData()
      formData.append('image', blob)

      const editor = document.getElementById('editor')
      if (!editor) return
      const quill = Quill.find(editor)

      await this.$store
        .dispatch('image/upload', blob)
        .then(async () => {
          const position =
            (quill.getSelection() || {}).index || quill.getLength()
          if (!position) return

          const url = await this.$store.dispatch('image/postItem')
          quill.insertEmbed(position, 'image', url)
        })
        .finally(() => {
          this.editable = true
          this.isImageDragoverBody = false
        })
    }
  }

  setTitle(title: string): void {
    this.$store.dispatch('editArticle/setTitle', title)
  }

  update(lineNumber: number, paragraph: string): void {
    this.$store.dispatch('editArticle/update', { lineNumber, paragraph })
  }

  async saveDraft(): Promise<void> {
    const title = this.title
    const body = this.getCurrentBody()

    await this.$store
      .dispatch('editArticle/saveDraft', { title, body })
      .then(() => {
        this.$store.dispatch('snackbar/setMessage', '下書きを保存しました')
        this.$store.dispatch('snackbar/snackOn')
      })
      .catch(() => {
        this.$store.dispatch(
          'snackbar/setMessage',
          '下書きを保存に失敗しました'
        )
        this.$store.dispatch('snackbar/snackOn')
      })
  }

  async saveDraftQuietly(): Promise<void> {
    const title = this.title
    const body = this.getCurrentBody()

    // snackbar での通知は保存に失敗したときだけにする
    await this.$store
      .dispatch('editArticle/saveDraft', { title, body })
      .catch(() => {
        this.$store.dispatch(
          'snackbar/setMessage',
          '下書きを保存に失敗しました'
        )
        this.$store.dispatch('snackbar/snackOn')
      })
  }

  isLinkContent(lineNumber: number): boolean {
    return Boolean(
      this.body[lineNumber].match(
        /^https?(:\/\/[-_.!~*\'()a-zA-Z0-9;\/?:\@&=+\$,%#]+)$/
      )
    )
  }

  isImageContent(lineNumber: number): boolean {
    return this.body[lineNumber].includes('<img')
  }

  isSmallImageContent(lineNumber: number): boolean {
    if (!this.$refs.paragraph) return false

    const img = this.$refs.paragraph[lineNumber].getElementsByTagName('img')
    if (img.length == 0) return false

    return img[0].classList.contains('small')
  }

  toggleImageScale(lineNumber: number): void {
    const img = this.$refs.paragraph[lineNumber].getElementsByTagName('img')[0]
    img.classList.toggle('small')
    this.$store.dispatch('editArticle/update', {
      lineNumber,
      paragraph: this.$refs.paragraph[lineNumber].innerHTML,
    })
  }

  deleteImage(lineNumber: number): void {
    if (!this.isImageContent(lineNumber)) return

    this.$store.dispatch('editArticle/update', {
      lineNumber,
      paragraph: '',
    })
  }
}
</script>

<style lang="scss">
/* global-style */
.articles__edit__image {
  cursor: pointer;
}

.articles__edit__image:not(.small) {
  max-width: 480px;
  max-height: 480px;
}

.articles__edit__image.small {
  max-width: 160px;
  max-height: 160px;
}

/* editor */
blockquote {
  background: $brain-gray-bg-color;
  padding: 32px;
}

// ql-editor の css
.quillWrapper {
  height: calc(100% + 20px);
}

.ql-toolbar.ql-snow {
  display: flex;
  overflow-x: auto;
  overflow-y: hidden;
  .ql-formats {
    display: flex;
  }
}

#edit-page {
  .v-input.v-text-field {
    .v-input__slot {
      padding-bottom: 4px;

      .v-text-field__slot {
        > input {
          font-size: 20px;
        }
      }
    }
  }
}

#editor {
  &.ql-container {
    margin-bottom: 60px; // footer
    height: calc(100% - 50px); // footer
  }

  .ql-editor {
    font-family: Hiragino Kaku Gothic ProN;
    font-size: 16px;

    &.ql-blank::before {
      font-style: normal;
      line-height: 2rem;
    }

    p,
    h1,
    h2,
    h3,
    pre {
      margin: 32px auto;
      &:first-child {
        margin: 0 auto 32px;
      }
    }

    p {
      line-height: 2rem;
    }

    blockquote {
      padding: 8px 32px;
      margin: 0;
      border: none;
    }
  }
}
</style>

<style lang="scss" module>
.edit_page {
  height: calc(100vh - 160px);
}

.main {
  max-width: $brain-max-width;
  margin: 0 auto;

  .eyecatch {
    max-width: $brain-max-width;
    max-height: 335px;
    width: 100%;
    height: 100%;
    background: $brain-button-disable-color;
    position: relative;
    display: block;

    &.eyecatch_selected {
      background: none;
    }

    &:before {
      content: '';
      display: block;
      padding-top: 52.3%;
    }

    .eyecatch_container {
      position: absolute;
      top: 0;
    }

    .unselected {
      width: 100%;

      background: $brain-button-disable-color;
      cursor: pointer;

      font-family: Hiragino Sans;
      font-weight: bold;
      font-size: 18px;
      color: #b5b5b5;

      border: 1px solid #ececec;
      box-sizing: border-box;
      border-radius: 2px;

      position: absolute;
      top: 0;
      bottom: 0;
      right: 0;
      left: 0;

      .attention {
        font-family: Hiragino Sans;
        font-weight: bold;
        font-size: 18px;
        color: #c2c2c2;
      }
    }

    .unselected_dragging {
      @extend .unselected;
      border: 3px dotted $brain-orange;
    }

    .eyecatch_wrapper {
      width: 100%;
      height: 100%;

      .line1 {
        display: flex;
        align-items: center;
      }

      .edit_add_icon {
        width: 16px;
        height: 16px;
        margin-right: 8px;
      }

      .line1_text {
        color: $brain-gray-text-color;
      }
    }

    .selected {
      width: $brain-max-width;
      max-width: 100%;
      object-fit: contain;
      background: $brain-button-disable-color;
      background-size: contain;
      position: relative;
    }

    .close_button {
      position: absolute;
      right: -12px;
      top: -12px;

      display: -webkit-flex;
      display: flex;
      justify-content: center;
      align-items: center;

      width: 28px;
      height: 28px;

      font-size: 24px;
      border-radius: 14px;

      background: #000000;
      color: #ffffff;
      cursor: pointer;
    }

    .close_button:after {
      content: '×';
    }
  }
}

.editor {
  height: 100%;

  .title {
    font-size: 26px;
    font-family: Hiragino Sans;
    color: $brain-text-color;
    max-width: $brain-max-width;
    width: 100%;
    margin: 32px 0;

    @include mobile {
      font-size: 20px;
    }
  }

  .title:not(:focus):empty {
    color: #b5b5b5;
  }

  .title:not(:focus):empty:after {
    content: '記事のタイトルを入力してください';
  }
}

.editor_alert {
  color: $brain-gray-text-color;
  line-height: 16px;
  margin-top: 10px;
  @include mobile {
    font-size: 12px;
  }
}

.image_dragging_on_editor {
  border: 3px dotted $brain-orange;
}
</style>
<style lang="css">
/* quillの定義上書き */
.ql-snow .ql-tooltip[data-mode='video']::before {
  content: 'YouTubeを埋め込む:' !important;
}
</style>
