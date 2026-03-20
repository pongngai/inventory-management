<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Budget Slider Card -->
    <div class="card">
      <div class="card-header">
        <span class="card-title">{{ t('restocking.budgetSlider') }}</span>
        <span class="budget-display">{{ currencySymbol }}{{ formattedBudget }}</span>
      </div>
      <div class="slider-wrapper">
        <span class="slider-min">{{ currencySymbol }}10,000</span>
        <input
          type="range"
          class="budget-slider"
          :min="10000"
          :max="500000"
          :step="5000"
          v-model.number="budget"
        />
        <span class="slider-max">{{ currencySymbol }}500,000</span>
      </div>
    </div>

    <!-- Success Message -->
    <div v-if="orderSuccess" class="success-banner">
      {{ t('restocking.orderSuccess') }}
    </div>

    <!-- Recommendations Table Card -->
    <div class="card">
      <div class="card-header">
        <span class="card-title">{{ t('restocking.recommendations') }}</span>
        <span v-if="!loading && recommendations.length" class="badge info">
          {{ recommendations.length }}
        </span>
      </div>

      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="!recommendations.length" class="empty-state">
        {{ t('restocking.noRecommendations') }}
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>{{ t('restocking.table.sku') }}</th>
              <th>{{ t('restocking.table.itemName') }}</th>
              <th>{{ t('restocking.table.demandGap') }}</th>
              <th>{{ t('restocking.table.currentStock') }}</th>
              <th>{{ t('restocking.table.qtyToOrder') }}</th>
              <th>{{ t('restocking.table.unitCost') }}</th>
              <th>{{ t('restocking.table.totalCost') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendations" :key="item.sku">
              <td><code class="sku-code">{{ item.sku }}</code></td>
              <td>{{ item.name }}</td>
              <td>
                <span class="badge danger">{{ item.demand_gap }}</span>
              </td>
              <td>{{ item.current_stock.toLocaleString() }}</td>
              <td><strong>{{ item.quantity_to_order.toLocaleString() }}</strong></td>
              <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
              <td class="cost-cell">{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Summary Bar -->
      <div v-if="!loading && !error && recommendations.length" class="summary-bar">
        <div class="summary-items">
          <div class="summary-item">
            <span class="summary-label">{{ t('restocking.totalCost') }}</span>
            <span class="summary-value" :class="{ 'over-budget': isOverBudget }">
              {{ currencySymbol }}{{ totalCost.toLocaleString() }}
            </span>
          </div>
          <div class="summary-divider">/</div>
          <div class="summary-item">
            <span class="summary-label">Budget</span>
            <span class="summary-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
          </div>
          <div class="summary-separator">|</div>
          <div class="summary-item">
            <span class="summary-label">{{ t('restocking.remainingBudget') }}</span>
            <span class="summary-value" :class="isOverBudget ? 'remaining-negative' : 'remaining-positive'">
              {{ isOverBudget ? '-' : '' }}{{ currencySymbol }}{{ Math.abs(remainingBudget).toLocaleString() }}
            </span>
          </div>
        </div>

        <!-- Progress bar -->
        <div class="budget-progress">
          <div
            class="budget-progress-fill"
            :class="{ 'over-budget-fill': isOverBudget }"
            :style="{ width: Math.min(budgetUsagePercent, 100) + '%' }"
          ></div>
        </div>
        <div class="budget-progress-label">
          {{ budgetUsagePercent.toFixed(1) }}% of budget utilized
        </div>
      </div>
    </div>

    <!-- Place Order Button -->
    <div class="action-row">
      <button
        class="btn-place-order"
        :disabled="!recommendations.length || loading || submitting"
        @click="placeOrder"
      >
        <span v-if="submitting">Placing order...</span>
        <span v-else>{{ t('restocking.placeOrder') }}</span>
      </button>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { useI18n } from '../composables/useI18n'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const budget = ref(50000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(false)

    // Currency symbol derived from currentCurrency
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Formatted budget label for slider display
    const formattedBudget = computed(() => budget.value.toLocaleString())

    // Aggregated totals
    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const isOverBudget = computed(() => totalCost.value > budget.value)

    const budgetUsagePercent = computed(() =>
      budget.value > 0 ? (totalCost.value / budget.value) * 100 : 0
    )

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (!recommendations.value.length) return

      submitting.value = true
      error.value = null
      try {
        const items = recommendations.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.quantity_to_order,
          unit_cost: item.unit_cost,
          total_cost: item.total_cost
        }))
        await api.submitRestockingOrder({ items, total_budget: budget.value })

        // Show success message and clear recommendations
        orderSuccess.value = true
        recommendations.value = []

        // Auto-hide success message after 4 seconds
        setTimeout(() => {
          orderSuccess.value = false
        }, 4000)
      } catch (err) {
        error.value = 'Failed to submit restocking order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    // Debounced watcher — reloads recommendations 300ms after budget changes
    let debounceTimer = null
    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    onMounted(() => loadRecommendations())

    return {
      t,
      budget,
      recommendations,
      loading,
      error,
      submitting,
      orderSuccess,
      currencySymbol,
      formattedBudget,
      totalCost,
      remainingBudget,
      isOverBudget,
      budgetUsagePercent,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Slider */
.slider-wrapper {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.5rem 0;
}

.slider-min,
.slider-max {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
  min-width: 70px;
}

.slider-max {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: transform 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  transform: scale(1.15);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-display {
  font-size: 1.25rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-weight: 500;
  font-size: 0.938rem;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 3rem 1.5rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* SKU code style */
.sku-code {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.cost-cell {
  font-weight: 600;
  color: #0f172a;
}

/* Summary bar */
.summary-bar {
  margin-top: 1.25rem;
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
}

.summary-items {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.875rem;
  flex-wrap: wrap;
}

.summary-item {
  display: flex;
  align-items: center;
  gap: 0.375rem;
}

.summary-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
}

.summary-value {
  font-size: 0.938rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-value.over-budget {
  color: #dc2626;
}

.summary-value.remaining-positive {
  color: #059669;
}

.summary-value.remaining-negative {
  color: #dc2626;
}

.summary-divider,
.summary-separator {
  color: #94a3b8;
  font-weight: 400;
}

/* Budget progress bar */
.budget-progress {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.375rem;
}

.budget-progress-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.4s ease;
}

.budget-progress-fill.over-budget-fill {
  background: #dc2626;
}

.budget-progress-label {
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Place order button */
.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 0.25rem;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
  letter-spacing: 0.01em;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
</style>
