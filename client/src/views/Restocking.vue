<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and get recommended items to restock based on inventory levels and demand trends.</p>
    </div>

    <!-- Success banner shown after order submission -->
    <div v-if="successBanner" class="success-banner">
      <div class="success-banner-content">
        <strong>Order placed successfully.</strong>
        Order <span class="order-number">{{ successBanner.order_number }}</span> submitted.
        Expected delivery: {{ successBanner.expected_delivery }}.
      </div>
      <button class="dismiss-btn" @click="successBanner = null">Dismiss</button>
    </div>

    <!-- Budget control card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
        <span class="budget-display">{{ formatCurrency(budget) }}</span>
      </div>
      <div class="slider-wrapper">
        <span class="slider-label">{{ formatCurrency(0) }}</span>
        <input
          type="range"
          class="budget-slider"
          min="0"
          max="500000"
          step="5000"
          :value="budget"
          @input="onBudgetInput"
        />
        <span class="slider-label">{{ formatCurrency(500000) }}</span>
      </div>
      <div class="slider-ticks">
        <span>$0</span>
        <span>$125K</span>
        <span>$250K</span>
        <span>$375K</span>
        <span>$500K</span>
      </div>
    </div>

    <!-- Recommendations section -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">
          Recommendations
          <span v-if="recommendations.length" class="item-count">({{ recommendations.length }} items)</span>
        </h3>
        <div v-if="selectedSkus.size > 0" class="selection-summary">
          {{ selectedSkus.size }} selected &mdash;
          <strong>{{ formatCurrency(selectedTotal) }}</strong>
          <span v-if="budget > 0" :class="['budget-status', selectedTotal > budget ? 'over-budget' : 'under-budget']">
            {{ selectedTotal > budget ? 'Over budget' : 'Within budget' }}
          </span>
        </div>
      </div>

      <!-- Loading state -->
      <div v-if="loading" class="loading">Loading recommendations...</div>

      <!-- Error state -->
      <div v-else-if="error" class="error">{{ error }}</div>

      <!-- Empty state: budget is 0 or no results -->
      <div v-else-if="recommendations.length === 0" class="empty-state">
        <div class="empty-state-icon">-</div>
        <p v-if="budget === 0" class="empty-state-text">Set a budget above to see restocking recommendations.</p>
        <p v-else class="empty-state-text">No items need restocking within the current budget.</p>
      </div>

      <!-- Recommendations table -->
      <div v-else class="table-container">
        <table class="restock-table">
          <thead>
            <tr>
              <th class="col-check">
                <input
                  type="checkbox"
                  :checked="allSelected"
                  :indeterminate.prop="someSelected && !allSelected"
                  @change="toggleAll"
                  title="Select all"
                />
              </th>
              <th class="col-sku">SKU</th>
              <th class="col-name">Name</th>
              <th class="col-category">Category</th>
              <th class="col-warehouse">Warehouse</th>
              <th class="col-qty">On Hand</th>
              <th class="col-qty">Reorder Point</th>
              <th class="col-qty">Qty to Order</th>
              <th class="col-cost">Unit Cost</th>
              <th class="col-cost">Est. Cost</th>
              <th class="col-trend">Trend</th>
              <th class="col-priority">Priority</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="item in recommendations"
              :key="item.sku"
              :class="{ 'row-selected': selectedSkus.has(item.sku) }"
            >
              <td class="col-check">
                <input
                  type="checkbox"
                  :checked="selectedSkus.has(item.sku)"
                  @change="toggleSelection(item.sku)"
                />
              </td>
              <td class="col-sku"><strong>{{ item.sku }}</strong></td>
              <td class="col-name">{{ item.name }}</td>
              <td class="col-category">{{ item.category }}</td>
              <td class="col-warehouse">{{ item.warehouse }}</td>
              <td class="col-qty">{{ item.quantity_on_hand }}</td>
              <td class="col-qty reorder-point">{{ item.reorder_point }}</td>
              <td class="col-qty order-qty">{{ item.quantity_to_order }}</td>
              <td class="col-cost">{{ formatCurrency(item.unit_cost) }}</td>
              <td class="col-cost est-cost">{{ formatCurrency(item.estimated_cost) }}</td>
              <td class="col-trend">
                <!-- Only show demand trend badge when value is not null -->
                <span
                  v-if="item.demand_trend !== null"
                  :class="['badge', getTrendClass(item.demand_trend)]"
                >
                  {{ item.demand_trend }}
                </span>
                <span v-else class="muted">-</span>
              </td>
              <td class="col-priority">
                <div class="priority-bar-container" :title="`Priority: ${(item.priority_score * 100).toFixed(0)}%`">
                  <div
                    class="priority-bar"
                    :style="{ width: (item.priority_score * 100).toFixed(0) + '%' }"
                    :class="getPriorityClass(item.priority_score)"
                  ></div>
                </div>
                <span class="priority-label">{{ (item.priority_score * 100).toFixed(0) }}%</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Footer: running total and place order button -->
      <div v-if="recommendations.length > 0" class="card-footer">
        <div class="footer-total">
          <span class="footer-label">Selected Total:</span>
          <span class="footer-value">{{ formatCurrency(selectedTotal) }}</span>
        </div>
        <button
          class="btn-place-order"
          :disabled="selectedSkus.size === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(0)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const successBanner = ref(null)

    // Set of selected SKUs — use a ref wrapping a Set; reassign to trigger reactivity
    const selectedSkus = ref(new Set())

    // Debounce timer handle
    let debounceTimer = null

    // Format a number as currency using the current symbol
    const formatCurrency = (value) => {
      const sym = currencySymbol.value
      return `${sym}${value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })}`
    }

    // Fetch recommendations from the API with debounce (~400ms)
    const fetchRecommendations = (budgetValue) => {
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(async () => {
        // Skip API call when budget is 0 — show empty state immediately
        if (budgetValue === 0) {
          recommendations.value = []
          selectedSkus.value = new Set()
          return
        }

        loading.value = true
        error.value = null
        try {
          const data = await api.getRestockingRecommendations(budgetValue)
          recommendations.value = data

          // Pre-select all returned items
          selectedSkus.value = new Set(data.map(item => item.sku))
        } catch (err) {
          error.value = 'Failed to load restocking recommendations: ' + err.message
          recommendations.value = []
        } finally {
          loading.value = false
        }
      }, 400)
    }

    // Slider input handler — update budget ref and trigger debounced fetch
    const onBudgetInput = (event) => {
      budget.value = Number(event.target.value)
      fetchRecommendations(budget.value)
    }

    // Computed: total estimated cost of selected items
    const selectedTotal = computed(() => {
      return recommendations.value
        .filter(item => selectedSkus.value.has(item.sku))
        .reduce((sum, item) => sum + item.estimated_cost, 0)
    })

    // Computed helpers for the header checkbox state
    const allSelected = computed(() => {
      return recommendations.value.length > 0 &&
        recommendations.value.every(item => selectedSkus.value.has(item.sku))
    })

    const someSelected = computed(() => {
      return recommendations.value.some(item => selectedSkus.value.has(item.sku))
    })

    // Toggle a single row's selection — must reassign Set to trigger reactivity
    const toggleSelection = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    // Toggle-all checkbox
    const toggleAll = () => {
      if (allSelected.value) {
        selectedSkus.value = new Set()
      } else {
        selectedSkus.value = new Set(recommendations.value.map(item => item.sku))
      }
    }

    // Map demand_trend string to badge class
    const getTrendClass = (trend) => {
      if (!trend) return ''
      const map = {
        increasing: 'increasing',
        stable: 'stable',
        decreasing: 'decreasing'
      }
      return map[trend.toLowerCase()] || 'info'
    }

    // Priority bar color class based on score
    const getPriorityClass = (score) => {
      if (score >= 0.7) return 'priority-high'
      if (score >= 0.4) return 'priority-medium'
      return 'priority-low'
    }

    // Submit the restocking order
    const placeOrder = async () => {
      if (selectedSkus.value.size === 0) return

      const selectedItems = recommendations.value
        .filter(item => selectedSkus.value.has(item.sku))
        .map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.quantity_to_order,
          unit_cost: item.unit_cost
        }))

      submitting.value = true
      error.value = null
      try {
        const result = await api.submitRestockingOrder(selectedItems, selectedTotal.value)

        // Calculate expected delivery: 7 days from now
        const deliveryDate = new Date()
        deliveryDate.setDate(deliveryDate.getDate() + 7)
        const formattedDelivery = deliveryDate.toLocaleDateString('en-US', {
          year: 'numeric',
          month: 'short',
          day: 'numeric'
        })

        successBanner.value = {
          order_number: result.order_number || result.id || 'N/A',
          expected_delivery: formattedDelivery
        }

        // Clear selections after success
        selectedSkus.value = new Set()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    // No initial fetch needed — wait for user to set a budget
    onMounted(() => {
      // Page loads with budget = 0, show empty state
    })

    return {
      budget,
      recommendations,
      loading,
      error,
      submitting,
      successBanner,
      selectedSkus,
      selectedTotal,
      allSelected,
      someSelected,
      formatCurrency,
      onBudgetInput,
      toggleSelection,
      toggleAll,
      getTrendClass,
      getPriorityClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Success banner */
.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #065f46;
  font-size: 0.938rem;
}

.success-banner-content .order-number {
  font-weight: 700;
}

.dismiss-btn {
  background: none;
  border: 1px solid #059669;
  color: #065f46;
  border-radius: 6px;
  padding: 0.375rem 0.75rem;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}

/* Budget card */
.budget-card .card-header {
  align-items: center;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-wrapper {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 0.5rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
  min-width: 50px;
}

.slider-label:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  background: linear-gradient(to right, #2563eb, #93c5fd);
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
  cursor: pointer;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
  cursor: pointer;
}

.slider-ticks {
  display: flex;
  justify-content: space-between;
  padding: 0.25rem 0.25rem 0;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Selection summary in card header */
.selection-summary {
  font-size: 0.875rem;
  color: #64748b;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.selection-summary strong {
  color: #0f172a;
}

.budget-status {
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
}

.budget-status.over-budget {
  background: #fecaca;
  color: #991b1b;
}

.budget-status.under-budget {
  background: #d1fae5;
  color: #065f46;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 3rem 1rem;
}

.empty-state-icon {
  font-size: 2rem;
  color: #cbd5e1;
  margin-bottom: 1rem;
}

.empty-state-text {
  color: #64748b;
  font-size: 0.938rem;
}

/* Restocking table layout */
.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 40px;
  text-align: center;
}

.col-sku {
  width: 110px;
}

.col-name {
  width: 200px;
}

.col-category {
  width: 150px;
}

.col-warehouse {
  width: 110px;
}

.col-qty {
  width: 80px;
  text-align: right;
}

.col-cost {
  width: 100px;
  text-align: right;
}

.col-trend {
  width: 110px;
  text-align: center;
}

.col-priority {
  width: 120px;
}

/* Highlight selected rows */
.row-selected {
  background: #eff6ff;
}

.row-selected:hover {
  background: #dbeafe !important;
}

/* Specific column emphasis */
.reorder-point {
  color: #ea580c;
}

.order-qty {
  font-weight: 600;
  color: #0f172a;
}

.est-cost {
  font-weight: 600;
  color: #0f172a;
}

.muted {
  color: #cbd5e1;
  font-size: 0.875rem;
}

/* Priority bar */
.priority-bar-container {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 0.25rem;
}

.priority-bar {
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s ease;
}

.priority-bar.priority-high {
  background: #ef4444;
}

.priority-bar.priority-medium {
  background: #f59e0b;
}

.priority-bar.priority-low {
  background: #3b82f6;
}

.priority-label {
  font-size: 0.75rem;
  color: #64748b;
}

/* Card footer with total and action button */
.card-footer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1.5rem;
  padding-top: 1rem;
  margin-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.footer-total {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.footer-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.footer-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

/* Place order button */
.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
</style>
