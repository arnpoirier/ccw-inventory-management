<template>
  <div class="restock-advisor">
    <!-- Page header -->
    <div class="page-header">
      <h2>Restock Advisor</h2>
      <p>Enter a budget to get prioritized restock recommendations for low-stock items.</p>
    </div>

    <!-- Budget input card (always visible) -->
    <div class="card budget-card">
      <div class="budget-row">
        <label class="budget-label">Available Budget</label>
        <div class="budget-input-group">
          <span class="currency-symbol">{{ currencySymbol }}</span>
          <input type="number" v-model.number="budgetInput" min="0" step="100" class="budget-input" />
        </div>
        <button @click="loadRecommendations" :disabled="loading" class="run-btn">
          {{ loading ? 'Calculating...' : 'Get Recommendations' }}
        </button>
      </div>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="loading">Calculating recommendations...</div>

    <!-- Error -->
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Results -->
    <template v-else-if="data">
      <!-- Empty: all stock healthy -->
      <div v-if="data.summary.total_items_needing_restock === 0" class="card empty-state">
        <p>All inventory items are above their reorder points for the selected filters.</p>
      </div>

      <!-- Empty: budget too small -->
      <div v-else-if="data.summary.items_within_budget === 0" class="card empty-state">
        <p>
          {{ data.summary.total_items_needing_restock }} items need restocking, but the budget of
          {{ currencySymbol }}{{ data.budget.toLocaleString() }} is too small to cover any single item.
        </p>
      </div>

      <template v-else>
        <!-- Summary stat cards -->
        <div class="stats-grid">
          <div class="stat-card info">
            <div class="stat-label">Items to Restock</div>
            <div class="stat-value">{{ data.summary.items_within_budget }}</div>
          </div>
          <div class="stat-card success">
            <div class="stat-label">Budget Allocated</div>
            <div class="stat-value">{{ currencySymbol }}{{ data.summary.budget_allocated.toLocaleString(undefined, {minimumFractionDigits: 2, maximumFractionDigits: 2}) }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Budget Remaining</div>
            <div class="stat-value">{{ currencySymbol }}{{ data.summary.budget_remaining.toLocaleString(undefined, {minimumFractionDigits: 2, maximumFractionDigits: 2}) }}</div>
          </div>
          <div v-if="data.summary.items_excluded > 0" class="stat-card warning">
            <div class="stat-label">Items Not Funded</div>
            <div class="stat-value">{{ data.summary.items_excluded }}</div>
          </div>
        </div>

        <!-- Recommendations table -->
        <div class="card">
          <div class="card-header">
            <h3 class="card-title">Recommendations ({{ data.recommendations.length }} items need restocking)</h3>
          </div>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>#</th>
                  <th>SKU</th>
                  <th>Item Name</th>
                  <th>Warehouse</th>
                  <th>On Hand</th>
                  <th>Reorder Point</th>
                  <th>Deficit</th>
                  <th>Trend</th>
                  <th>Restock Qty</th>
                  <th>Unit Cost</th>
                  <th>Restock Cost</th>
                  <th>Status</th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="(item, index) in data.recommendations"
                  :key="item.item_id"
                  :class="{ 'row-excluded': !item.allocated }"
                >
                  <td>{{ index + 1 }}</td>
                  <td><strong>{{ item.sku }}</strong></td>
                  <td>{{ item.name }}</td>
                  <td>{{ item.warehouse }}</td>
                  <td>{{ item.quantity_on_hand }}</td>
                  <td>{{ item.reorder_point }}</td>
                  <td><span class="badge danger">{{ item.deficit }}</span></td>
                  <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                  <td><strong>{{ item.restock_quantity }}</strong></td>
                  <td>{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                  <td><strong>{{ currencySymbol }}{{ item.restock_cost.toLocaleString(undefined, {minimumFractionDigits: 2, maximumFractionDigits: 2}) }}</strong></td>
                  <td>
                    <span v-if="item.allocated" class="badge success">Funded</span>
                    <span v-else class="badge-excluded">Excluded</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Shortfall notice -->
        <div v-if="data.summary.items_excluded > 0" class="card shortfall-notice">
          <strong>{{ data.summary.items_excluded }} item{{ data.summary.items_excluded !== 1 ? 's' : '' }} could not be funded.</strong>
          An additional {{ currencySymbol }}{{ (data.summary.total_restock_cost - data.summary.budget_allocated).toLocaleString(undefined, {minimumFractionDigits: 2, maximumFractionDigits: 2}) }} would be needed to cover all qualifying items.
        </div>
      </template>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'RestockAdvisor',
  setup() {
    const { currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const budgetInput = ref(5000)
    const loading = ref(false)
    const error = ref(null)
    const data = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const filters = getCurrentFilters()
        data.value = await api.getRestockRecommendations({
          budget: budgetInput.value,
          warehouse: filters.warehouse,
          category: filters.category,
        })
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Auto-reload when warehouse/category filter changes (if we already have results)
    watch([selectedLocation, selectedCategory], () => {
      if (data.value !== null) {
        loadRecommendations()
      }
    })

    return {
      currencySymbol,
      budgetInput,
      loading,
      error,
      data,
      loadRecommendations,
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
  white-space: nowrap;
}

.budget-input-group {
  display: flex;
  align-items: center;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  background: #f8fafc;
  overflow: hidden;
}

.currency-symbol {
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 600;
  background: #f1f5f9;
  border-right: 1px solid #cbd5e1;
}

.budget-input {
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  color: #0f172a;
  border: none;
  background: transparent;
  outline: none;
  width: 150px;
}

.budget-input:focus {
  outline: none;
}

.run-btn {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  white-space: nowrap;
}

.run-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.run-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.row-excluded {
  opacity: 0.4;
}

.badge-excluded {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  background: #f1f5f9;
  color: #64748b;
}

.shortfall-notice {
  background: #fffbeb;
  border-color: #fde68a;
  color: #92400e;
}
</style>
