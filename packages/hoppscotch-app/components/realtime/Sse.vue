<template>
  <AppPaneLayout>
    <template #primary>
      <div
        class="sticky top-0 z-10 flex flex-shrink-0 p-4 overflow-x-auto space-x-2 bg-primary hide-scrollbar"
      >
        <div class="inline-flex flex-1 space-x-2">
          <div class="flex flex-1">
            <input
              id="server"
              v-model="server"
              type="url"
              autocomplete="off"
              :class="{ error: !serverValid }"
              class="flex flex-1 w-full px-4 py-2 border rounded-l bg-primaryLight border-divider text-secondaryDark"
              :placeholder="$t('sse.url')"
              :disabled="connectionSSEState"
              @keyup.enter="serverValid ? toggleSSEConnection() : null"
            />
            <label
              for="event-type"
              class="px-4 py-2 font-semibold truncate border-t border-b bg-primaryLight border-divider text-secondaryLight"
            >
              {{ $t("sse.event_type") }}
            </label>
            <input
              id="event-type"
              v-model="eventType"
              class="flex flex-1 w-full px-4 py-2 border rounded-r bg-primaryLight border-divider text-secondaryDark"
              spellcheck="false"
              :disabled="connectionSSEState"
              @keyup.enter="serverValid ? toggleSSEConnection() : null"
            />
          </div>
          <ButtonPrimary
            id="start"
            :disabled="!serverValid"
            name="start"
            class="w-32"
            :label="
              !connectionSSEState ? $t('action.start') : $t('action.stop')
            "
            :loading="connectingState"
            @click.native="toggleSSEConnection"
          />
        </div>
      </div>
    </template>
    <template #secondary>
      <RealtimeLog
        :title="$t('sse.log')"
        :log="log"
        @delete="clearLogEntries()"
      />
    </template>
  </AppPaneLayout>
</template>

<script>
import { defineComponent } from "@nuxtjs/composition-api"
import "splitpanes/dist/splitpanes.css"
import debounce from "lodash/debounce"
import { logHoppRequestRunToAnalytics } from "~/helpers/fb/analytics"
import {
  SSEEndpoint$,
  setSSEEndpoint,
  SSEEventType$,
  setSSEEventType,
  SSESocket$,
  setSSESocket,
  SSEConnectingState$,
  SSEConnectionState$,
  setSSEConnectionState,
  setSSEConnectingState,
  SSELog$,
  setSSELog,
  addSSELogLine,
} from "~/newstore/SSESession"
import { useStream } from "~/helpers/utils/composables"

export default defineComponent({
  setup() {
    return {
      connectionSSEState: useStream(
        SSEConnectionState$,
        false,
        setSSEConnectionState
      ),
      connectingState: useStream(
        SSEConnectingState$,
        false,
        setSSEConnectingState
      ),
      server: useStream(SSEEndpoint$, "", setSSEEndpoint),
      eventType: useStream(SSEEventType$, "", setSSEEventType),
      sse: useStream(SSESocket$, null, setSSESocket),
      log: useStream(SSELog$, [], setSSELog),
    }
  },
  data() {
    return {
      isUrlValid: true,
    }
  },
  computed: {
    serverValid() {
      return this.isUrlValid
    },
  },
  watch: {
    server() {
      this.debouncer()
    },
  },
  created() {
    if (process.browser) {
      this.worker = this.$worker.createRejexWorker()
      this.worker.addEventListener("message", this.workerResponseHandler)
    }
  },
  destroyed() {
    this.worker.terminate()
  },
  methods: {
    debouncer: debounce(function () {
      this.worker.postMessage({ type: "sse", url: this.server })
    }, 1000),
    workerResponseHandler({ data }) {
      if (data.url === this.url) this.isUrlValid = data.result
    },
    toggleSSEConnection() {
      // If it is connecting:
      if (!this.connectionSSEState) return this.start()
      // Otherwise, it's disconnecting.
      else return this.stop()
    },
    start() {
      this.connectingState = true
      this.log = [
        {
          payload: this.$t("state.connecting_to", { name: this.server }),
          source: "info",
          event: "connecting",
          ts: Date.now(),
        },
      ]
      if (typeof EventSource !== "undefined") {
        try {
          this.sse = new EventSource(this.server)
          this.sse.onopen = () => {
            this.connectingState = false
            this.connectionSSEState = true
            this.log = [
              {
                payload: this.$t("state.connected_to", { name: this.server }),
                source: "info",
                event: "connected",
                ts: Date.now(),
              },
            ]
            this.$toast.success(this.$t("state.connected"))
          }
          this.sse.onerror = () => {
            this.handleSSEError()
          }
          this.sse.onclose = () => {
            this.connectionSSEState = false
            addSSELogLine({
              payload: this.$t("state.disconnected_from", {
                name: this.server,
              }),
              source: "disconnected",
              event: "disconnected",
              ts: Date.now(),
            })
            this.$toast.error(this.$t("state.disconnected"))
          }
          this.sse.addEventListener(this.eventType, ({ data }) => {
            addSSELogLine({
              payload: data,
              source: "server",
              ts: Date.now(),
            })
          })
        } catch (e) {
          this.handleSSEError(e)
          this.$toast.error(this.$t("error.something_went_wrong"))
        }
      } else {
        this.log = [
          {
            payload: this.$t("error.browser_support_sse"),
            source: "disconnected",
            event: "error",
            ts: Date.now(),
          },
        ]
      }

      logHoppRequestRunToAnalytics({
        platform: "sse",
      })
    },
    handleSSEError(error) {
      this.stop()
      this.connectionSSEState = false
      addSSELogLine({
        payload: this.$t("error.something_went_wrong"),
        source: "disconnected",
        event: "error",
        ts: Date.now(),
      })
      if (error !== null)
        addSSELogLine({
          payload: error,
          source: "disconnected",
          event: "error",
          ts: Date.now(),
        })
    },
    stop() {
      this.sse.close()
      this.sse.onclose()
    },
    clearLogEntries() {
      this.log = []
    },
  },
})
</script>
