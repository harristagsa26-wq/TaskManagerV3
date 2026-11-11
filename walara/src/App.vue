<script setup lang="ts">

import { computed, ref, watch } from 'vue'
import {
  Sidebar,
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarGroupContent,
  SidebarGroupLabel,
  SidebarHeader,
  SidebarInset,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarProvider,
  SidebarRail,
  SidebarTrigger,
} from '@/components/ui/sidebar'
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from '@/components/ui/alert-dialog'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import {
  GalleryVerticalEnd,
  List,
  Clock,
  CalendarOff,
  CheckCheck,
  Pencil,
  Trash2,
  Plus,
  Type,
  Search,
  Sun,
  Moon,
} from 'lucide-vue-next'
import { toast } from 'vue-sonner'
import { Toaster } from '@/components/ui/sonner'


type Task = {
  id: number
  title: string
  type: 'deadline' | 'none'
  deadline: string 
  done: boolean
}

const isDark = ref(localStorage.getItem('theme') === 'dark')

watch(
  isDark,
  (value) => {
    document.documentElement.classList.toggle('dark', value)
    localStorage.setItem('theme', value ? 'dark' : 'light')
  },
  { immediate: true },
)

function toggleDark() {
  isDark.value = !isDark.value
}

function loadTask(): Task[] {
  try {
    return JSON.parse(localStorage.getItem('tasks') || '[]')
  } catch {
    return[]
  }
}

const tasks = ref<Task[]>(loadTask())

watch(
  tasks,
  (value) => localStorage.setItem('tasks', JSON.stringify(value)),
  { deep: true },
)

const filters = ['All', 'Deadline', 'No deadline', 'Completed task']
const filter = ref('All')

const filterFuncs: Record<string, (t: Task) => boolean> = {
  All: (t) => !t.done,
  Deadline: (t) => t.type === 'deadline' && !t.done,
  'No deadline': (t) => t.type === 'none' && !t.done,
  'Completed task': (t) => t.done,
}

const search = ref('')

const filteredTasks = computed(() => {
    const predicate = filterFuncs[filter.value] ?? filterFuncs.All!
    const query = search.value.trim().toLowerCase()

    return tasks.value.filter(predicate)
    .filter((t) => t.title.toLowerCase().includes(query))
    .sort((a, b) => {
      const aHas = a.type === 'deadline' && a.deadline
      const bHas = b.type === 'deadline' && b.deadline

      if(aHas && bHas) return a.deadline.localeCompare(b.deadline)
      if(aHas) return -1
      if(bHas) return 1
      return 0 
    })
})

function today() {
  return new Date().toLocaleDateString('en-CA')
}

function isOverdue(t: Task) {
  return t.type === 'deadline' 
  && 
  !t.done
  && 
  !!t.deadline 
  && 
  t.deadline < today()
}

const remaining = computed(() => tasks.value.filter((t) => !t.done).length)
const overdueCount = computed(() => tasks.value.filter(isOverdue).length)

// Add button and the editing

const dialogOpen = ref(false)
const editingId = ref<number | null>(null)
const title = ref('')
const hasDeadline = ref(false)
const deadline = ref('')
const error = ref('')

function openAdd() {
  editingId.value = null
  title.value = ''
  hasDeadline.value = false
  deadline.value = ''
  error.value = ''
  dialogOpen.value = true
}

function openEdit(task: Task) {
  if(task.done) return
  editingId.value = task.id
  title.value = task.title
  hasDeadline.value = task.type === 'deadline'
  deadline.value = task.deadline
  error.value = ''
  dialogOpen.value = true
}

function saveTask() {
  const  text = title.value.trim()

  if(!text) {
    error.value = 'Please enter a task'
    return
  }  
  if(hasDeadline.value && !deadline.value) {
    error.value = 'Please pick a deadline date'
    return
  }
    error.value = ''
  
  const type: Task['type'] = hasDeadline.value ? 'deadline' : 'none'
  const date = hasDeadline.value ? deadline.value : ''

  if(editingId.value === null) {
    tasks.value.push({
      id: Date.now(), 
      title: text, type, 
      deadline: date, 
      done: false
    })
    toast.success('Task added!')
  } else {
    const task = tasks.value.find((t) => t.id === editingId.value)
    if(task && !task.done) {
      task.title = text
      task.type = type
      task.deadline = date
      toast.success('Task Updated!')
    }
  }
  dialogOpen.value = false
 }

function deleteTask(task: Task) {
  if(task.done) return 
  tasks.value = tasks.value.filter((t) => t.id !== task.id)
  toast.success('Task deleted!')
}

const totalCounts = computed(() => tasks.value.length)

function clearTask() {
  tasks.value = []
  toast.success('All tasks cleared!')
}

</script>

<template>
  <SidebarProvider>
    <Sidebar>
      <SidebarHeader>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton size="lg">
              <div class="flex aspect-square 
              size-8 
              items-center 
              justify-center
              rounded-lg
              bg-sidebar-primary
              text-sidebar-primary-foreground">
              <GalleryVerticalEnd class="size-4"/>        
              </div>
              <span class="truncate font-bold select-none">My Task Manager</span>
            </SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarHeader>

      <SidebarContent>
        <SidebarGroup>
          <SidebarGroupLabel class="select-none">Filters</SidebarGroupLabel>
             <SidebarGroupContent>
              <SidebarMenu>
                <SidebarMenuItem v-for="f in filters" :key="f">
                  <SidebarMenuButton :is-active="filter === f" @click="filter = f" class="cursor-pointer select-none">
                    <List v-if="f === 'All'" />
                      <Clock v-else-if="f === 'Deadline'" /> 
                      <CalendarOff v-else-if="f === 'No deadline'"/>
                      <CheckCheck v-else />
                        <span>
                          {{ f }}
                        </span>
                  </SidebarMenuButton>
                </SidebarMenuItem>
              </SidebarMenu>
          </SidebarGroupContent>
        </SidebarGroup>
      </SidebarContent>

      <SidebarFooter>
        <div class="px-2 text-sm text-muted-foreground select-none">
          {{ remaining }} remaining 
          <span v-if="overdueCount" class="text-destructive select-none">
            . {{ overdueCount }} overdue
          </span>
        </div>
      </SidebarFooter>
      <SidebarRail/>
    </Sidebar>

    <SidebarInset class="min-h-svh">
      <header class="flex h-16 shrink-0 items-center justify-between gap-2 px-4">
        <SidebarTrigger class="-ml-1 cursor-pointer" />
        <header class="flex h-16 shrink-0 items-center gap-2 px-4">
      </header>

      <Button class="cursor-pointer" variant="outline" size="icon" @click="toggleDark">
      <Sun v-if="isDark" class="size-4" />
      <Moon v-else class="size-4" />
      </Button>

      <div class="ml-auto flex items-center gap-2">
      <Search />
      <Input
      v-model="search"
      type="search"
      placeholder="Search tasks..."
      class="w-40 sm:w-64"
    />
        <Button class="select-none cursor-pointer" @click="openAdd">
        <Plus class="size-4"/>
          Add Task
        </Button>

      <AlertDialog>
        <AlertDialogTrigger as-child>
          <Button class="select-none cursor-pointer" :disabled="totalCounts === 0">
            <Trash2 class="size-4"/>
            Clear tasks
          </Button>
        </AlertDialogTrigger>

        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>
              Clear all tasks?
            </AlertDialogTitle>
            <AlertDialogDescription>
              All {{ totalCounts }} task(s) will be remove including completed ones.
              This can't be undone
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel class="cursor-pointer">
              Cancel
            </AlertDialogCancel>
            <AlertDialogAction class="bg-destructive text-white hover:bg-destructive/90 select-none cursor-pointer"
            @click="clearTask">
            Clear all
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
      </div>
      </header>
      
    <div class="mx-auto flex w-full max-w-2xl flex-1 flex-col p-4 pt-0">

      <div v-if="filteredTasks.length === 0"
      class="flex flex-1 flex-col items-center justify-center gap-2 pb-16 text-center">
      <h1 class="mb-4 text-2xl font-semibold select-none">
        {{ filter }}
      </h1>

      <p v-if="filteredTasks.length === 0" 
      class="text-muted-foreground flex min-h [50vh] items-center justify-center select-none">
          No tasks yet.
        </p>
        </div>
        <template v-else>
          <h1 class="mb-4 text-2xl font-semibold justify-center select-none">
            {{ filter }}
          </h1>
        </template>

        <ul class="flex flex-col gap-2">
          <li v-for="task in filteredTasks"
          :key="task.id"
          class="flex items-center gap-3 rounded-lg border p-3"
          :class="{'bg-muted/50' :task.done}">

          <input type="checkbox" v-model="task.done" class="size-4 cursor-pointer"/>

          <div class="flex-1" :class="{'opacity-50': task.done}">
            <p :class="{'line-through': task.done}">
              {{ task.title }}
            </p>
            <p v-if="task.type === 'deadline'"
            class="text-xs select-none"
            :class="isOverdue(task) ? 'text-destructive' : 'text-muted-foreground'">
            Deadline: {{ task.deadline }}
            <span v-if="isOverdue(task)">(overdue)</span>
          </p>
          </div>

          <Button class="cursor-pointer" variant="outline"
          size="icon"
          :disabled="task.done"
          @click="openEdit(task)"
          >

          <Pencil class="size-4"/>
          </Button>
          
          <AlertDialog>
            <AlertDialogTrigger as-child>
              <Button class="cursor-pointer" variant="outline" size="icon" :disabled="task.done">
                <Trash2 class="size-4"/>
              </Button>
            </AlertDialogTrigger>
            <AlertDialogContent>
              <AlertDialogHeader>
                <AlertDialogTitle>
                  Delete this task?
                  <AlertDialogDescription>
                    "{{ task.title }}" will be removed. This can't be undone.
                  </AlertDialogDescription>
                </AlertDialogTitle>
              </AlertDialogHeader>
              <AlertDialogFooter>
              <AlertDialogCancel>
                Cancel
              </AlertDialogCancel>
              <AlertDialogAction
              class="bg-destructive text-white hover:bg-destructive/90"
              @click="deleteTask(task)">
                 Delete
              </AlertDialogAction>
            </AlertDialogFooter>
            </AlertDialogContent>
          </AlertDialog>
        </li>
        </ul>
    </div>
  </SidebarInset>

  <Dialog v-model:open="dialogOpen">
    <DialogContent class="gap-6 sm:max-w-md">
      <DialogHeader class="text-left">
        <DialogTitle class="text-xl">
          {{ editingId === null ? 'Add new task' : 'Edit Task' }}
        </DialogTitle>
        <DialogDescription>
          Enter a task and an optional deadline.
        </DialogDescription>
      </DialogHeader>

      <form class="grid gap-6" @submit.prevent="saveTask">
        <div class="grid gap-2">
          <Label for="title">Task</Label>
            <Input id="title" v-model="title" type="text" placeholder="Enter a task"/>
        </div>

        <label class="flex items-center gap-2 text-sm">
          <input type="checkbox" v-model="hasDeadline" class="size-4"/>
          This task has a deadline
        </label>

        <div v-if="hasDeadline" class="grid gap-2">
          <Label for="deadline">Deadline</Label>
          <Input id="deadline" v-model="deadline" type="date" />
        </div>

        <p v-if="error" class="text-sm text-destructive">
          {{ error }}
        </p>
        <DialogFooter>
          <Button type="submit">{{ editingId === null ? 'Add' : 'Save' }}</Button>
        </DialogFooter>
      </form>
    </DialogContent>
  </Dialog>
  <Toaster />
  </SidebarProvider>
</template> 