<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
        </div>

        <div class="budget-display">{{ formattedBudget }}</div>

        <div class="slider-wrapper">
          <input
            type="range"
            class="budget-slider"
            :min="0"
            :max="500000"
            :step="1000"
            :value="budget"
            @input="budget = Number($event.target.value)"
          />
          <div class="slider-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}500,000</span>
          </div>
        </div>

        <div class="budget-summary">
          {{ selectedItems.length }} {{ t('restocking.itemsSelected') }} &middot;
          {{ formatCurrency(totalCost) }} {{ t('restocking.budgetUsed') }}
        </div>

        <div v-if="showSuccess" class="success-message">
          {{ t('restocking.orderPlaced') }}
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('demand.demandForecasts') }}</h3>
        </div>

        <div v-if="recommendations.length === 0" class="no-data">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.columns.item') }}</th>
                <th>{{ t('restocking.columns.sku') }}</th>
                <th class="col-number">{{ t('restocking.columns.demandGap') }}</th>
                <th class="col-number">{{ t('restocking.columns.unitCost') }}</th>
                <th class="col-number">{{ t('restocking.columns.restockQty') }}</th>
                <th class="col-number">{{ t('restocking.columns.totalCost') }}</th>
                <th class="col-status">{{ t('restocking.columns.included') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ selected: isSelected(item.sku) }"
              >
                <td>{{ item.item_name }}</td>
                <td><code class="sku">{{ item.sku }}</code></td>
                <td class="col-number">{{ item.demand_gap }}</td>
                <td class="col-number">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-number">{{ item.restock_qty }}</td>
                <td class="col-number">{{ formatCurrency(item.total_cost) }}</td>
                <td class="col-status">
                  <span v-if="isSelected(item.sku)" class="badge success">Yes</span>
                  <span v-else class="badge">No</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="card-footer">
          <button
            class="btn-primary"
            :disabled="selectedItems.length === 0 || orderLoading"
            @click="placeOrder"
          >
            {{ orderLoading ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>
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
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(50000)
    const showSuccess = ref(false)
    const orderLoading = ref(false)

    const selectedItems = computed(() => {
      let remaining = budget.value
      const selected = []
      for (const item of recommendations.value) {
        if (item.total_cost <= remaining) {
          selected.push(item)
          remaining -= item.total_cost
        }
      }
      return selected
    })

    const totalCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const formattedBudget = computed(() => {
      return formatCurrency(budget.value)
    })

    const isSelected = (sku) => {
      return selectedItems.value.some(item => item.sku === sku)
    }

    const formatCurrency = (value) => {
      if (currentCurrency.value === 'JPY') {
        return '¥' + Math.round(value).toLocaleString()
      }
      return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })
    }

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = t('common.error') + ': Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      try {
        orderLoading.value = true
        await api.createRestockingOrder({ items: selectedItems.value })
        showSuccess.value = true
        setTimeout(() => {
          showSuccess.value = false
        }, 3000)
      } catch (err) {
        error.value = t('common.error') + ': Failed to place order'
        console.error(err)
      } finally {
        orderLoading.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      t,
      loading,
      error,
      recommendations,
      budget,
      showSuccess,
      orderLoading,
      selectedItems,
      totalCost,
      formattedBudget,
      currencySymbol,
      isSelected,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.03em;
  margin-bottom: 1.25rem;
}

.slider-wrapper {
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  accent-color: #2563eb;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-summary {
  font-size: 0.938rem;
  color: #475569;
  font-weight: 500;
}

.success-message {
  margin-top: 0.875rem;
  padding: 0.75rem 1rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  color: #15803d;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
}

.no-data {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.restocking-table {
  table-layout: auto;
  width: 100%;
}

.col-number {
  text-align: right;
}

.col-status {
  text-align: center;
  width: 100px;
}

tbody tr.selected {
  background-color: #dcfce7;
}

tbody tr.selected:hover {
  background-color: #bbf7d0;
}

.sku {
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.card-footer {
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
}

.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}
</style>
