<script setup>
import { ref, computed, watch } from 'vue'

const rooms = ref([])
const bookings = ref([])

const load = async () => {
	const res = await fetch("http://172.18.17.2:5174/api/rooms.json")
	rooms.value = await res.json()

	const res2 = await fetch("http://172.18.17.2:5174/api/bookings.json")
	bookings.value = await res2.json()
}
// load() // load from localstorage first

const DAYS = 15;
const start = ref(new Date('2024-06-01'))

const adjustStart = (d) => {
	const nd = new Date(d)
	if (d.getDate() <= 7) {
		nd.setDate(1)
	} else if (d.getDate() >= 24) {
		nd.setDate(16)
	} else {
		nd.setDate(d.getDate() - 7)
	}
	start.value = nd
}

const days = computed(() => {
	return [...Array(DAYS)].map((_, i) => {
		const d = new Date(start.value)
		d.setDate(d.getDate() + i)
		return d
	})
})

const toStr = (date) => date.toISOString().slice(0, 10)

//TODO: refactor later
const roomBookings = (room, date) => {
	const bs = bookings.value.filter(b => b.roomId === room.id)
	return bs.filter(b => {
		const s = new Date(b.checkInDate)
		const e = new Date(b.checkOutDate)
		return (date >= s && date <= e)
	})
}

const bookingStyle = (b) => {
	const [x, y] = b.id.slice(-2).split("")
	return {
		background: `#DD${x}D${y}D`
	}
}


// ---- filter rooms by room type
const roomTypes = computed(() => {
	return new Set(rooms.value.map(r => r.roomType))
})

const roomTypeColors = {
	"Deluxe": "#3d5875",
	"Standard": "#548c7f",
	"Suite": "#3081ed",
	"Single": "#00b2e6",
	"Double": "#e86f8e"
}
const currentRoomType = ref("Deluxe")
const currentRooms = computed(() => {
	if (currentRoomType.value) {
		return rooms.value.filter(r => r.roomType === currentRoomType.value)
	} else {
		return rooms.value
	}
})

// ---- unassigned bookign list
const unassignedBookings = computed(() => {
	return bookings.value.filter(b => b.roomId === "" && b.roomType === currentRoomType.value)
})

const adjustToBookingDays = (b) => {
	const s = new Date(b.checkInDate)
	const e = new Date(b.checkOutDate)
	const ds = (e - s) / 3600 / 24 / 1000
	const diff = (DAYS - ds) / 2 - 1

	s.setDate(s.getDate() - diff)
	start.value = s
}


// ---- Drag and Drop Functionality
const dragging = ref([])
const dropOnRoom = (room) => {
	dragging.value.forEach(b => {
		// check room available
		const s = new Date(b.checkInDate)
		const e = new Date(b.checkOutDate)
		const ds = (e - s) / 3600 / 24 / 1000

		let available = true
		for (let i = 0; i < ds; i++) {
			const nd = new Date(s)
			nd.setDate(nd.getDate() + i)
			if (roomBookings(room, nd).length > 0) {
				available = false
				break
			}
		}

		if (available) {
			b.roomId = room.id
		}
	})

	dragging.value = []
}

const dropOnUnssigned = () => {
	dragging.value.forEach(b => b.roomId = "")
	dragging.value = []
}

document.addEventListener('mouseup', () => dragging.value = [])


// ---- Reset Button
const reset = () => {
	load()
	start.value = new Date('2024-06-01')
	currentRoomType.value = "Deluxe"
}

// ---- Persistent State
const store = () => {
	const data = {
		start: start.value,
		currentRoomType: currentRoomType.value,
		rooms: rooms.value,
		bookings: bookings.value
	}
	localStorage.setItem('data', JSON.stringify(data))
}

watch(() => [start, currentRoomType, rooms, bookings], store, { deep: true })

const loadStorage = () => {
	const data = localStorage.getItem('data')
	if (typeof data === 'string') {
		const { start: s, currentRoomType: c, rooms: rs, bookings: bs } = JSON.parse(data)
		start.value = new Date(s)
		currentRoomType.value = c
		rooms.value = rs
		bookings.value = bs
	} else {
		load()
	}
}
loadStorage()



// ---- Export/Import Functionality
const exportBookings = () => {
	const data = bookings.value.map(b => `${b.id}, ${b.roomId}`)
	const dataStr = ["bookingId, roomId", ...data].join("\r\n")

	const blob = new Blob([dataStr], { type: 'text/csv' })
	const url = URL.createObjectURL(blob)
	const a = document.createElement('a')
	a.href = url
	a.download = "hhh-bookings.csv"
	a.click()
	URL.revokeObjectURL(url);
}

const importFromCSV = (csvStr) => {
	const data = csvStr.split("\r\n").slice(1).map(line => line.split(", "))
	data.forEach(([bookingId, roomId]) => {
		const b = bookings.value.find(b => b.id === bookingId)

		// load the roomId
		if (b) b.roomId = roomId
	})
}

// move reader out of importBooking func, only have to create once.
const csvReader = new FileReader();
csvReader.addEventListener("load", () => {
	importFromCSV(csvReader.result)
})

// the same with fileInput, only 1 global object requried
const fileInput = document.createElement('input')
fileInput.type = "file"

fileInput.addEventListener("change", () => {
	const [file] = fileInput.files

	if (file) {
		csvReader.readAsText(file);

		// little trick, for detect change event everytime select the same file
		fileInput.value = null
	}
})

const importBookings = () => {
	fileInput.click()
}
</script>

<template>
	Calendar

	<div clsas="btn-group">
		<a href="#" @click.prevent="exportBookings" class="btn">Export</a>
		<a href="#" @click.prevent="importBookings" class="btn">Import</a>
		<a href="#" @click.prevent="reset" class="btn">Reset</a>
		<a href="#" @click.prevent class="btn">Fullscreen</a>
	</div>

	<div class="calendar">
		<div>
			<div class="day-wrapper">
				<div class="room-placeholder"></div>
				<div v-for="d in days" class="day" @click="adjustStart(d)">
					{{ d.getDate() }}
				</div>
			</div>

			{{ rooms[0] }} <br>
			{{ bookings[0] }}

			<div>
				<div v-for="room in currentRooms" class="room" @drop="dropOnRoom(room)" @dragover.prevent>
					<div class="room-id">{{ room.id }}</div>
					<div v-for="d in days" class="day-box">
						<div v-if="roomBookings(room, d).length === 0" class="booking">
							---
						</div>
						<div v-else v-for="b in roomBookings(room, d)" draggable="true" class="booking"
							:style="bookingStyle(b)" @dragstart="dragging = [b]">
							{{ b.id.slice(-3) }}
						</div>
					</div>
				</div>
			</div>
		</div>
		<aside>
			<div class="btn-group room-types">
				<a href="#" v-for="roomType in roomTypes" @click.prevent="currentRoomType = roomType" class="btn"
					:class="{ active: currentRoomType === roomType }" :style="{ background: roomTypeColors[roomType] }">
					{{ roomType }}
				</a>
			</div>

			<div class="unassigned-bookings" @drop="dropOnUnssigned" @dragover.prevent>
				<div v-for="b in unassignedBookings" draggable="true" class="unassigned-booking"
					:style="{ borderColor: roomTypeColors[b.roomType] }" @click="adjustToBookingDays(b)"
					@dragstart="dragging = [b]">
					{{ b.id }}
					{{ b.guestName }}
					<span>
						{{ b.checkInDate }}
						{{ b.checkOutDate }}
						{{ b.numberOfGuests }}
					</span>
				</div>
			</div>
		</aside>
	</div>
</template>

<style>
.calendar {
	display: flex;
}

.calendar aside {
	width: 40%;
}

.day-wrapper {
	display: flex;
	gap: 0 .5rem;
}

.day-box,
.day {
	flex: 0 0 40px;
	height: 40px;
	border: 1px solid #ccc;

	display: grid;
	place-items: center;

	cursor: pointer;
}

.room {
	display: flex;
	gap: 0 .5rem;
	margin: .5rem 0;
}

.room-placeholder,
.room-id {
	background: #3d5875;
	color: #fff;
	flex: 0 0 100px;
	display: grid;
	place-items: center;
}

.room-placeholder {
	background: transparent;
}

.day-box {
	background: lightgray;
	border: 1px solid darkgray;

	display: flex;
	flex-direction: column;
}

.booking {
	width: 100%;
	display: grid;
	place-items: center;
	flex: 1;

	font-size: smaller;
}

.btn-group {
	display: flex;
	gap: 0 .5rem;
}

.btn {
	display: inline-grid;
	place-items: center;

	height: 40px;
	padding: 0 1rem;

	border: 1px solid darkgray;
	/* box-sizing: border-box; */
}

.btn.active {
	border-width: 6px;
}

.room-types .btn {
	color: #fff;
}

.unassigned-booking {
	border: 3px solid #ccc;
	margin: .5rem 0;
	padding: .25rem;
	display: flex;
	gap: 0 .5rem;

	cursor: pointer;
}

.unassigned-booking span {
	margin-left: auto;
}
</style>