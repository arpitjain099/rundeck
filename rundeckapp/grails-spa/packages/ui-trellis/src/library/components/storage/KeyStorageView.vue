<template>
  <div>
    <div v-if="errorMsg !== ''" class="alert alert-warning">
      <span>{{ errorMsg }}</span>
    </div>

    <div class="card">
      <div class="card-content">
        <div class="row text-info">
          <div
            class="form-group col-sm-12"
            :class="[invalid === true ? 'has-error' : '']"
          >
            <div class="input-group">
              <div v-if="staticRoot" class="input-group-addon bg-3">
                <span>{{ rootPath }}</span>
              </div>
              <input
                v-model="inputPath"
                type="text"
                class="form-control bg-2"
                style="padding-left: 18px"
                placeholder="Enter a path"
                @keyup.enter="loadDirInputPath()"
              />
              <div
                v-if="!isProject && !readOnly"
                class="input-group-btn"
                :class="
                  isDropdownOpen ? 'open input-group-btn' : 'input-group-btn'
                "
              >
                <button
                  type="button"
                  class="btn btn-default dropdown-toggle"
                  :aria-expanded="isDropdownOpen"
                  @click="toggleDropdown"
                >
                  <span>{{ linksTitle }}</span>
                  <span class="caret"></span>
                </button>
                <ul
                  v-if="isDropdownOpen"
                  class="dropdown-menu dropdown-menu-right"
                >
                  <li v-for="link in jumpLinks" :key="link.path">
                    <a
                      href="#"
                      data-testid="load-dir-link"
                      @click="loadDir(link.path)"
                      >{{ link.name }}</a
                    >
                  </li>
                </ul>
              </div>
              <div v-if="isRunner" class="input-group-btn">
                <button
                  type="button"
                  class="btn btn-default"
                  @click="loadDir(path, true)"
                >
                  <span>{{ "Reload" }}</span>
                </button>
              </div>
            </div>
          </div>
        </div>
        <div class="row keySelector">
          <div class="col-sm-12">
            <div class="keySelector-button-group" style="margin-bottom: 1em">
              <button
                type="button"
                class="btn btn-sm btn-default"
                :disabled="upPath === ''"
                @click="loadDir(upPath)"
              >
                <i class="glyphicon glyphicon-folder-open"></i>
                <i class="glyphicon glyphicon-arrow-up"></i>
                <span>{{ showUpPath() }}</span>
              </button>
              <button
                v-if="!readOnly || allowUpload === true"
                class="btn btn-sm btn-cta"
                data-testid="add-key-btn"
                @click="actionUpload"
              >
                <i class="glyphicon glyphicon-plus"></i>
                Add or Upload a Key
              </button>

              <button
                v-if="
                  allowUpload === true && isSelectedKey === true && !readOnly
                "
                class="btn btn-sm btn-warning"
                data-testid="overwrite-key-btn"
                @click="actionUploadModify"
              >
                <i class="glyphicon glyphicon-pencil"></i>
                Overwrite Key
              </button>
              <button
                v-if="
                  selectedKey && selectedKey.path && isSelectedKey && !readOnly
                "
                class="btn btn-sm btn-danger"
                data-testid="delete-key-btn"
                @click="deleteKey"
              >
                <i class="glyphicon glyphicon-trash"></i>
                {{ "Delete" }}
              </button>
            </div>

            <div
              v-if="loading"
              class="loading-area text-info"
              style="
                width: 100%;
                height: 200px;
                padding: 50px;
                background-color: #eee;
              "
            >
              <i class="glyphicon glyphicon-time">{{ "Loading..." }}</i>
              <div v-if="isRunner">
                <span v-if="countDownLimit > 0">
                  Reload from the remote Runner in {{ countDownLimit }} seconds
                </span>
                <span v-if="countDownLimit === 0"> Reload </span>
              </div>
            </div>
            <table v-if="!loading" class="table table-hover table-condensed">
              <tbody>
                <tr>
                  <td colspan="2" class="text-strong">
                    <span v-if="files.length < 1">No keys</span>
                    <span v-if="files.length > 0">
                      <span>{{ files.length }}</span>
                      keys
                    </span>
                  </td>
                </tr>
              </tbody>
              <tbody>
                <tr
                  v-for="key in files"
                  :key="key.name"
                  :class="[
                    selectedKey && key.path === selectedKey.path
                      ? selectedClass
                      : '',
                    'action',
                  ]"
                  @click="selectKey(key)"
                >
                  <td>
                    <i
                      :class="[
                        key.path === selectedKey.path
                          ? 'glyphicon glyphicon-ok'
                          : 'glyphicon glyphicon-unchecked',
                      ]"
                    ></i>

                    <span
                      v-if="isPrivateKey(key)"
                      title="This path contains a private key that can be used for remote node execution."
                    >
                      <i class="glyphicon glyphicon-lock"></i>
                    </span>
                    <span v-if="isPublicKey(key)">
                      <i class="glyphicon glyphicon-eye-open"></i>
                    </span>
                    <span
                      v-if="isPassword(key)"
                      title="This path contains a password that can be used for remote node execution."
                    >
                      <i class="glyphicon glyphicon-lock"></i>
                    </span>
                    <span data-testid="created-key">{{ key.name }}</span
                    ><span v-if="key.expired">{{ "[CACHE EXPIRED]" }}</span>
                  </td>
                  <td class="text-strong">
                    <span class="pull-right">
                      <span
                        v-if="isPrivateKey(key)"
                        title="This path contains a private key that can be used for remote node execution."
                      >
                        Private Key
                      </span>
                      <span v-if="isPublicKey(key)"> Public Key </span>
                      <span
                        v-if="isPassword(key)"
                        title="This path contains a password that can be used for remote node execution."
                      >
                        Password
                      </span>
                    </span>
                  </td>
                </tr>
              </tbody>

              <tbody v-if="notFound() === true">
                <tr>
                  <td colspan="2">
                    <span class="text-strong"
                      >Nothing found at this path.
                    </span>
                  </td>
                </tr>
              </tbody>
              <tbody>
                <tr v-for="directory in directories" :key="directory.name">
                  <td
                    data-testid="keyDirectoryButton"
                    class="action"
                    colspan="2"
                    @click="loadDir(directory.path)"
                  >
                    <i class="glyphicon glyphicon-arrow-down"></i>
                    <i class="glyphicon glyphicon-folder-close"></i>
                    <span>{{ dirNameString(directory.path) }}</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
        <modal
          v-if="isConfirmingDeletion === true"
          id="storagedeletekey"
          ref="modalDelete"
          v-model="isConfirmingDeletion"
          title="Delete Selected Key"
          auto-focus
          append-to-body
          :footer="false"
        >
          <div class="modal-body">
            <p>{{ "Really delete the selected key at this path?" }}</p>

            <p>
              <strong class="text-info"> {{ selectedKey.path }}</strong>
            </p>
          </div>
          <div class="modal-footer">
            <button
              type="button"
              data-testid="confirm-delete-btn"
              class="btn btn-sm btn-danger obs-storagedelete-select"
              @click="confirmDeleteKey"
            >
              {{ "Delete" }}
            </button>
            <button
              type="button"
              class="pull-right btn btn-sm btn-default"
              @click="cancelDeleteKey"
            >
              {{ "Cancel" }}
            </button>
          </div>
        </modal>
        <div v-if="isSelectedKey && selectedKey.type === 'file'" class="row">
          <div class="col-sm-12">
            <div class="well">
              <div>
                Storage path:
                <code class="text-success">{{ selectedKey.path }}</code>
                <a href="#" data-bind="attr: { href: selectedPathUrl() }">
                  <i class="glyphicon glyphicon-link"></i>
                </a>
              </div>
              <div v-if="createdTime !== ''">
                <div>
                  Created:
                  <span class="timeabs text-strong">
                    {{ formatKeyStorageCreatedTime }}
                  </span>

                  <span v-if="createdUsername() !== ''">
                    by:
                    <span class="text-strong">{{ createdUsername() }}</span>
                  </span>
                </div>
              </div>
              <div v-if="wasModified() !== ''">
                <div>
                  Modified:
                  <span class="timeago text-strong">
                    {{ formatHumanizedModifiedTimeAgoDuration }} ago
                  </span>

                  <span v-if="modifiedUsername() !== ''">
                    by:
                    <span class="text-strong">{{ modifiedUsername() }}</span>
                  </span>
                </div>
              </div>
              <div
                v-if="selectedKey && isPublicKey(selectedKey)"
                class="pull-right"
              >
                <span>
                  <a :href="downloadUrl()">
                    <i class="glyphicon glyphicon-download"></i>
                    {{ "Download" }}</a
                  >
                </span>
              </div>
              <div v-if="selectedKey && selectedKey.expired" class="pull-right">
                <span>
                  <a
                    :href="'javascript:void(0)'"
                    @click="loadKeys(selectedKey)"
                  >
                    <i class="glyphicon glyphicon-download"></i>
                    {{ "Refresh Expired Key" }}</a
                  >
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="card-footer">
        <hr />
        <span class="text-info">
          {{
            $t(
              "Key Storage provides a global directory-like structure to save Public and Private Keys and Passwords, for use with Node Execution authentication.",
            )
          }}
        </span>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { listProjects } from "../../services/projects";
import {
  storageKeyDelete,
  storageKeyGetMetadata,
  StorageKeyListResponse,
} from "../../services/storage";
import moment from "moment";
import { getRundeckContext } from "../../index";
import { defineComponent, PropType } from "vue";
import KeyType from "../../types/KeyType";
import InputType from "../../types/InputType";
import {
  formatHumanizedDuration,
  formatKeyStorageDate,
} from "../../utilities/DateTimeFormatters";

export default defineComponent({
  name: "KeyStorageView",
  props: {
    readOnly: Boolean,
    allowUpload: Boolean,
    modelValue: String,
    storageFilter: String,
    rootPath: String,
    createdKey: {},
    runnerId: String,
    getKeyMetadata: {
      type: Function as PropType<
        (path: string) => Promise<StorageKeyListResponse>
      >,
      default: storageKeyGetMetadata,
      required: true,
    },
  },
  emits: ["update:modelValue", "openEditor"],
  data() {
    return {
      errorMsg: "",
      isDropdownOpen: false,
      modalOpen: false,
      invalid: false,
      project: "",
      staticRoot: true,
      inputPath: "",
      upPath: "",
      path: "",
      isConfirmingDeletion: false,
      modalEdit: false,
      upload: {} as any,
      selectedKey: {} as any,
      isSelectedKey: false,
      files: [] as any,
      selectedClass: "success",
      directories: [] as any,
      uploadErrors: {} as any,
      selectedIsDownloadable: true,
      loading: true,
      linksTitle: "",
      projectList: [],
      jumpLinks: [] as Array<{ name: string | undefined; path: string }>,
      countDownLimit: 0,
      countDownInterval: 0,
    };
  },
  computed: {
    formatKeyStorageCreatedTime() {
      return formatKeyStorageDate(this.createdTime);
    },
    formatHumanizedModifiedTimeAgoDuration() {
      return formatHumanizedDuration(this.modifiedTimeAgoText);
    },
    isProject(): boolean {
      return this.rootPath?.startsWith("keys/project") || false;
    },
    createdTime() {
      let value = "";
      if (
        this.selectedKey != null &&
        this.selectedKey.meta != null &&
        this.selectedKey.meta["Rundeck-content-creation-time"] != null
      ) {
        value = this.selectedKey.meta["Rundeck-content-creation-time"];
      }
      return value;
    },
    modifiedTimeAgoText() {
      let value = 0;
      if (
        this.selectedKey != null &&
        this.selectedKey.meta != null &&
        this.selectedKey.meta["Rundeck-content-modify-time"] != null
      ) {
        const time = this.selectedKey.meta["Rundeck-content-modify-time"];
        value = this.duration(time);
      }
      return value;
    },
    isRunner(): boolean {
      return this.rootPath ? this.rootPath.startsWith("runner:") : false;
    },
  },
  watch: {
    createdKey: function (newValue) {
      if (newValue !== null) {
        this.selectKey(newValue);
      }
    },
    rootPath: function (newValue: string) {
      // Reset current path when rootPath changed.

      this.path = "";
      this.inputPath = "";
      this.selectedKey = {};
      this.loadKeys();
    },
  },
  mounted() {
    this.loadKeys();
    this.loadProjectNames();
  },
  unmounted() {
    if (this.countDownInterval > 0) {
      clearInterval(this.countDownInterval);
      this.countDownInterval = 0;
    }
  },
  methods: {
    downloadUrl(): string {
      const downloadBaseUrl = "storage/download/keys";
      const rundeckContext = getRundeckContext();

      if (!this.selectedKey || !this.selectedKey.path) return "#";

      return `${rundeckContext.rdBase}/${downloadBaseUrl}?resourcePath=${encodeURIComponent(this.selectedKey.path)}`;
    },
    async loadProjectNames() {
      try {
        const response = await listProjects();

        this.linksTitle = "Projects";
        this.jumpLinks = response.map((v: any) => {
          return { name: v.name, path: "keys/project/" + v.name };
        });
      } catch (error) {
        console.error("Error loading project names:", error);
      }
    },
    toggleDropdown() {
      this.isDropdownOpen = !this.isDropdownOpen;
    },
    deleteKey() {
      this.isConfirmingDeletion = true;
    },
    async confirmDeleteKey() {
      this.isConfirmingDeletion = false;

      try {
        const resp = await storageKeyDelete(this.selectedKey.path.slice(5));
        if (!resp) {
          this.errorMsg = "Not found";
          return;
        }
      } catch (e) {
        this.errorMsg = e.message;
        return;
      }
      this.selectedKey = {};
      this.isSelectedKey = false;

      this.loadKeys();
    },
    cancelDeleteKey() {
      this.isConfirmingDeletion = false;
    },
    calcBrowsePath(path: string) {
      let browse = path;

      if (this.isRunner) {
        return path;
      }

      if (this.rootPath != "keys/" && this.rootPath != "keys") {
        browse = this.rootPath + "/" + path;
        browse = browse.substring(5);
      }
      return browse;
    },
    countDown(selectedKey?: any) {
      if (this.countDownLimit > 0) return;
      this.countDownLimit = 5;
      // @ts-ignore
      this.countDownInterval = setInterval(() => {
        this.countDownLimit--;

        if (this.countDownLimit <= 0) {
          // @ts-ignore
          clearInterval(this.countDownInterval);
          this.countDownInterval = 0;

          const delayExec = setTimeout(() => {
            this.loadKeys(selectedKey);
            clearTimeout(delayExec);
          }, 600); // Delay 600ms to execute to give better user experience.
        }
      }, 1000);
    },
    loadKeys(selectedKey?: any, forceRefresh?: boolean) {
      if (selectedKey) {
        this.selectedKey = selectedKey;
      }
      this.loading = true;

      const getPath = this.calcBrowsePath(this.path);

      const requestOptions = {
        queryParameters: forceRefresh ? { refresh: "true" } : {},
      };
      this.getKeyMetadata(getPath, requestOptions)
        .then((result: any) => {
          this.directories = [];
          this.files = [];

          if (result.cacheStatus === "LOADING") {
            this.loading = true;
            this.countDown(selectedKey);
            return;
          }

          if (result.resources != null) {
            result.resources.forEach((resource: any) => {
              if (resource.type === "directory") {
                this.directories.push(resource);
                if (this.directories.length > 1) {
                  this.directories.sort((obj1: any, obj2: any) => {
                    if (obj1.path > obj2.path) {
                      return 1;
                    }

                    if (obj1.path < obj2.path) {
                      return -1;
                    }
                    return 0;
                  });
                }
              }

              if (resource.type === "file") {
                if (this.storageFilter != null && this.storageFilter !== "") {
                  if (this.allowedResource(resource.meta)) {
                    this.files.push(resource);
                  }
                } else {
                  this.files.push(resource);
                }

                this.files.sort((obj1: any, obj2: any) => {
                  if (obj1.path > obj2.path) {
                    return 1;
                  }

                  if (obj1.path < obj2.path) {
                    return -1;
                  }
                  return 0;
                });
              }
            });
          }

          this.loading = false;
        })
        .catch((err: Error) => {
          try {
            const jsonError = JSON.parse(err.message);

            if (
              jsonError.message ===
              "Error: Key browsing is not allowed in CCP mode."
            ) {
              // Generate sample key path string
              const sampleKeyPath = this.rootPath.endsWith("/")
                ? `${this.rootPath}${this.path}`
                : `${this.rootPath}/${this.path}`;

              // Remove the Error prefix
              const softenedMessage = jsonError.message.startsWith("Error: ")
                ? jsonError.message.substring("Error: ".length)
                : jsonError.message;

              this.errorMsg = `${softenedMessage} If you need to select a key, please type the path of the key with the prefix "${sampleKeyPath}" into the form field. E.g. ${sampleKeyPath}/path/to/the/key`;
            } else {
              this.errorMsg = jsonError.message;
            }
          } catch (parseError) {
            this.errorMsg = err.message;
          }

          this.loading = false;
        });
    },
    allowedResource(meta: any) {
      const filterArray = this.storageFilter?.split("=") || ["", ""];
      const key = filterArray[0];
      const value = filterArray[1];
      if (key == "Rundeck-key-type") {
        if (
          value === meta["rundeckKeyType"] ||
          value === meta["Rundeck-key-type"]
        ) {
          return true;
        }
      } else {
        if (key == "Rundeck-data-type") {
          if (value === meta["Rundeck-data-type"]) {
            return true;
          }
        }
      }
      return false;
    },
    duration(start: any) {
      return moment().diff(moment(start));
    },
    modifiedUsername() {
      const value = "";
      if (
        this.selectedKey != null &&
        this.selectedKey.meta != null &&
        this.selectedKey.meta["Rundeck-auth-modified-username"]
      ) {
        return this.selectedKey.meta["Rundeck-auth-modified-username"];
      }
      return value;
    },
    wasModified() {
      if (
        this.selectedKey != null &&
        this.selectedKey.meta != null &&
        this.selectedKey.meta["Rundeck-content-creation-time"] != null &&
        this.selectedKey.meta["Rundeck-content-modify-time"] != null
      ) {
        return (
          this.selectedKey.meta["Rundeck-content-creation-time"] !=
          this.selectedKey.meta["Rundeck-content-modify-time"]
        );
      }
      return false;
    },
    createdUsername() {
      const value = "";
      if (
        this.selectedKey != null &&
        this.selectedKey.meta != null &&
        this.selectedKey.meta["Rundeck-auth-created-username"] != null
      ) {
        return this.selectedKey.meta["Rundeck-auth-created-username"];
      }
      return value;
    },
    dirNameString(path: any) {
      if (path.lastIndexOf("/") >= 0) {
        return path.substring(path.lastIndexOf("/") + 1);
      } else {
        return path;
      }
    },
    isPrivateKey(key: any) {
      let keyType = key.meta["rundeckKeyType"];
      if (keyType == null) {
        keyType = key.meta["Rundeck-key-type"];
      }

      return key && key.meta && keyType && keyType === "private";
    },
    isPublicKey(key: any) {
      let keyType = key.meta["rundeckKeyType"];
      if (keyType == null) {
        keyType = key.meta["Rundeck-key-type"];
      }
      return key && key.meta && keyType && keyType === "public";
    },
    isPassword(key: any) {
      return key && key.meta && key.meta["Rundeck-data-type"] === "password";
    },
    notFound() {
      return false;
    },
    selectKey(key: any) {
      if (this.selectedKey.path === key.path && this.isSelectedKey == false) {
        this.isSelectedKey = true;
      } else if (this.selectedKey.path === key.path) {
        this.selectedKey = {};
        this.isSelectedKey = false;
      } else {
        this.selectedKey = key;
        this.isSelectedKey = true;
      }

      this.$emit(
        "update:modelValue",
        this.selectedKey ? this.selectedKey.path : "",
      );
    },
    parentDirString(path: any) {
      if (null != path && path.lastIndexOf("/") >= 0) {
        return path.substring(0, path.lastIndexOf("/"));
      } else {
        return "";
      }
    },
    actionUploadModify() {
      // Default to KeyType.Password
      let type = KeyType.Password;
      if (this.isPrivateKey(this.selectedKey)) {
        type = KeyType.Private;
      }
      if (this.isPublicKey(this.selectedKey)) {
        type = KeyType.Public;
      }

      const inputPath = this.relativePath(
        this.parentDirString(this.selectedKey.path),
      );
      const inputType = InputType.Text;

      const upload = {
        modifyMode: true,
        keyType: type,
        inputPath: inputPath,
        inputType: inputType,
        fileName: this.selectedKey.name,
        file: "",
        fileContent: "",
        textArea: "",
        password: "",
        status: "update",
        errorMsg: null as any,
      };
      this.$emit("openEditor", upload);
    },
    clean() {
      this.directories = [];
      this.files = [];
      this.selectedKey = {};
      this.isSelectedKey = false;
      this.inputPath = "";
      this.invalid = false;
      this.errorMsg = "";
      this.uploadErrors = {} as any;
    },
    loadDirInputPath() {
      if (this.isRunner) {
        this.loadDir(this.inputPath);
      } else {
        this.loadDir(this.rootPath + "/" + this.inputPath);
      }
    },
    defaultSelectKey(path: any) {
      storageKeyGetMetadata(this.calcBrowsePath(path)).then((result: any) => {
        if (result.resources != null) {
          result.resources.forEach((resource: any) => {
            if (resource.type === "file") {
              if (resource.path === path) {
                this.selectedKey = resource;
                this.isSelectedKey = true;
              }
            }
          });
        }
      });
    },
    loadUpPath() {
      let upPath = "";
      if (this.isRunner) {
        // upPath is the path without trailing part
        if (!this.path) this.upPath = "";
        else {
          const lastIndexOfSlash = this.path.lastIndexOf("/");
          if (lastIndexOfSlash >= 0)
            this.upPath = this.path.substring(0, lastIndexOfSlash);
          else this.upPath = "";
        }
      } else {
        if (
          this.path != "" &&
          this.path != this.rootPath &&
          this.path != this.rootPath + "/"
        ) {
          if (this.path.indexOf("/") >= 0) {
            upPath = this.parentDirString(this.path);
          } else {
            upPath = this.rootPath || "";
          }

          if (upPath != this.rootPath) {
            this.upPath = this.absolutePath(upPath);
          } else {
            this.upPath = this.rootPath;
          }
        } else {
          this.upPath = upPath;
        }
      }
    },
    absolutePath(relpath: any) {
      if (this.isRunner) {
        return "/" + relpath;
      }

      if (this.staticRoot === false) {
        return relpath;
      }
      return this.rootPath + "/" + relpath;
    },
    loadDir(selectedPath: any, forceRefresh?: boolean) {
      this.isDropdownOpen = false;
      this.clean();
      let path = "";

      if (this.isRunner) {
        path = selectedPath;
      } else {
        if (selectedPath != null) {
          path = this.relativePath(selectedPath);
        }
      }

      this.path = path;
      this.inputPath = path;

      this.loadUpPath();

      this.loadKeys(path, forceRefresh);
    },
    showUpPath() {
      if (this.isRunner) {
        return this.upPath;
      }

      if (this.upPath != this.rootPath) {
        return this.relativePath(this.upPath);
      } else {
        return this.upPath;
      }
    },
    relativePath(path: any) {
      const root = this.rootPath;
      const statroot = this.staticRoot;

      if (!statroot) {
        return path;
      }
      let newpath = "";
      if (path != null && root != null) {
        path = this.cleanPath(path);

        newpath = this.cleanPath(path.substring(root.length));
      }
      return newpath;
    },

    cleanPath(path: any) {
      if (path != null) {
        while (path.indexOf("/") == 0) {
          path = path.substring(1);
        }
      } else {
        return "";
      }
      return path;
    },
    actionUpload() {
      const upload = {
        modifyMode: false,
        keyType: KeyType.Private,
        inputPath: this.inputPath,
        inputType: InputType.Text,
        fileName: null,
        file: "",
        fileContent: "",
        textArea: "",
        password: "",
        status: "new",
        errorMsg: null as any,
      };

      this.$emit("openEditor", upload);
    },
  },
});
</script>

<style>
.keySelector span {
  content: " ";
  margin: 0 2px;
}

.keySelector i {
  content: " ";
  margin: 0 1px;
}

.keySelector-button-group button {
  content: " ";
  margin: 0 2px;
}

.label-key {
  vertical-align: middle;
}

.input-group-addon {
  color: var(--font-color);
}
</style>
