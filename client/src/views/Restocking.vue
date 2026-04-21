<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Auto-select items to restock based on demand forecasts and inventory levels</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success Banner -->
      <div v-if="orderSuccess" class="success-banner">
        Restocking order {{ lastOrderNumber }} placed successfully. Items have been queued for procurement.
      </div>

      <!-- Budget Controls -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
          <div class="budget-summary">
            <span class="budget-label">Remaining:</span>
            <span :class="['budget-remaining', { 'over-budget': budgetRemaining < 0 }]">
              ${{ budgetRemaining.toLocaleString() }}
            </span>
            <span class="budget-label">of ${{ budget.toLocaleString() }}</span>
          </div>
        </div>
        <div class="budget-slider-row">
          <span class="slider-bound">$0</span>
          <input
            type="range"
            v-model.number="budget"
            min="0"
            max="100000"
            step="1000"
            class="budget-slider"
            @input="onBudgetChange"
          />
          <span class="slider-bound">$100,000</span>
          <span class="budget-value">${{ budget.toLocaleString() }}</span>
        </div>
        <div class="budget-bar-track">
          <div
            class="budget-bar-fill"
            :style="{ width: Math.min(100, (selectedTotal / budget) * 100) + '%' }"
            :class="{ 'over-budget-bar': budgetRemaining < 0 }"
          ></div>
        </div>
      </div>

      <!-- Controls Row -->
      <div class="controls-row">
        <div class="controls-left">
          <button class="btn-secondary" @click="runAutoSelect">Auto-Select</button>
          <button class="btn-ghost" @click="clearAll">Clear All</button>
        </div>
        <div class="controls-right">
          <span class="selection-summary">
            {{ selectedSkus.size }} item{{ selectedSkus.size !== 1 ? 's' : '' }} selected &mdash;
            <strong>${{ selectedTotal.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
          </span>
          <button
            class="btn-primary"
            :disabled="selectedSkus.size === 0 || isSubmitting"
            @click="placeOrder"
          >
            {{ isSubmitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>

      <!-- Items Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Items to Restock ({{ enrichedItems.length }})</h3>
          <span class="table-hint">Items with demand forecasts joined with inventory data</span>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-sku">SKU</th>
                <th class="col-trend">Trend</th>
                <th class="col-stock">Stock Status</th>
                <th class="col-qty">Restock Qty</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-total">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in enrichedItems"
                :key="item.item_sku"
                :class="{
                  'row-selected': selectedSkus.has(item.item_sku),
                  'row-low-stock': item.isBelowReorder
                }"
                @click="toggleItem(item.item_sku)"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    @change.stop="toggleItem(item.item_sku)"
                    @click.stop
                  />
                </td>
                <td class="col-name">
                  <strong>{{ item.item_name }}</strong>
                </td>
                <td class="col-sku">
                  <code>{{ item.item_sku }}</code>
                </td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-stock">
                  <span v-if="item.quantity_on_hand !== null">
                    <span :class="['stock-indicator', { 'stock-low': item.isBelowReorder }]">
                      {{ item.quantity_on_hand }} on hand
                    </span>
                    <span class="reorder-point"> (reorder at {{ item.reorder_point }})</span>
                  </span>
                  <span v-else class="text-muted">No inventory data</span>
                </td>
                <td class="col-qty">
                  {{ item.forecasted_demand.toLocaleString() }}
                </td>
                <td class="col-cost">
                  <span v-if="item.unit_cost !== null">${{ item.unit_cost.toFixed(2) }}</span>
                  <span v-else class="text-muted">N/A</span>
                </td>
                <td class="col-total">
                  <span v-if="item.lineTotal !== null">
                    <strong>${{ item.lineTotal.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
                  </span>
                  <span v-else class="text-muted">N/A</span>
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
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const isSubmitting = ref(false)
    const orderSuccess = ref(false)
    const lastOrderNumber = ref(null)

    const demandForecasts = ref([])
    const inventoryItems = ref([])

    const budget = ref(50000)
    const selectedSkus = ref(new Set())
    const manuallyDeselected = ref(new Set())

    // Join demand forecasts with inventory data on item_sku === sku
    const enrichedItems = computed(() => {
      const inventoryMap = new Map(inventoryItems.value.map(inv => [inv.sku, inv]))
      return demandForecasts.value.map(forecast => {
        const inv = inventoryMap.get(forecast.item_sku) || null
        const unit_cost = inv ? inv.unit_cost : null
        const quantity_on_hand = inv ? inv.quantity_on_hand : null
        const reorder_point = inv ? inv.reorder_point : null
        const isBelowReorder = inv ? inv.quantity_on_hand < inv.reorder_point : false
        const lineTotal = unit_cost !== null ? forecast.forecasted_demand * unit_cost : null
        return {
          forecasted_demand: forecast.forecasted_demand,
          item_sku: forecast.item_sku,
          item_name: forecast.item_name,
          unit_cost,
          quantity_on_hand,
          reorder_point,
          isBelowReorder,
          lineTotal,
          trend: forecast.trend
        }
      })
    })

    const selectedTotal = computed(() => {
      return enrichedItems.value
        .filter(item => selectedSkus.value.has(item.item_sku) && item.lineTotal !== null)
        .reduce((sum, item) => sum + item.lineTotal, 0)
    })

    const budgetRemaining = computed(() => budget.value - selectedTotal.value)

    // Priority order: increasing(0) > below reorder(1) > stable(2) > decreasing(3)
    const trendPriority = (item) => {
      if (item.trend === 'increasing') return 0
      if (item.isBelowReorder) return 1
      if (item.trend === 'stable') return 2
      return 3 // decreasing
    }

    const runAutoSelect = () => {
      const sorted = [...enrichedItems.value].sort((a, b) => {
        const pa = trendPriority(a)
        const pb = trendPriority(b)
        if (pa !== pb) return pa - pb
        // Within same priority bucket: cheaper first
        const la = a.lineTotal !== null ? a.lineTotal : Infinity
        const lb = b.lineTotal !== null ? b.lineTotal : Infinity
        return la - lb
      })

      const newSelected = new Set()
      let remaining = budget.value

      for (const item of sorted) {
        if (item.unit_cost === null) continue
        if (manuallyDeselected.value.has(item.item_sku)) continue
        if (item.lineTotal !== null && item.lineTotal <= remaining) {
          newSelected.add(item.item_sku)
          remaining -= item.lineTotal
        }
      }

      selectedSkus.value = newSelected
    }

    const onBudgetChange = () => {
      manuallyDeselected.value = new Set()
      runAutoSelect()
    }

    const toggleItem = (sku) => {
      const isCurrentlySelected = selectedSkus.value.has(sku)
      const newSelected = new Set(selectedSkus.value)

      if (isCurrentlySelected) {
        // Remove from selected, add to manually deselected
        newSelected.delete(sku)
        const newDeselected = new Set(manuallyDeselected.value)
        newDeselected.add(sku)
        manuallyDeselected.value = newDeselected
      } else {
        // Re-add to selected, remove from manually deselected
        newSelected.add(sku)
        const newDeselected = new Set(manuallyDeselected.value)
        newDeselected.delete(sku)
        manuallyDeselected.value = newDeselected
      }

      selectedSkus.value = newSelected
    }

    const clearAll = () => {
      selectedSkus.value = new Set()
      manuallyDeselected.value = new Set(enrichedItems.value.map(i => i.item_sku))
    }

    const placeOrder = async () => {
      if (selectedSkus.value.size === 0 || isSubmitting.value) return

      isSubmitting.value = true
      error.value = null

      try {
        const items = enrichedItems.value
          .filter(item => selectedSkus.value.has(item.item_sku))
          .map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_price: item.unit_cost
          }))

        const response = await api.createRestockingOrder({ items })
        orderSuccess.value = true
        lastOrderNumber.value = response.order_number || response.id || 'N/A'
        selectedSkus.value = new Set()
        manuallyDeselected.value = new Set()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        isSubmitting.value = false
      }
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecastsData
        inventoryItems.value = inventoryData
        // Run initial auto-select after data loaded
        runAutoSelect()
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      isSubmitting,
      orderSuccess,
      lastOrderNumber,
      budget,
      selectedSkus,
      enrichedItems,
      selectedTotal,
      budgetRemaining,
      onBudgetChange,
      runAutoSelect,
      toggleItem,
      clearAll,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* inherits .main-content padding from App.vue */
}

/* ---- Success Banner ---- */
.success-banner {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.5rem;
  font-size: 0.938rem;
  font-weight: 500;
}

/* ---- Budget Card ---- */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-summary {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.938rem;
}

.budget-label {
  color: #64748b;
}

.budget-remaining {
  font-size: 1.25rem;
  font-weight: 700;
  color: #059669;
}

.budget-remaining.over-budget {
  color: #dc2626;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.slider-bound {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-weight: 700;
  color: #0f172a;
  font-size: 1rem;
  min-width: 90px;
  text-align: right;
}

.budget-bar-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 3px;
  transition: width 0.2s ease;
}

.budget-bar-fill.over-budget-bar {
  background: #dc2626;
}

/* ---- Controls Row ---- */
.controls-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.25rem;
  gap: 1rem;
}

.controls-left {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}

.controls-right {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.selection-summary {
  font-size: 0.938rem;
  color: #64748b;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-secondary {
  background: white;
  color: #2563eb;
  border: 1px solid #2563eb;
  padding: 0.625rem 1.25rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-secondary:hover {
  background: #eff6ff;
}

.btn-ghost {
  background: transparent;
  color: #64748b;
  border: 1px solid #e2e8f0;
  padding: 0.625rem 1.25rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-ghost:hover {
  background: #f8fafc;
  color: #0f172a;
}

/* ---- Table ---- */
.table-hint {
  font-size: 0.813rem;
  color: #64748b;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check { width: 40px; }
.col-name  { width: 220px; }
.col-sku   { width: 110px; }
.col-trend { width: 110px; }
.col-stock { width: 200px; }
.col-qty   { width: 110px; }
.col-cost  { width: 100px; }
.col-total { width: 130px; }

/* Row states */
.row-selected {
  background: #eff6ff !important;
}

.row-low-stock {
  border-left: 3px solid #f59e0b;
}

tbody tr {
  cursor: pointer;
}

/* Stock indicators */
.stock-indicator {
  font-size: 0.875rem;
  color: #334155;
}

.stock-indicator.stock-low {
  color: #dc2626;
  font-weight: 600;
}

.reorder-point {
  font-size: 0.813rem;
  color: #64748b;
}

.text-muted {
  color: #94a3b8;
  font-size: 0.813rem;
}

code {
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}
</style>
