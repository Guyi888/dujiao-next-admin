<script setup lang="ts">
import { reactive, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { Package } from 'lucide-vue-next'
import { adminAPI } from '@/api/admin'
import type { AdminOrder, AdminOrderItem, AdminFulfillment, AdminProcurementOrder, AdminPayment } from '@/api/types'
import IdCell from '@/components/IdCell.vue'
import { Button } from '@/components/ui/button'
import { Dialog, DialogScrollContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'
import { Input } from '@/components/ui/input'
import { Card, CardContent } from '@/components/ui/card'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import {
  orderStatusClass,
  orderStatusLabel,
  paymentStatusClass as paymentStatusClassMap,
  paymentStatusLabel as paymentStatusLabelMap,
} from '@/utils/status'
import {
  fulfillmentStatusLabel as fulfillmentStatusLabelMap,
  fulfillmentTypeLabel as fulfillmentTypeLabelMap,
} from '@/utils/fulfillment'
import { formatDate, formatMoney, getLocalizedText, hasPositiveAmount } from '@/utils/format'
import { resolveSkuCodeFromSnapshot, resolveSkuSpecFromSnapshot } from '@/utils/sku'

const props = defineProps<{
  modelValue: boolean
  order: AdminOrder | null
  siteCurrency: string
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', value: boolean): void
  (e: 'refresh'): void
  (e: 'openFulfillment', order: AdminOrder, parentId?: number): void
}>()

const { t, locale } = useI18n()

const detailLoading = ref(false)
const detailError = ref('')
const selectedOrder = ref<AdminOrder | null>(null)
const procurementOrder = ref<AdminProcurementOrder | null | undefined>(null)

const refundSubmitting = ref(false)
const refundError = ref('')
const refundSuccess = ref('')
const refundForm = reactive({
  amount: '',
  remark: '',
})

const adminPath = import.meta.env.VITE_ADMIN_PATH || ''

const userDetailLink = (userId: number) => `${adminPath}/users/${userId}`
const productLink = (productId: number) => `${adminPath}/products?product_id=${productId}`
const couponCodeLink = (code: string) => `${adminPath}/coupons?code=${encodeURIComponent(code)}`
const promotionLink = (promotionId: number) => `${adminPath}/promotions?id=${promotionId}`
const paymentLink = (paymentId: number) => `${adminPath}/payments?payment_id=${paymentId}`

const statusLabel = (status: string) => orderStatusLabel(t, status)
const statusClass = (status: string) => orderStatusClass(status)
const fulfillmentTypeLabel = (type: string) => fulfillmentTypeLabelMap(t, type, 'admin.orders')
const fulfillmentStatusLabel = (status: string) => fulfillmentStatusLabelMap(t, status, 'admin.orders')
const paymentStatusLabel = (status: string) => paymentStatusLabelMap(t, status)
const paymentStatusClass = (status: string) => paymentStatusClassMap(status)

const providerTypeLabel = (value?: string) => {
  const map: Record<string, string> = {
    official: t('admin.paymentChannels.providerTypes.official'),
    epay: t('admin.paymentChannels.providerTypes.epay'),
    epusdt: t('admin.paymentChannels.providerTypes.epusdt'),
    tokenpay: t('admin.paymentChannels.providerTypes.tokenpay'),
    wallet: t('admin.paymentChannels.providerTypes.wallet'),
  }
  if (!value) return '-'
  return map[value] || value
}

const channelTypeLabel = (value?: string) => {
  const map: Record<string, string> = {
    wechat: t('admin.paymentChannels.channelTypes.wechat'),
    alipay: t('admin.paymentChannels.channelTypes.alipay'),
    qqpay: t('admin.paymentChannels.channelTypes.qqpay'),
    paypal: t('admin.paymentChannels.channelTypes.paypal'),
    stripe: t('admin.paymentChannels.channelTypes.stripe'),
    usdt: t('admin.paymentChannels.channelTypes.usdt'),
    'usdt-trc20': t('admin.paymentChannels.channelTypes.usdtTrc20'),
    'usdc-trc20': t('admin.paymentChannels.channelTypes.usdcTrc20'),
    trx: t('admin.paymentChannels.channelTypes.trx'),
    balance: t('admin.paymentChannels.channelTypes.balance'),
  }
  if (!value) return '-'
  return map[value] || value
}

const hasWalletPayment = (order: AdminOrder) => hasPositiveAmount(order?.wallet_paid_amount)
const hasOnlinePayment = (order: AdminOrder) => hasPositiveAmount(order?.online_paid_amount)

const orderPaymentMethodLabel = (order: AdminOrder) => {
  if (hasWalletPayment(order) && hasOnlinePayment(order)) {
    return t('admin.orders.paymentMethodMixed')
  }
  if (hasWalletPayment(order)) {
    return t('admin.orders.paymentMethodWalletOnly')
  }
  if (hasOnlinePayment(order)) {
    return t('admin.orders.paymentMethodOnlineOnly')
  }
  return t('admin.orders.paymentMethodUnknown')
}

const canCreateFulfillment = (order: AdminOrder | null) => {
  if (!order) return false
  if (order.fulfillment) return false
  if (order.parent_id == null && Array.isArray(order.children) && order.children.length > 0) return false
  if (Array.isArray(order.items) && order.items.length > 0 && !order.items.every((item: AdminOrderItem) => item.fulfillment_type === 'manual')) {
    return false
  }
  return order.status === 'paid' || order.status === 'fulfilling'
}

const canCreateChildFulfillment = (order: AdminOrder | null) => {
  if (!canCreateFulfillment(order)) return false
  if (!order || !order.items || !order.items.length) return false
  return order.items.every((item: AdminOrderItem) => item.fulfillment_type === 'manual')
}

const formatManualValue = (value: unknown) => {
  if (Array.isArray(value)) {
    return value.map((item) => String(item)).join(', ')
  }
  if (value === null || value === undefined) {
    return '-'
  }
  if (typeof value === 'object') {
    try {
      return JSON.stringify(value)
    } catch {
      return String(value)
    }
  }
  return String(value)
}

const manualSubmissionRows = (submission: Record<string, unknown> | null | undefined) => {
  if (!submission || typeof submission !== 'object') return []
  return Object.entries(submission)
    .filter(([key]) => String(key).trim() !== '')
    .map(([key, value]) => ({
      key: String(key),
      value: formatManualValue(value),
    }))
}

const parseOrderItemSkuId = (item: AdminOrderItem & Record<string, unknown>) => {
  const snapshot = item?.sku_snapshot as Record<string, unknown> | undefined
  const value = Number(item?.sku_id || snapshot?.sku_id || 0)
  if (!Number.isFinite(value)) return 0
  const normalized = Math.trunc(value)
  return normalized > 0 ? normalized : 0
}

const orderItemSkuCodeText = (item: AdminOrderItem & Record<string, unknown>) => {
  const code = resolveSkuCodeFromSnapshot(item?.sku_snapshot, {
    defaultLabel: t('admin.orders.itemSkuDefaultCode'),
  })
  if (code) return code
  const skuId = parseOrderItemSkuId(item)
  if (skuId > 0) return `#${skuId}`
  return t('admin.orders.itemSkuUnknown')
}

const orderItemSkuSpecText = (item: AdminOrderItem & Record<string, unknown>) => {
  const specText = resolveSkuSpecFromSnapshot(item?.sku_snapshot, locale.value)
  if (specText) return specText
  return t('admin.orders.itemSkuSpecEmpty')
}

const fulfillmentDeliveryLines = (fulfillment: AdminFulfillment & Record<string, unknown>) => {
  const lines: string[] = []
  const logistics = (fulfillment?.delivery_data || fulfillment?.logistics) as Record<string, unknown> | undefined
  if (logistics && typeof logistics === 'object') {
    const note = String(logistics.note || '').trim()
    if (note) {
      lines.push(note)
    }
    const entries = Array.isArray(logistics.entries) ? logistics.entries : []
    entries.forEach((item: Record<string, unknown>) => {
      const key = String(item?.key || '').trim()
      const value = String(item?.value || '').trim()
      if (!key && !value) {
        return
      }
      if (!key) {
        lines.push(value)
      } else if (!value) {
        lines.push(key)
      } else {
        lines.push(`${key}: ${value}`)
      }
    })

    Object.entries(logistics).forEach(([key, value]) => {
      if (key === 'note' || key === 'entries') return
      const text = String(value ?? '').trim()
      if (!text) return
      lines.push(`${key}: ${text}`)
    })
  }
  return lines
}

const parseMoneyValue = (value?: string | number) => {
  if (value === null || value === undefined || value === '') return 0
  const parsed = Number(value)
  if (Number.isNaN(parsed)) return 0
  return parsed
}

const isPaidOrder = (order: AdminOrder | null) => {
  if (!order) return false
  return Boolean(order.paid_at)
}

const refundableAmountValue = (order: AdminOrder | null) => {
  if (!order) return 0
  if (!isPaidOrder(order)) return 0
  const total = parseMoneyValue(order.total_amount)
  const refunded = parseMoneyValue(order.refunded_amount)
  const value = total - refunded
  if (!Number.isFinite(value)) return 0
  return Number(Math.max(value, 0).toFixed(2))
}

const refundableAmountDisplay = (order: AdminOrder | null) => formatMoney(refundableAmountValue(order).toFixed(2), order?.currency)

const canRefundToWallet = (order: AdminOrder | null) => {
  if (!order) return false
  if (!order.user_id) return false
  if (!isPaidOrder(order)) return false
  return refundableAmountValue(order) > 0
}

const resetRefundForm = () => {
  refundForm.amount = ''
  refundForm.remark = ''
  refundError.value = ''
  refundSuccess.value = ''
}

const formatFeeRate = (channel: AdminPayment | { fee_rate: number | string; fixed_fee?: number | string }) => {
  const feeRate = channel.fee_rate
  const fixedFee = channel.fixed_fee

  let display = '-'
  if (feeRate !== undefined && feeRate !== null && feeRate !== '') {
    const rateParsed = Number(feeRate)
    if (!Number.isNaN(rateParsed)) {
      display = `${rateParsed.toFixed(2)}%`
    }
  }

  if (fixedFee !== undefined && fixedFee !== null && fixedFee !== '') {
    const fixedParsed = Number(fixedFee)
    if (!Number.isNaN(fixedParsed) && fixedParsed > 0) {
      if (display === '-') {
        display = fixedParsed.toFixed(2)
      } else {
        display += ` + ${fixedParsed.toFixed(2)}`
      }
    }
  }

  return display
}

const fetchOrderDetail = async (orderId: number) => {
  detailLoading.value = true
  detailError.value = ''
  selectedOrder.value = null
  procurementOrder.value = null
  try {
    const response = await adminAPI.getOrder(orderId)
    selectedOrder.value = response.data.data
    try {
      const procRes = await adminAPI.getProcurementOrders({ order_no: selectedOrder.value?.order_no, page_size: 1 })
      const procList = procRes.data.data
      if (Array.isArray(procList) && procList.length > 0) {
        procurementOrder.value = procList[0]
      }
    } catch {
      // silently ignore - procurement order may not exist
    }
  } catch (err: any) {
    detailError.value = err?.message || t('admin.orders.detailFetchFailed')
  } finally {
    detailLoading.value = false
  }
}

const submitRefundToWallet = async () => {
  if (!selectedOrder.value) return
  refundError.value = ''
  refundSuccess.value = ''
  if (!canRefundToWallet(selectedOrder.value)) {
    refundError.value = t('admin.orders.refundNoRemaining')
    return
  }
  const amount = refundForm.amount.trim()
  const value = Number(amount)
  if (!amount || Number.isNaN(value) || value <= 0) {
    refundError.value = t('admin.orders.refundInvalidAmount')
    return
  }
  if (value > refundableAmountValue(selectedOrder.value)) {
    refundError.value = t('admin.orders.refundExceeded')
    return
  }

  refundSubmitting.value = true
  try {
    await adminAPI.refundOrderToWallet(Number(selectedOrder.value.id), {
      amount,
      remark: refundForm.remark.trim() || undefined,
    })
    refundSuccess.value = t('admin.orders.refundSuccess')
    refundForm.amount = ''
    refundForm.remark = ''
    await fetchOrderDetail(Number(selectedOrder.value.id))
    emit('refresh')
  } catch (err: any) {
    refundError.value = err?.message || t('admin.orders.refundFailed')
  } finally {
    refundSubmitting.value = false
  }
}

const handleClose = () => {
  emit('update:modelValue', false)
  selectedOrder.value = null
  detailError.value = ''
  resetRefundForm()
}

const handleOpenFulfillment = (order: AdminOrder, parentId?: number) => {
  handleClose()
  emit('openFulfillment', order, parentId)
}

// Watch for the order prop to trigger detail fetch
watch(
  () => props.order,
  (newOrder) => {
    if (newOrder && props.modelValue) {
      resetRefundForm()
      fetchOrderDetail(newOrder.id)
    }
  }
)

watch(
  () => props.modelValue,
  (open) => {
    if (open && props.order) {
      resetRefundForm()
      fetchOrderDetail(props.order.id)
    }
    if (!open) {
      handleClose()
    }
  }
)
</script>

<template>
  <Dialog :open="modelValue" @update:open="(value) => { if (!value) handleClose() }">
    <DialogScrollContent class="w-full max-w-5xl">
      <DialogHeader>
        <DialogTitle>{{ t('admin.orders.detailTitle') }}</DialogTitle>
      </DialogHeader>
      <div class="space-y-6">
        <div v-if="detailLoading" class="h-32 rounded-lg border border-border bg-muted/40 animate-pulse"></div>
        <div v-else-if="detailError" class="rounded-lg border border-destructive/30 bg-destructive/10 p-3 text-sm text-destructive">
          {{ detailError }}
        </div>
        <div v-else-if="selectedOrder" class="space-y-6">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 text-sm">
            <Card class="rounded-lg border-border bg-background shadow-none">
              <CardContent class="p-3">
                <div class="text-xs text-muted-foreground">{{ t('admin.orders.table.id') }}</div>
                <div class="text-foreground font-mono mt-1">
                  <IdCell :value="selectedOrder.id" />
                </div>
              </CardContent>
            </Card>
            <Card class="rounded-lg border-border bg-background shadow-none">
              <CardContent class="p-3">
                <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailOrderNo') }}</div>
                <div class="text-foreground font-mono mt-1">{{ selectedOrder.order_no }}</div>
              </CardContent>
            </Card>
            <Card class="rounded-lg border-border bg-background shadow-none">
              <CardContent class="p-3">
                <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailUser') }}</div>
                <div class="text-sm text-foreground">
                  <span v-if="selectedOrder.user_id">
                    {{ t('admin.orders.userLabel') }}:
                    <a :href="userDetailLink(selectedOrder.user_id)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline">
                      #{{ selectedOrder.user_id }}
                    </a>
                  </span>
                  <span v-else>{{ t('admin.orders.guestLabel') }}: {{ selectedOrder.guest_email || '-' }}</span>
                </div>
              </CardContent>
            </Card>
            <Card class="rounded-lg border-border bg-background shadow-none">
              <CardContent class="p-3">
                <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailStatus') }}</div>
                <div class="text-sm text-foreground">{{ statusLabel(selectedOrder.status) }}</div>
              </CardContent>
            </Card>
            <Card class="rounded-lg border-border bg-background shadow-none">
              <CardContent class="p-3">
                <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailCreatedAt') }}</div>
                <div class="text-sm text-foreground">{{ formatDate(selectedOrder.created_at) }}</div>
              </CardContent>
            </Card>
            <Card class="rounded-lg border-border bg-background shadow-none">
              <CardContent class="p-3">
                <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailClientIp') }}</div>
                <div class="text-sm text-foreground">{{ selectedOrder.client_ip || '-' }}</div>
              </CardContent>
            </Card>
          </div>

          <div class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('orderDetail.amountTitle') }}</h3>
            <div class="grid grid-cols-1 md:grid-cols-3 gap-4 text-sm">
              <Card class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('orderDetail.amountOriginal') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.original_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('orderDetail.amountDiscount') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.discount_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('orderDetail.amountTotal') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.total_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card v-if="hasPositiveAmount(selectedOrder.wallet_paid_amount)" class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailWalletPaid') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.wallet_paid_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card v-if="hasPositiveAmount(selectedOrder.online_paid_amount)" class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailOnlinePaid') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.online_paid_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card v-if="hasPositiveAmount(selectedOrder.refunded_amount)" class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailRefunded') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.refunded_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailPaymentMethod') }}</div>
                  <div class="text-foreground mt-1">{{ orderPaymentMethodLabel(selectedOrder) }}</div>
                </CardContent>
              </Card>
            </div>
          </div>

          <div class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('admin.orders.detailDiscountTitle') }}</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 text-sm">
              <Card class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailCouponDiscount') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.discount_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
              <Card class="rounded-lg border-border bg-background shadow-none">
                <CardContent class="p-3">
                  <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailPromotionDiscount') }}</div>
                  <div class="text-foreground font-mono mt-1">{{ formatMoney(selectedOrder.promotion_discount_amount, selectedOrder.currency) }}</div>
                </CardContent>
              </Card>
            </div>
          </div>

          <div class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('orderDetail.itemsTitle') }}</h3>
            <div v-if="selectedOrder.items && selectedOrder.items.length" class="space-y-3">
              <div v-for="item in selectedOrder.items" :key="item.id" class="flex flex-col gap-3 border-b border-border/60 pb-3 text-sm text-muted-foreground">
                <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-3">
                  <div>
                    <a :href="productLink(item.product_id)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline font-medium">
                      {{ getLocalizedText(item.title) }}
                    </a>
                    <div class="text-xs text-muted-foreground mt-1">#{{ item.product_id }}</div>
                    <div class="text-xs text-muted-foreground">{{ t('orderDetail.quantityLabel') }}：{{ item.quantity }}</div>
                    <div class="mt-2 rounded-lg border border-border/70 bg-background/90 px-3 py-2">
                      <div class="flex flex-wrap items-center gap-2">
                        <span class="inline-flex items-center gap-1 rounded-full border border-emerald-200 bg-emerald-50 px-2 py-0.5 text-[11px] font-medium text-emerald-700">
                          <Package class="h-3.5 w-3.5" />
                          {{ t('admin.orders.itemSkuLabel') }}
                        </span>
                        <span class="font-mono text-xs text-foreground">{{ orderItemSkuCodeText(item) }}</span>
                      </div>
                      <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.orders.itemSkuSpec') }}：{{ orderItemSkuSpecText(item) }}</div>
                    </div>
                    <div class="text-xs text-muted-foreground mt-1">
                      {{ t('admin.orders.itemCouponCode') }}：
                      <a v-if="selectedOrder.coupon_code" :href="couponCodeLink(selectedOrder.coupon_code)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline">
                        {{ selectedOrder.coupon_code }}
                      </a>
                      <span v-else>-</span>
                    </div>
                    <div class="text-xs text-muted-foreground mt-1">
                      {{ t('admin.orders.itemPromotionName') }}：
                      <a v-if="item.promotion_id" :href="promotionLink(item.promotion_id)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline">
                        {{ item.promotion_name || `#${item.promotion_id}` }}
                      </a>
                      <span v-else>-</span>
                    </div>
                    <div v-if="item.tags && item.tags.length" class="mt-2 flex flex-wrap gap-2">
                      <span
                        v-for="(tag, index) in item.tags"
                        :key="index"
                        class="rounded-full border border-border/70 bg-background px-2 py-0.5 text-[11px] text-muted-foreground"
                      >
                        {{ tag }}
                      </span>
                    </div>
                    <div v-if="manualSubmissionRows(item.manual_form_submission).length" class="mt-3 rounded-lg border border-border bg-background p-3">
                      <div class="text-xs font-semibold text-muted-foreground mb-2">{{ t('admin.orders.manualSubmissionTitle') }}</div>
                      <div class="space-y-1 text-xs text-muted-foreground">
                        <div v-for="row in manualSubmissionRows(item.manual_form_submission)" :key="`${item.id}-${row.key}`">
                          <span class="text-foreground">{{ row.key }}</span>：{{ row.value }}
                        </div>
                      </div>
                    </div>
                  </div>
                  <div class="text-right space-y-1">
                    <div>{{ t('orderDetail.unitPriceLabel') }}：{{ formatMoney(item.unit_price, selectedOrder.currency) }}</div>
                    <div>{{ t('orderDetail.totalPriceLabel') }}：{{ formatMoney(item.total_price, selectedOrder.currency) }}</div>
                    <div v-if="hasPositiveAmount(item.coupon_discount_amount)">
                      {{ t('orderDetail.couponDiscountLabel') }}：{{ formatMoney(item.coupon_discount_amount, selectedOrder.currency) }}
                    </div>
                    <div v-if="hasPositiveAmount(item.promotion_discount_amount)">
                      {{ t('orderDetail.promotionDiscountLabel') }}：{{ formatMoney(item.promotion_discount_amount, selectedOrder.currency) }}
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div v-else class="text-xs text-muted-foreground">{{ t('orderDetail.noItems') }}</div>
          </div>

          <div v-if="selectedOrder.children && selectedOrder.children.length" class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('orderDetail.childOrdersTitle') }}</h3>
            <div class="space-y-4">
              <div v-for="child in selectedOrder.children" :key="child.id" class="rounded-xl border border-border bg-background p-4">
                <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-3">
                  <div>
                    <div class="text-xs text-muted-foreground">{{ t('orderDetail.childOrderNo') }}：{{ child.order_no }}</div>
                    <div class="text-xs text-muted-foreground mt-1">{{ t('orderDetail.childOrderAmount') }}：{{ formatMoney(child.total_amount, child.currency || selectedOrder.currency) }}</div>
                  </div>
                  <div class="flex items-center gap-2">
                    <span class="inline-flex rounded-full border px-2.5 py-1 text-xs" :class="statusClass(child.status)">
                      {{ statusLabel(child.status) }}
                    </span>
                    <Button v-if="canCreateChildFulfillment(child)" size="sm" variant="secondary" @click="handleOpenFulfillment(child, selectedOrder.id)">
                      {{ t('admin.orders.fulfillmentCreate') }}
                    </Button>
                  </div>
                </div>
                <div class="mt-4">
                  <h4 class="text-xs font-semibold text-muted-foreground mb-2">{{ t('orderDetail.childItemsTitle') }}</h4>
                  <div v-if="child.items && child.items.length" class="space-y-3">
                    <div v-for="item in child.items" :key="item.id" class="flex flex-col md:flex-row md:items-center md:justify-between gap-3 border-b border-border/60 pb-3 text-sm text-muted-foreground">
                      <div>
                        <a :href="productLink(item.product_id)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline font-medium">
                          {{ getLocalizedText(item.title) }}
                        </a>
                        <div class="text-xs text-muted-foreground mt-1">#{{ item.product_id }}</div>
                        <div class="text-xs text-muted-foreground">{{ t('orderDetail.quantityLabel') }}：{{ item.quantity }}</div>
                        <div class="mt-2 rounded-lg border border-border/70 bg-muted/20 px-3 py-2">
                          <div class="flex flex-wrap items-center gap-2">
                            <span class="inline-flex items-center gap-1 rounded-full border border-emerald-200 bg-emerald-50 px-2 py-0.5 text-[11px] font-medium text-emerald-700">
                              <Package class="h-3.5 w-3.5" />
                              {{ t('admin.orders.itemSkuLabel') }}
                            </span>
                            <span class="font-mono text-xs text-foreground">{{ orderItemSkuCodeText(item) }}</span>
                          </div>
                          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.orders.itemSkuSpec') }}：{{ orderItemSkuSpecText(item) }}</div>
                        </div>
                        <div v-if="manualSubmissionRows(item.manual_form_submission).length" class="mt-3 rounded-lg border border-border bg-muted/20 p-3">
                          <div class="text-xs font-semibold text-muted-foreground mb-2">{{ t('admin.orders.manualSubmissionTitle') }}</div>
                          <div class="space-y-1 text-xs text-muted-foreground">
                            <div v-for="row in manualSubmissionRows(item.manual_form_submission)" :key="`${item.id}-${row.key}`">
                              <span class="text-foreground">{{ row.key }}</span>：{{ row.value }}
                            </div>
                          </div>
                        </div>
                      </div>
                      <div class="text-right space-y-1">
                        <div>{{ t('orderDetail.unitPriceLabel') }}：{{ formatMoney(item.unit_price, selectedOrder.currency) }}</div>
                        <div>{{ t('orderDetail.totalPriceLabel') }}：{{ formatMoney(item.total_price, selectedOrder.currency) }}</div>
                        <div v-if="hasPositiveAmount(item.coupon_discount_amount)">
                          {{ t('orderDetail.couponDiscountLabel') }}：{{ formatMoney(item.coupon_discount_amount, selectedOrder.currency) }}
                        </div>
                        <div v-if="hasPositiveAmount(item.promotion_discount_amount)">
                          {{ t('orderDetail.promotionDiscountLabel') }}：{{ formatMoney(item.promotion_discount_amount, selectedOrder.currency) }}
                        </div>
                      </div>
                    </div>
                  </div>
                  <div v-else class="text-xs text-muted-foreground">{{ t('orderDetail.noItems') }}</div>
                </div>
                <div class="mt-4">
                  <h4 class="text-xs font-semibold text-muted-foreground mb-2">{{ t('orderDetail.childFulfillmentTitle') }}</h4>
                  <div v-if="child.fulfillment">
                    <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailFulfillmentType') }}：{{ fulfillmentTypeLabel(child.fulfillment.type) }}</div>
                    <div class="text-xs text-muted-foreground">{{ t('admin.orders.detailFulfillmentStatus') }}：{{ fulfillmentStatusLabel(child.fulfillment.status) }}</div>
                    <div v-if="fulfillmentDeliveryLines(child.fulfillment).length" class="mt-3 rounded-lg border border-border bg-muted/30 p-3 text-xs text-foreground space-y-1">
                      <div v-for="(line, lineIndex) in fulfillmentDeliveryLines(child.fulfillment)" :key="`child-fulfillment-${child.id}-${lineIndex}`">{{ line }}</div>
                    </div>
                    <div v-else-if="child.fulfillment.payload" class="mt-3 rounded-lg border border-border bg-muted/30 p-3 text-xs text-foreground whitespace-pre-wrap">
                      {{ child.fulfillment.payload }}
                    </div>
                  </div>
                  <div v-else class="text-xs text-muted-foreground">{{ t('orderDetail.childFulfillmentEmpty') }}</div>
                </div>
              </div>
            </div>
          </div>

          <div v-if="selectedOrder.fulfillment" class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('admin.orders.detailFulfillmentTitle') }}</h3>
            <div class="text-sm text-muted-foreground">{{ t('admin.orders.detailFulfillmentType') }}：{{ fulfillmentTypeLabel(selectedOrder.fulfillment.type) }}</div>
            <div class="text-sm text-muted-foreground">{{ t('admin.orders.detailFulfillmentStatus') }}：{{ fulfillmentStatusLabel(selectedOrder.fulfillment.status) }}</div>
            <div v-if="fulfillmentDeliveryLines(selectedOrder.fulfillment).length" class="mt-3 rounded-lg border border-border bg-background p-3 text-xs text-muted-foreground space-y-1">
              <div v-for="(line, lineIndex) in fulfillmentDeliveryLines(selectedOrder.fulfillment)" :key="`fulfillment-${selectedOrder.id}-${lineIndex}`">{{ line }}</div>
            </div>
            <div v-else class="mt-3 rounded-lg border border-border bg-background p-3 text-xs text-muted-foreground whitespace-pre-wrap">
              {{ selectedOrder.fulfillment.payload }}
            </div>
          </div>

          <div v-if="procurementOrder" class="rounded-xl border border-indigo-200 bg-indigo-50/50 p-4">
            <h3 class="text-sm font-semibold text-indigo-800 mb-3">{{ t('orderDetail.procurementTitle') }}</h3>
            <div class="grid grid-cols-2 gap-3 text-sm">
              <div>
                <span class="text-xs text-muted-foreground">{{ t('procurement.columns.id') }}:</span>
                <span class="ml-1 font-mono">{{ procurementOrder.id }}</span>
              </div>
              <div>
                <span class="text-xs text-muted-foreground">{{ t('procurement.columns.status') }}:</span>
                <span
                  class="ml-1 inline-flex rounded-full border px-2 py-0.5 text-xs"
                  :class="{
                    'text-yellow-700 border-yellow-200 bg-yellow-50': procurementOrder.status === 'pending',
                    'text-blue-700 border-blue-200 bg-blue-50': ['submitted', 'accepted'].includes(procurementOrder.status),
                    'text-emerald-700 border-emerald-200 bg-emerald-50': ['fulfilled', 'completed'].includes(procurementOrder.status),
                    'text-red-700 border-red-200 bg-red-50': ['failed', 'rejected'].includes(procurementOrder.status),
                    'text-muted-foreground border-border bg-muted/30': procurementOrder.status === 'canceled',
                  }"
                >
                  {{ t('procurement.status.' + procurementOrder.status) }}
                </span>
              </div>
              <div>
                <span class="text-xs text-muted-foreground">{{ t('procurement.columns.connection') }}:</span>
                <span class="ml-1">{{ procurementOrder.connection?.name || procurementOrder.connection_id || '-' }}</span>
              </div>
              <div>
                <span class="text-xs text-muted-foreground">{{ t('procurement.columns.upstreamOrderNo') }}:</span>
                <span class="ml-1 font-mono">{{ procurementOrder.upstream_order_no || '-' }}</span>
              </div>
              <div>
                <span class="text-xs text-muted-foreground">{{ t('procurement.columns.upstreamAmount') }}:</span>
                <span class="ml-1">{{ procurementOrder.upstream_amount || '-' }}</span>
              </div>
              <div v-if="procurementOrder.error_message">
                <span class="text-xs text-muted-foreground">{{ t('procurement.columns.errorMessage') }}:</span>
                <span class="ml-1 text-red-600 text-xs">{{ procurementOrder.error_message }}</span>
              </div>
            </div>
          </div>

          <div class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('admin.orders.detailPayments') }}</h3>
            <div
              v-if="hasWalletPayment(selectedOrder) && (!selectedOrder.payments || selectedOrder.payments.length === 0)"
              class="mb-3 rounded-lg border border-amber-200 bg-amber-50 p-3 text-xs text-amber-700"
            >
              {{ t('admin.orders.detailWalletOnlyNoOnlinePayments') }}
            </div>
            <div v-if="selectedOrder.payments && selectedOrder.payments.length">
              <Table>
                <TableHeader class="bg-muted/40 text-xs uppercase text-muted-foreground">
                  <TableRow>
                    <TableHead class="px-4 py-3">{{ t('admin.payments.table.paymentId') }}</TableHead>
                    <TableHead class="px-4 py-3">{{ t('admin.payments.table.channel') }}</TableHead>
                    <TableHead class="px-4 py-3">{{ t('admin.payments.table.status') }}</TableHead>
                    <TableHead class="px-4 py-3">{{ t('admin.payments.table.feeRate') }}</TableHead>
                    <TableHead class="px-4 py-3">{{ t('admin.payments.table.amount') }}</TableHead>
                    <TableHead class="px-4 py-3">{{ t('admin.payments.table.createdAt') }}</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody class="divide-y divide-border">
                  <TableRow v-for="payment in selectedOrder.payments" :key="payment.id" class="hover:bg-muted/30">
                    <TableCell class="px-4 py-3">
                      <div class="flex items-center gap-2">
                        <IdCell :value="payment.id" />
                        <a :href="paymentLink(payment.id)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline text-xs">
                          {{ t('admin.payments.view') }}
                        </a>
                      </div>
                    </TableCell>
                    <TableCell class="px-4 py-3 text-xs">
                      <div class="text-foreground">{{ payment.channel_name || '-' }}</div>
                      <div class="text-muted-foreground">{{ providerTypeLabel(payment.provider_type) }} / {{ channelTypeLabel(payment.channel_type) }}</div>
                    </TableCell>
                    <TableCell class="px-4 py-3 text-xs">
                      <span class="inline-flex rounded-full border px-2.5 py-1 text-xs" :class="paymentStatusClass(payment.status)">
                        {{ paymentStatusLabel(payment.status) }}
                      </span>
                    </TableCell>
                    <TableCell class="px-4 py-3 text-xs text-muted-foreground">{{ formatFeeRate(payment) }}</TableCell>
                    <TableCell class="px-4 py-3 font-mono text-foreground">{{ formatMoney(payment.amount, payment.currency) }}</TableCell>
                    <TableCell class="px-4 py-3 text-xs text-muted-foreground">{{ formatDate(payment.created_at) }}</TableCell>
                  </TableRow>
                </TableBody>
              </Table>
            </div>
            <div v-else class="text-xs text-muted-foreground">{{ t('admin.payments.empty') }}</div>
          </div>

          <div v-if="selectedOrder.user_id" class="rounded-xl border border-border bg-muted/20 p-4">
            <h3 class="text-sm font-semibold text-foreground mb-3">{{ t('admin.orders.refundToWalletTitle') }}</h3>
            <div class="mb-3 text-xs text-muted-foreground">
              {{ t('admin.orders.refundableAmount') }}：{{ refundableAmountDisplay(selectedOrder) }}
            </div>
            <form class="grid grid-cols-1 gap-3 md:grid-cols-[1fr_2fr_auto]" @submit.prevent="submitRefundToWallet">
              <Input
                v-model="refundForm.amount"
                :placeholder="t('admin.orders.refundAmountPlaceholder')"
                :disabled="refundSubmitting || !canRefundToWallet(selectedOrder)"
              />
              <Input
                v-model="refundForm.remark"
                :placeholder="t('admin.orders.refundRemarkPlaceholder')"
                :disabled="refundSubmitting || !canRefundToWallet(selectedOrder)"
              />
              <Button
                type="submit"
                :disabled="refundSubmitting || !canRefundToWallet(selectedOrder)"
              >
                {{ refundSubmitting ? t('admin.orders.refunding') : t('admin.orders.refundSubmit') }}
              </Button>
            </form>
            <div v-if="!canRefundToWallet(selectedOrder)" class="mt-3 text-xs text-muted-foreground">
              {{ t('admin.orders.refundNoRemaining') }}
            </div>
            <div v-if="refundError" class="mt-3 rounded-lg border border-destructive/30 bg-destructive/10 p-3 text-sm text-destructive">
              {{ refundError }}
            </div>
            <div v-if="refundSuccess" class="mt-3 rounded-lg border border-emerald-200 bg-emerald-50 p-3 text-sm text-emerald-700">
              {{ refundSuccess }}
            </div>
          </div>
        </div>
      </div>
    </DialogScrollContent>
  </Dialog>
</template>
