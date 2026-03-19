<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

const budget = ref(5000)
const recommendations = ref([])
const loading = ref(false)
const error = ref(null)
const submitting = ref(false)
const lastSubmittedOrder = ref(null)

// Included items fit within budget (marked by backend)
const includedItems = computed(() => recommendations.value.filter(r => r.included))
const excludedItems = computed(() => recommendations.value.filter(r => !r.included))

// Sum of included items' total_cost
const totalCost = computed(() =>
  includedItems.value.reduce((sum, r) => sum + r.total_cost, 0)
)

const loadRecommendations = async () => {
  loading.value = true
  error.value = null
  try {
    recommendations.value = await api.getRestockingRecommendations(budget.value)
  } catch (err) {
    error.value = 'Failed to load recommendations'
    console.error(err)
  } finally {
    loading.value = false
  }
}

// Manual debounce — re-fetch 300ms after the slider stops moving
let debounceTimer = null
watch(budget, () => {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    loadRecommendations()
  }, 300)
})

const placeOrder = async () => {
  submitting.value = true
  error.value = null
  try {
    const items = includedItems.value.map(r => ({
      item_sku: r.item_sku,
      item_name: r.item_name,
      quantity: r.recommended_quantity,
      unit_cost: r.unit_cost,
      total_cost: r.total_cost
    }))
    const result = await api.createRestockingOrder(items)
    lastSubmittedOrder.value = result
  } catch (err) {
    error.value = 'Failed to place order'
    console.error(err)
  } finally {
    submitting.value = false
  }
}

const resetOrder = () => {
  lastSubmittedOrder.value = null
  loadRecommendations()
}

const formatCurrency = (value) =>
  '$' + value.toLocaleString('en-US', { minimumFractionDigits: 0 })

const formatDate = (dateString) => {
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
}

onMounted(() => loadRecommendations())
</script>

<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set a budget to see which items can be restocked based on demand forecasts.</p>
    </div>

    <!-- Budget control card -->
    <div class="card">
      <div class="budget-label">
        <span class="budget-text">Budget</span>
        <span class="budget-value">{{ formatCurrency(budget) }}</span>
      </div>
      <input
        type="range"
        min="0"
        max="15000"
        step="500"
        v-model.number="budget"
        class="budget-slider"
      />
      <p class="budget-summary">
        {{ includedItems.length }} item{{ includedItems.length !== 1 ? 's' : '' }} recommended
        &mdash; Total: {{ formatCurrency(totalCost) }}
      </p>
    </div>

    <!-- Loading / error states -->
    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Recommendations table -->
    <div v-else class="card">
      <div class="card-header">
        <h3 class="card-title">Recommendations</h3>
      </div>
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th>SKU</th>
              <th>Item</th>
              <th>Trend</th>
              <th>Demand Gap</th>
              <th>Unit Cost</th>
              <th>Qty</th>
              <th>Total Cost</th>
            </tr>
          </thead>
          <tbody>
            <!-- Included rows — green left border accent -->
            <tr
              v-for="item in includedItems"
              :key="item.item_sku"
              class="row-included"
            >
              <td>{{ item.item_sku }}</td>
              <td>{{ item.item_name }}</td>
              <td>
                <span :class="['badge', item.trend.toLowerCase()]">{{ item.trend }}</span>
              </td>
              <td>{{ item.demand_gap }}</td>
              <td>{{ formatCurrency(item.unit_cost) }}</td>
              <td>{{ item.recommended_quantity }}</td>
              <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
            </tr>

            <!-- Divider row when there are excluded items -->
            <tr v-if="excludedItems.length > 0" class="row-divider">
              <td colspan="7" class="divider-cell">Over Budget — Excluded</td>
            </tr>

            <!-- Excluded rows — muted -->
            <tr
              v-for="item in excludedItems"
              :key="item.item_sku"
              class="row-excluded"
            >
              <td>{{ item.item_sku }}</td>
              <td>{{ item.item_name }}</td>
              <td>
                <span :class="['badge', item.trend.toLowerCase()]">{{ item.trend }}</span>
              </td>
              <td>{{ item.demand_gap }}</td>
              <td>{{ formatCurrency(item.unit_cost) }}</td>
              <td>{{ item.recommended_quantity }}</td>
              <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
            </tr>

            <!-- Empty state -->
            <tr v-if="recommendations.length === 0">
              <td colspan="7" class="empty-cell">No recommendations available.</td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Place Order button -->
      <div class="order-action">
        <button
          class="btn-place-order"
          :disabled="includedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing...' : 'Place Order' }}
        </button>
      </div>
    </div>

    <!-- Order confirmation card -->
    <div v-if="lastSubmittedOrder" class="card confirmation-card">
      <div class="card-header">
        <h3 class="card-title">Order Submitted</h3>
      </div>
      <div class="confirmation-details">
        <div class="detail-row">
          <span class="detail-label">Order Number</span>
          <span class="detail-value"><strong>{{ lastSubmittedOrder.order_number }}</strong></span>
        </div>
        <div class="detail-row">
          <span class="detail-label">Submitted</span>
          <span class="detail-value">{{ formatDate(lastSubmittedOrder.submission_date) }}</span>
        </div>
        <div class="detail-row">
          <span class="detail-label">ETA</span>
          <span class="detail-value">{{ formatDate(lastSubmittedOrder.eta) }}</span>
        </div>
        <div class="detail-row">
          <span class="detail-label">Total Cost</span>
          <span class="detail-value"><strong>{{ formatCurrency(lastSubmittedOrder.total_cost) }}</strong></span>
        </div>
      </div>
      <div class="order-action">
        <button class="btn-place-order" @click="resetOrder">
          Place Another Order
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget card internals */
.budget-label {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.75rem;
}

.budget-text {
  font-weight: 600;
  color: #475569;
  font-size: 0.875rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  margin-bottom: 0.75rem;
  cursor: pointer;
}

.budget-summary {
  color: #64748b;
  font-size: 0.875rem;
}

/* Table row variants */
.row-included {
  box-shadow: inset 3px 0 0 #22c55e;
}

.row-excluded td {
  color: #94a3b8;
}

.row-excluded .badge {
  opacity: 0.6;
}

.row-divider {
  background: #f8fafc;
}

.divider-cell {
  color: #94a3b8;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  padding: 0.5rem 0.75rem;
  border-top: 2px dashed #e2e8f0;
}

.empty-cell {
  color: #94a3b8;
  text-align: center;
  padding: 2rem;
}

/* Place order button */
.order-action {
  margin-top: 1rem;
  display: flex;
  justify-content: flex-end;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Confirmation card */
.confirmation-card {
  border-color: #22c55e;
  background: #f0fdf4;
}

.confirmation-details {
  display: flex;
  flex-direction: column;
  gap: 0.625rem;
  margin-bottom: 0.5rem;
}

.detail-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  font-size: 0.875rem;
}

.detail-label {
  color: #64748b;
  font-weight: 500;
}

.detail-value {
  color: #0f172a;
}
</style>
