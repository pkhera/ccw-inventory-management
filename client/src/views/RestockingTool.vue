<template>
  <div class="restocking-tool">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Enter a budget to get a prioritized restock plan based on stock urgency and demand trends.</p>
    </div>

    <div class="card">
      <div class="budget-row">
        <label class="budget-label" for="budget-input">Budget (USD)</label>
        <div class="budget-input-row">
          <span class="budget-prefix">$</span>
          <input
            id="budget-input"
            v-model.number="budget"
            type="number"
            min="1"
            step="100"
            class="border border-slate-300 rounded-lg px-3 py-2 text-sm w-48 focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <button
            @click="fetchRecommendations"
            :disabled="loading"
            class="bg-blue-600 hover:bg-blue-700 text-white font-semibold px-5 py-2 rounded-lg disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
          >
            {{ loading ? 'Loading...' : 'Get Recommendations' }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading">Calculating recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <template v-if="hasResult && !loading">
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Budget</div>
          <div class="stat-value">${{ result.total_cost.toLocaleString() }} used</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Items Selected</div>
          <div class="stat-value">{{ result.items_count }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Remaining Budget</div>
          <div class="stat-value">${{ result.remaining_budget.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Items Skipped</div>
          <div class="stat-value">{{ result.items_skipped }}</div>
          <div class="stat-sublabel">didn't fit budget</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocks</h3>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within this budget. Try increasing the budget amount.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>#</th>
                <th>Product</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th>On Hand</th>
                <th>Target</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Trend</th>
                <th>Backlog</th>
                <th>Score</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(item, index) in recommendations" :key="item.id">
                <td>{{ index + 1 }}</td>
                <td>
                  <div><strong>{{ item.sku }}</strong></div>
                  <div class="product-name">{{ item.name }}</div>
                </td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>
                  {{ item.target_quantity }}
                  <div class="target-sublabel">2x reorder pt</div>
                </td>
                <td>{{ item.restock_quantity }}</td>
                <td>${{ item.unit_cost.toFixed(2) }}</td>
                <td><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
                <td>
                  <span
                    v-if="item.demand_trend === 'no data'"
                    class="badge info"
                  >No Data</span>
                  <span
                    v-else
                    :class="['badge', item.demand_trend]"
                  >{{ item.demand_trend }}</span>
                </td>
                <td>
                  <span v-if="item.in_backlog" class="badge danger">Backlog</span>
                  <span v-else class="backlog-none">&mdash;</span>
                </td>
                <td>
                  <span :class="scoreClass(item.priority_score)">
                    {{ item.priority_score.toFixed(2) }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>

    <div v-if="!hasResult && !loading && !error" class="card initial-empty-state">
      Enter a budget above and click Get Recommendations to see your restocking plan.
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue'
import { api } from '../api'

export default {
  name: 'RestockingTool',
  setup() {
    const budget = ref(10000)
    const loading = ref(false)
    const error = ref(null)
    const result = ref(null)

    const recommendations = computed(() => result.value?.recommendations ?? [])
    const hasResult = computed(() => result.value !== null)

    const fetchRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        result.value = data
      } catch (err) {
        error.value = 'Failed to fetch recommendations: ' + (err.message || 'Unknown error')
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const scoreClass = (score) => {
      if (score > 1.0) return 'text-red-600 font-semibold'
      if (score > 0.5) return 'text-orange-500 font-semibold'
      return 'text-slate-500'
    }

    return {
      budget,
      loading,
      error,
      result,
      recommendations,
      hasResult,
      fetchRecommendations,
      scoreClass
    }
  }
}
</script>

<style scoped>
.restocking-tool {
  padding: 2rem;
}

.budget-row {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 1rem 1.5rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
}

.budget-input-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.budget-prefix {
  font-size: 1rem;
  color: #64748b;
  font-weight: 500;
}

.product-name {
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 2px;
}

.target-sublabel {
  font-size: 0.7rem;
  color: #94a3b8;
  margin-top: 2px;
}

.stat-sublabel {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 4px;
}

.backlog-none {
  color: #94a3b8;
}

.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.95rem;
}

.initial-empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.95rem;
}
</style>
