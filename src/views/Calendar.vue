<script setup>
    import { ref, computed } from 'vue'

    const rooms = ref([])
    const bookings = ref([])

    const load = async () => {   
        const res = await fetch("http://localhost:5173/api/rooms.json")
        rooms.value = await res.json()

        const res2 = await fetch("http://localhost:5173/api/bookings.json")
        bookings.value = await res2.json()
    }
    load()

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

    const toStr = (date) => date.toISOString().slice(0,10)

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
</script>

<template>
    Calendar

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
                <div v-for="room in currentRooms" class="room">
                    <div class="room-id">{{ room.id }}</div>
                    <div v-for="d in days" class="day-box">
                        <div v-if="roomBookings(room, d).length === 0" class="booking">
                            ---
                        </div>
                        <div v-else v-for="b in roomBookings(room, d)" class="booking" :style="bookingStyle(b)">
                            {{ b.id.slice(-3) }}
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <aside>
            <div class="btn-group">
                <a href="#" @click.prevent="currentRoomType = roomType" 
                    v-for="roomType in roomTypes" 
                    class="btn" 
                    :class="{active: currentRoomType === roomType}"
                    :style="{background: roomTypeColors[roomType]}">
                    {{ roomType }}
                </a>
            </div>

            <div class="unassigned-bookings">
                <div v-for="b in unassignedBookings" 
                    class="unassigned-booking"
                    :style="{borderColor: roomTypeColors[b.roomType]}">
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
        display:flex;
        gap: 0 .5rem;
    }

    .day-box,
    .day {
        flex: 0 0 40px;
        height: 40px;
        border: 1px solid #ccc;

        display:grid;
        place-items: center;

        cursor: pointer;
    }

    .room {
        display: flex;
        gap: 0 .5rem;
        margin: .5rem 0;
    }
    .room-placeholder,
    .room-id{
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
        border: 1px solid  darkgray;

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
        box-sizing: border-box;

        color: #fff;
    }
    .btn.active {
        border-width: 6px;
    }

    .unassigned-booking {
        border: 3px solid #ccc;
        margin: .5rem 0;
        padding: .25rem;
        display: flex;
        gap: 0 .5rem;
    }

    .unassigned-booking span {
        margin-left: auto;
    }
</style>