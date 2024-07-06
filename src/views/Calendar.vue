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
</script>

<template>
    Calendar

    <div class="day-wrapper">
        <div class="room-placeholder"></div>
        <div v-for="d in days" class="day">
            {{ d.getDate() }}
        </div>
    </div>

    {{ rooms[0] }} <br>
    {{ bookings[0] }}

    <div>
        <div v-for="room in rooms" class="room">
            <div class="room-id">{{ room.id }}</div>
            <div v-for="d in days" class="day-box">
                <template v-if="roomBookings(room, d).length === 0">
                    ---
                </template>
                <div v-else v-for="b in roomBookings(room, d)" class="booking" :style="bookingStyle(b)">
                    {{ b.id.slice(-3) }}
                </div>
            </div>
        </div>
    </div>
</template>

<style>
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
</style>