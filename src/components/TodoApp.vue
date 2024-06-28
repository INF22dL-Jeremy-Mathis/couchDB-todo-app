<template>
  <div>
    <h1>Todo List</h1>
    <input type="text" v-model="newTask" placeholder="New Task" />
    <button @click="addTask">Add Task</button>
    <ul>
      <li v-for="task in tasks" :key="task._id">
        <input type="text" v-model="task.title" @change="updateTask(task._id, task.title)" />
        <button @click="deleteTask(task._id)">Delete</button>
      </li>
    </ul>
  </div>
</template>

<script>
/* eslint-disable */
import axios from 'axios';

// Set the base URL for Axios
axios.defaults.baseURL = 'http://localhost:5984/';
axios.defaults.headers.post['Content-Type'] = 'application/json';

export default {
  data() {
    return {
      tasks: [],
      newTask: ''
    };
  },
  created() {
    this.initializeDatabase();
  },
  methods: {
    initializeDatabase() {
      axios.put('todos', {}, {
        auth: {
          username: 'admin',
          password: 'admin'
        }
      })
        .then(() => {
          console.log('Database created or already exists.');
          this.fetchTasks();
        })
        .catch(error => {
          if (error.response && error.response.status === 412) {
            // Database already exists
            console.log('Database already exists.');
            this.fetchTasks();
          } else {
            console.error('Error initializing database:', error.response ? error.response.data : error.message);
          }
        });
    },
    fetchTasks() {
      axios.get('todos/_all_docs?include_docs=true', {
        auth: {
          username: 'admin',
          password: 'admin'
        }
      })
        .then(response => {
          this.tasks = response.data.rows.map(row => row.doc);
        })
        .catch(error => {
          console.error('Error fetching tasks:', error);
        });
    },
    addTask() {
      const newTask = { title: this.newTask };

      // Step 1: Get a UUID from CouchDB
      axios.get('_uuids', {
        auth: {
          username: 'admin',
          password: 'admin'
        }
      })
        .then(response => {
          const uuid = response.data.uuids[0];

          // Step 2: Insert the new task using the UUID
          axios.put(`todos/${uuid}`, newTask, {
            headers: {
              'Content-Type': 'application/json'
            },
            auth: {
              username: 'admin',
              password: 'admin'
            }
          })
            .then(response => {
              newTask._id = response.data.id;
              newTask._rev = response.data.rev;
              this.tasks.push(newTask);
              this.newTask = '';
            })
            .catch(error => {
              console.error('Error adding task:', error.response ? error.response.data : error.message);
            });
        })
        .catch(error => {
          console.error('Error generating UUID:', error.response ? error.response.data : error.message);
        });
    },
    updateTask(id, title) {
      const task = this.tasks.find(task => task._id === id);
      if (task) {
        task.title = title;
        axios.put(`todos/${id}`, task, {
          headers: {
            'Content-Type': 'application/json'
          },
          auth: {
            username: 'admin',
            password: 'admin'
          }
        })
          .then(response => {
            task._rev = response.data.rev;
          })
          .catch(error => {
            console.error('Error updating task:', error.response ? error.response.data : error.message);
          });
      }
    },
    deleteTask(id) {
      const task = this.tasks.find(task => task._id === id);
      if (task) {
        axios.delete(`todos/${id}?rev=${task._rev}`, {
          auth: {
            username: 'admin',
            password: 'admin'
          }
        })
          .then(response => {
            this.tasks = this.tasks.filter(task => task._id !== id);
          })
          .catch(error => {
            console.error('Error deleting task:', error.response ? error.response.data : error.message);
          });
      }
    }
  }
};
</script>

<style scoped>
/* Add any styles you need here */

/**/
</style>
