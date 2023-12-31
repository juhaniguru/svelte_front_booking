<script>
    import { onMount, tick } from "svelte";

    // ajat alkaa 8 päättyy 16
    let start = 8;
    let end = 16;

    // interval on se funktio joka päivittää 2 sekunnin välein ajat
    let interval = null;

    // tehdaan 8 * 4 slottia aikoja varten, koska varattavat ajat on 15 minuutin välein (4 aikaa tunnissa)
    let times = new Array((end - start) * 4);
    times = times.fill(0).slice();
    let hour = start;
    let hourChange = 1;

    // jos usePolling on true socketit ei ole käytössä
    let usePolling = true;

    // times-array sisältää ajat näin:
    //[{t: 8:00, own: false}, {t: 8:15, own: false}, {t:8:30, own: false}, {t: 8:45, own: false}, {t:9:00, own: false}] ...jne
    times = times.map(() => {
        if (hourChange % 5 == 0) {
            hour += 1;
            hourChange = 1;
        }

        // vaihdetaan tasatuntia 4 välein
        let min = (hourChange - 1) * 15;
        if (min == 0) {
            min = "00";
        }

        hourChange++;

        return { t: `${hour}:${min}`, own: false };
    });

    // kopioidaan defaultTimes muuttujaan oletusajat ennen yhtään varausta, jotta niihin voidaan palata
    let defaultTimes = times.slice();

    // odotetaan, että sivu on kunnolla latautunut, ennen kuin päivitetään varaustilanne
    onMount(async () => {
        await tick();
        await _getBookings();

        // jos usePolling on true, päivitetään varaustilanne 2 sekunnin välein
        // tämä siksi, että jos usePolling on false, käytetään websocket-yhteyttä, mutta sitä ei ole vielä tehty
        if (usePolling) {
            interval = setInterval(() => {
                console.log("getBookings");
                _getBookings();
            }, 2000);
        } else {
            // tähän websocket-yhteys
        }
    });

    // splitataan aikaleima (esim 8:15) kaksoispisteestä, ja läheteään vuosi, kuukausi, päivä, tunti, minuutti ja booking_reference backendiin
    // jos varattua aikaa ollaan peruuttamassa tämä funktio poistaa jo varatun ajan appointments-taulusta
    // jos taas aika on ollu aiemmin vapaana, aika lisätään booking_referencen osoittamaan varaukseen
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
                "http://localhost:8001/api/v1/booking/appointment",
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
                        // localstoragen sijaan, tämä olisi hyvä tallentaa keksiin, mutta tässä esimerkissä tietoturva ei ole tärkein pointti
                        // menin sen vuoksi helpommalla tavalla
                        booking_reference: localStorage.getItem("booking_ref"),
                    }),
                },
            );

            if (!res.ok) {
                throw new Error(res.statusText);
            }

            // koska samaa funktiota käytetään sekä ajan lisäämiseen varaukseen ja ajan poistamiseen varauksesta,
            // tässä vaihdetaan napin klikin jälkeen own-attribuutin arvo joko truesta falseksi tai toisinpäin.
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

    // poistaa bookinging tietokannasta käyttäen x-booking-ref headerissa olevaa jwt tokenia, jossa booking_ref
    // jos bookingissa ei ole yhtään varattua aikaa
    async function _removeBooking() {
        try {
            if (localStorage.getItem("booking_ref") == null) {
                return;
            }
            const res = await fetch("http://localhost:8001/api/v1/booking/", {
                headers: {
                    "content-type": "application/json",
                    "x-booking-ref": `Bearer ${localStorage.getItem(
                        "booking_ref",
                    )}`,
                },
                method: "DELETE",
            });

            if (!res.ok) {
                throw new Error(res.statusText);
            }
            localStorage.removeItem("booking_ref");
        } catch (e) {
            console.log(e);
            localStorage.removeItem("booking_ref");
        }
    }

    // tämä funktio hakee sekä omat varatut ajat, että muiden varaamat ajat ja palauttaa ne tällaisessa muodossa
    // [{hour: 8, min: 15, own: true / false}]
    async function _getBookings() {
        try {
            let headers = {
                "content-type": "application/json",
            };

            // jos olet ensimmäistä kertaa varamassa aikaa, sinulla ei ole oo. booking_refiä,
            // ja siksi x-booking-ref lisätään vain jos jwt löytyy localstoragesta
            if (localStorage.getItem("booking_ref") != null) {
                headers["x-booking-ref"] = `Bearer ${localStorage.getItem(
                    "booking_ref",
                )}`;
            }

            const res = await fetch("http://localhost:8001/api/v1/booking", {
                headers,
            });

            if (!res.ok) {
                throw new Error(res.statusText);
            }

            const bookings = await res.json();

            // poistetaan muiden varaamat ajat kokonaan listasta, koska et voi varata niitä
            times = defaultTimes.filter((time) => {
                let splitTime = time.t.split(":");
                let isBookedByOterhs = bookings.bookings.find((b) => {
                    return (
                        b.hour == splitTime[0] &&
                        b.min == splitTime[1] &&
                        !b.own
                    );
                });

                return !isBookedByOterhs;
            });

            // sen jälkeen, muutetaan kaikkien omien varausten own-attribuutin arvoksi true
            // koska itselle varaamat ajat näkyvät vihreänä näytöllä
            // vapaat ajat sinisinä ja muiden varaamat ajat eivät ole ollenkaan näkyvissä
            times = times.map((time) => {
                let splitTime = time.t.split(":");
                let isMine = bookings.bookings.find((b) => {
                    return (
                        b.hour == splitTime[0] && b.min == splitTime[1] && b.own
                    );
                });

                return isMine == null ? time : { ...time, own: true };
            });
        } catch (e) {
            console.log(e);
        } finally {
            let hasAppointments = times.find((time) => time.own);
            if (!hasAppointments) {
                await _removeBooking();
            }
        }
    }

    // book luo booking-tauluun uuden rivin, jolla on uuid booking_reference (random merkkjono)
    // ja palauttaa jwt-tokenin, jonka sub on tuo booking_reference
    // jos varaus on luotu jo aiemmin (eli localstoragesta löytyy booking_ref-niminen token)
    // lisätään uusia aikoja tähän oo. varaukseen tai poistetaan siitä aiemmin varattuja aikoja
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
</script>

<div class="container">
    <div class="row">
        {#each times as time (time.t)}
            <div class="col">
                <button
                    on:click={() => book(time)}
                    class={time.own
                        ? "btn btn-sm btn-success mb-2 mt-2"
                        : "btn btn-sm btn-primary mb-2 mt-2"}>{time.t}</button
                >
            </div>
        {/each}
    </div>
</div>
