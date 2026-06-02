<template>
  <div class="restocking">

    <!-- Success state: replaces normal UI -->
    <div v-if="orderSuccess" class="success-card">
      <div class="success-icon-wrap">
        <svg class="success-icon" viewBox="0 0 48 48" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
          <circle cx="24" cy="24" r="24" fill="#d1fae5"/>
          <path d="M14 25l7 7 13-14" stroke="#059669" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
      </div>
      <h2 class="success-heading">{{ t('restocking.orderSuccess') }}</h2>
      <div class="success-details">
        <div class="success-row">
          <span class="success-label">{{ t('restocking.orderNumber') }}</span>
          <span class="success-value">{{ lastOrder && lastOrder.order_number }}</span>
        </div>
        <div class="success-row">
          <span class="success-label">{{ t('restocking.leadTime') }}</span>
          <span class="success-value">{{ lastOrder && t('restocking.leadTimeDays', { days: lastOrder.lead_time_days }) }}</span>
        </div>
        <div class="success-row">
          <span class="success-label">{{ t('restocking.expectedDelivery') }}</span>
          <span class="success-value">{{ lastOrder && formatDate(lastOrder.expected_delivery) }}</span>
        </div>
      </div>
      <button class="btn-primary" @click="goToOrders">{{ t('restocking.viewOrders') }}</button>
    </div>

    <!-- Normal UI -->
    <template v-else>
      <div class="page-header">
        <h2>{{ t('restocking.title') }}</h2>
        <p>{{ t('restocking.description') }}</p>
      </div>

      <!-- Budget card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-amount">{{ formatCurrency(selectedBudget) }}</div>
          <input
            type="range"
            class="budget-slider"
            v-model.number="selectedBudget"
            min="10000"
            max="500000"
            step="5000"
          />
          <div class="slider-labels">
            <span>{{ t('restocking.budgetMin') }}</span>
            <span>{{ t('restocking.budgetMax') }}</span>
          </div>
          <div class="budget-bar-track">
            <div
              class="budget-bar-fill"
              :class="{ amber: budgetUtilizationPct >= 90 }"
              :style="{ width: budgetUtilizationPct + '%' }"
            ></div>
          </div>
          <div class="budget-used-label">
            {{ t('restocking.budgetUsed') }}: {{ formatCurrency(totalCost) }} of {{ formatCurrency(selectedBudget) }}
          </div>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>

        <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
        <div v-else-if="error && recommendations.length === 0" class="error">{{ error }}</div>
        <div v-else-if="recommendations.length === 0" class="no-recs">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>{{ t('restocking.table.itemName') }}</th>
                  <th>{{ t('restocking.table.sku') }}</th>
                  <th>{{ t('restocking.table.trend') }}</th>
                  <th>{{ t('restocking.table.demandGap') }}</th>
                  <th>{{ t('restocking.table.recommendedQty') }}</th>
                  <th>{{ t('restocking.table.unitCost') }}</th>
                  <th>{{ t('restocking.table.subtotal') }}</th>
                  <th>{{ t('restocking.table.leadTime') }}</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendations" :key="item.item_sku">
                  <td>{{ item.item_name }}</td>
                  <td><strong>{{ item.item_sku }}</strong></td>
                  <td>
                    <span :class="['badge', trendBadgeClass(item.trend)]">{{ item.trend }}</span>
                  </td>
                  <td>{{ item.demandGap }}</td>
                  <td><strong>{{ item.recommendedQty }}</strong></td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td>{{ formatCurrency(item.subtotal) }}</td>
                  <td>{{ t('restocking.leadTimeDays', { days: item.leadTimeDays }) }}</td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Order summary panel -->
          <div class="order-summary">
            <div class="summary-title">{{ t('restocking.orderSummary') }}</div>
            <div class="summary-row">
              <span class="summary-label">{{ t('restocking.totalCost') }}</span>
              <span class="summary-total">{{ formatCurrency(totalCost) }}</span>
            </div>
            <div class="summary-row">
              <span class="summary-label">{{ t('restocking.remainingBudget') }}</span>
              <span class="summary-remaining">{{ formatCurrency(remainingBudget) }}</span>
            </div>
            <div v-if="error" class="error">{{ error }}</div>
            <button
              class="btn-primary btn-full"
              :disabled="recommendations.length === 0 || submitting"
              @click="placeOrder"
            >
              <span v-if="submitting">{{ t('restocking.placing') }}</span>
              <span v-else>{{ t('restocking.placeOrder') }}</span>
            </button>
          </div>
        </div>
      </div>
    </template>

  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

// Category-specific lead times in days
const CATEGORY_LEAD_TIMES = {
  'Sensors': 14,
  'Controllers': 21,
  'Actuators': 10,
  'Circuit Boards': 28
}
const DEFAULT_LEAD_TIME = 14

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()
    const router = useRouter()

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const lastOrder = ref(null)
    const forecasts = ref([])
    const selectedBudget = ref(50000)

    // Greedy budget allocation: favour increasing trend, then largest demand gap
    const recommendations = computed(() => {
      if (selectedBudget.value === 0 || forecasts.value.length === 0) return []

      const scored = forecasts.value
        .filter(f => f.trend !== 'decreasing')
        .map(f => {
          const demandGap = Math.max(0, f.forecasted_demand - f.current_demand)
          const trendScore = f.trend === 'increasing' ? 2 : 1
          return { ...f, demandGap, trendScore }
        })
        .sort((a, b) => {
          if (b.trendScore !== a.trendScore) return b.trendScore - a.trendScore
          return b.demandGap - a.demandGap
        })

      let remaining = selectedBudget.value
      const result = []
      for (const item of scored) {
        if (remaining <= 0) break
        if (!item.unit_cost || item.unit_cost <= 0) continue
        const maxAffordable = Math.floor(remaining / item.unit_cost)
        if (maxAffordable === 0) continue
        const targetQty = item.demandGap > 0 ? item.demandGap : Math.ceil(item.current_demand * 0.2)
        const qty = Math.min(targetQty, maxAffordable)
        const subtotal = qty * item.unit_cost
        const leadDays = CATEGORY_LEAD_TIMES[item.category] ?? DEFAULT_LEAD_TIME
        result.push({ ...item, recommendedQty: qty, subtotal, leadTimeDays: leadDays })
        remaining -= subtotal
      }
      return result
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.subtotal, 0)
    )

    const remainingBudget = computed(() => selectedBudget.value - totalCost.value)

    // Clamp to [0, 100] to keep the bar within bounds
    const budgetUtilizationPct = computed(() =>
      Math.min(100, (totalCost.value / selectedBudget.value) * 100)
    )

    const formatCurrency = (val) =>
      val.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })

    const formatDate = (dateStr) => {
      if (!dateStr) return ''
      const d = new Date(dateStr)
      if (isNaN(d.getTime())) return dateStr
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    // Map trend value to badge modifier class
    const trendBadgeClass = (trend) => {
      if (trend === 'increasing') return 'success'
      if (trend === 'stable') return 'info'
      return 'danger'
    }

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts'
        console.error('Demand forecast load error:', err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return
      try {
        submitting.value = true
        error.value = null
        const payload = {
          items: recommendations.value.map(r => ({
            sku: r.item_sku,
            name: r.item_name,
            quantity: r.recommendedQty,
            unit_cost: r.unit_cost
          })),
          total_value: totalCost.value,
          warehouse: 'San Francisco'
        }
        const result = await api.createRestockingOrder(payload)
        lastOrder.value = result
        orderSuccess.value = true
      } catch (err) {
        error.value = 'Failed to place order. Please try again.'
        console.error('Restocking order error:', err)
      } finally {
        submitting.value = false
      }
    }

    const goToOrders = () => router.push('/orders')

    onMounted(loadForecasts)

    return {
      t,
      loading,
      error,
      submitting,
      orderSuccess,
      lastOrder,
      forecasts,
      selectedBudget,
      recommendations,
      totalCost,
      remainingBudget,
      budgetUtilizationPct,
      formatCurrency,
      formatDate,
      trendBadgeClass,
      placeOrder,
      goToOrders
    }
  }
}
</script>

<style scoped>
/* ---- Budget card body ---- */
.budget-body {
  padding: 0.25rem 0 0.5rem;
}

.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.03em;
  margin-bottom: 1.25rem;
}

/* Range slider — remove native appearance, use custom track/thumb */
.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  transition: background 0.2s, transform 0.1s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #2563eb;
  transform: scale(1.1);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  transition: background 0.2s;
}

.budget-slider::-moz-range-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1.25rem;
}

/* Budget utilisation bar */
.budget-bar-track {
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  overflow: hidden;
  margin-bottom: 0.625rem;
}

.budget-bar-fill {
  height: 100%;
  border-radius: 4px;
  background: #10b981;
  transition: width 0.3s ease, background 0.3s ease;
}

.budget-bar-fill.amber {
  background: #f59e0b;
}

.budget-used-label {
  font-size: 0.813rem;
  color: #64748b;
}

/* ---- No-recommendations message ---- */
.no-recs {
  padding: 2.5rem;
  text-align: center;
  color: #94a3b8;
  font-size: 0.938rem;
}

/* ---- Order summary panel ---- */
.order-summary {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1.5rem;
  margin-top: 1.25rem;
}

.summary-title {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1rem;
  padding-bottom: 0.75rem;
  border-bottom: 1px solid #e2e8f0;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.625rem;
}

.summary-label {
  font-size: 0.875rem;
  color: #64748b;
}

.summary-total {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.summary-remaining {
  font-size: 1rem;
  font-weight: 600;
  color: #10b981;
}

/* ---- Primary button ---- */
.btn-primary {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  background: #3b82f6;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
  margin-top: 1rem;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

.btn-full {
  width: 100%;
}

/* ---- Success card ---- */
.success-card {
  max-width: 560px;
  margin: 3rem auto;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-left: 5px solid #10b981;
  border-radius: 10px;
  padding: 2.5rem 2rem;
  text-align: center;
}

.success-icon-wrap {
  display: flex;
  justify-content: center;
  margin-bottom: 1.25rem;
}

.success-icon {
  width: 56px;
  height: 56px;
}

.success-heading {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1.5rem;
  letter-spacing: -0.025em;
}

.success-details {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1.25rem;
  margin-bottom: 1.5rem;
  text-align: left;
}

.success-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid #f1f5f9;
}

.success-row:last-child {
  border-bottom: none;
  padding-bottom: 0;
}

.success-label {
  font-size: 0.875rem;
  color: #64748b;
}

.success-value {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}
</style>
