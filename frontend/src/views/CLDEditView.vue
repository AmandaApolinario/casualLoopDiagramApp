<template>
  <div class="cld-edit-container">
    <NavBar />
    <div class="cld-edit-content">
      <h1>Edit CLD</h1>
      
      <div v-if="loading" class="loading-message">Loading CLD...</div>
      <div v-else-if="error" class="error-message">{{ error }}</div>
      <form v-else @submit.prevent="handleSubmit" class="cld-form">
        <!-- Basic CLD Info -->
        <div class="form-section card">
          <h2 class="section-title">Basic Information</h2>
          <div class="form-grid">
            <div class="form-group">
              <label for="name">CLD Name</label>
              <input
                type="text"
                id="name"
                v-model="diagram.title"
                required
                placeholder="Enter CLD name"
                class="form-input"
              />
            </div>

            <div class="form-group">
              <label for="date">Date</label>
              <input
                type="date"
                id="date"
                v-model="diagram.createdAt"
                required
                class="form-input"
                @focus="ensureDateFormat"
              />
            </div>
          </div>

          <div class="form-group">
            <label for="description">Description</label>
            <textarea
              id="description"
              v-model="diagram.description"
              placeholder="Enter CLD description"
              rows="4"
              class="form-textarea"
            ></textarea>
          </div>
        </div>

        <!-- Relationships Section -->
        <div class="form-section card">
          <h2 class="section-title">Relationships</h2>
          
          <div class="relationships-container">
            <div v-for="(edge, index) in diagram.edges" :key="index" class="relationship-card">
              <div class="relationship-controls">
                <select v-model="edge.source" 
                       @change="edge.target && checkForConflictingRelationship(index)"
                       required 
                       class="form-select">
                  <option value="" disabled>Select source</option>
                  <option v-for="variable in variables" :key="variable.id" :value="variable.id">
                    {{ variable.name }}
                  </option>
                </select>
                
                <select v-model="edge.polarity" 
                       @change="edge.source && edge.target && checkForConflictingRelationship(index)"
                       required 
                       class="form-select type-select">
                  <option value="positive">+ (Positive)</option>
                  <option value="negative">- (Negative)</option>
                </select>
                
                <select v-model="edge.target" 
                       @change="checkForConflictingRelationship(index)"
                       required 
                       class="form-select">
                  <option value="" disabled>Select target</option>
                  <option v-for="variable in filteredTargetVariables(edge.source)" :key="variable.id" :value="variable.id">
                    {{ variable.name }}
                  </option>
                </select>
              </div>
              <button 
                type="button" 
                @click="() => removeEdge(index)" 
                class="btn-icon"
                aria-label="Remove relationship"
              >
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" viewBox="0 0 16 16">
                  <path d="M4.646 4.646a.5.5 0 0 1 .708 0L8 7.293l2.646-2.647a.5.5 0 0 1 .708.708L8.707 8l2.647 2.646a.5.5 0 0 1-.708.708L8 8.707l-2.646 2.647a.5.5 0 0 1-.708-.708L7.293 8 4.646 5.354a.5.5 0 0 1 0-.708z"/>
                </svg>
              </button>
            </div>
            
            <button 
              type="button" 
              @click="addEdge" 
              class="btn-add-relationship"
            >
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" viewBox="0 0 16 16" class="icon">
                <path d="M8 4a.5.5 0 0 1 .5.5v3h3a.5.5 0 0 1 0 1h-3v3a.5.5 0 0 1-1 0v-3h-3a.5.5 0 0 1 0-1h3v-3A.5.5 0 0 1 8 4z"/>
              </svg>
              Add Relationship
            </button>
          </div>
        </div>

        <!-- Form Actions -->
        <div class="form-actions">
          <button 
            type="button" 
            @click="cancelEdit" 
            class="btn-secondary"
          >
            Cancel
          </button>
          <button 
            type="submit" 
            :disabled="saving" 
            class="btn-primary"
          >
            <span v-if="saving">
              <svg class="spinner" viewBox="0 0 50 50">
                <circle class="path" cx="25" cy="25" r="20" fill="none" stroke-width="5"></circle>
              </svg>
              Saving...
            </span>
            <span v-else>Save Changes</span>
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup>
import { onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import NavBar from '../components/NavBar.vue'
import { useCLDEditorViewModel } from '@/viewmodels/CLDEditorViewModel'

// Initialize route and router for navigation
const route = useRoute()
const router = useRouter()

// Initialize the CLD Editor ViewModel
const { 
  diagram,
  variables,
  loading,
  saving,
  error,
  successMessage,
  fetchDiagram,
  updateDiagram,
  addEdge,
  removeEdge,
  filteredTargetVariables,
  validateDiagram
} = useCLDEditorViewModel()

// Make sure date is in the proper format for date input (YYYY-MM-DD)
const ensureDateFormat = () => {
  if (!diagram.value || !diagram.value.createdAt) {
    // If date is missing, set it to today
    diagram.value.createdAt = new Date().toISOString().split('T')[0];
    return;
  }
  
  try {
    // Check if it's already in the right format (YYYY-MM-DD)
    if (!/^\d{4}-\d{2}-\d{2}$/.test(diagram.value.createdAt)) {
      // Try to convert it to a date object and then format it
      const date = new Date(diagram.value.createdAt);
      diagram.value.createdAt = date.toISOString().split('T')[0];
    }
  } catch (err) {
    console.error('Error formatting date:', err);
    // Fallback to today's date
    diagram.value.createdAt = new Date().toISOString().split('T')[0];
  }
}

// Load the diagram data on component mount
onMounted(async () => {
  await fetchDiagram(route.params.id)
  
  // Ensure date is properly formatted after loading
  ensureDateFormat();
})

// Handle form submission
const handleSubmit = async () => {
  // Validate the diagram data
  const validationError = validateDiagram()
  if (validationError) {
    error.value = validationError
    return
  }
  
  // Update the diagram
  const updatedDiagram = await updateDiagram(route.params.id, diagram.value)
  if (updatedDiagram) {
    // Redirect back to diagram detail after successful update
    setTimeout(() => {
      router.push(`/cld/${route.params.id}`)
    }, 1000)
  }
}

// Cancel edit and go back
const cancelEdit = () => {
  router.go(-1)
}

// Add the checkForConflictingRelationship method
const checkForConflictingRelationship = (index) => {
  const currentEdge = diagram.value.edges[index];
  
  // Skip if the edge is not fully defined
  if (!currentEdge.source || !currentEdge.target || !currentEdge.polarity) {
    return;
  }
  
  // Check if there's already a relationship between these variables with a different polarity
  for (let i = 0; i < diagram.value.edges.length; i++) {
    if (i === index) continue; // Skip the current edge
    
    const otherEdge = diagram.value.edges[i];
    
    // Skip incomplete edges
    if (!otherEdge.source || !otherEdge.target || !otherEdge.polarity) {
      continue;
    }
    
    // Check if exactly the same variables are connected in the same direction
    const sameDirectionalRelationship = 
      (currentEdge.source === otherEdge.source && currentEdge.target === otherEdge.target);
    
    if (sameDirectionalRelationship && currentEdge.polarity !== otherEdge.polarity) {
      error.value = `Conflicting relationship: Cannot have both positive and negative relationships between the same variables in the same direction`;
      
      // Reset the last selection 
      currentEdge.target = '';
      
      // Clear the error after 5 seconds
      setTimeout(() => {
        error.value = null;
      }, 5000);
      
      break;
    }
  }
}
</script>

<style scoped>
.cld-edit-container {
  min-height: 100vh;
  background-color: #f8fafc;
  padding-bottom: 2rem;
}

.cld-edit-content {
  max-width: 1000px;
  margin: 0 auto;
  padding: 1.5rem;
}

h1 {
  color: #1e293b;
  font-size: 1.8rem;
  font-weight: 600;
  margin-bottom: 2rem;
}

.card {
  background-color: white;
  border-radius: 0.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #1e293b;
  margin-bottom: 1rem;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.selection-count {
  font-size: 0.875rem;
  color: #64748b;
  background-color: #f1f5f9;
  padding: 0.25rem 0.5rem;
  border-radius: 0.25rem;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
  margin-bottom: 1rem;
}

.form-group {
  margin-bottom: 1rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-size: 0.875rem;
  font-weight: 500;
  color: #334155;
}

.form-input, .form-select, .form-textarea {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  transition: border-color 0.2s;
  background-color: #fff;
}

.form-input:focus, .form-select:focus, .form-textarea:focus {
  outline: none;
  border-color: #6366f1;
  box-shadow: 0 0 0 2px rgba(99, 102, 241, 0.1);
}

.form-textarea {
  min-height: 120px;
  resize: vertical;
}

/* Variables Section */
.variables-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 0.75rem;
}

.variable-card {
  display: flex;
  align-items: center;
  padding: 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.375rem;
  transition: all 0.2s;
}

.variable-card:hover {
  border-color: #cbd5e1;
  background-color: #f8fafc;
}

.variable-checkbox {
  margin-right: 0.75rem;
  width: 1rem;
  height: 1rem;
  accent-color: #6366f1;
}

.variable-label {
  display: flex;
  flex-direction: column;
  cursor: pointer;
}

.variable-name {
  font-weight: 500;
  color: #1e293b;
  margin-bottom: 0.25rem;
}

.variable-description {
  font-size: 0.75rem;
  color: #64748b;
  line-height: 1.4;
}

/* Relationships Section */
.relationships-container {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.relationship-card {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.relationship-controls {
  display: flex;
  flex-grow: 1;
  gap: 0.75rem;
}

.type-select {
  max-width: 120px;
}

.btn-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 2rem;
  height: 2rem;
  border-radius: 0.375rem;
  background-color: #fee2e2;
  color: #dc2626;
  border: none;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-icon:hover {
  background-color: #fecaca;
}

.btn-add-relationship {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background-color: #e0e7ff;
  color: #4f46e5;
  border: none;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 0.5rem;
}

.btn-add-relationship:hover {
  background-color: #c7d2fe;
}

.icon {
  width: 1rem;
  height: 1rem;
}

/* Form Actions */
.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  margin-top: 1.5rem;
}

.btn-primary, .btn-secondary {
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.btn-primary {
  background-color: #4f46e5;
  color: white;
  border: none;
}

.btn-primary:hover {
  background-color: #4338ca;
}

.btn-primary:disabled {
  background-color: #c7d2fe;
  cursor: not-allowed;
}

.btn-secondary {
  background-color: white;
  color: #64748b;
  border: 1px solid #e2e8f0;
}

.btn-secondary:hover {
  background-color: #f1f5f9;
}

/* Loading and Error States */
.loading-message, .error-message {
  padding: 2rem;
  text-align: center;
  border-radius: 0.5rem;
}

.loading-message {
  color: #334155;
  background-color: #f1f5f9;
}

.error-message {
  color: #dc2626;
  background-color: #fee2e2;
}

/* Spinner */
.spinner {
  animation: rotate 2s linear infinite;
  width: 1rem;
  height: 1rem;
}

.spinner .path {
  stroke: #ffffff;
  stroke-linecap: round;
  animation: dash 1.5s ease-in-out infinite;
}

@keyframes rotate {
  100% {
    transform: rotate(360deg);
  }
}

@keyframes dash {
  0% {
    stroke-dasharray: 1, 150;
    stroke-dashoffset: 0;
  }
  50% {
    stroke-dasharray: 90, 150;
    stroke-dashoffset: -35;
  }
  100% {
    stroke-dasharray: 90, 150;
    stroke-dashoffset: -124;
  }
}
</style>