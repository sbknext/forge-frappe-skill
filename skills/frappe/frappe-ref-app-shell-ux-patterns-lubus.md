---
title: Frappe Ref — App Shell UX Patterns (lubus)
category: frappe
tags: ['frappe', 'frappe-ui', 'ux', 'app-shell', 'layout']
source: lubus
---

# App Shell Patterns

Detailed layouts for Frappe app shells derived from official apps.

## CRM Shell Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🔲 Frappe CRM        Search...                          🔔  👤 User Menu  │
├──────────────┬──────────────────────────────────────────────────────────────┤
│              │                                                              │
│  📊 Leads    │  Leads                      [+ New Lead]                    │
│     (124)    │  ─────────────────────────────────────────                  │
│              │  🔍 Search    [Status ▾] [Source ▾]    [List|Kanban|Grid]   │
│  💰 Deals    │                                                              │
│     (67)     │  ┌─────────────────────────────────────────────────────┐    │
│              │  │ ☑ │ 👤 John Doe           │ Hot    │ 2 hours ago    │    │
│  👥 Contacts │  │   │    Acme Corp          │        │                │    │
│              │  ├───┼─────────────────────────────────────────────────┤    │
│  🏢 Orgs     │  │ ☐ │ 👤 Jane Smith         │ Warm   │ Yesterday      │    │
│              │  │   │    Tech Inc           │        │                │    │
│  📅 Activities│  └───┴─────────────────────────────────────────────────┘    │
│              │                                                              │
│  ──────────  │                                                              │
│  ⚙️ Settings │                                                              │
│              │                                                              │
└──────────────┴──────────────────────────────────────────────────────────────┘
```

## CRM with Detail Panel

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🔲 Frappe CRM        Search...                          🔔  👤 User Menu  │
├──────────────┬─────────────────────────────┬────────────────────────────────┤
│              │                             │                                │
│  📊 Leads    │  Leads              [+ New] │  ✕  Lead: John Doe            │
│     (124)    │  ─────────────────────────  │  ────────────────────────────  │
│              │                             │                                │
│  💰 Deals    │  ┌─────────────────────┐   │  [Details] [Activity] [Notes]  │
│     (67)     │  │ 👤 John Doe    Hot  │◀──│                                │
│              │  │    Acme Corp        │   │  Name: John Doe                │
│  👥 Contacts │  ├─────────────────────┤   │  Email: john@acme.com          │
│              │  │ 👤 Jane Smith  Warm │   │  Phone: +1 555-0123            │
│  🏢 Orgs     │  │    Tech Inc         │   │  Company: Acme Corp            │
│              │  ├─────────────────────┤   │  Status: Hot 🔥                │
│              │  │ 👤 Bob Wilson  Cold │   │  Source: Website               │
│              │  │    StartupXYZ       │   │                                │
│              │  └─────────────────────┘   │  [Convert to Deal]             │
│              │                             │                                │
└──────────────┴─────────────────────────────┴────────────────────────────────┘
```

## Helpdesk Shell Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🎫 Helpdesk          Search tickets...                  🔔  👤 Agent      │
├──────────────┬──────────────────────────────────────────────────────────────┤
│              │                                                              │
│  📥 Tickets  │  Tickets                    [+ New Ticket]                  │
│              │  ─────────────────────────────────────────                  │
│  VIEWS       │                                                              │
│  All (234)   │  ┌─────────────────────────────────────────────────────┐    │
│  Mine (12)   │  │ #1234 │ Cannot login           │ 🔴 High │ 2h ago  │    │
│  Unassigned  │  │       │ john@customer.com      │ Open    │ SLA: 4h │    │
│    (45)      │  ├───────┼────────────────────────┼─────────┼─────────┤    │
│              │  │ #1233 │ Payment issue          │ 🟡 Med  │ 5h ago  │    │
│  PRIORITY    │  │       │ jane@customer.com      │ Working │ SLA: OK │    │
│  🔴 High (8) │  ├───────┼────────────────────────┼─────────┼─────────┤    │
│  🟡 Med (23) │  │ #1232 │ Feature request        │ 🟢 Low  │ 1d ago  │    │
│  🟢 Low (45) │  │       │ bob@customer.com       │ Open    │         │    │
│              │  └───────┴────────────────────────┴─────────┴─────────┘    │
│  ──────────  │                                                              │
│  📊 Reports  │                                                              │
│  ⚙️ Settings │                                                              │
└──────────────┴──────────────────────────────────────────────────────────────┘
```

## HRMS Shell Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  👥 HRMS               Search...                         🔔  👤 Employee   │
├──────────────┬──────────────────────────────────────────────────────────────┤
│              │                                                              │
│  🏠 Home     │  My Dashboard                                               │
│              │  ─────────────────────────────────────────                  │
│  ME          │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
│  📅 Leave    │  │ Leave Balance │  │ Attendance    │  │ Pending       │   │
│  ⏰ Attendance│  │               │  │               │  │ Approvals     │   │
│  💰 Payslips │  │  PL: 12 days  │  │  Present: 22  │  │               │   │
│  📝 Requests │  │  SL: 5 days   │  │  Absent: 1    │  │    3          │   │
│              │  │  CL: 3 days   │  │  WFH: 2       │  │               │   │
│  TEAM        │  └───────────────┘  └───────────────┘  └───────────────┘   │
│  👥 Directory│                                                              │
│  📊 Reports  │  Recent Activity                                            │
│              │  ─────────────────                                          │
│  ──────────  │  • Leave request approved - 2 hours ago                     │
│  ⚙️ Settings │  • Payslip generated - Yesterday                            │
│              │  • Attendance marked - Today 9:00 AM                        │
└──────────────┴──────────────────────────────────────────────────────────────┘
```

## Vue Implementation

### App.vue (Root shell)

```vue
<template>
  <div class="h-screen flex flex-col bg-white">
    <!-- Header -->
    <header class="h-12 border-b flex items-center px-4 justify-between">
      <div class="flex items-center gap-4">
        <button @click="sidebarOpen = !sidebarOpen" class="lg:hidden">
          <FeatherIcon name="menu" class="w-5 h-5" />
        </button>
        <router-link to="/" class="font-semibold">{{ appTitle }}</router-link>
      </div>
      
      <div class="flex-1 max-w-md mx-4">
        <CommandPalette />
      </div>
      
      <div class="flex items-center gap-2">
        <NotificationsDropdown />
        <UserDropdown />
      </div>
    </header>
    
    <!-- Body -->
    <div class="flex-1 flex overflow-hidden">
      <!-- Sidebar (responsive) -->
      <Transition name="slide">
        <aside 
          v-show="sidebarOpen || !isMobile"
          :class="[
            'w-56 border-r bg-gray-50 flex flex-col overflow-y-auto',
            isMobile && 'fixed inset-y-0 left-0 z-40 pt-12'
          ]"
        >
          <Sidebar />
        </aside>
      </Transition>
      
      <!-- Backdrop for mobile -->
      <div 
        v-if="sidebarOpen && isMobile"
        class="fixed inset-0 bg-black/20 z-30"
        @click="sidebarOpen = false"
      />
      
      <!-- Main content -->
      <main class="flex-1 overflow-auto">
        <router-view v-slot="{ Component }">
          <Transition name="fade" mode="out-in">
            <component :is="Component" />
          </Transition>
        </router-view>
      </main>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useMediaQuery } from '@vueuse/core'

const sidebarOpen = ref(false)
const isMobile = useMediaQuery('(max-width: 1024px)')
const appTitle = 'My App'
</script>
```

### Sidebar.vue

```vue
<template>
  <div class="flex flex-col h-full">
    <!-- Navigation -->
    <nav class="flex-1 p-2 space-y-1">
      <SidebarItem
        v-for="item in navigation"
        :key="item.name"
        :item="item"
      />
    </nav>
    
    <!-- Footer -->
    <div class="p-2 border-t">
      <SidebarItem :item="settingsItem" />
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

const navigation = [
  { name: 'Leads', icon: 'users', to: '/leads', count: 124 },
  { name: 'Deals', icon: 'dollar-sign', to: '/deals', count: 67 },
  { name: 'Contacts', icon: 'user', to: '/contacts' },
  { name: 'Organizations', icon: 'briefcase', to: '/organizations' },
]

const settingsItem = { name: 'Settings', icon: 'settings', to: '/settings' }
</script>
```

### SidebarItem.vue

```vue
<template>
  <router-link
    :to="item.to"
    :class="[
      'flex items-center gap-3 px-3 py-2 rounded-md text-sm transition-colors',
      isActive 
        ? 'bg-gray-900 text-white' 
        : 'text-gray-700 hover:bg-gray-200'
    ]"
  >
    <FeatherIcon :name="item.icon" class="w-4 h-4" />
    <span class="flex-1">{{ item.name }}</span>
    <Badge v-if="item.count" :variant="isActive ? 'solid' : 'subtle'">
      {{ item.count }}
    </Badge>
  </router-link>
</template>

<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const props = defineProps({
  item: { type: Object, required: true }
})

const route = useRoute()
const isActive = computed(() => route.path.startsWith(props.item.to))
</script>
```

## Layout Variations

### Full-width detail (no split)

For complex forms or dashboards:

```vue
<template>
  <div class="flex-1 overflow-auto">
    <div class="max-w-4xl mx-auto py-6 px-4">
      <Breadcrumb :items="breadcrumbs" class="mb-4" />
      <DetailView :doc="doc" />
    </div>
  </div>
</template>
```

### Three-column layout

For email/messaging apps:

```
┌──────────┬────────────────┬──────────────────────────────┐
│ Folders  │ Message List   │ Message Content              │
│          │                │                              │
│ Inbox    │ Subject 1      │ From: sender@example.com     │
│ Sent     │ Preview...     │ To: me@example.com           │
│ Drafts   │                │                              │
│ Archive  │ Subject 2      │ Lorem ipsum dolor sit amet   │
│          │ Preview...     │ consectetur adipiscing...    │
└──────────┴────────────────┴──────────────────────────────┘
```

### Dashboard layout

For overview/home pages:

```vue
<template>
  <div class="p-6">
    <h1 class="text-xl font-semibold mb-6">Dashboard</h1>
    
    <!-- Stats row -->
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 mb-6">
      <StatCard v-for="stat in stats" :key="stat.label" :stat="stat" />
    </div>
    
    <!-- Content grid -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
      <div class="lg:col-span-2">
        <Card title="Recent Activity">
          <ActivityFeed :items="recentActivities" />
        </Card>
      </div>
      <div>
        <Card title="Quick Actions">
          <QuickActionList :actions="quickActions" />
        </Card>
      </div>
    </div>
  </div>
</template>
```
