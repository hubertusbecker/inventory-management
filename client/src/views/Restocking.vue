<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Order inventory based on forecasted demand. Select a budget to see recommended items.</p>
    </div>

    <div v-if="successMessage" class="success-banner">
      {{ successMessage }}
    </div>

    <div v-if="error" class="error">{{ error }}</div>

    <!-- Budget Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-body">
        <div class="budget-label-row">
          <span class="budget-amount">Budget: {{ formatCurrency(budget) }}</span>
        </div>
        <input
          type="range"
          min="0"
          max="500000"
          step="1000"
          :value="budget"
          @input="onBudgetInput"
          class="budget-slider"
        />
        <div class="budget-range-labels">
          <span>{{ formatCurrency(0) }}</span>
          <span>{{ formatCurrency(500000) }}</span>
        </div>
        <div class="budget-stats">
          <span class="budget-stat">
            <span class="stat-label-text">Allocated:</span>
            <strong>{{ formatCurrency(allocatedTotal) }}</strong>
          </span>
          <span class="budget-stat">
            <span class="stat-label-text">Remaining:</span>
            <strong :class="{ 'over-budget-text': remaining < 0 }">{{ formatCurrency(remaining) }}</strong>
          </span>
        </div>
      </div>
    </div>

    <!-- Recommendations Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items <span class="card-title-sub">(increasing demand, sorted by demand gap)</span></h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="recommendations.length === 0" class="empty-state">
        No items with increasing demand found for the selected budget.
      </div>
      <div v-else class="table-container">
        <table class="reco-table">
          <thead>
            <tr>
              <th class="col-sku">SKU</th>
              <th class="col-name">Item</th>
              <th class="col-warehouse">Warehouse</th>
              <th class="col-num">Current Demand</th>
              <th class="col-num">Forecasted</th>
              <th class="col-gap">Gap</th>
              <th class="col-cost">Total Cost</th>
              <th class="col-incl">Incl.</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="rec in recommendations"
              :key="rec.sku"
              :class="{ 'row-over-budget': !rec.fits_budget }"
            >
              <td class="col-sku"><code>{{ rec.sku }}</code></td>
              <td class="col-name">{{ rec.name }}</td>
              <td class="col-warehouse">{{ rec.warehouse }}</td>
              <td class="col-num">{{ rec.current_demand.toLocaleString() }}</td>
              <td class="col-num">{{ rec.forecasted_demand.toLocaleString() }}</td>
              <td class="col-gap">
                <span class="demand-gap">+{{ rec.demand_gap.toLocaleString() }}</span>
              </td>
              <td class="col-cost">{{ formatCurrency(rec.total_cost) }}</td>
              <td class="col-incl">
                <input
                  type="checkbox"
                  :checked="checkedSkus.has(rec.sku)"
                  @change="toggleSku(rec.sku)"
                />
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Place Order Footer -->
    <div class="order-footer" v-if="!loading && recommendations.length > 0">
      <button
        class="place-order-btn"
        :disabled="checkedSkus.size === 0 || submitting"
        @click="placeOrder"
      >
        {{ submitting ? 'Submitting...' : 'Place Order' }}
      </button>
      <div class="order-footer-meta">
        <span>Total: <strong>{{ formatCurrency(allocatedTotal) }}</strong></span>
        <span class="meta-sep">·</span>
        <span>{{ checkedSkus.size }} item{{ checkedSkus.size !== 1 ? 's' : '' }} selected</span>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'

export default {
  name: 'Restocking',
  setup() {
    const { selectedLocation } = useFilters()

    const budget = ref(100000)
    const recommendations = ref([])
    const checkedSkus = ref(new Set())
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)
    let debounceTimer = null

    const formatCurrency = (value) => {
      return '$' + Math.round(value).toLocaleString('en-US')
    }

    const allocatedTotal = computed(() => {
      return recommendations.value
        .filter(r => checkedSkus.value.has(r.sku))
        .reduce((sum, r) => sum + r.total_cost, 0)
    })

    const remaining = computed(() => budget.value - allocatedTotal.value)

    const fetchRecommendations = () => {
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(async () => {
        loading.value = true
        error.value = null
        try {
          const data = await api.getRestockingRecommendations(budget.value)
          recommendations.value = data
          // Auto-check items that fit in budget
          const autoChecked = new Set()
          data.forEach(rec => {
            if (rec.fits_budget) autoChecked.add(rec.sku)
          })
          checkedSkus.value = autoChecked
        } catch (err) {
          error.value = 'Failed to load recommendations: ' + err.message
          console.error(err)
        } finally {
          loading.value = false
        }
      }, 300)
    }

    const fetchImmediately = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data
        const autoChecked = new Set()
        data.forEach(rec => {
          if (rec.fits_budget) autoChecked.add(rec.sku)
        })
        checkedSkus.value = autoChecked
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const onBudgetInput = (event) => {
      budget.value = Number(event.target.value)
      fetchRecommendations()
    }

    const toggleSku = (sku) => {
      const next = new Set(checkedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      checkedSkus.value = next
    }

    const placeOrder = async () => {
      if (checkedSkus.value.size === 0 || submitting.value) return

      submitting.value = true
      error.value = null
      successMessage.value = null

      const selectedRecs = recommendations.value.filter(r => checkedSkus.value.has(r.sku))
      const warehouse = selectedLocation.value !== 'all' ? selectedLocation.value : null

      const payload = {
        items: selectedRecs.map(rec => ({
          sku: rec.sku,
          name: rec.name,
          quantity: rec.quantity,
          unit_price: rec.unit_cost
        })),
        warehouse,
        order_type: 'restocking'
      }

      try {
        const order = await api.createRestockingOrder(payload)
        successMessage.value = `Order ${order.order_number} submitted — expected delivery in 14 days.`
        await fetchImmediately()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(fetchImmediately)

    return {
      budget,
      recommendations,
      checkedSkus,
      loading,
      error,
      submitting,
      successMessage,
      allocatedTotal,
      remaining,
      formatCurrency,
      onBudgetInput,
      toggleSku,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}

/* Budget card body */
.budget-body {
  padding: 0.5rem 0;
}

.budget-label-row {
  margin-bottom: 0.75rem;
}

.budget-amount {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  accent-color: #3b82f6;
  cursor: pointer;
  height: 6px;
  margin-bottom: 0.375rem;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1rem;
}

.budget-stats {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.budget-stat {
  display: flex;
  align-items: center;
  gap: 0.375rem;
  font-size: 0.875rem;
}

.stat-label-text {
  color: #64748b;
}

.budget-stat strong {
  color: #0f172a;
}

.over-budget-text {
  color: #dc2626 !important;
}

/* Recommendations table */
.reco-table {
  table-layout: auto;
  width: 100%;
}

.col-sku {
  width: 100px;
}

.col-name {
  min-width: 180px;
}

.col-warehouse {
  width: 130px;
}

.col-num {
  width: 120px;
  text-align: right;
}

.col-gap {
  width: 80px;
  text-align: right;
}

.col-cost {
  width: 120px;
  text-align: right;
}

.col-incl {
  width: 60px;
  text-align: center;
}

.col-incl input[type="checkbox"] {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #3b82f6;
}

.demand-gap {
  color: #059669;
  font-weight: 600;
}

.row-over-budget {
  opacity: 0.4;
}

code {
  font-family: 'Courier New', monospace;
  font-size: 0.8rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

/* Card title sub-label */
.card-title-sub {
  font-size: 0.813rem;
  font-weight: 400;
  color: #64748b;
  margin-left: 0.25rem;
}

/* Empty state */
.empty-state {
  padding: 2.5rem 1rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Order footer */
.order-footer {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 1.25rem;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  margin-bottom: 1.25rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  border: none;
  cursor: pointer;
  transition: background 0.15s ease;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.order-footer-meta {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.938rem;
  color: #334155;
}

.meta-sep {
  color: #cbd5e1;
}
</style>
