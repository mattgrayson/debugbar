<!-- This is the main component of the debugbar. -->
<!--It's imported in the different App*.vue components.-->

<script setup lang="ts">
import { createConsumer } from "@rails/actioncable"
import { computed, reactive } from "vue"
import { CodeBracketIcon, XCircleIcon, PauseIcon, PlayIcon, TrashIcon } from "@heroicons/vue/16/solid"
import TabButton from "@/components/TabButton.vue"
import { useRequestsStore } from "@/stores/RequestsStore.ts"
import { useConfigStore } from "@/stores/configStore.ts"
import StatusCode from "@/components/ui/StatusCode.vue"
import RequestListItem from "@/components/RequestListItem.vue"
import RequestTimings from "@/components/RequestTimings.vue"
import HttpVerb from "@/components/ui/HttpVerb.vue"
import DebugbarBody from "@/DebugbarBody.vue"

let requestsStore = useRequestsStore()
let configStore = useConfigStore()

const state = reactive({
  mode: configStore.config.mode,
  activeTab: "",
  isConnected: false,
  isPolling: configStore.config.mode === "poll",
})

const defaultTabName = "queries"

const isActive = computed(() => {
  return state.activeTab != ""
})

const devMode = computed(() => {
  return import.meta.env.DEV
})

let debugbarChannel = null

if (configStore.config.mode === "ws") {
  debugbarChannel = createConsumer(configStore.config.actionCableUrl).subscriptions.create(
    { channel: configStore.config.cable.channelName },
    {
      connected() {
        state.isConnected = true
        debugbarChannel.send({ ids: [] })
      },

      disconnected() {
        state.isConnected = false
      },
      received(data) {
        if (data.length == 0) {
          return
        }

        const ids = requestsStore.addRequests(data)

        if (!isActive.value) {
          requestsStore.setCurrentRequestById(ids[ids.length - 1])
        }

        setTimeout(() => {
          debugbarChannel.send({ ids: ids })
        }, 50)
      },
    }
  )
} else if (configStore.config.mode === "poll") {
  console.log(
    `Using debugbar in "polling mode". Consider using "ws" mode for better performance (requires ActionCable).`
  )
  setInterval(() => {
    if (!state.isPolling) {
      return
    }

    fetch(configStore.config.pollUrl)
      .then((response) => response.json())
      .then((data) => {
        if (data.length == 0) {
          return
        }

        console.log(data)

        const ids = requestsStore.addRequests(data)

        if (!isActive.value) {
          requestsStore.setCurrentRequestById(ids[ids.length - 1])
        }

        fetch(configStore.config.pollUrl + "/confirm", {
          // mode: "no-cors",
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({ ids: ids }),
        })
      })
  }, configStore.config.poll.interval)
} else {
  console.log(`Using debugbar in "offline mode", ideal for demos using fixtures.`)
}

const clearRequests = () => {
  state.activeTab = ""
  requestsStore.clearRequests()
  debugbarChannel?.send({ clear: true })
  state.isPolling = true
}

const togglePolling = () => {
  state.isPolling = !state.isPolling
}

const setActiveTab = (tab) => {
  state.activeTab = tab
}
</script>

<template>
  <div class="">
    <div class="px-1 flex items-center justify-between w-full bg-stone-700 text-white">
      <div class="py-0.5">{{ requestsStore.requestCount }} requests</div>
      <div class="pl-1.5 flex items-center space-x-2">
        <div
          v-if="state.mode == 'ws'"
          class="rounded-full w-2 h-2"
          :class="{
            'bg-emerald-500': state.isConnected,
            'bg-rose-500': !state.isConnected,
          }"
          :title="state.isConnected ? 'Connected to backend via websocket' : 'Cannot connect to backend via websocket'"
        ></div>
        <button
          @click="togglePolling"
          v-if="state.mode == 'poll'"
          :title="state.isPolling ? 'Pause polling' : 'Resume polling'"
        >
          <pause-icon v-if="state.isPolling" class="size-4" />
          <play-icon v-if="!state.isPolling" class="size-4" />
        </button>

        <button @click="clearRequests" title="Clear all requests (frontend and backend)">
          <trash-icon class="size-3.5" />
        </button>
      </div>
    </div>

    <!--  No request yet, the debugbar is full width but empty  -->
    <div v-if="requestsStore.currentRequest == null">
      <p class="px-1 py-2">
        No request detected yet.
        <a class="text-blue-700 font-medium underline" href="https://debugbar.dev/docs/troubleshooting/">Learn more</a>
      </p>
    </div>

    <div v-if="requestsStore.currentRequest" class="z-[9999] text-stone-900 w-full">
      <div id="debugbar-header" class="flex flex-col px-1 items-center justify-between bg-stone-50">
        <!--      THE LIST-->
        <div v-if="!isActive" class="w-full space-y-1">
          <div
            v-for="r in requestsStore.requests"
            class="cursor-pointer"
            @click="
              (_event) => {
                requestsStore.setCurrentRequestById(r.id)
                state.activeTab = defaultTabName
              }
            "
          >
            <request-list-item :request="r" class="hover:bg-stone-200" />
          </div>
        </div>

        <!--      THE PANEL-->
        <div v-if="isActive" class="w-full">
          <div class="flex w-full justify-between items-center">
            <div class="py-1.5 w-full flex items-center">
              <http-verb :verb="requestsStore.currentRequest.meta.method" />
              <div
                class="pl-1 grow text-nowrap overflow-hidden font-mono"
                :title="requestsStore.currentRequest.meta.path"
              >
                {{ requestsStore.currentRequest.meta.path }}
              </div>

              <request-timings :request="requestsStore.currentRequest" />
            </div>

            <div class="pl-1.5 flex items-center space-x-1">
              <button
                v-if="configStore.config.mode == 'poll'"
                @click="togglePolling"
                :title="state.isPolling ? 'Pause polling' : 'Resume polling'"
              >
                <pause-icon v-if="state.isPolling" class="size-4" />
                <play-icon v-if="!state.isPolling" class="size-4" />
              </button>

              <button @click="state.activeTab = ''" title="Close">
                <x-circle-icon class="size-4" />
              </button>
            </div>
          </div>

          <div class="flex w-full justify-end flex-wrap-reverse items-center">
            <div class="flex grow justify-start items-center">
              <tab-button
                v-for="(v, k) in requestsStore.currentRequest.dataForTabs"
                key="k"
                :label="v.label"
                :count="v?.count"
                :is-active="k === state.activeTab"
                :disabled="v.count == 0"
                @click="setActiveTab(k)"
                >{{ v.label }}</tab-button
              >

              <button
                v-if="devMode"
                @click="setActiveTab('debug')"
                class="px-3 py-1.5 text-stone-600"
                :class="{ 'bg-stone-300': state.activeTab == 'debug' }"
              >
                <CodeBracketIcon class="size-4" />
              </button>
            </div>

            <div class="flex items-center space-x-3">
              <div @click="setActiveTab('request')" class="flex cursor-pointer">
                <span class="text-sm text-stone-600 font-medium tracking-wide">
                  {{ requestsStore.currentRequest.routeAlias }}
                </span>
                <status-code :code="requestsStore.currentRequest.meta.status" class="ml-3" />
              </div>
            </div>
          </div>
        </div>
      </div>

      <debugbar-body :request="requestsStore.currentRequest" :active-tab="state.activeTab" />
    </div>
  </div>
</template>
