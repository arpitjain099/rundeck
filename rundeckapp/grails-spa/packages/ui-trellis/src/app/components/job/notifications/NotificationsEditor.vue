<template>
  <div>
    <div class="help-block">
      Notifications can be triggered by different events during the Job
      Execution.
    </div>
    <div>
      <undo-redo :event-bus="eventBus" />

      <div v-if="notifications.length < 1">
        <p class="text-muted">
          No Notifications are defined. Click an event below to add a
          Notification for that Trigger.
        </p>
      </div>
      <div class="main-section">
        <div v-for="trigger in notifyTypes">
          <div :id="'job-notifications-' + trigger" class="list-group">
            <div
              class="list-group-item flex-container flex-align-items-baseline flex-justify-space-between"
            >
              <span
                class="flex-item"
                :class="{
                  'text-secondary': !hasNotificationsForTrigger(trigger),
                }"
              >
                <i class="fas" :class="triggerIcons[trigger]"></i>
                {{ $t("notification.event." + trigger) }}
              </span>
              <btn
                type="default"
                class="flex-item"
                size="sm"
                @click="addNotification(trigger)"
              >
                <i class="fas fa-plus"></i>
                Add Notification
              </btn>
            </div>
            <div
              v-if="trigger === 'onavgduration'"
              class="list-group-item form-inline"
            >
              <div class="form-group">
                <div class="col-sm-12">
                  <div class="input-group">
                    <label
                      class="input-group-addon"
                      for="schedJobNotifyAvgDurationThreshold"
                    >
                      {{
                        $t(
                          "scheduledExecution.property.notifyAvgDurationThreshold.label",
                        )
                      }}
                    </label>
                    <input
                      id="schedJobNotifyAvgDurationThreshold"
                      v-model="notifyAvgDurationThreshold"
                      type="text"
                      name="notifyAvgDurationThreshold"
                      class="form-control"
                      :placeholder="$t('jobAverageDurationPlaceholder')"
                      size="40"
                    />
                    <span
                      id="jobAvgInfoBtn"
                      class="input-group-addon btn btn-info btn-md"
                    >
                      <i class="glyphicon glyphicon-question-sign"></i>
                    </span>
                  </div>

                  <popover
                    :title="
                      $t(
                        'scheduledExecution.property.notifyAvgDurationThreshold.label',
                      )
                    "
                    target="#jobAvgInfoBtn"
                  >
                    <template #popover>
                      <VMarkdownView
                        class="markdown-body"
                        :content="
                          $t(
                            'scheduledExecution.property.notifyAvgDurationThreshold.description',
                          )
                        "
                        mode=""
                      />
                    </template>
                  </popover>
                </div>
              </div>
            </div>
            <template v-if="getNotificationsForTrigger(trigger)">
              <div
                v-for="(notif, i) in getNotificationsForTrigger(trigger)"
                class="list-group-item flex-container flex-justify-start"
              >
                <div style="margin-right: 10px">
                  <dropdown ref="dropdown" append-to-body>
                    <btn
                      type="simple"
                      class="btn-hover btn-secondary dropdown-toggle"
                    >
                      <span class="caret"></span>
                    </btn>
                    <template #dropdown>
                      <li @click="doCopyNotification(notif)">
                        <a role="button"> {{ $t("duplicate") }}... </a>
                      </li>
                      <li role="separator" class="divider"></li>
                      <li @click="doDeleteNotification(notif)">
                        <a role="button">
                          {{ $t("Delete") }}
                        </a>
                      </li>
                    </template>
                  </dropdown>
                </div>
                <div class="flex-item flex-grow-1">
                  <plugin-config
                    :key="'g_' + i + '/' + notif.type + ':config'"
                    service-name="Notification"
                    :provider="notif.type"
                    :config="notif.config"
                    mode="show"
                    :show-title="true"
                    :show-description="true"
                    scope="Instance"
                    default-scope="Instance"
                  />
                </div>

                <btn type="default" size="sm" @click="doEditNotification(notif)"
                  >Edit</btn
                >
              </div>
            </template>
          </div>
        </div>
      </div>
    </div>

    <modal
      id="job-notifications-edit-modal"
      v-model="editModal"
      :title="$t(editIndex < 0 ? 'notification.create' : 'notification.edit')"
      size="lg"
      append-to-body
    >
      <div>
        <div class="form-group">
          <label class="col-sm-2 control-label"> Trigger </label>
          <div class="col-sm-10 form-control-static">
            <dropdown id="notification-edit-trigger-dropdown" ref="dropdown">
              <btn
                type="simple"
                class="btn-hover btn-secondary dropdown-toggle"
              >
                <span class="caret"></span>
                &nbsp;
                <span v-if="editNotificationTrigger" class="text-strong">
                  <i
                    class="fas"
                    :class="triggerIcons[editNotificationTrigger]"
                  ></i>
                  {{ $t("notification.event." + editNotificationTrigger) }}
                </span>
                <span v-else> Select a Trigger </span>
              </btn>
              <template #dropdown>
                <li
                  v-for="trigger in notifyTypes"
                  :data-trigger="trigger"
                  @click="setEditNotificationTrigger(trigger)"
                >
                  <a role="button">
                    <i class="fas" :class="triggerIcons[trigger]"></i>
                    {{ $t("notification.event." + trigger) }}
                  </a>
                </li>
              </template>
            </dropdown>
          </div>
        </div>

        <div v-if="editNotificationTrigger">
          <div class="form-group">
            <label class="col-sm-2 control-label"> Notification Type </label>
            <div class="col-sm-10 form-control-static">
              <dropdown id="notification-edit-type-dropdown" ref="dropdown">
                <btn
                  type="simple"
                  class="btn-simple btn-hover btn-secondary dropdown-toggle"
                >
                  <span class="caret"></span>
                  &nbsp;
                  <span
                    v-if="
                      editNotification.type &&
                      getProviderFor(editNotification.type)
                    "
                  >
                    <plugin-info
                      :detail="getProviderFor(editNotification.type)"
                      :show-description="false"
                      :show-extended="false"
                      description-css="help-block"
                    >
                    </plugin-info>
                  </span>
                  <span v-else> Select a Notification </span>
                </btn>
                <template #dropdown>
                  <li v-for="plugin in sortedProviders" :key="plugin.name">
                    <a
                      role="button"
                      :data-plugin-type="plugin.name"
                      @click="setEditNotificationType(plugin.name)"
                    >
                      <plugin-info
                        :detail="plugin"
                        :show-description="true"
                        :show-extended="false"
                        description-css="help-block"
                      >
                      </plugin-info>
                    </a>
                  </li>
                </template>
              </dropdown>
              <div
                v-if="
                  editNotification.type && getProviderFor(editNotification.type)
                "
              >
                <plugin-info
                  :detail="getProviderFor(editNotification.type)"
                  :show-description="true"
                  :show-extended="false"
                  :show-title="false"
                  :show-icon="false"
                  description-css="help-block"
                >
                </plugin-info>
              </div>
            </div>
          </div>
        </div>

        <plugin-config
          id="notification-edit-config"
          :key="'edit_config' + editIndex + '/' + editNotification.type"
          v-model="editNotification"
          :mode="editIndex === -1 ? 'create' : 'edit'"
          :service-name="'Notification'"
          :show-title="false"
          :show-description="false"
          :context-autocomplete="true"
          :validation="editValidation"
          scope="Instance"
          default-scope="Instance"
          :autocomplete-callback="autocompleteCallback"
        ></plugin-config>
      </div>

      <template #footer>
        <div>
          <btn
            id="job-notifications-edit-modal-btn-cancel"
            @click="cancelEditNotification"
            >{{ $t("Cancel") }}</btn
          >
          &nbsp;
          <btn
            id="job-notifications-edit-modal-btn-save"
            type="primary"
            :disabled="!editNotificationTrigger || !editNotification.type"
            @click="saveNotification"
            >{{ $t("Save") }}</btn
          >
          <span
            v-if="editValidation && !editValidation.valid"
            class="text-warning"
          >
            Please correct the highlighted errors.
          </span>
          <span v-if="editError" class="text-warning">
            {{ editError }}
          </span>
        </div>
      </template>
    </modal>
  </div>
</template>
<script lang="ts">
import { defineComponent } from "vue";

import PluginInfo from "../../../../library/components/plugins/PluginInfo.vue";
import PluginConfig from "../../../../library/components/plugins/pluginConfig.vue";
import pluginService from "../../../../library/modules/pluginService";
import ExtendedDescription from "../../../../library/components/utils/ExtendedDescription.vue";
import UndoRedo from "../../util/UndoRedo.vue";
import { VMarkdownView } from "vue3-markdown";
import mitt from "mitt";
const localEB = mitt();
export default defineComponent({
  name: "NotificationsEditor",
  components: {
    PluginInfo,
    PluginConfig,
    ExtendedDescription,
    UndoRedo,
    VMarkdownView,
  },
  props: ["notificationData"],
  emits: ["changed"],
  data() {
    return {
      notifyAvgDurationThreshold: null,
      pluginProviders: [],
      pluginLabels: {},
      notifyTypes: [
        "onstart",
        "onsuccess",
        "onfailure",
        "onretryablefailure",
        "onavgduration",
      ],
      triggerIcons: {
        onsuccess: "fa-check-square text-success",
        onfailure: "fa-times-circle text-danger",
        onstart: "fa-play text-info",
        onavgduration: "fa-clock text-secondary",
        onretryablefailure: "fa-redo text-warning",
      },
      notifications: [],
      editNotificationTrigger: null,
      editNotification: {},
      editValidation: null,
      editError: null,
      editIndex: -1,
      editModal: false,
      autocompleteCallback: {
        type: Function,
      },
      eventBus: localEB,
    };
  },
  computed: {
    sortedProviders() {
      const prov = [this.getProviderFor("email"), this.getProviderFor("url")];
      const other = this.pluginProviders.filter(
        (x) => x.name !== "email" && x.name !== "url",
      );
      return prov.concat(other);
    },
    groupedNotifications() {
      const grouped = {};
      this.notifyTypes.forEach((trigger) => {
        const found = this.notifications.filter((s) => s.trigger === trigger);
        if (found && found.length > 0) {
          grouped[trigger] = found;
        }
      });
      return grouped;
    },
  },
  watch: {
    notifications: {
      handler() {
        this.$emit("changed", this.notifications);
      },
      deep: true,
    },
  },
  async mounted() {
    this.autocompleteCallback = window.notificationAutocomplete;

    this.notifications = [].concat(this.notificationData.notifications || []);
    this.notifyAvgDurationThreshold =
      this.notificationData.notifyAvgDurationThreshold;
    this.eventBus.on("undo", this.doUndo);
    this.eventBus.on("redo", this.doRedo);
    pluginService.getPluginProvidersForService("Notification").then((data) => {
      if (data.service) {
        this.pluginProviders = data.descriptions;
        this.pluginLabels = data.labels;
      }
    });
  },
  beforeUnmount() {
    this.eventBus.off("undo");
    this.eventBus.off("redo");
  },
  methods: {
    async addNotification(trigger) {
      this.editNotificationTrigger = trigger;
      this.editNotification = { type: null, config: {} };
      this.editIndex = -1;
      this.editValidation = null;
      this.editError = null;
      this.editModal = true;
    },
    async doEditNotification(notif) {
      this.editIndex = this.notifications.findIndex((n) => n === notif);
      if (this.editIndex < 0) {
        return;
      }
      this.editNotificationTrigger = this.notifications[this.editIndex].trigger;
      this.editNotification = this.notifications[this.editIndex];
      this.editValidation = null;
      this.editError = null;
      this.editModal = true;
    },
    async operationDelete(index) {
      return this.notifications.splice(index, 1);
    },
    async operationCreate(value) {
      this.notifications.push(value);
    },
    async operationInsert(index, value) {
      this.notifications.splice(index, 0, value);
    },
    async operationModify(index, value) {
      this.notifications.splice(index, 1, value);
    },
    async doDelete(index) {
      const oldval = this.doClone(this.notifications[index]);
      await this.operationDelete(index);
      this.eventBus.emit("change", {
        index: index,
        value: oldval,
        operation: "delete",
        undo: "insert",
      });
    },
    async doCreate(value) {
      await this.operationCreate(value);
      const value1 = this.doClone(value);
      const index = this.notifications.length - 1;
      this.eventBus.emit("change", {
        index: index,
        value: value1,
        operation: "insert",
        undo: "delete",
      });
    },
    async doModify(index, value) {
      const oldval = this.doClone(this.notifications[index]);
      await this.operationModify(index, value);
      const clone = this.doClone(value);
      this.eventBus.emit("change", {
        index: index,
        value: clone,
        orig: oldval,
        operation: "modify",
        undo: "modify",
      });
    },
    async doDeleteNotification(notif) {
      const ndx = this.notifications.findIndex((n) => n === notif);
      if (ndx >= 0) {
        return this.doDelete(ndx);
      }
    },
    async doCopyNotification(notif) {
      this.editNotificationTrigger = notif.trigger;
      this.editNotification = {
        type: notif.type,
        config: Object.assign({}, notif.config),
      };
      this.editIndex = -1;
      this.editValidation = null;
      this.editError = null;
      this.editModal = true;
    },
    async cancelEditNotification() {
      this.editModal = false;
      this.editIndex = -1;
      this.editNotification = { type: null, config: {} };
      this.editNotificationTrigger = null;
      this.editValidation = null;
      this.editError = null;
    },
    async setEditNotificationTrigger(name) {
      this.editError = null;
      this.editNotificationTrigger = name;
    },
    async setEditNotificationType(name) {
      this.editValidation = null;
      this.editError = null;
      this.editNotification.type = name;
    },
    doClone(notif) {
      return {
        type: notif.type,
        trigger: notif.trigger,
        config: Object.assign({}, notif.config),
      };
    },
    async saveNotification() {
      if (!this.editNotificationTrigger) {
        this.editError = "Choose a Trigger";
        return;
      }
      if (!this.editNotification.type) {
        this.editError = "Choose a Notification Type";
        return;
      }

      if (this.editNotification.config.recipients != null) {
        const recipientsStr = this.editNotification.config.recipients;
        this.editNotification.config.recipients = recipientsStr
          .split(",")
          .map((mail) => mail.trim())
          .join(",");
      }

      const validation = await pluginService.validatePluginConfig(
        "Notification",
        this.editNotification.type,
        this.editNotification.config,
        "Project",
      );
      if (!validation.valid) {
        this.editValidation = validation;
        return;
      }
      this.editModal = false;
      this.editNotification.trigger = this.editNotificationTrigger;
      if (this.editIndex < 0) {
        await this.doCreate(this.editNotification);
      } else {
        await this.doModify(this.editIndex, this.editNotification);
      }
      this.editIndex = -1;
      this.editNotification = {};
    },
    getProviderFor(name) {
      return this.pluginProviders.find((p) => p.name === name);
    },
    getNotificationsForTrigger(trigger) {
      return this.notifications.filter((s) => s.trigger === trigger);
    },
    hasNotificationsForTrigger(trigger) {
      return this.notifications.findIndex((s) => s.trigger === trigger) >= 0;
    },
    doUndo(change) {
      console.log("do undo");
      this.perform(change.undo, {
        index: change.index,
        value: change.orig || change.value,
      });
    },
    doRedo(change) {
      console.log("do redo");
      this.perform(change.operation, change);
    },
    async perform(operation, change) {
      if (operation === "create") {
        return this.operationCreate(change.value);
      }
      if (operation === "insert") {
        return this.operationInsert(change.index, change.value);
      } else if (operation === "modify") {
        return this.operationModify(change.index, change.value);
      } else if (operation === "delete") {
        return this.operationDelete(change.index);
      }
    },
  },
});
</script>
<style lang="scss" scoped>
.list-group-item {
  &.list-group-item-secondary {
    border-width: 0;
  }
}
.list-placeholder {
  margin-bottom: 20px;
}
.main-section {
  margin-top: 20px;
}
</style>
