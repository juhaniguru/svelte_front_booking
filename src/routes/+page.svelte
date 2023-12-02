<script>
    import { onMount, tick } from "svelte";

    let start = 8;
    let end = 16;
    let times = new Array((end - start) * 4);
    times = times.fill(0).slice();
    let hour = start;
    let hourChange = 1;

    let interval = null;
    let usePolling = false;
    let reconnectInterval = null;
    let socket = null;

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

    let defaultTimes = times.slice();

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
            socket.send(JSON.stringify({ tag: "update" }));
        } catch (e) {
            console.log(e);
        }
    }

    async function _getBookings() {
        console.log("getBookings");
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
            let isBooked = bookings.bookings.find((b) => {
                return (
                    b.hour == splitTime[0] && b.min == splitTime[1] && !b.own
                );
            });

            return !isBooked;
        });

        times = times.map((time) => {
            let splitTime = time.t.split(":");
            let isMine = bookings.bookings.find((b) => {
                return b.hour == splitTime[0] && b.min == splitTime[1] && b.own;
            });

            return isMine == null ? time : { ...time, own: true };
        });

        console.log(times);
    }

    onMount(async () => {
        await tick();
        await _getBookings();

        if (usePolling) {
            interval = setInterval(async () => {
                console.log("interval");
                await _getBookings();
            }, 2000);
        } else {
            connect();
        }
    });

    function connect() {
        socket = new WebSocket("ws://localhost:8001/ws");

        // tämä callback suoritetaan silloin, kun yhteys saadaan
        socket.onopen = () => {
            // jos interval ei ole null, se tarkoitaa, että serveri on sammunut ja on yritetty autom. yhdistää uudelleen
            // nyt kun yhteys on saatu takaisin, voidaan intervalli lopettaa
            if (reconnectInterval != null) {
                clearTimeout(reconnectInterval);
                reconnectInterval = null;
            }
            if (socket.readyState == 1) {
                // tässä client lähettää severille viestin, että on liitytty keskusteluun
                // serveri sitten lähettää tämän viestin kaikille osallistujille
                // esim. juhani liittyi keskusteluun
            }
        };

        // jos tapahtuu virhe, tullaan tänne
        socket.onerror = (e) => {
            console.log("socket error", e);
        };

        // kun yhteys sulkeutuu, tullaan tänne
        socket.onclose = (e) => {
            // wasClean on boolean, jos true, se tarkoittaa, että yhteys katkesi toivotusti (esim. käyttäjä poistui chatista)
            if (e.wasClean) {
                console.log("bye bye");
            } else {
                // jos wasClean on false, se tarkoittaa, että esim. serveri on nurin
                console.log("something bad");
                // tästä syystä yritetään yhdistää uudelleen 2 sekunnin välein
                reconnectInterval = setTimeout(() => {
                    console.log("reconnecting");
                    connect();
                }, 2000);
            }
        };

        // serveriltä tulee viesti, niin tullaan tänne
        socket.onmessage = (m) => {
            times = defaultTimes.slice();
            _getBookings();
        };
    }
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
