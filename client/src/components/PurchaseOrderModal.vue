<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Create mode: form -->
            <template v-if="mode === 'create'">
              <div class="context-banner">
                <div class="context-row">
                  <span class="context-label">Item</span>
                  <span class="context-value">{{ backlogItem.item_name }}</span>
                </div>
                <div class="context-row">
                  <span class="context-label">SKU</span>
                  <span class="context-value mono">{{ backlogItem.item_sku }}</span>
                </div>
                <div class="context-row">
                  <span class="context-label">Quantity Needed</span>
                  <span class="context-value">{{ backlogItem.quantity_needed }} units</span>
                </div>
              </div>

              <form class="po-form" @submit.prevent="submitForm">
                <div class="form-group">
                  <label class="form-label" for="supplier_name">Supplier Name <span class="required">*</span></label>
                  <input
                    id="supplier_name"
                    v-model="form.supplier_name"
                    type="text"
                    class="form-input"
                    :class="{ 'input-error': errors.supplier_name }"
                    placeholder="Enter supplier name"
                    required
                  />
                  <span v-if="errors.supplier_name" class="error-text">{{ errors.supplier_name }}</span>
                </div>

                <div class="form-row">
                  <div class="form-group">
                    <label class="form-label" for="quantity">Quantity <span class="required">*</span></label>
                    <input
                      id="quantity"
                      v-model.number="form.quantity"
                      type="number"
                      class="form-input"
                      :class="{ 'input-error': errors.quantity }"
                      min="1"
                      placeholder="0"
                      required
                    />
                    <span v-if="errors.quantity" class="error-text">{{ errors.quantity }}</span>
                  </div>

                  <div class="form-group">
                    <label class="form-label" for="unit_cost">Unit Cost ($) <span class="required">*</span></label>
                    <input
                      id="unit_cost"
                      v-model.number="form.unit_cost"
                      type="number"
                      class="form-input"
                      :class="{ 'input-error': errors.unit_cost }"
                      min="0"
                      step="0.01"
                      placeholder="0.00"
                      required
                    />
                    <span v-if="errors.unit_cost" class="error-text">{{ errors.unit_cost }}</span>
                  </div>
                </div>

                <div class="form-group">
                  <label class="form-label" for="expected_delivery_date">Expected Delivery Date <span class="required">*</span></label>
                  <input
                    id="expected_delivery_date"
                    v-model="form.expected_delivery_date"
                    type="date"
                    class="form-input"
                    :class="{ 'input-error': errors.expected_delivery_date }"
                    required
                  />
                  <span v-if="errors.expected_delivery_date" class="error-text">{{ errors.expected_delivery_date }}</span>
                </div>

                <div class="form-group">
                  <label class="form-label" for="notes">Notes</label>
                  <textarea
                    id="notes"
                    v-model="form.notes"
                    class="form-textarea"
                    placeholder="Optional notes about this purchase order"
                    rows="3"
                  ></textarea>
                </div>

                <div v-if="totalCost > 0" class="total-cost-banner">
                  <span class="total-label">Estimated Total Cost</span>
                  <span class="total-value">${{ totalCost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
                </div>

                <div v-if="submitError" class="submit-error">{{ submitError }}</div>
              </form>
            </template>

            <!-- View mode: read-only PO details -->
            <template v-else-if="mode === 'view'">
              <div v-if="loadingPO" class="loading-state">
                <div class="spinner"></div>
                <span>Loading purchase order...</span>
              </div>

              <div v-else-if="fetchError" class="fetch-error">
                {{ fetchError }}
              </div>

              <div v-else-if="existingPO" class="po-details">
                <div class="po-header-row">
                  <div class="po-id-block">
                    <div class="info-label">Purchase Order ID</div>
                    <div class="info-value mono">{{ existingPO.id }}</div>
                  </div>
                  <span class="status-badge" :class="existingPO.status">{{ existingPO.status }}</span>
                </div>

                <div class="info-grid">
                  <div class="info-item">
                    <div class="info-label">Supplier</div>
                    <div class="info-value">{{ existingPO.supplier_name }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Quantity</div>
                    <div class="info-value">{{ existingPO.quantity }} units</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Unit Cost</div>
                    <div class="info-value">${{ Number(existingPO.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Total Cost</div>
                    <div class="info-value highlight">${{ (existingPO.quantity * existingPO.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Expected Delivery</div>
                    <div class="info-value">{{ formatDate(existingPO.expected_delivery_date) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Created Date</div>
                    <div class="info-value">{{ formatDate(existingPO.created_date) }}</div>
                  </div>
                </div>

                <div v-if="existingPO.notes" class="notes-section">
                  <div class="info-label">Notes</div>
                  <div class="notes-body">{{ existingPO.notes }}</div>
                </div>
              </div>
            </template>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    mode: {
      type: String,
      default: 'create',
      validator: (v) => ['create', 'view'].includes(v)
    }
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    // Form state (create mode)
    const form = ref({
      supplier_name: '',
      quantity: 0,
      unit_cost: 0,
      expected_delivery_date: '',
      notes: ''
    })
    const errors = ref({})
    const submitting = ref(false)
    const submitError = ref(null)

    // View mode state
    const existingPO = ref(null)
    const loadingPO = ref(false)
    const fetchError = ref(null)

    const totalCost = computed(() => {
      const qty = Number(form.value.quantity)
      const cost = Number(form.value.unit_cost)
      if (!qty || !cost || qty <= 0 || cost <= 0) return 0
      return qty * cost
    })

    const resetForm = () => {
      form.value = {
        supplier_name: '',
        // Pre-fill quantity from backlog item when available
        quantity: props.backlogItem?.quantity_needed ?? 0,
        unit_cost: 0,
        expected_delivery_date: '',
        notes: ''
      }
      errors.value = {}
      submitError.value = null
    }

    const fetchExistingPO = async () => {
      if (!props.backlogItem?.id) return
      loadingPO.value = true
      fetchError.value = null
      existingPO.value = null
      try {
        existingPO.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
      } catch (err) {
        fetchError.value = 'Failed to load purchase order details.'
        console.error(err)
      } finally {
        loadingPO.value = false
      }
    }

    // When modal opens, initialise state depending on mode
    watch(
      () => props.isOpen,
      (isOpen) => {
        if (!isOpen) return
        if (props.mode === 'create') {
          resetForm()
        } else if (props.mode === 'view') {
          fetchExistingPO()
        }
      }
    )

    // Also react to mode changes while open (e.g. parent swaps mode without closing)
    watch(
      () => props.mode,
      (mode) => {
        if (!props.isOpen) return
        if (mode === 'create') {
          resetForm()
        } else if (mode === 'view') {
          fetchExistingPO()
        }
      }
    )

    const validate = () => {
      const errs = {}
      if (!form.value.supplier_name.trim()) {
        errs.supplier_name = 'Supplier name is required.'
      }
      if (!form.value.quantity || form.value.quantity <= 0) {
        errs.quantity = 'Quantity must be greater than 0.'
      }
      if (!form.value.unit_cost || form.value.unit_cost <= 0) {
        errs.unit_cost = 'Unit cost must be greater than 0.'
      }
      if (!form.value.expected_delivery_date) {
        errs.expected_delivery_date = 'Expected delivery date is required.'
      }
      errors.value = errs
      return Object.keys(errs).length === 0
    }

    const submitForm = async () => {
      if (!validate()) return
      submitting.value = true
      submitError.value = null
      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name.trim(),
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes.trim() || null
        }
        const created = await api.createPurchaseOrder(payload)
        emit('po-created', created)
      } catch (err) {
        submitError.value = 'Failed to create purchase order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const close = () => {
      emit('close')
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    return {
      form,
      errors,
      submitting,
      submitError,
      existingPO,
      loadingPO,
      fetchError,
      totalCost,
      submitForm,
      close,
      formatDate
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.75rem;
}

.modal-footer {
  padding: 1.25rem 1.75rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

/* Context banner (create mode) */
.context-banner {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.context-row {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.context-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  min-width: 120px;
}

.context-value {
  font-size: 0.875rem;
  color: #0f172a;
  font-weight: 500;
}

.context-value.mono {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

/* Form styles */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #334155;
}

.required {
  color: #ef4444;
}

.form-input,
.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
}

.form-input:focus,
.form-textarea:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-input.input-error,
.form-textarea.input-error {
  border-color: #ef4444;
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.error-text {
  font-size: 0.75rem;
  color: #ef4444;
}

.total-cost-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
}

.total-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #1e40af;
}

.total-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #1d4ed8;
}

.submit-error {
  font-size: 0.875rem;
  color: #ef4444;
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
}

/* View mode: loading / error */
.loading-state {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 2rem 0;
  color: #64748b;
  font-size: 0.875rem;
}

.spinner {
  width: 20px;
  height: 20px;
  border: 2px solid #e2e8f0;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.fetch-error {
  font-size: 0.875rem;
  color: #ef4444;
  padding: 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
}

/* View mode: PO details */
.po-details {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.po-header-row {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
}

.po-id-block {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.mono {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.info-value.highlight {
  color: #0f172a;
  font-weight: 700;
}

/* Status badge */
.status-badge {
  display: inline-block;
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: capitalize;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.status-badge.pending {
  background: #fef9c3;
  color: #854d0e;
}

.status-badge.approved {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.ordered {
  background: #e0f2fe;
  color: #075985;
}

.status-badge.received {
  background: #dcfce7;
  color: #166534;
}

.status-badge.cancelled {
  background: #f1f5f9;
  color: #475569;
}

/* Notes section */
.notes-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.notes-body {
  font-size: 0.875rem;
  color: #334155;
  line-height: 1.6;
  white-space: pre-wrap;
}

/* Buttons */
.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #0f172a;
  border: 1px solid #0f172a;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1e293b;
  border-color: #1e293b;
}

.btn-primary:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

/* Modal transition */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
