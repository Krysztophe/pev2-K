<script lang="ts" setup>
import { inject, nextTick, onMounted, provide, reactive, ref, watch } from "vue"
import type { Ref } from "vue"
import PlanNodeDetail from "@/components/PlanNodeDetail.vue"
import NodeBadges from "@/components/NodeBadges.vue"
import type { IPlan, Node, ViewOptions } from "@/interfaces"
import {
  HighlightedNodeIdKey,
  PlanKey,
  SelectedNodeIdKey,
  SelectNodeKey,
  ViewOptionsKey,
} from "@/symbols"
import { keysToString, sortKeys } from "@/filters"
import { HighlightType, NodeProp } from "@/enums"
import { findNodeBySubplanName } from "@/services/help-service"
import useNode from "@/node"
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome"
import {
  faChevronDown,
  faChevronUp,
  faSearch,
} from "@fortawesome/free-solid-svg-icons"

const outerEl = ref<Element | null>(null) // The outer Element, useful for CTE and subplans

const selectedNodeId = inject(SelectedNodeIdKey)
if (!selectedNodeId) {
  throw new Error(`Could not resolve ${SelectedNodeIdKey.description}`)
}
const highlightedNodeId = inject(HighlightedNodeIdKey)
const selectNode = inject(SelectNodeKey)
if (!selectNode) {
  throw new Error(`Could not resolve ${SelectNodeKey.description}`)
}
const viewOptions = inject(ViewOptionsKey) as ViewOptions

interface Props {
  node: Node
}
const props = defineProps<Props>()

const showDetails = ref<boolean>(false)

const node = reactive<Node>(props.node)
const plan = inject(PlanKey) as Ref<IPlan>
const updateNodeSize =
  inject<(node: Node, size: [number, number]) => null>("updateNodeSize")

const {
  nodeName,
  barWidth,
  barColor,
  highlightValue,
  isNeverExecuted,
  workersLaunchedCount,
  workersPlannedCount,
  workersPlannedCountReversed,
} = useNode(plan, node, viewOptions)

onMounted(async () => {
  updateSize(node)
})

function updateSize(node: Node) {
  const rect = outerEl.value?.getBoundingClientRect()
  if (rect) {
    updateNodeSize?.(node, [rect.width, rect.height])
  }
}
provide("updateSize", updateSize)

watch(showDetails, () => {
  window.setTimeout(() => updateSize(node), 1)
})

watch(viewOptions, () => {
  // Using nextTick has the same effect as a debounce making all nodes
  // size to be updated all at once
  nextTick(() => {
    updateSize(node)
  })
})

watch(selectedNodeId, () => {
  if (selectedNodeId.value == node.nodeId) {
    showDetails.value = true
  }
})

function centerCte() {
  const cteNode = findNodeBySubplanName(
    plan.value,
    node[NodeProp.CTE_NAME] as string,
  )
  if (cteNode) {
    selectNode?.(cteNode.nodeId, true)
  }
}
</script>

<template>
  <div ref="outerEl" @mousedown.stop>
    <div
      :class="[
        'text-start plan-node',
        {
          detailed: showDetails,
          'never-executed': isNeverExecuted,
          parallel: workersPlannedCount,
          selected: selectedNodeId == node.nodeId,
          highlight: highlightedNodeId == node.nodeId,
        },
      ]"
    >
      <div v-if="node[NodeProp.SUBPLAN_NAME]" class="fixed-bottom text-center">
        <b class="subplan-name fst-italic px-1">
          {{ node[NodeProp.SUBPLAN_NAME] }}
        </b>
      </div>
      <div class="workers text-secondary py-0 px-1" v-if="workersPlannedCount">
        <div
          v-for="index in workersPlannedCountReversed"
          :key="index"
          :style="{
            top: 1 + index * 2 + 'px',
            left: 1 + (index + 1) * 3 + 'px',
          }"
          :class="{ 'border-dashed': index >= workersLaunchedCount }"
        >
          {{ index }}
        </div>
      </div>
      <div
        class="plan-node-body card"
        @mouseenter="highlightedNodeId = node.nodeId"
        @mouseleave="highlightedNodeId = undefined"
      >
        <div class="card-body header no-focus-outline">
          <header class="mb-0 d-flex justify-content-between">
            <h4
              class="text-body overflow-hidden btn btn-light text-start py-0 px-1"
              @click.prevent.stop="showDetails = !showDetails"
            >
              <span class="text-secondary">
                <FontAwesomeIcon
                  fixed-width
                  :icon="faChevronUp"
                  v-if="showDetails"
                ></FontAwesomeIcon>
                <FontAwesomeIcon
                  fixed-width
                  :icon="faChevronDown"
                  v-else
                ></FontAwesomeIcon>
              </span>
              {{ nodeName }}
            </h4>
            <div class="text-nowrap">
              <node-badges :node="node" />
              <a
                class="fw-normal small ms-1"
                href=""
                @click.prevent.stop="selectNode(node.nodeId, true)"
              >
                #{{ node.nodeId }}
              </a>
            </div>
          </header>
          <div class="text-start font-monospace">
            <div
              v-if="
                node[NodeProp.RELATION_NAME] || node[NodeProp.FUNCTION_NAME]
              "
              :class="{ 'line-clamp-2': !showDetails }"
            >
              <span class="text-secondary">on</span>
              <span v-if="node[NodeProp.SCHEMA]"
                >{{ node[NodeProp.SCHEMA] }}.</span
              >{{ node[NodeProp.RELATION_NAME] }}
              {{ node[NodeProp.FUNCTION_NAME] }}
              <span v-if="node[NodeProp.ALIAS]">
                <span class="text-secondary">as</span>
                {{ node[NodeProp.ALIAS] }}
              </span>
            </div>
            <div
              v-else-if="node[NodeProp.ALIAS]"
              :class="{ 'line-clamp-2': !showDetails }"
            >
              <span class="text-secondary">on</span>
              <span
                v-html="keysToString(node[NodeProp.ALIAS] as string)"
              ></span>
            </div>
            <div
              v-if="node[NodeProp.GROUP_KEY]"
              :class="{ 'line-clamp-2': !showDetails }"
            >
              <span class="text-secondary">by</span>
              <span
                v-html="keysToString(node[NodeProp.GROUP_KEY] as string)"
              ></span>
            </div>
            <div
              v-if="node[NodeProp.SORT_KEY]"
              :class="{ 'line-clamp-2': !showDetails }"
            >
              <span class="text-secondary">by</span>
              <span
                v-html="
                  sortKeys(
                    node[NodeProp.SORT_KEY] as string[],
                    node[NodeProp.PRESORTED_KEY] as string[],
                  )
                "
              ></span>
            </div>
            <div v-if="node[NodeProp.JOIN_TYPE]">
              {{ node[NodeProp.JOIN_TYPE] }}
              <span class="text-secondary">join</span>
            </div>
            <div
              v-if="node[NodeProp.INDEX_NAME]"
              :class="{ 'line-clamp-2': !showDetails }"
            >
              <span class="text-secondary">using</span>
              <span
                v-html="keysToString(node[NodeProp.INDEX_NAME] as string)"
              ></span>
            </div>
            <div
              v-if="node[NodeProp.HASH_CONDITION]"
              :class="{ 'line-clamp-2': !showDetails }"
            >
              <span class="text-secondary">on</span>
              <span
                v-html="keysToString(node[NodeProp.HASH_CONDITION] as string)"
              ></span>
            </div>
            <div v-if="node[NodeProp.CTE_NAME]">
              <a class="text-reset" href="" @click.prevent.stop="centerCte">
                <FontAwesomeIcon
                  :icon="faSearch"
                  class="text-secondary"
                ></FontAwesomeIcon>
                <span class="text-secondary">CTE</span>
                {{ node[NodeProp.CTE_NAME] }}
              </a>
            </div>
          </div>

          <div
            v-if="
              viewOptions.highlightType !== HighlightType.NONE &&
              highlightValue !== null
            "
          >
            <div class="progress node-bar-container" style="height: 5px">
              <div
                class="progress-bar"
                role="progressbar"
                :style="{
                  width: barWidth + '%',
                  'background-color': barColor,
                }"
                aria-valuenow="0"
                aria-valuemin="0"
                aria-valuemax="100"
              ></div>
            </div>
            <span class="node-bar-label">
              <span class="text-secondary"
                >{{ viewOptions.highlightType }}:</span
              >
              <span v-html="highlightValue"></span>
            </span>
          </div>
        </div>
        <plan-node-detail :node="node" v-if="showDetails"></plan-node-detail>
      </div>
    </div>
  </div>
</template>
