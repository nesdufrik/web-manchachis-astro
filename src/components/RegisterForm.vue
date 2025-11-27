<script setup>
import { useSupabase } from '../db/supabase'
import { computed, onMounted, ref } from 'vue'
import { EVENT_ID } from '../config'

const { supabase } = useSupabase()
const currentEventId = EVENT_ID
const startGroups = ref([])

const newParticipant = ref({
	event_id: currentEventId,
	category_id: '',
	full_name: '',
	email: '',
	phone: '+591',
	birth_date: '',
	gender: '',
	start_group_id: '',
	terms: false,
	receipt: null,
	receipt_url: '',
})

const loadEvent = async (idEvent) => {
	const { data: groups, error } = await supabase
		.from('start_groups')
		.select(`*, categories(*)`)
		.eq('event_id', idEvent)
	if (error) {
		console.error(error)
	}
	startGroups.value = groups
}

const categories = computed(() => {
	newParticipant.value.category_id = ''
	return startGroups.value.find(
		(group) => group.id === newParticipant.value.start_group_id
	)?.categories
})

const isSubmitting = ref(false)

const submitForm = async () => {
	if (isSubmitting.value) return
	isSubmitting.value = true

	try {
		console.log('Registrando participante...', newParticipant.value)

		let receiptUrl = null

		// 1. Subir el comprobante si existe
		if (newParticipant.value.receipt) {
			const file = newParticipant.value.receipt
			const fileExt = file.name.split('.').pop()
			const fileName = `${Date.now()}_${Math.floor(
				Math.random() * 1000
			)}.${fileExt}`
			const filePath = `${fileName}`

			// Asegúrate de crear un bucket llamado 'receipts' en tu Supabase Storage
			const { data: uploadData, error: uploadError } = await supabase.storage
				.from('receipts')
				.upload(filePath, file)

			if (uploadError) throw uploadError

			// 2. Obtener la URL pública
			const { data: urlData } = supabase.storage
				.from('receipts')
				.getPublicUrl(filePath)

			receiptUrl = urlData.publicUrl
		}

		// 3. Guardar en la base de datos
		const participantData = {
			...newParticipant.value,
			receipt_url: receiptUrl, // Asegúrate de tener esta columna en tu tabla 'participants'
		}

		// Eliminamos el objeto File antes de enviar a la BD
		delete participantData.receipt
		delete participantData.start_group_id

		/* const { data, error } = await supabase
			.from('participants')
			.insert([participantData])
			.select() */
		const { data, error } = await supabase.rpc('register_participant', {
			p_event_id: participantData.event_id,
			p_category_id: participantData.category_id,
			p_full_name: participantData.full_name,
			p_email: participantData.email,
			p_phone: participantData.phone,
			p_birth_date: participantData.birth_date,
			p_gender: participantData.gender,
			p_terms: participantData.terms,
			p_receipt_url: participantData.receipt_url,
		})

		if (error) throw error

		console.log('Registro exitoso:', data)
		alert('¡Registro completado exitosamente!')

		// Opcional: Limpiar formulario o redirigir
		newParticipant.value = {
			event_id: currentEventId,
			category_id: '',
			full_name: '',
			email: '',
			phone: '+591',
			birth_date: '',
			gender: '',
			start_group_id: '',
			terms: false,
			receipt: null,
			receipt_url: '',
		}
	} catch (error) {
		console.error('Error:', error)
		alert('Hubo un error al registrar: ' + (error.message || error))
	} finally {
		isSubmitting.value = false
	}
}

const genderQR = [
	{
		gender: 'male',
		qr: '/qr/qr-varones.png',
		price: 50,
	},
	{
		gender: 'female',
		qr: '/qr/qr-damas.png',
		price: 35,
	},
]

onMounted(() => {
	loadEvent(currentEventId)
})
</script>
<template>
	<div
		class="rounded-xl border border-slate-200 bg-background-light shadow-lg dark:border-slate-800 dark:bg-background-dark/70 dark:backdrop-blur-sm"
	>
		<form class="p-6 space-y-8" @submit.prevent="submitForm">
			<div class="space-y-6">
				<h2 class="text-xl font-bold text-slate-900 dark:text-slate-50">
					Información Personal
				</h2>
				<div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Nombre Completo
						</p>
						<input
							class="form-input flex w-full min-w-0 flex-1 resize-none overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 placeholder:text-slate-400 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50 dark:placeholder:text-slate-500"
							placeholder="Ingresa tu nombre"
							type="text"
							v-model="newParticipant.full_name"
						/>
					</label>
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Género
						</p>
						<select
							class="form-select flex w-full min-w-0 flex-1 overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50"
							v-model="newParticipant.gender"
						>
							<option value="" disabled>Seleccionar...</option>
							<option value="male">Masculino</option>
							<option value="female">Femenino</option>
						</select>
					</label>
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Fecha de Nacimiento
						</p>
						<input
							class="form-input flex w-full min-w-0 flex-1 resize-none overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 placeholder:text-slate-400 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50 dark:placeholder:text-slate-500"
							type="date"
							v-model="newParticipant.birth_date"
							required
						/>
					</label>
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Club/Equipo
						</p>
						<input
							class="form-input flex w-full min-w-0 flex-1 resize-none overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 placeholder:text-slate-400 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50 dark:placeholder:text-slate-500"
							placeholder="Ingresa el nombre de tu club"
							type="text"
							v-model="newParticipant.team"
						/>
					</label>
				</div>
			</div>

			<div class="space-y-6">
				<h2 class="text-xl font-bold text-slate-900 dark:text-slate-50">
					Información de Contacto
				</h2>
				<div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Correo Electrónico
						</p>
						<input
							class="form-input flex w-full min-w-0 flex-1 resize-none overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 placeholder:text-slate-400 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50 dark:placeholder:text-slate-500"
							placeholder="tu@email.com"
							type="email"
							v-model="newParticipant.email"
						/>
					</label>
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Número de Teléfono
						</p>
						<input
							class="form-input flex w-full min-w-0 flex-1 resize-none overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 placeholder:text-slate-400 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50 dark:placeholder:text-slate-500"
							placeholder="+591 12345678"
							type="tel"
							v-model="newParticipant.phone"
						/>
					</label>
				</div>
			</div>

			<div class="space-y-6">
				<h2 class="text-xl font-bold text-slate-900 dark:text-slate-50">
					Detalles de la Competición
				</h2>
				<div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Grupo
						</p>
						<select
							class="form-select flex w-full min-w-0 flex-1 overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50"
							v-model="newParticipant.start_group_id"
						>
							<option value="" disabled>Selecciona un grupo...</option>
							<option v-for="group in startGroups" :value="group.id">
								{{ group.name }}
							</option>
						</select>
					</label>
					<label class="flex flex-col">
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Categoría
						</p>
						<select
							class="form-select flex w-full min-w-0 flex-1 overflow-hidden rounded-lg border border-slate-300 bg-background-light p-3 text-base text-slate-900 focus:border-primary focus:outline-0 focus:ring-2 focus:ring-primary/20 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-50"
							v-model="newParticipant.category_id"
						>
							<option value="" disabled>Selecciona una categoria...</option>
							<option v-for="category in categories" :value="category.id">
								{{ category.name }}
							</option>
						</select>
					</label>
				</div>
			</div>

			<div class="space-y-6">
				<h2 class="text-xl font-bold text-slate-900 dark:text-slate-50">
					Legal y Pago
				</h2>
				<div class="space-y-4">
					<label class="flex items-start gap-3">
						<input
							class="form-checkbox mt-0.5 size-5 rounded border-slate-300 bg-background-light text-primary focus:ring-2 focus:ring-primary/50 dark:border-slate-600 dark:bg-slate-800"
							type="checkbox"
							v-model="newParticipant.terms"
						/>
						<span class="text-sm text-slate-700 dark:text-slate-300"
							>He leído y acepto el
							<a href="/reglas" target="_blank" class="text-primary"
								>reglamento</a
							>
							y la
							<a href="/reglas#absolucion" target="_blank" class="text-primary"
								>absolución de responsabilidad</a
							>.</span
						>
					</label>
				</div>
				<div
					class="rounded-lg border border-primary/20 bg-primary/10 p-4 dark:bg-primary/20"
				>
					<div class="flex items-center justify-between">
						<p
							class="text-base font-semibold text-slate-800 dark:text-slate-100"
						>
							Total a Pagar:
						</p>
						<p class="text-2xl font-bold text-secondary">
							Bs.
							{{
								newParticipant.gender
									? genderQR.find((p) => p.gender === newParticipant.gender)
											.price
									: 0.0
							}}
						</p>
					</div>
				</div>
			</div>

			<div class="space-y-6 grid grid-cols-1 md:grid-cols-2 gap-6">
				<div
					class="flex flex-col items-center justify-center space-y-4 rounded-lg border border-slate-200 bg-slate-50 p-6 dark:border-slate-700 dark:bg-slate-800/50"
					v-show="newParticipant.terms && newParticipant.gender"
				>
					<p
						class="text-center text-sm font-medium text-slate-600 dark:text-slate-400"
					>
						Escanea el código QR para realizar el pago
					</p>
					<div class="overflow-hidden rounded-xl bg-white p-2 shadow-sm">
						<!-- Reemplazar src con la ruta real de la imagen del QR -->
						<img
							:src="
								genderQR.find((p) => p.gender === newParticipant.gender)?.qr
							"
							alt="QR de Pago"
							class="h-48 w-48 object-contain"
						/>
						<a
							:href="
								genderQR.find((p) => p.gender === newParticipant.gender)?.qr
							"
							download="QR-Pago-Desafio-Manchachis.png"
							class="mt-2 flex w-full items-center justify-center gap-2 rounded-md bg-slate-100 py-2 text-sm font-medium text-slate-700 hover:bg-slate-200 dark:bg-slate-700 dark:text-slate-200 dark:hover:bg-slate-600 transition-colors"
						>
							<span class="material-symbols-outlined">download</span>
							Descargar QR
						</a>
					</div>
				</div>

				<div>
					<label
						class="flex flex-col"
						v-show="newParticipant.terms && newParticipant.gender"
					>
						<p
							class="text-sm font-medium pb-2 text-slate-800 dark:text-slate-200"
						>
							Comprobante de Pago
						</p>
						<input
							class="block w-full cursor-pointer rounded-lg border border-slate-300 bg-background-light text-sm text-slate-500 file:mr-4 file:cursor-pointer file:border-0 file:bg-primary file:px-4 file:py-3 file:text-sm file:font-semibold file:text-white hover:file:bg-primary/90 focus:outline-none dark:border-slate-700 dark:bg-slate-800 dark:text-slate-400"
							type="file"
							accept="image/*,application/pdf"
							@change="(e) => (newParticipant.receipt = e.target.files[0])"
						/>
						<span class="pt-1 text-xs text-slate-500 dark:text-slate-400">
							Sube una imagen o PDF del comprobante de la transferencia.
						</span>
					</label>

					<div class="flex justify-end w-full pt-4">
						<button
							class="flex w-full cursor-pointer items-center justify-center overflow-hidden rounded-lg h-12 px-6 bg-primary text-base font-bold text-slate-50 transition-colors duration-200 hover:bg-primary/90 disabled:cursor-not-allowed disabled:bg-primary/10 disabled:text-slate-500"
							:disabled="!newParticipant.terms || isSubmitting"
						>
							<span class="truncate" v-if="!isSubmitting"
								>Completar Registro y Pagar</span
							>
							<span class="truncate" v-else>Enviando...</span>
						</button>
					</div>
				</div>
			</div>
		</form>
	</div>
</template>
