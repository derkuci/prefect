<template>
  <p-context-sidebar class="context-sidebar">
    <template #header>
      <router-link :to="routes.root()" class="context-sidebar__logo-link">
        <p-icon icon="Prefect" class="context-sidebar__logo-icon" />
      </router-link>
    </template>
    <p-context-nav-item title="Dashboard" :to="routes.dashboard()" />
    <p-context-nav-item title="Flow Runs" :to="routes.flowRuns()" />
    <p-context-nav-item title="Flows" :to="routes.flows()" />
    <p-context-nav-item title="Deployments" :to="routes.deployments()" />
    <p-context-nav-item v-if="canSeeWorkPools" title="Work Pools" :to="routes.workPools()" />
    <p-context-nav-item v-if="!canSeeWorkPools" title="Work Queues" :to="routes.workQueues()" />
    <p-context-nav-item title="Blocks" :to="routes.blocks()" />
    <p-context-nav-item :title="localization.info.variables" :to="routes.variables()" />
    <p-context-nav-item title="Notifications" :to="routes.notifications()" />
    <p-context-nav-item title="Concurrency" :to="routes.concurrencyLimits()" />
    <p-context-nav-item v-if="canSeeArtifacts" title="Artifacts" :to="routes.artifacts()" />

    <template #footer>
      <p-context-nav-item title="Settings" :to="routes.settings()" />
    </template>
  </p-context-sidebar>
</template>

<script lang="ts" setup>
  import { PContextSidebar, PContextNavItem } from '@prefecthq/prefect-design'
  import { localization } from '@prefecthq/prefect-ui-library'
  import { computed } from 'vue'
  import { useCan } from '@/compositions/useCan'
  import { routes } from '@/router'

  const can = useCan()
  const canSeeWorkPools = computed(() => can.access.work_pools && can.read.work_pool)
  const canSeeArtifacts = computed(() => can.access.artifacts)
</script>

<style>
.context-sidebar__logo-link { @apply
  outline-none
  rounded-md
  focus:ring-spacing-focus-ring
  focus:ring-focus-ring
}

.context-sidebar__logo-link:focus:not(:focus-visible) { @apply
  ring-transparent
}

.context-sidebar__logo-icon { @apply
  !w-[42px]
  !h-[42px]
}
</style>