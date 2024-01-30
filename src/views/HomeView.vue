<template>
  <div id="app">
    <div class="lg:w-8/12 w-11/12 flex flex-col mx-auto">
      <div class="add-todo flex space-x-3 mb-5">
        <input
          v-model="newTodoText"
          @keyup.enter="addToDoItem"
          placeholder="Add a new to-do item"
          type="text"
          id="first_name"
          class="inputcustom flex-1 border rounded-md block w-full p-2.5 bg-gray-700 border-gray-600 placeholder-[#ffffff35] ring-transparent text-white hover:bg-gray-600 text-base tracking-normal"
          required
        />

        <input
          v-model="newTodoTimeToCompletion"
          type="time"
          @keyup.enter="addToDoItem"
          class="border rounded-md block p-2.5 bg-gray-700 border-gray-600 placeholder-[#ffffff35] ring-transparent text-white hover:bg-gray-600 text-base tracking-normal"
        />

        <button
          @click="addToDoItem"
          class="flex-0 w-32 border text-base rounded-md block p-2.5 bg-[#2C5364] border-[#356275] text-white focus:ring-[#3d7188] focus:border-[#3d7188] hover:bg-[#213f4c] hover:border-[#2a5162]"
        >
          Add Item
        </button>
      </div>
      <div class="list w-full flex flex-col space-y-4 mx-auto">
        <div
          v-for="(todo, index) in todoEntries"
          :key="todo.id"
          class="item bg-gradient-to-br from-[#4c7886] via-[#3b7586] to-[#3d6673] text-white px-4 py-2 rounded-lg flex justify-between items-center"
        >
          <div v-if="!todo.editing && !todo.timeEditing" @click="startEditing(todo)">
            {{ todo.text }}
          </div>
          <input
            v-else
            :ref="(el) => (editItemRefs[index] = el as HTMLInputElement | null)"
            v-model="todo.text"
            @blur="stopEditing(todo)"
            @keyup.enter="stopEditing(todo)"
            type="text"
            class="bg-[#ffffff5d] text-white cursor-text border-none focus:outline-none p-1 rounded-md"
          />
          <div className="flex items-center space-x-4">
            <div v-if="!todo.editing && !todo.timeEditing" @click="startEditingTime(todo)">
              {{ todo.timeToCompletion ? 'Due at ' + todo.timeToCompletion : 'Set due time' }}
            </div>
            <input
              v-else
              v-model="todo.timeToCompletion"
              type="time"
              class="bg-[#ffffff5d] flex-1 text-white cursor-text border-none focus:outline-none p-1 rounded-md"
              @blur="stopEditingTime(todo)"
              @keyup.enter="stopEditingTime(todo)"
            />
            <span
              @click="deleteToDoItem(todo.id)"
              class="flex-0 rounded-full border border-[#4b8fb4] hover:border-[#5499be] hover:text-[#ffffff] uppercase bg-gradient-to-tr from-[#182d38] to-[#05324b] hover:from-[#234a55] hover:to-[#0f333e] cursor-pointer text-xs text-[#ffffff] px-2 py-2 tracking-tighter"
            >
              <TrashIcon class="icon" />
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script lang="ts">
import { v4 as uuidv4 } from 'uuid'
import { TrashIcon } from '@heroicons/vue/24/solid'
import { createClient } from '@supabase/supabase-js'
import { toast } from 'vue3-toastify'
import 'vue3-toastify/dist/index.css'

interface ToDoItem {
  id: string
  text: string
  user_id: string
  editing: boolean
  timeToCompletion?: string
  timeEditing?: boolean // Add this property for time editing
}

const supabaseUrl = import.meta.env.VITE_URL_SUPABASE
const supabaseKey = import.meta.env.VITE_APIKEY_SUPABASE
const supabase = createClient(supabaseUrl, supabaseKey)

export default {
  name: 'App',
  data() {
    return {
      newTodoText: '',
      newTodoTimeToCompletion: '', // Add this line for time input
      todoEntries: [] as ToDoItem[],
      editItemRefs: [] as (HTMLInputElement | null)[],
      user_id: 'a0841002-a7e7-4dbc-a090-1784dd80b6cc'
    }
  },
  components: { TrashIcon },
  methods: {
    async addToDoItem() {
      const trimmedText = this.newTodoText.trim()

      if (trimmedText !== '') {
        const newTodoId = uuidv4()
        const time_to_completion = this.newTodoTimeToCompletion.trim() // Get the time value

        try {
          const { error: insertError } = await supabase.from('todo_items').insert([
            {
              id: newTodoId,
              text: trimmedText,
              user_id: this.user_id,
              time_to_completion: time_to_completion
            }
          ])

          if (insertError) {
            console.error('Error adding to-do item:', insertError)
          } else {
            // Perform a select query to get the newly added to-do item
            const { data: insertedData, error: selectError } = await supabase
              .from('todo_items')
              .select('*')
              .eq('id', newTodoId)

            if (selectError) {
              console.error('Error fetching newly added to-do item:', selectError)
            } else if (Array.isArray(insertedData) && insertedData.length > 0) {
              this.todoEntries.unshift({
                id: newTodoId,
                text: trimmedText,
                user_id: this.user_id,
                timeToCompletion: time_to_completion,
                editing: false,
                timeEditing: false
              })

              this.newTodoText = ''
              this.newTodoTimeToCompletion = '' // Clear the time input
            } else {
              console.error('No data received after adding to-do item.')
            }
          }
        } catch (error) {
          console.error('Error adding to-do item:', error)
        }
      }
    },
    async deleteToDoItem(id: string) {
      try {
        const { error } = await supabase.from('todo_items').delete().eq('id', id)

        if (error) {
          console.error('Error deleting to-do item:', error)
        } else {
          this.todoEntries = this.todoEntries.filter((todo) => todo.id !== id)
        }
      } catch (error) {
        console.error('Error deleting to-do item:', error)
      }
    },
    startEditing(todo: ToDoItem) {
      todo.editing = true

      this.$nextTick(() => {
        const index = this.todoEntries.indexOf(todo)
        const inputElement = this.editItemRefs[index]

        if (inputElement instanceof HTMLInputElement) {
          inputElement.focus()
        }
      })
    },
    async stopEditing(todo: ToDoItem) {
      const trimmedText = todo.text.trim()

      // Check if the trimmed text is empty, and revert if it is
      if (trimmedText === '') {
        toast('Text cannot be empty or contain only white space.', {
          theme: 'dark',
          type: 'warning',
          transition: 'flip',
          dangerouslyHTMLString: true
        })
        return
      } else {
        try {
          const { error } = await supabase
            .from('todo_items')
            .update({ text: trimmedText })
            .eq('id', todo.id)

          if (error) {
            console.error('Error updating to-do item:', error)
          } else {
            // Fetch the updated record
            const { data: updatedData, error: fetchError } = await supabase
              .from('todo_items')
              .select('*')
              .eq('id', todo.id)

            if (fetchError) {
              console.error('Error fetching updated to-do item:', fetchError)
            } else if (Array.isArray(updatedData) && updatedData.length > 0) {
              // Update the todo.text with the fetched data
              todo.text = updatedData[0].text
              todo.editing = false
            } else {
              console.error('No data received after updating to-do item.')
            }
          }
        } catch (error) {
          console.error('Error updating to-do item:', error)
        }
      }
    },
    startEditingTime(todo: ToDoItem) {
      todo.timeEditing = true

      this.$nextTick(() => {
        const index = this.todoEntries.indexOf(todo)
        const inputElement = this.editItemRefs[index]

        if (inputElement instanceof HTMLInputElement) {
          inputElement.focus()
        }
      })
    },
    async stopEditingTime(todo: ToDoItem) {
      const trimmedTime = todo.timeToCompletion ? todo.timeToCompletion.trim() : ''

      // Check if the trimmed time is empty, and revert if it is
      if (trimmedTime === '') {
        toast('Time cannot be empty or contain only white space.', {
          theme: 'dark',
          type: 'warning',
          transition: 'flip',
          dangerouslyHTMLString: true
        })
        return
      } else {
        todo.timeEditing = false

        try {
          const { error } = await supabase
            .from('todo_items')
            .update({ time_to_completion: trimmedTime })
            .eq('id', todo.id)

          if (error) {
            console.error('Error updating to-do item time:', error)
          } else {
            // Fetch the updated record
            const { data: updatedData, error: fetchError } = await supabase
              .from('todo_items')
              .select('*')
              .eq('id', todo.id)

            if (fetchError) {
              console.error('Error fetching updated to-do item:', fetchError)
            } else if (Array.isArray(updatedData) && updatedData.length > 0) {
              // Update the todo.timeToCompletion with the fetched data
              todo.timeToCompletion = updatedData[0].time_to_completion
            } else {
              console.error('No data received after updating to-do item time.')
            }
          }
        } catch (error) {
          console.error('Error updating to-do item time:', error)
        }
      }
    },

    async fetchData() {
      try {
        const { data, error } = await supabase
          .from('todo_items')
          .select('*')
          .eq('user_id', this.user_id)

        if (error) {
          console.error('Error fetching to-do items:', error)
        } else {
          // Initialize the timeToCompletion property for each item
          this.todoEntries = data.map((item: any) => ({
            ...item,
            timeToCompletion: item.time_to_completion || '', // Use an empty string if time_to_completion is null
            editing: false,
            timeEditing: false
          }))
        }
      } catch (error) {
        console.error('Error fetching to-do items:', error)
      }
    }
  },
  created() {
    this.fetchData()
  }
}
</script>

<style>
.icon {
  width: 15px;
  height: 15px;
  color: #ffffff;
}
</style>
