<template>
  <v-container>
    <v-row>
      <v-col cols="12">
        <v-card class="pa-4 mb-4">
          <v-card-title class="text-h5">
            {{ editingPost ? 'Edit Post' : 'Create New Post' }}
          </v-card-title>
          <v-card-text>
            <v-form @submit.prevent="editingPost ? updatePost() : createPost()">
              <v-text-field
                v-model="post.title"
                label="Title"
                required
                variant="outlined"
                class="mb-4"
              ></v-text-field>
              
              <v-textarea
                v-model="post.body"
                label="Content"
                required
                variant="outlined"
                rows="4"
                class="mb-4"
              ></v-textarea>
              
              <v-btn
                type="submit"
                color="primary"
                class="mr-2"
              >
                {{ editingPost ? 'Update' : 'Create' }}
              </v-btn>
              
              <v-btn
                v-if="editingPost"
                @click="cancelEdit"
                color="grey"
              >
                Cancel
              </v-btn>
            </v-form>
          </v-card-text>
        </v-card>
        
        <v-card>
          <v-card-title class="text-h5">Posts</v-card-title>
          <v-card-text>
            <v-progress-circular
              v-if="loading"
              indeterminate
              color="primary"
            ></v-progress-circular>
            
            <v-list v-else>
              <v-list-item
                v-for="post in posts"
                :key="post.id"
                class="mb-2"
              >
                <template v-slot:prepend>
                  <v-icon icon="mdi-text"></v-icon>
                </template>
                
                <v-list-item-title>{{ post.title }}</v-list-item-title>
                <v-list-item-subtitle>{{ post.body }}</v-list-item-subtitle>
                
                <template v-slot:append>
                  <v-btn
                    @click="editPost(post)"
                    icon="mdi-pencil"
                    color="warning"
                    variant="text"
                    class="mr-2"
                  ></v-btn>
                  <v-btn
                    @click="deletePost(post.id)"
                    icon="mdi-delete"
                    color="error"
                    variant="text"
                  ></v-btn>
                </template>
              </v-list-item>
            </v-list>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
axios.defaults.withCredentials = true;
const API_URL = import.meta.env.VITE_API_URL

// const API_URL = import.meta.env.VITE_API_URL
console.log('Env vars:', import.meta.env) 
const posts = ref([])
const loading = ref(false)
const editingPost = ref(false)
const post = ref({
  id: null,
  title: '',
  body: ''
})

// Generate a temporary ID for new posts
const generateTempId = () => {
  return 'temp-' + Date.now() + '-' + Math.floor(Math.random() * 1000)
}

// Fetch all posts
const fetchPosts = async () => {
  try {
    loading.value = true
    const response = await axios.get(API_URL)
    // Filter out any temporary posts that might have been created
    posts.value = response.data.filter(p => !p.id.toString().startsWith('temp-')).slice(0, 5)
  } catch (error) {
    console.error('Error fetching posts:', error)
  } finally {
    loading.value = false
  }
}

// Create a new post
const createPost = async () => {
  try {
    // Generate temporary ID for optimistic UI update
    const tempId = generateTempId()
    const tempPost = { ...post.value, id: tempId }
    
    // Optimistically add to UI immediately
    posts.value = [tempPost, ...posts.value]
    
    // Send to server (without the temporary ID)
    const { id, ...postData } = post.value
    const response = await axios.post(API_URL, postData)
    
    // Replace temporary post with server response
    const index = posts.value.findIndex(p => p.id === tempId)
    if (index !== -1) {
      posts.value[index] = response.data
    }
    
    resetForm()
  } catch (error) {
    console.error('Error creating post:', error)
    // Remove the temporary post if creation failed
    posts.value = posts.value.filter(p => !p.id.toString().startsWith('temp-'))
  }
}

// Update an existing post
const updatePost = async () => {
  try {
    // Check if we're updating a temporary post (not yet saved to server)
    const isTempPost = post.value.id.toString().startsWith('temp-')
    
    if (isTempPost) {
      // For temporary posts, treat as a new creation
      await createPost()
      return
    }
    
    // For existing posts, perform update
    const response = await axios.put(`${API_URL}${post.value.id}/`, post.value)
    
    // Update local state with server response
    const index = posts.value.findIndex(p => p.id === post.value.id)
    if (index !== -1) {
      posts.value[index] = response.data
    }
    
    resetForm()
  } catch (error) {
    console.error('Error updating post:', error)
    // Fallback to local update if API fails
    const index = posts.value.findIndex(p => p.id === post.value.id)
    if (index !== -1) {
      posts.value[index] = { ...post.value }
    }
    resetForm()
  }
}

// Delete a post
const deletePost = async (id) => {
  try {
    // Don't try to delete temporary posts from server
    if (!id.toString().startsWith('temp-')) {
      await axios.delete(`${API_URL}${id}/`)
    }
    
    // Remove from local state in any case
    posts.value = posts.value.filter(post => post.id !== id)
  } catch (error) {
    console.error('Error deleting post:', error)
  }
}

// Edit a post
const editPost = (postToEdit) => {
  post.value = { ...postToEdit }
  editingPost.value = true
}

// Cancel edit mode
const cancelEdit = () => {
  resetForm()
}

// Reset form
const resetForm = () => {
  post.value = {
    id: null,
    title: '',
    body: ''
  }
  editingPost.value = false
}

// Fetch posts when component mounts
onMounted(() => {
  fetchPosts()
})
</script>

<style>
/* Styles here if needed */
</style>