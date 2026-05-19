<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Smart inventory replenishment based on stock levels and demand trends</p>
    </div>

    <!-- Budget Section -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-section">
        <label class="budget-label" for="budget-input">Max Budget ($)</label>
        <input
          id="budget-input"
          type="number"
          v-model.number="budget"
          min="0"
          step="500"
          class="budget-input"
        />
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="500000"
          step="500"
          class="budget-slider"
        />
        <div class="budget-display">Budget: ${{ budget.toLocaleString() }} selected</div>
      </div>
    </div>

    <!-- Recommendations Table -->
    <div class="card">
      <div class="card-header">
        <div class="card-header-text">
          <h3 class="card-title">Recommended Items</h3>
          <p class="card-subtitle">Ranked by combined low-stock urgency and demand trend score</p>
        </div>
        <div class="selected-total">
          Selected Total: <strong>${{ selectedTotal.toLocaleString() }}</strong>
        </div>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>Item Name</th>
              <th>SKU</th>
              <th>Warehouse</th>
              <th>Category</th>
              <th>Stock (on hand / reorder pt)</th>
              <th>Trend</th>
              <th>Qty to Order</th>
              <th>Unit Cost</th>
              <th>Line Total</th>
              <th>Score</th>
              <th>Select</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendations" :key="item.sku">
              <td>{{ item.name }}</td>
              <td class="sku-cell">{{ item.sku }}</td>
              <td>{{ item.warehouse }}</td>
              <td>{{ item.category }}</td>
              <td class="stock-cell">
                {{ item.stock_on_hand }}
                <span class="stock-divider">/</span>
                <span class="reorder-pt">{{ item.reorder_point }}</span>
              </td>
              <td>
                <span :class="['badge', item.trend]">{{ item.trend }}</span>
              </td>
              <td>{{ item.qty_to_order }}</td>
              <td>${{ (item.unit_cost || 0).toLocaleString() }}</td>
              <td>${{ (item.line_total || 0).toLocaleString() }}</td>
              <td>{{ Math.round((item.score || 0) * 100) }}%</td>
              <td>
                <input
                  type="checkbox"
                  :checked="checkedSkus.has(item.sku)"
                  @change="toggleSku(item.sku)"
                  class="row-checkbox"
                />
              </td>
            </tr>
            <tr v-if="recommendations.length === 0">
              <td colspan="11" class="empty-state">No restocking recommendations at this time.</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Delivery Date + Place Order -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Place Order</h3>
      </div>

      <div v-if="successMessage" class="success-banner">{{ successMessage }}</div>
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <div class="order-form">
        <div class="form-group">
          <label for="delivery-date">Expected Delivery Date</label>
          <input
            id="delivery-date"
            type="date"
            v-model="expectedDelivery"
            :min="minDeliveryDate"
            class="date-input"
          />
        </div>
        <button
          class="btn-primary"
          :disabled="!canSubmit"
          @click="submitOrder"
        >
          {{ submitting ? 'Submitting...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)

    // Budget — single value bound to both the number input and the range slider
    const budget = ref(50000)

    // Checked SKUs (Set; always reassign to trigger reactivity)
    const checkedSkus = ref(new Set())

    // Delivery date — default to today + 7 days
    const computeMinDate = () => {
      const d = new Date()
      d.setDate(d.getDate() + 7)
      return d.toISOString().split('T')[0]
    }
    const minDeliveryDate = computeMinDate()
    const expectedDelivery = ref(minDeliveryDate)

    // Submit state
    const submitting = ref(false)
    const successMessage = ref('')
    const submitError = ref('')

    // Greedy algorithm: check highest-score rows until budget is exhausted
    // Recommendations are already sorted by score (highest first) from backend
    const computeDefaultChecked = () => {
      const checked = new Set()
      let remaining = budget.value
      for (const item of recommendations.value) {
        const lineTotal = item.line_total || 0
        if (lineTotal <= remaining) {
          checked.add(item.sku)
          remaining -= lineTotal
        }
      }
      checkedSkus.value = checked
    }

    // Recompute default selection whenever budget changes
    watch(budget, computeDefaultChecked)

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations()
        // Set initial default checked state once data is loaded
        computeDefaultChecked()
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const toggleSku = (sku) => {
      // Rebuild the Set so Vue detects the change
      const next = new Set(checkedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      checkedSkus.value = next
    }

    const selectedTotal = computed(() => {
      return recommendations.value
        .filter(item => checkedSkus.value.has(item.sku))
        .reduce((sum, item) => sum + (item.line_total || 0), 0)
    })

    const selectedItems = computed(() => {
      return recommendations.value.filter(item => checkedSkus.value.has(item.sku))
    })

    const canSubmit = computed(() => {
      return checkedSkus.value.size > 0 && expectedDelivery.value !== '' && !submitting.value
    })

    const submitOrder = async () => {
      if (!canSubmit.value) return
      submitting.value = true
      successMessage.value = ''
      submitError.value = ''
      try {
        const payload = {
          items: selectedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.qty_to_order,
            unit_cost: item.unit_cost,
            line_total: item.line_total
          })),
          total_value: selectedTotal.value,
          expected_delivery: expectedDelivery.value
        }
        const result = await api.submitRestockingOrder(payload)
        successMessage.value = `Order ${result.order_number} submitted successfully.`
        // Clear selections; keep table visible
        checkedSkus.value = new Set()
      } catch (err) {
        submitError.value =
          'Failed to submit order: ' +
          (err.response?.data?.detail || err.message || 'Unknown error')
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      recommendations,
      loading,
      error,
      budget,
      minDeliveryDate,
      expectedDelivery,
      submitting,
      successMessage,
      submitError,
      checkedSkus,
      selectedTotal,
      canSubmit,
      toggleSku,
      submitOrder
    }
  }
}
</script>

<style scoped>
/* Budget section */
.budget-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  max-width: 480px;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

.budget-input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
  background: #f8fafc;
  outline: none;
  transition: border-color 0.15s;
}

.budget-input:focus {
  border-color: #2563eb;
  background: white;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-display {
  font-size: 0.938rem;
  font-weight: 600;
  color: #2563eb;
}

/* Card header with subtitle */
.card-header-text {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.card-subtitle {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 400;
}

/* Selected total in header */
.selected-total {
  font-size: 0.938rem;
  color: #334155;
  white-space: nowrap;
}

.selected-total strong {
  color: #0f172a;
  font-size: 1rem;
}

/* Table cells */
.sku-cell {
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.813rem;
  color: #64748b;
}

.stock-cell {
  white-space: nowrap;
}

.stock-divider {
  margin: 0 0.25rem;
  color: #94a3b8;
}

.reorder-pt {
  color: #94a3b8;
  font-size: 0.813rem;
}

.row-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

.empty-state {
  text-align: center;
  color: #94a3b8;
  padding: 2rem 0;
  font-size: 0.938rem;
}

/* Order form */
.order-form {
  display: flex;
  align-items: flex-end;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-group label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

.date-input {
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
  background: #f8fafc;
  outline: none;
  transition: border-color 0.15s;
}

.date-input:focus {
  border-color: #2563eb;
  background: white;
}

.btn-primary {
  padding: 0.563rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
  white-space: nowrap;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
