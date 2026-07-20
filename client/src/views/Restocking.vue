<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetTitle') }}</h3>
        </div>
        <div class="budget-control">
          <div class="budget-display">
            <span class="budget-label">{{ t('restocking.currentBudget') }}:</span>
            <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
          </div>
          <div class="slider-container">
            <span class="slider-label">{{ currencySymbol }}0</span>
            <input
              type="range"
              v-model.number="budget"
              min="0"
              max="10000"
              step="100"
              class="budget-slider"
            />
            <span class="slider-label">{{ currencySymbol }}10,000</span>
          </div>
        </div>
      </div>

      <!-- Stats Grid -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.totalItems') }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.estimatedCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ budgetRemaining.toLocaleString() }}</div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ selectedItems.length }} {{ t('common.selected') }})</h3>
          <button
            class="btn btn-primary"
            :disabled="selectedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('common.submitting') : t('restocking.placeOrder') }}
          </button>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-select">
                  <input
                    type="checkbox"
                    :checked="allSelected"
                    @change="toggleSelectAll"
                  />
                </th>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-name">{{ t('restocking.table.itemName') }}</th>
                <th class="col-number">{{ t('restocking.table.onHand') }}</th>
                <th class="col-number">{{ t('restocking.table.forecasted') }}</th>
                <th class="col-number">{{ t('restocking.table.gap') }}</th>
                <th class="col-number">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-quantity">{{ t('restocking.table.quantity') }}</th>
                <th class="col-number">{{ t('restocking.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ selected: selectedSkus.has(item.sku) }"
              >
                <td class="col-select">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.sku)"
                    @change="toggleSelect(item.sku)"
                  />
                </td>
                <td class="col-sku"><strong>{{ item.sku }}</strong></td>
                <td class="col-name">{{ translateProductName(item.item_name) }}</td>
                <td class="col-number">{{ item.quantity_on_hand }}</td>
                <td class="col-number">{{ item.forecasted_demand }}</td>
                <td class="col-number">
                  <span class="gap-value">{{ item.demand_gap }}</span>
                </td>
                <td class="col-number">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-quantity">
                  <input
                    type="number"
                    v-model.number="quantities[item.sku]"
                    min="0"
                    :max="item.recommended_quantity"
                    class="quantity-input"
                    @input="updateQuantity(item.sku)"
                  />
                </td>
                <td class="col-number">
                  <strong>{{ currencySymbol }}{{ (quantities[item.sku] * item.unit_cost).toFixed(2) }}</strong>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const budget = ref(5000)
    const recommendations = ref([])
    const selectedSkus = ref(new Set())
    const quantities = ref({})
    const submitting = ref(false)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const response = await api.getRestockingRecommendations(budget.value)

        recommendations.value = response.items || []

        // Initialize quantities with recommended values
        const initialQuantities = {}
        recommendations.value.forEach(item => {
          initialQuantities[item.sku] = item.recommended_quantity
        })
        quantities.value = initialQuantities

        // Clear selections when budget changes
        selectedSkus.value = new Set()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error('Load error:', err)
      } finally {
        loading.value = false
      }
    }

    // Watch budget changes and reload recommendations
    watch(budget, () => {
      loadRecommendations()
    })

    const selectedItems = computed(() => {
      return recommendations.value.filter(item => selectedSkus.value.has(item.sku))
    })

    const totalCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => {
        return sum + (quantities.value[item.sku] * item.unit_cost)
      }, 0)
    })

    const budgetRemaining = computed(() => {
      return Math.max(0, budget.value - totalCost.value)
    })

    const allSelected = computed(() => {
      return recommendations.value.length > 0 &&
             selectedSkus.value.size === recommendations.value.length
    })

    const toggleSelect = (sku) => {
      const newSet = new Set(selectedSkus.value)
      if (newSet.has(sku)) {
        newSet.delete(sku)
      } else {
        newSet.add(sku)
      }
      selectedSkus.value = newSet
    }

    const toggleSelectAll = () => {
      if (allSelected.value) {
        selectedSkus.value = new Set()
      } else {
        selectedSkus.value = new Set(recommendations.value.map(item => item.sku))
      }
    }

    const updateQuantity = (sku) => {
      // Ensure quantity stays within bounds
      const item = recommendations.value.find(i => i.sku === sku)
      if (item) {
        const qty = quantities.value[sku]
        if (qty < 0) {
          quantities.value[sku] = 0
        } else if (qty > item.recommended_quantity) {
          quantities.value[sku] = item.recommended_quantity
        }
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return

      try {
        submitting.value = true

        // Build order items array
        const orderItems = selectedItems.value.map(item => ({
          sku: item.sku,
          name: item.item_name,
          quantity: quantities.value[item.sku],
          unit_cost: item.unit_cost
        }))

        const orderData = {
          items: orderItems,
          total_value: totalCost.value
        }

        await api.submitRestockingOrder(orderData)

        alert(`Order placed successfully! Total: ${currencySymbol.value}${totalCost.value.toLocaleString()}`)

        // Clear selections and reload recommendations
        selectedSkus.value = new Set()
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place order'
        console.error('Order error:', err)
        alert('Failed to place order. Please try again.')
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => {
      loadRecommendations()
    })

    return {
      t,
      loading,
      error,
      budget,
      recommendations,
      selectedSkus,
      quantities,
      submitting,
      selectedItems,
      totalCost,
      budgetRemaining,
      allSelected,
      currencySymbol,
      toggleSelect,
      toggleSelectAll,
      updateQuantity,
      placeOrder,
      translateProductName
    }
  }
}
</script>

<style scoped>
/* Budget Card */
.budget-card {
  margin-bottom: 2rem;
}

.budget-control {
  padding: 1rem;
}

.budget-display {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.budget-label {
  font-size: 0.875rem;
  color: #64748b;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
}

.slider-container {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  min-width: 60px;
}

.slider-label:first-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  transition: background 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #2563eb;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: none;
  transition: background 0.2s;
}

.budget-slider::-moz-range-thumb:hover {
  background: #2563eb;
}

/* Table styles */
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-select {
  width: 50px;
  text-align: center;
}

.col-sku {
  width: 120px;
}

.col-name {
  width: 200px;
}

.col-number {
  width: 100px;
  text-align: right;
}

.col-quantity {
  width: 100px;
  text-align: center;
}

.restocking-table tbody tr {
  transition: background-color 0.2s;
}

.restocking-table tbody tr.selected {
  background-color: #eff6ff;
}

.restocking-table tbody tr:hover {
  background-color: #f8fafc;
}

.gap-value {
  color: #ef4444;
  font-weight: 600;
}

.quantity-input {
  width: 80px;
  padding: 0.375rem 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  font-size: 0.875rem;
  text-align: center;
}

.quantity-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

/* Button */
.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-primary {
  background: #3b82f6;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}

/* Card header with button */
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>
