<template>
    <div class="p-6 max-w-7xl mx-auto">
        <h2 class="text-3xl font-bold mb-6">📊 Market Report (Territory-wise) with Month-Year Subtotals</h2>
        <FilterComponent v-model="filters" />
        <div class="overflow-x-auto border rounded shadow">
            <table class="min-w-full table-fixed border-collapse text-sm">
                <thead class="bg-blue-600 text-white sticky top-0 z-10">
                    <tr>
                        <th class="border border-blue-700 px-4 py-2 w-40 text-left">Company</th>
                        <th class="border border-blue-700 px-4 py-2 w-36 text-left">Zone</th>
                        <th class="border border-blue-700 px-4 py-2 w-36 text-left">Wing</th>
                        <th class="border border-blue-700 px-4 py-2 w-36 text-left">Division</th>
                        <th class="border border-blue-700 px-4 py-2 w-36 text-left">Region</th>
                        <th class="border border-blue-700 px-4 py-2 w-36 text-left">Area</th>
                        <th class="border border-blue-700 px-4 py-2 w-36 text-left">Territory</th>
                        <th class="border border-blue-700 px-4 py-2 w-32 text-left">Month-Year</th>
                        <th class="border border-blue-700 px-4 py-2 w-24 text-right">Qty</th>
                    </tr>
                </thead>
                <tbody>
                    <template v-for="(group, monthYear) in groupedRows" :key="monthYear">
                        <tr v-for="(row, i) in group" :key="row.id || i"
                            :class="i % 2 === 0 ? 'bg-white' : 'bg-gray-50'">
                            <td class="border border-gray-300 px-4 py-2 font-semibold text-gray-800">{{ row.company }}
                            </td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700">{{ row.zone }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700">{{ row.wing }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700">{{ row.division }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700">{{ row.region }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700">{{ row.area }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700">{{ row.territory }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-gray-700 font-medium">{{ row.month }}-{{
                                row.year }}</td>
                            <td class="border border-gray-300 px-4 py-2 text-right font-mono text-gray-900">{{
                                formatNumber(row.qty) }}</td>
                        </tr>

                        <!-- Subtotal row -->
                        <tr class="bg-blue-100 font-semibold text-blue-900">
                            <td class="border border-gray-300 px-4 py-2 text-right" colspan="8">
                                Subtotal for {{ monthYear }}:
                            </td>
                            <td class="border border-gray-300 px-4 py-2 text-right">
                                {{ formatNumber(monthYearSubtotals[monthYear]) }}
                            </td>
                        </tr>
                    </template>
                </tbody>
            </table>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onMounted,watchEffect } from 'vue'
import FilterComponent from '../components/filter/FilterComponent.vue'

const rawData = ref([])
const flatRows = ref([])
const filters = ref({})

function formatNumber(value) {
    return value
        ? value.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })
        : '0.00'
}

function flattenData(data) {
    const rows = []
    for (const company of data) {
        const companyName = company.company_name || '(No Company)'
        if (!Array.isArray(company.zones)) continue

        for (const zone of company.zones) {
            const zoneName = zone.zone_name || '(No Zone)'
            if (!Array.isArray(zone.wings)) continue

            for (const wing of zone.wings) {
                const wingName = wing.wing || '(No Wing)'
                if (!Array.isArray(wing.divisions)) continue

                for (const division of wing.divisions) {
                    const divisionName = division.division || '(No Division)'
                    if (!Array.isArray(division.regions)) continue

                    for (const region of division.regions) {
                        const regionName = region.region || '(No Region)'
                        if (!Array.isArray(region.areas)) continue

                        for (const area of region.areas) {
                            const areaName = area.area || '(No Area)'
                            if (!Array.isArray(area.territories)) continue

                            for (const territory of area.territories) {
                                const territoryName = territory.territory || '(No Territory)'
                                if (!Array.isArray(territory.months)) continue

                                for (const month of territory.months) {
                                    rows.push({
                                        id: `${companyName}-${zoneName}-${wingName}-${divisionName}-${regionName}-${areaName}-${territoryName}-${month.month}-${month.year}`,
                                        company: companyName,
                                        zone: zoneName,
                                        wing: wingName,
                                        division: divisionName,
                                        region: regionName,
                                        area: areaName,
                                        territory: territoryName,
                                        month: month.month || '',
                                        year: month.year || '',
                                        qty: month.do_qty || 0,
                                    })
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    return rows
}

const groupedRows = computed(() => {
    const groups = {}
    for (const r of flatRows.value) {
        const key = `${r.month}-${r.year}`
        if (!groups[key]) groups[key] = []
        groups[key].push(r)
    }
    return groups
})

const monthYearSubtotals = computed(() => {
    const subtotals = {}
    for (const monthYear in groupedRows.value) {
        subtotals[monthYear] = groupedRows.value[monthYear].reduce((sum, r) => sum + r.qty, 0)
    }
    return subtotals
})

async function fetchData() {
    try {
        const params = new URLSearchParams()

        // Convert filters object into query parameters
        for (const [key, value] of Object.entries(filters.value)) {
            if (value) params.append(key, value)
        }
        const res = await fetch(`http://127.0.0.1:8000/api/market/territory?${params.toString()}`) // Adjust to your endpoint
        const json = await res.json()
        const treeData = json.tree || json
        rawData.value = treeData
        flatRows.value = flattenData(treeData)
    } catch (e) {
        console.error('Fetch error:', e)
    }
}
watchEffect(() => {
  // only run if filters.value is ready
  if (filters.value) {
    console.log('Filters changed (watchEffect):', filters.value)
    fetchData()
  }
})
onMounted(() => fetchData())
</script>
