<script>
    import { onMount, tick } from "svelte";

    let start = 8;
    let end = 16;
    let times = new Array((end - start) * 4);
    times = times.fill(0).slice();
    let hour = start;
    let hourChange = 1;

    let interval = null;
    let usePolling = true;

    times = times.map(() => {
        if (hourChange % 5 == 0) {
            hour += 1;
            hourChange = 1;
        }
        let min = (hourChange - 1) * 15;
        if (min == 0) {
            min = "00";
        }
        hourChange++;

        return { t: `${hour}:${min}`, own: false };
    });

    async function _book(time) {
        try {
            const now = new Date();
            let splitTime = time.split(":");
            let hour = parseInt(splitTime[0]);
            let m = 0;
            if (splitTime[1] != "00") {
                m = parseInt(splitTime[1]);
            }
            const res = await fetch(
                "http://localhost:8001/api/v1/booking/appointment/",
                {
                    headers: {
                        "content-type": "application/json",
                    },
                    method: "POST",
                    body: JSON.stringify({
                        year: now.getFullYear(),
                        month: now.getMonth() + 1,
                        day: now.getDay(),
                        hour: hour,
                        min: m,
                        booking_reference: localStorage.getItem("booking_ref"),
                    }),
                },
            );
            if (!res.ok) {
                throw new Error(res.statusText);
            }

            times = times.map((time) => {
                let splitTime = time.t.split(":");
                if (splitTime[0] == hour && splitTime[1] == m) {
                    time = { ...time, own: !time.own };
                }

                return time;
            });
        } catch (e) {
            console.log(e);
        }
    }

    async function book(time) {
        let bookingRef = localStorage.getItem("booking_ref");
        try {
            if (bookingRef == null) {
                const res = await fetch(
                    "http://localhost:8001/api/v1/booking/",
                    {
                        headers: {
                            "content-type": "application/json",
                        },
                        method: "POST",
                        body: null,
                    },
                );
                if (!res.ok) {
                    throw new Error(res.statusText);
                }

                const tokenRes = await res.json();
                localStorage.setItem("booking_ref", tokenRes.token);
            }

            await _book(time.t);
        } catch (e) {
            console.log(e);
        }
    }

    async function _getBookings() {
        let headers = {
            "content-type": "application/json",
        };

        if (localStorage.getItem("booking_ref") != null) {
            headers["x-booking-ref"] = `Bearer ${localStorage.getItem(
                "booking_ref",
            )}`;
        }
        const res = await fetch("http://localhost:8001/api/v1/booking/", {
            headers,
        });

        const bookings = await res.json();

        times = times.filter((time) => {
            let splitTime = time.t.split(":");
            let isBookedByOthers = bookings.bookings.find((b) => {
                return (
                    b.hour == splitTime[0] && b.min == splitTime[1] && !b.own
                );
            });

            return !isBookedByOthers;
        });

        times = times.map((time) => {
            let splitTime = time.t.split(":");
            let isMine = bookings.bookings.find((b) => {
                return b.hour == splitTime[0] && b.min == splitTime[1] && b.own;
            });

            return isMine == null ? time : { ...time, own: true };
        });
    }

    onMount(async () => {
        await tick();
        await _getBookings();

        if (usePolling) {
            interval = setInterval(async () => {
                console.log("interval");
                await _getBookings();
            }, 2000);
        }
    });
</script>

<div class="container">
    <div class="row">
        {#each times as time (time)}
            <div class="col">
                <button
                    on:click={() => {
                        book(time);
                    }}
                    class={time.own
                        ? "btn btn-sm btn-success mb-2 mt-2"
                        : "btn btn-sm btn-primary mb-2 mt-2"}>{time.t}</button
                >
            </div>
        {/each}
    </div>
</div>
