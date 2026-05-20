<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Review items at or below reorder point, set a budget, and submit a restock order.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget slider card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-body">
          <div class="budget-row">
            <input
              type="range"
              :min="0"
              :max="budgetMax"
              :step="1000"
              v-model.number="budget"
              class="budget-slider"
            />
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
          <p class="budget-hint">
            {{ checkedCount }} item{{ checkedCount !== 1 ? 's' : '' }} fit within budget
          </p>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendations.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items at or below reorder point — nothing to restock.
        </div>

        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">SKU</th>
                <th class="col-name">Name</th>
                <th class="col-category">Category</th>
                <th class="col-num">On Hand</th>
                <th class="col-num">Reorder Point</th>
                <th class="col-num">Forecasted Demand</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-qty">Qty</th>
                <th class="col-num">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="row in recommendations"
                :key="row.sku"
                :class="{ 'row-selected': row.selected, 'row-over': row.selected && row.quantity * row.unit_cost + (grandTotal - (row.selected ? row.quantity * row.unit_cost : 0)) > budget }"
              >
                <td class="col-check">
                  <input type="checkbox" v-model="row.selected" />
                </td>
                <td class="col-sku"><strong>{{ row.sku }}</strong></td>
                <td class="col-name">{{ row.name }}</td>
                <td class="col-category">
                  <span class="badge info">{{ row.category }}</span>
                </td>
                <td class="col-num">{{ row.quantity_on_hand }}</td>
                <td class="col-num">{{ row.reorder_point }}</td>
                <td class="col-num">
                  <span v-if="row.forecasted_demand !== null">{{ row.forecasted_demand }}</span>
                  <span v-else class="no-forecast">—</span>
                </td>
                <td class="col-num">${{ row.unit_cost.toFixed(2) }}</td>
                <td class="col-qty">
                  <input
                    type="number"
                    v-model.number="row.quantity"
                    min="1"
                    class="qty-input"
                    :disabled="!row.selected"
                  />
                </td>
                <td class="col-num">
                  <strong :class="{ 'over-budget-text': row.selected && grandTotal > budget }">
                    {{ formatCurrency(row.quantity * row.unit_cost) }}
                  </strong>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Success banner -->
        <div v-if="lastOrder" class="success-banner">
          Order {{ lastOrder.order_number }} placed &mdash; Expected arrival in {{ lastOrder.daysUntil }} day{{ lastOrder.daysUntil !== 1 ? 's' : '' }}
        </div>

        <!-- Footer -->
        <div v-if="recommendations.length > 0" class="order-footer">
          <div class="footer-summary" :class="{ 'over-budget': grandTotal > budget }">
            {{ checkedCount }} item{{ checkedCount !== 1 ? 's' : '' }} &middot;
            {{ formatCurrency(grandTotal) }} of {{ formatCurrency(budget) }}
            <span v-if="grandTotal > budget" class="over-label"> (over budget)</span>
          </div>
          <button
            class="place-order-btn"
            :disabled="grandTotal === 0 || grandTotal > budget || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
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
    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const lastOrder = ref(null)

    // Mutable row state — kept as ref array so user edits persist across slider moves
    const recommendations = ref([])

    const budget = ref(0)
    const budgetMax = ref(50000)

    const grandTotal = computed(() => {
      return recommendations.value.reduce((sum, row) => {
        return sum + (row.selected ? row.quantity * row.unit_cost : 0)
      }, 0)
    })

    const checkedCount = computed(() => {
      return recommendations.value.filter(r => r.selected).length
    })

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    // Greedy fit: iterate rows in display order, check each if it fits under budget
    const applyGreedyFit = (budgetValue) => {
      let running = 0
      for (const row of recommendations.value) {
        const lineCost = row.quantity * row.unit_cost
        if (running + lineCost <= budgetValue) {
          row.selected = true
          running += lineCost
        } else {
          row.selected = false
        }
      }
    }

    watch(budget, (newBudget) => {
      applyGreedyFit(newBudget)
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [inventoryData, forecastsData] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])

        // Build SKU and name lookup maps from forecasts
        const forecastBySku = {}
        const forecastByName = {}
        for (const f of forecastsData) {
          forecastBySku[f.item_sku] = f
          forecastByName[f.item_name.toLowerCase()] = f
        }

        // Filter to items at or below reorder point, then attach forecast
        const rows = inventoryData
          .filter(item => item.quantity_on_hand <= item.reorder_point)
          .map(item => {
            const forecast = forecastBySku[item.sku] || forecastByName[item.name.toLowerCase()] || null
            const forecastedDemand = forecast ? forecast.forecasted_demand : null
            const defaultQty = Math.max(
              item.reorder_point * 2,
              forecastedDemand || 0
            ) - item.quantity_on_hand
            return {
              sku: item.sku,
              name: item.name,
              category: item.category,
              quantity_on_hand: item.quantity_on_hand,
              reorder_point: item.reorder_point,
              unit_cost: item.unit_cost,
              forecasted_demand: forecastedDemand,
              quantity: Math.max(1, defaultQty),
              selected: false
            }
          })

        // Sort by forecasted_demand desc; no-forecast items sort to the end
        rows.sort((a, b) => {
          const aVal = a.forecasted_demand ?? -1
          const bVal = b.forecasted_demand ?? -1
          return bVal - aVal
        })

        recommendations.value = rows

        // Compute slider max from full restock cost, rounded up to nearest $10K
        const totalFullRestockCost = rows.reduce((sum, r) => sum + r.quantity * r.unit_cost, 0)
        const computedMax = totalFullRestockCost > 0
          ? Math.ceil(totalFullRestockCost / 10000) * 10000
          : 50000
        budgetMax.value = computedMax

        // Default budget at ~50% of max, rounded to nearest $1K
        budget.value = Math.round(computedMax * 0.5 / 1000) * 1000

        // Apply initial greedy fit
        applyGreedyFit(budget.value)
      } catch (err) {
        error.value = 'Failed to load restocking data'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      placing.value = true
      lastOrder.value = null
      try {
        const items = recommendations.value
          .filter(r => r.selected)
          .map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.quantity,
            unit_price: r.unit_cost
          }))

        const created = await api.createOrder({ items })

        // Compute days until delivery for the banner
        const deliveryDate = new Date(created.expected_delivery)
        const daysUntil = isNaN(deliveryDate.getTime())
          ? null
          : Math.max(0, Math.ceil((deliveryDate.getTime() - Date.now()) / 86_400_000))

        lastOrder.value = {
          order_number: created.order_number,
          daysUntil
        }

        // Reset selections after successful order
        for (const row of recommendations.value) {
          row.selected = false
        }
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      placing,
      lastOrder,
      recommendations,
      budget,
      budgetMax,
      grandTotal,
      checkedCount,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-body {
  padding: 1.25rem 1.5rem;
}

.budget-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 110px;
  text-align: right;
}

.budget-hint {
  margin-top: 0.625rem;
  font-size: 0.875rem;
  color: #64748b;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check { width: 40px; }
.col-sku { width: 110px; }
.col-name { width: 220px; }
.col-category { width: 160px; }
.col-num { width: 120px; }
.col-qty { width: 90px; }

.qty-input {
  width: 80px;
  text-align: right;
  padding: 0.25rem 0.375rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #f8fafc;
}

.qty-input:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.qty-input:focus {
  outline: none;
  border-color: #3b82f6;
  background: white;
}

.row-selected {
  background: #f0f9ff;
}

.no-forecast {
  color: #94a3b8;
}

.over-budget-text {
  color: #dc2626;
}

.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  margin: 1rem 1.5rem 0;
  padding: 0.75rem 1rem;
  background: #d1fae5;
  color: #065f46;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  border: 1px solid #a7f3d0;
}

.order-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.footer-summary {
  font-size: 0.938rem;
  color: #475569;
  font-weight: 500;
}

.footer-summary.over-budget {
  color: #dc2626;
  font-weight: 600;
}

.over-label {
  font-weight: 700;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
