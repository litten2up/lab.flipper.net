<template>
  <q-page class="column items-center q-pa-md" :class="$q.screen.width > 960 && $q.screen.height > 500 ? 'q-mt-xl' : 'q-mt-xs'">
    <div
      v-if="!connected || !flags.rpcActive || flags.rpcToggling"
      class="column flex-center q-my-xl"
    >
      <q-spinner
        color="primary"
        size="3em"
        class="q-mb-md"
      ></q-spinner>
      <p>Waiting for Flipper...</p>
    </div>
    <div v-if="connected && flags.rpcActive" class="file-container">
      <div class="file-menu flex no-wrap q-pa-xs rounded-borders">
        <q-btn
          flat
          dense
          icon="arrow_back_ios_new"
          class="q-mr-xs"
          style="width: 24px;"
          :disabled="path === '/'"
          @click="itemClicked({ name: '..' })"
        ></q-btn>
        <code
          class="q-py-xs q-px-sm bg-grey-3 rounded-borders overflow-hidden-y"
        >{{ path }}</code>
        <q-space></q-space>
        <q-btn flat dense icon="mdi-plus" :disabled="path === '/'">
          <q-menu auto-close self="top middle">
            <q-list style="min-width: 100px">
              <q-item clickable @click="flags.uploadPopup = true; uploadedFiles = null">
                <q-item-section avatar>
                  <q-icon name="mdi-file-upload-outline"/>
                </q-item-section>
                <q-item-section>
                  Upload file
                </q-item-section>
              </q-item>
              <q-item clickable @click="flags.uploadFolderPopup = true; uploadedFiles = null">
                <q-item-section avatar>
                  <q-icon name="mdi-folder-upload-outline"/>
                </q-item-section>
                <q-item-section>
                  Upload folder
                </q-item-section>
              </q-item>
              <q-item clickable @click="flags.mkdirPopup = true; editorText = ''">
                <q-item-section avatar>
                  <q-icon name="mdi-folder-plus-outline"/>
                </q-item-section>
                <q-item-section>
                  Create folder
                </q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </q-btn>
      </div>
      <q-list class="file-grid">
        <q-item
          v-for="item in dir"
          :key="item.name"
          v-bind="item"
          dense
          clickable
          style="border-radius: 3px;"
        >
          <q-item-section avatar @click="itemClicked(item)">
            <q-icon :name="itemIconSwitcher(item)"/>
          </q-item-section>

          <q-item-section @click="itemClicked(item)">
            <q-item-label>
              {{ item.name }}
            </q-item-label>
            <span v-if="path === '/' && item.name === 'int'">Flipper internal storage</span>
            <span v-if="path === '/' && item.name === 'ext'">SD card</span>
            <span v-if="item.type !== 1 && item.size" class="text-weight-light">{{ item.size }} bytes</span>
          </q-item-section>

          <q-item-section avatar>
            <q-btn v-if="path !== '/'" flat dense round icon="more_vert">
              <q-menu auto-close self="top middle">
                <q-list style="min-width: 100px">
                  <q-item v-if="item.type === 0" clickable @click="itemClicked(item)">
                    <q-item-section avatar>
                      <q-icon name="mdi-download-outline"/>
                    </q-item-section>
                    <q-item-section>
                      Download
                    </q-item-section>
                  </q-item>
                  <q-item v-if="item.name.endsWith('.sub') || item.name.endsWith('.ir')" clickable @click="openFileIn(item, 'pulse-plotter')">
                    <q-item-section avatar>
                      <q-icon name="mdi-share-outline"/>
                    </q-item-section>
                    <q-item-section>
                      Open in Pulse plotter
                    </q-item-section>
                  </q-item>
                  <q-item clickable @click="editorText = item.name; oldName = item.name; flags.renamePopup = true">
                    <q-item-section avatar>
                      <q-icon name="mdi-pencil-outline"/>
                    </q-item-section>
                    <q-item-section>
                      Rename
                    </q-item-section>
                  </q-item>
                  <q-item clickable class="text-negative" @click="flags.deletePopup = true; itemToDelete = item">
                    <q-item-section avatar>
                      <q-icon name="mdi-delete-outline"/>
                    </q-item-section>
                    <q-item-section>
                      Delete
                    </q-item-section>
                  </q-item>
                </q-list>
              </q-menu>
            </q-btn>
          </q-item-section>
        </q-item>
        <q-item v-if="dir.length === 0 && path !== '/'">
          <q-item-section avatar class="q-ml-xs">
            <q-icon name="mdi-folder-outline"/>
          </q-item-section>

          <q-item-section>
            <q-item-label>
              Empty folder
            </q-item-label>
          </q-item-section>
        </q-item>
      </q-list>
      <q-dialog v-model="flags.uploadFolderPopup">
        <q-card>
          <q-card-section class="q-pt-none">
            <q-file
              outlined
              multiple
              webkitdirectory
              v-model="uploadedFiles"
              label="Drop or select folder"
              class="q-pt-md folder-upload"
              :style="$q.screen.width > 380 ? 'width: 300px;' : ''"
            >
              <template v-slot:prepend>
                <q-icon name="mdi-folder-upload-outline"></q-icon>
              </template>
            </q-file>
          </q-card-section>

          <q-card-actions align="right">
            <q-btn
              flat
              label="Upload"
              v-close-popup
              @click="upload"
            ></q-btn>
            <q-btn
              flat
              label="Cancel"
              color="negative"
              v-close-popup
            ></q-btn>
          </q-card-actions>
        </q-card>
      </q-dialog>
      <q-dialog v-model="flags.deletePopup">
        <q-card>
          <q-card-section>
            <div class="text-subtitle1">Are you sure you want to delete <b>{{ itemToDelete.name }}</b>?</div>
            This action is permanent and can't be undone.
          </q-card-section>

          <q-card-actions align="right">
            <q-btn
              flat
              label="Cancel"
              v-close-popup
            ></q-btn>
            <q-btn
              flat
              label="Delete"
              color="negative"
              v-close-popup
              @click="remove(path + '/' + itemToDelete.name, !!itemToDelete.type)"
            ></q-btn>
          </q-card-actions>
        </q-card>
      </q-dialog>
      <q-dialog v-model="flags.uploadPopup">
        <q-card>
          <q-card-section class="q-pt-none">
            <q-file
              outlined
              multiple
              v-model="uploadedFiles"
              label="Drop or select files"
              class="q-pt-md"
              :style="$q.screen.width > 380 ? 'width: 300px;' : ''"
            >
              <template v-slot:prepend>
                <q-icon name="mdi-file-upload-outline"></q-icon>
              </template>
            </q-file>
          </q-card-section>

          <q-card-actions align="right">
            <q-btn
              flat
              label="Upload"
              v-close-popup
              @click="upload"
            ></q-btn>
            <q-btn
              flat
              label="Cancel"
              color="negative"
              v-close-popup
            ></q-btn>
          </q-card-actions>
        </q-card>
      </q-dialog>
      <q-dialog v-model="flags.renamePopup">
        <q-card>
          <q-card-section>
            <q-input
              v-model="editorText"
              :label="'Rename ' + oldName"
              :style="$q.screen.width > 380 ? 'width: 300px;' : ''"
            ></q-input>
          </q-card-section>

          <q-card-actions align="right">
            <q-btn
              flat
              label="Save"
              v-close-popup
              @click="rename(path, oldName, editorText)"
            ></q-btn>
            <q-btn
              flat
              label="Cancel"
              color="negative"
              v-close-popup
              @click="editorText = ''"
            ></q-btn>
          </q-card-actions>
        </q-card>
      </q-dialog>
      <q-dialog v-model="flags.mkdirPopup">
        <q-card>
          <q-card-section>
            <q-input
              v-model="editorText"
              label="Folder name"
              :style="$q.screen.width > 380 ? 'width: 300px;' : ''"
            ></q-input>
          </q-card-section>

          <q-card-actions align="right">
            <q-btn
              flat
              label="Create"
              v-close-popup
              @click="mkdir(path + '/' + editorText)"
            ></q-btn>
            <q-btn
              flat
              label="Cancel"
              color="negative"
              v-close-popup
              @click="editorText = ''"
            ></q-btn>
          </q-card-actions>
        </q-card>
      </q-dialog>
      <q-dialog v-model="flags.blockingOperationPopup" persistent>
        <q-card>
          <q-card-section>
            <div class="text-h6">File operation in progress</div>
          </q-card-section>
          <q-card-section v-if="file.name.length > 0">
            <ProgressBar
              :title="file.name"
              :progress="file.progress"
            />
          </q-card-section>
        </q-card>
      </q-dialog>
    </div>
  </q-page>
</template>

<script>
import { defineComponent, ref } from 'vue'
import { exportFile, useQuasar } from 'quasar'
import ProgressBar from 'components/ProgressBar.vue'
import asyncSleep from 'simple-async-sleep'
const flipperIcons = {
  'archive:new': 'img:icons/flipper/action-new.svg',
  'archive:remove': 'img:icons/flipper/action-remove.svg',
  'archive:rename': 'img:icons/flipper/action-rename.svg',
  'archive:save': 'img:icons/flipper/action-save.svg',
  'archive:sdcard': 'img:icons/flipper/location-sdcard.svg',
  'archive:internal': 'img:icons/flipper/location-internal.svg',
  'archive:file': 'img:icons/flipper/file.svg',
  'archive:folder': 'img:icons/flipper/folder.svg',
  'archive:badusb': 'img:icons/flipper/badusb.svg',
  'archive:ibutton': 'img:icons/flipper/ibutton.svg',
  'archive:infrared': 'img:icons/flipper/infrared.svg',
  'archive:nfc': 'img:icons/flipper/nfc.svg',
  'archive:rfid': 'img:icons/flipper/rfid.svg',
  'archive:subghz': 'img:icons/flipper/subghz.svg',
  'archive:u2f': 'img:icons/flipper/u2f.svg'
}

export default defineComponent({
  name: 'PageArchive',

  components: {
    ProgressBar
  },

  props: {
    flipper: Object,
    connected: Boolean,
    rpcActive: Boolean
  },

  setup () {
    const $q = useQuasar()

    $q.iconMapFn = (iconName) => {
      const icon = flipperIcons[iconName]
      if (icon !== void 0) {
        return { icon: icon }
      }
    }
    return {
      path: ref('/'),
      dir: ref([]),
      flags: ref({
        rpcActive: false,
        rpcToggling: false,
        uploadPopup: false,
        uploadFolderPopup: false,
        renamePopup: false,
        mkdirPopup: false,
        blockingOperationPopup: false,
        deletePopup: false
      }),
      uploadedFiles: ref(null),
      editorText: ref(''),
      oldName: ref(''),
      file: ref({
        name: '',
        progress: 0
      }),
      itemToDelete: ref(null)
    }
  },

  watch: {
    async connected (newStatus, oldStatus) {
      if (newStatus) {
        await this.start()
      }
    }
  },

  methods: {
    async startRpc () {
      this.flags.rpcToggling = true
      const ping = await this.flipper.commands.startRpcSession(this.flipper)
      if (!ping.resolved || ping.error) {
        this.$emit('showNotif', {
          message: 'Unable to start RPC session. Reload the page or reconnect Flipper manually.',
          color: 'negative',
          reloadBtn: true
        })
        this.$emit('log', {
          level: 'error',
          message: 'Archive: Couldn\'t start rpc session'
        })
        throw new Error('Couldn\'t start rpc session')
      }
      this.flags.rpcActive = true
      this.flags.rpcToggling = false
      this.$emit('setRpcStatus', true)
      this.$emit('log', {
        level: 'info',
        message: 'Archive: RPC started'
      })
    },
    async stopRpc () {
      this.flags.rpcToggling = true
      await this.flipper.commands.stopRpcSession()
      this.flags.rpcActive = false
      this.flags.rpcToggling = false
      this.$emit('setRpcStatus', false)
      this.$emit('log', {
        level: 'info',
        message: 'Archive: RPC stopped'
      })
    },
    async restartRpc (force) {
      if (this.connected && (this.rpcActive || force)) {
        this.flags.restarting = true
        await this.flipper.closeReader()
        await asyncSleep(300)
        await this.flipper.disconnect()
        await asyncSleep(300)
        await this.flipper.connect()
        await this.startRpc()
      }
    },

    async list () {
      let res = await this.flipper.commands.storage.list(this.path)
        .catch(error => this.rpcErrorHandler(error, 'storage.list'))
        .finally(() => {
          this.$emit('log', {
            level: 'debug',
            message: 'Archive: storage.list: ' + this.path
          })
        })
      if (res.length === 0) {
        this.dir = res
        return
      }
      if (res === 'empty response' || res[0] === undefined) {
        return setTimeout(this.list, 300)
      }
      if (this.path === '/') {
        res = res.filter(e => e.name !== 'any')
      }
      this.dir = res.sort((a, b) => { return (b.type || 0) - a.type })
    },

    async read (path, preventDownload) {
      this.flags.blockingOperationPopup = true
      this.file.name = path.slice(path.lastIndexOf('/') + 1)
      const file = this.dir.find(e => e.name === this.file.name && !e.type)
      const total = file.size
      let maxChunks = 0
      const unbind = this.flipper.emitter.on('storageReadRequest/progress', chunks => {
        if (maxChunks < chunks) {
          maxChunks = chunks
          this.file.progress = Math.min(maxChunks * 512, total) / total
        }
      })
      const res = await this.flipper.commands.storage.read(path)
        .catch(error => this.rpcErrorHandler(error, 'storage.read'))
        .finally(() => {
          this.$emit('log', {
            level: 'debug',
            message: 'Archive: storage.read: ' + path
          })
        })
      const s = path.split('/')
      if (!preventDownload) {
        exportFile(s[s.length - 1], res)
      }
      unbind()
      this.flags.blockingOperationPopup = false
      if (preventDownload) {
        return res
      }
    },

    async remove (path, isRecursive) {
      await this.flipper.commands.storage.remove(path, isRecursive)
        .catch(error => this.rpcErrorHandler(error, 'storage.remove'))
        .finally(() => {
          this.$emit('log', {
            level: 'debug',
            message: `Archive: storage.remove: ${path}, recursive: ${isRecursive}`
          })
        })
      this.list()
    },

    async rename (path, oldName, newName) {
      await this.flipper.commands.storage.rename(path, oldName, newName)
        .catch(error => this.rpcErrorHandler(error, 'storage.rename'))
        .finally(() => {
          this.$emit('log', {
            level: 'debug',
            message: `Archive: storage.rename: ${path}, old name: ${oldName}, new name: ${newName}`
          })
        })
      this.list()
    },

    async mkdir (path) {
      await this.flipper.commands.storage.mkdir(path)
        .catch(error => this.rpcErrorHandler(error, 'storage.mkdir'))
        .finally(() => {
          this.$emit('log', {
            level: 'debug',
            message: 'Archive: storage.mkdir: ' + path
          })
        })
      this.list()
    },

    async upload () {
      if (!this.uploadedFiles || this.uploadedFiles.length === 0) {
        return
      }
      this.flags.blockingOperationPopup = true
      for (const file of this.uploadedFiles) {
        this.file.name = file.name
        let dir = this.path

        if (file.webkitRelativePath?.length > 0) {
          const path = file.webkitRelativePath.split('/')
          path.pop()
          while (path.length > 0) {
            dir += '/' + path.shift()
            await this.flipper.commands.storage.stat(dir).catch(async e => {
              if (e === 'ERROR_STORAGE_NOT_EXIST') {
                await this.flipper.commands.storage.mkdir(dir)
              } else {
                throw e
              }
            })
          }
        }

        const unbind = this.flipper.emitter.on('storageWriteRequest/progress', e => {
          this.file.progress = e.progress / e.total
        })
        await this.flipper.commands.storage.write(dir + '/' + file.name, await file.arrayBuffer())
          .catch(error => this.rpcErrorHandler(error, 'storage.write'))
          .finally(() => {
            this.$emit('log', {
              level: 'debug',
              message: 'Archive: storage.write: ' + this.path + '/' + file.name
            })
          })
        unbind()
      }
      this.file.name = ''
      this.list()
      this.flags.blockingOperationPopup = false
    },

    itemClicked (item) {
      if (item.type === 1) {
        if (!this.path.endsWith('/')) {
          this.path += '/'
        }
        this.path += item.name
        this.list()
      } else if (item.name === '..') {
        this.path = this.path.slice(0, this.path.lastIndexOf('/'))
        if (this.path.length === 0) {
          this.path = '/'
        }
        this.list()
      } else {
        this.read(this.path + '/' + item.name)
      }
    },

    async openFileIn (item, path) {
      const res = await this.read(this.path + '/' + item.name, true)
      this.$emit('openFileIn', {
        path,
        file: {
          name: item.name,
          data: res
        }
      })
    },

    itemIconSwitcher (item) {
      if (this.path === '/' && item.name === 'int') {
        return 'archive:internal'
      } else if (this.path === '/' && item.name === 'ext') {
        return 'archive:sdcard'
      } else if (item.type === 1) {
        return 'mdi-folder-outline'
      } else if (item.name.endsWith('.badusb')) {
        return 'archive:badusb'
      } else if (item.name.endsWith('.ibtn')) {
        return 'archive:ibutton'
      } else if (item.name.endsWith('.ir')) {
        return 'archive:infrared'
      } else if (item.name.endsWith('.nfc')) {
        return 'archive:nfc'
      } else if (item.name.endsWith('.rfid')) {
        return 'archive:rfid'
      } else if (item.name.endsWith('.sub')) {
        return 'archive:subghz'
      } else if (item.name.endsWith('.u2f')) {
        return 'archive:u2f'
      } else {
        return 'mdi-file-outline'
      }
    },

    rpcErrorHandler (error, command) {
      error = error.toString()
      this.$emit('showNotif', {
        message: `RPC error in command '${command}': ${error}`,
        color: 'negative'
      })
      this.$emit('log', {
        level: 'error',
        message: `Archive: RPC error in command '${command}': ${error}`
      })
    },

    async start () {
      this.flags.rpcActive = this.rpcActive
      if (!this.rpcActive) {
        setTimeout(() => {
          if (!this.rpcActive) {
            return this.restartRpc(true)
          }
        }, 1000)
        await this.startRpc()
      }
      await this.list()
    }
  },

  async mounted () {
    if (this.connected) {
      await this.start()
    }
  }
})
</script>
