<script setup>
import { onMounted, onUnmounted, ref, computed } from 'vue'
import { useSupabase } from '../db/supabase'

const { supabase } = useSupabase()
const currentEventId = '9e838735-1bd0-4efb-9a30-1ce9cc08b0b9'

const fetchLiveRanking = async () => {
	const { data, error } = await supabase
		.from('live_ranking')
		.select('*')
		.eq('event_id', currentEventId)
		.neq('status', null)
		.order('finish_time', { ascending: true, nullsFirst: false })

	if (error) {
		console.error('Error fetching live ranking:', error)
		return
	}

	results.value = data
}

const results = ref([])
const searchQuery = ref('')

const filteredResults = computed(() => {
	if (!searchQuery.value) {
		return results.value
	}

	return results.value.filter((result) => {
		return (
			result.full_name
				.toLowerCase()
				.includes(searchQuery.value.toLowerCase()) ||
			result.race_number.toString().includes(searchQuery.value) ||
			result.category_name
				.toLowerCase()
				.includes(searchQuery.value.toLowerCase()) ||
			result.start_group_name
				.toLowerCase()
				.includes(searchQuery.value.toLowerCase())
		)
	})
})

const channel = supabase
	.channel('public-race-results')
	.on(
		'postgres_changes',
		{
			event: '*',
			schema: 'public',
			table: 'race_results',
		},
		(payload) => {
			console.log('¡Nuevo resultado detectado!', payload)

			fetchLiveRanking()
		}
	)
	.subscribe()

onMounted(() => {
	fetchLiveRanking()
})

onUnmounted(() => {
	channel.unsubscribe()
})
</script>
<template>
	<div class="mb-6 flex flex-row items-center gap-4">
		<label class="relative flex flex-col min-w-40 h-12 w-full max-w-lg">
			<div class="absolute inset-y-0 left-0 flex items-center pl-4">
				<span class="material-symbols-outlined text-gray-500 dark:text-gray-400"
					>search</span
				>
			</div>
			<input
				class="form-input flex w-full min-w-0 flex-1 resize-none overflow-hidden rounded-lg text-gray-900 dark:text-white focus:outline-0 focus:ring-2 focus:ring-primary/50 border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 h-full placeholder:text-gray-500 dark:placeholder:text-gray-400 pl-12 pr-4 font-normal leading-normal text-sm"
				placeholder="Buscar por nombre, dorsal, grupo o categoría"
				v-model="searchQuery"
			/>
		</label>
	</div>

	<div class="@container">
		<div
			class="overflow-hidden rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800"
		>
			<div class="overflow-x-auto">
				<table class="w-full min-w-[600px]">
					<thead>
						<tr class="bg-gray-50 dark:bg-gray-900/50">
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 text-sm font-medium tracking-wider uppercase"
							>
								Nro
							</th>
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 w-1/4 text-sm font-medium tracking-wider uppercase"
							>
								Nombre
							</th>
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 w-auto text-sm font-medium tracking-wider uppercase"
							>
								Equipo
							</th>
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 w-auto text-sm font-medium tracking-wider uppercase"
							>
								Grupo
							</th>
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 w-auto text-sm font-medium tracking-wider uppercase"
							>
								Categoría
							</th>
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 w-28 text-sm font-medium tracking-wider uppercase"
							>
								Tiempo
							</th>
							<th
								class="px-4 py-3 text-left text-gray-600 dark:text-gray-400 w-16 text-sm font-medium tracking-wider uppercase"
							>
								Pos
							</th>
						</tr>
					</thead>
					<tbody class="divide-y divide-gray-200 dark:divide-gray-700">
						<tr
							class="hover:bg-primary/10 dark:hover:bg-primary/10 transition-colors"
							v-if="results && results.length > 0"
							v-for="result in filteredResults"
						>
							<td
								class="h-[72px] px-4 py-2 w-16 text-gray-800 dark:text-gray-200 text-sm font-semibold"
							>
								{{ result.race_number }}
							</td>
							<td
								class="h-[72px] px-4 py-2 w-1/4 text-gray-900 dark:text-white text-sm font-medium"
							>
								{{ result.full_name }}
							</td>
							<td
								class="h-[72px] px-4 py-2 w-auto text-gray-600 dark:text-gray-300 text-sm"
							>
								{{ result.team }}
							</td>
							<td
								class="h-[72px] px-4 py-2 w-auto text-gray-600 dark:text-gray-300 text-sm"
							>
								{{ result.start_group_name }}
							</td>
							<td
								class="h-[72px] px-4 py-2 w-auto text-gray-600 dark:text-gray-300 text-sm"
							>
								{{ result.category_name }}
							</td>
							<td
								class="h-[72px] px-4 py-2 w-32 text-gray-600 dark:text-gray-300 text-sm"
							>
								{{
									result.total_time_ms
										? `${Math.floor(
												result.total_time_ms / 3600000
										  )}:${Math.floor((result.total_time_ms % 3600000) / 60000)
												.toString()
												.padStart(2, '0')}:${Math.floor(
												(result.total_time_ms % 60000) / 1000
										  )
												.toString()
												.padStart(2, '0')}`
										: '--'
								}}
							</td>
							<td
								class="h-[72px] px-4 py-2 w-16 text-gray-800 dark:text-gray-200 text-sm font-semibold flex flex-row items-center gap-2"
							>
								<span>
									{{ result.position_in_category }}
								</span>
								<span
									class="material-symbols-outlined"
									v-if="
										result.position_in_category === 1 ||
										result.position_in_category === 2 ||
										result.position_in_category === 3
									"
									:class="`
									${
										result.position_in_category === 1
											? ' text-yellow-500 font-bold'
											: result.position_in_category === 2
											? ' text-gray-500'
											: result.position_in_category === 3
											? ' text-amber-800'
											: ''
									}
									`"
								>
									social_leaderboard
								</span>
							</td>
						</tr>
						<tr v-else>
							<td
								colspan="5"
								class="px-4 py-8 text-center text-gray-600 dark:text-gray-400"
							>
								No hay resultados disponibles aún
							</td>
						</tr>
					</tbody>
				</table>
			</div>
		</div>
	</div>
</template>
