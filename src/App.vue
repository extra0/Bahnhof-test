<script setup lang="ts">
	import CityName from '@/components/CityName.vue' // City name component
	import TheSpinner from '@/components/TheSpinner.vue' // Loader component
	import { ref, computed } from 'vue' // vue

	interface WeatherData {
		current: {
			temp_c: number
			feelslike_c: number
			condition: {
				text: string
				icon: string
			}
		}
	}

	interface CityInfo {
		query: {
			pages: Record<string, {
				extract: string
			}>
		}
	}

	const weatherData = ref<WeatherData | null>(null)
	const cityInfo = ref<CityInfo | null>(null)
	const cityName = ref<string>('Lviv')
	const isWaitingForReponse = ref<boolean>(true)

	const cityDescription = computed(() => { // Get city description
		if (!cityInfo.value) {
			return ''
		}

		return Object.values(cityInfo.value?.query?.pages)[0]?.extract || ''
	})
	const iconSrc = computed(() => weatherData.value?.current?.condition?.icon || '') // get weather icon src
	const iconDescription = computed(() => weatherData.value?.current?.condition?.text || '') // get weather icon description (for alt)
	const currentTemp = computed(() => weatherData.value?.current?.temp_c || '') // get current temperature
	const feelsLikeTemp = computed(() => weatherData.value?.current?.feelslike_c || '') // get feels like temperature
	const isWeaterAvailable = computed(() => !!iconSrc.value && !!currentTemp.value && !!feelsLikeTemp.value) // check wweather data

	/**
	 * Send requests
	 */
	const fetchData = async () => {
		isWaitingForReponse.value = true

		const urls = {
			weatherData: `http://api.weatherapi.com/v1/current.json?key=${import.meta.env.VITE_WEATHER_API_KEY}&q=${cityName.value}`,
			cityInfo: `https://en.wikipedia.org/w/api.php?action=query&titles=${cityName.value}&prop=extracts&exlimit=max&origin=*&format=json&formatversion=2`,
		};

		try {
			const responses = await Promise.allSettled(
				Object.entries(urls).map(([key, url]) => fetch(url).then(res => res.json()).then(data => ({
					key,
					data,
				})))
			);

			responses.forEach((r) => {
				if (r.status === 'fulfilled') {
					if (r.value.key === 'weatherData') {
						weatherData.value = r.value.data;
					}

					if (r.value.key === 'cityInfo') {
						cityInfo.value = r.value.data;
					}
				} else {
					console.error('Error fetching data from URL:', r.reason)
				}
			})
		} catch (error) {
			console.error('Unexpected error:', error)
		} finally {
			isWaitingForReponse.value = false
		}
	}

	fetchData()
</script>

<template>
	<template v-if="isWaitingForReponse">
		<CityName :city-name="cityName" />
		<TheSpinner />
	</template>
	<template v-else>
		<header :class="{ 'error': !isWeaterAvailable }">
			<CityName :city-name="cityName" />
			<template v-if="isWeaterAvailable">
				<img v-if="iconSrc" :src="iconSrc" :alt="iconDescription" />
				Temperature: {{ currentTemp }}&nbsp;°C (Feels like: {{ feelsLikeTemp }}&nbsp;°C)
			</template>
			<span v-else class="error">Unfortunately, it was not possible to obtain weather data for the city</span>
		</header>

		<main>
			<div class="wrapper">
				<div v-if="cityDescription" v-html="cityDescription" />
				<div v-else class="error">Unfortunately, it was not possible to obtain a description of the city</div>
			</div>
		</main>
	</template>
</template>

<style scoped>
	header {
		line-height: 1.5;
	}

	header.error {
		flex-direction: column;
	}

	@media (min-width: 1024px) {
		header {
			display: flex;
			place-items: center;
		}
	}
</style>
