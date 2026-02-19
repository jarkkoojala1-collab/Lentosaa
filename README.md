
<!DOCTYPE html>

<html lang="fi">
<head>
<meta charset="UTF-8">
<title>Sääennusteiden tutkija</title>

<style>
body{
    font-family: Arial;
    background:#1e2430;
    color:white;
    text-align:center;
}

.container{
    max-width:900px;
    margin:auto;
}
input, button{
    padding:10px;
    font-size:16px;
    margin:5px;
    border-radius:6px;
    border:none;
}

button{
    background:#4fc3f7;
    cursor:pointer;
}

.card{
    background:#2b3245;
    padding:15px;
    margin:10px;
    border-radius:10px;
    display:inline-block;
    width:120px;
}

h2{
    margin-top:25px;
}
</style>

</head>

<body>
<div class="container">

<h1>🌦️ Sääennusteiden tutkija</h1>

<input id="city" placeholder="Kaupunki (esim Helsinki)">
<button onclick="haeSaa()">Hae sää</button>

<h2 id="otsikko"></h2>

<div id="rinne"></div>
<div id="ppg"></div>

<h2>Seuraavat tunnit</h2>
<div id="tunnit"></div>

</div>

<script>

async function geokoodaa(kaupunki){
    const url = `https://geocoding-api.open-meteo.com/v1/search?name=${kaupunki}&count=1&language=fi`;
    const res = await fetch(url);
    const data = await res.json();
    return data.results?.[0];
}

async function haeSaa(){
    const kaupunki = document.getElementById("city").value;
    if(!kaupunki) return;

    const paikka = await geokoodaa(kaupunki);

    if(!paikka){
        alert("Paikkaa ei löytynyt");
        return;
    }

    document.getElementById("otsikko").innerText =
        paikka.name + ", " + paikka.country;

    const lat = paikka.latitude;
    const lon = paikka.longitude;

    const url =
    `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}
    &hourly=temperature_2m,cloudcover,windspeed_10m,winddirection_10m,windgusts_10m,cape,precipitation
    &timezone=auto`;

    const res = await fetch(url);
    const data = await res.json();

    analysoiRinne(data.hourly);
    analysoiPPG(data.hourly);
    naytaTunnit(data.hourly);
}

function analysoiRinne(h){

    let tulos = "Huono";
    let vari = "red";
    let syy = "Liian heikko tai puuskainen tuuli";

    for(let i=0;i<12;i++){
        const wind = h.windspeed_10m[i];
        const gust = h.windgusts_10m[i];
        const clouds = h.cloudcover[i];
        const cape = h.cape?.[i] ?? 0;

        if(
            wind>=4 && wind<=10 &&
            (gust-wind)<4 &&
            clouds>20 && clouds<90 &&
            cape<300
        ){
            tulos="Hyvä";
            vari="lime";
            syy="Tasainen nostotuuli";
            break;
        }
        else if(wind>=3 && wind<=12){
            tulos="Mahdollinen";
            vari="orange";
            syy="Rajakelpo";
        }
    }

    document.getElementById("rinne").innerHTML =
    `<h2 style="color:${vari}">🪂 Rinnelento: ${tulos}</h2><p>${syy}</p>`;
}

function analysoiPPG(h){

    let tulos = "Huono";
    let vari = "red";
    let syy = "Liian tuulinen tai terminen";

    for(let i=0;i<6;i++){
        const wind = h.windspeed_10m[i];
        const gust = h.windgusts_10m[i];
        const cape = h.cape?.[i] ?? 0;
        const rain = h.precipitation[i];

        if(
            wind<=4 &&
            (gust-wind)<2 &&
            cape<100 &&
            rain==0
        ){
            tulos="Hyvä";
            vari="lime";
            syy="Tyyni ilta/aamu";
            break;
        }
        else if(wind<=6){
            tulos="Mahdollinen";
            vari="orange";
            syy="Kokenut pilotti";
        }
    }

    document.getElementById("ppg").innerHTML =
    `<h2 style="color:${vari}">🛩️ Paramoottori: ${tulos}</h2><p>${syy}</p>`;
}

function naytaTunnit(hourly){
    const div = document.getElementById("tunnit");
    div.innerHTML="";

    for(let i=0;i<12;i++){
        const time = hourly.time[i].slice(11,16);
        const temp = hourly.temperature_2m[i];
        const wind = hourly.windspeed_10m[i];

        div.innerHTML +=
        `<div class="card">
            <b>${time}</b><br>
            ${temp}°C<br>
            💨 ${wind} m/s
        </div>`;
    }
}

</script>
</body>
</html>
:::