<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Pilot Sunrise/Sunset Predictor (Offline)</title>
  <style>
    body {
      font-family: 'Segoe UI', Roboto, sans-serif;
      background-color: #121212;
      color: #e0e0e0;
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 20px;
    }
    h1 {
      color: #03DAC6;
      margin-bottom: 10px;
    }
    .container {
      background: #1f1f1f;
      padding: 20px 30px;
      border-radius: 12px;
      box-shadow: 0 4px 15px rgba(0,0,0,0.5);
      max-width: 400px;
      width: 100%;
    }
    label {
      display: block;
      margin-top: 12px;
      color: #BB86FC;
      font-size: 0.9em;
    }
    input {
      width: 100%;
      padding: 10px;
      margin-top: 4px;
      border-radius: 6px;
      border: 1px solid #333;
      background-color: #2c2c2c;
      color: #e0e0e0;
      font-size: 1em;
    }
    button {
      margin-top: 20px;
      width: 100%;
      padding: 12px;
      border: none;
      border-radius: 6px;
      background-color: #03DAC6;
      color: #000;
      font-weight: bold;
      font-size: 1em;
      cursor: pointer;
      transition: 0.3s;
    }
    button:hover {
      background-color: #018786;
    }
    .result {
      margin-top: 20px;
      padding: 15px;
      background-color: #2a2a2a;
      border-radius: 8px;
      color: #ffffff;
      text-align: center;
      font-size: 1.2em;
      box-shadow: 0 2px 10px rgba(0,0,0,0.3);
    }
    footer {
      margin-top: 40px;
      font-size: 0.8em;
      color: #888;
    }
  </style>
</head>
<body>
  <h1>Sunrise/Sunset Predictor (Offline) by Chris</h1>

  <div class="container">
    <label>Latitude (°)</label>
    <input type="number" id="lat" value="25.2048" step="0.0001">

    <label>Longitude (°)</label>
    <input type="number" id="lon" value="55.2708" step="0.0001">

    <label>Altitude (feet)</label>
    <input type="number" id="altitude" value="35000">

    <label>Groundspeed (knots)</label>
    <input type="number" id="gs" value="480">

    <label>Track (degrees)</label>
    <input type="number" id="track" value="90">

    <label>Date</label>
    <input type="date" id="date" value="2025-03-18">

    <label>Timezone Offset (hours)</label>
    <input type="number" id="tz" value="4">

    <button onclick="predictSunEvent()">Predict</button>

    <div class="result" id="result">Enter your flight details and click Predict</div>
  </div>

  <footer>© 2025 Chris Sykes</footer>

  <script>
    function predictSunEvent() {
      let lat = parseFloat(document.getElementById("lat").value);
      let lon = parseFloat(document.getElementById("lon").value);
      const altitudeFt = parseFloat(document.getElementById("altitude").value);
      const gs = parseFloat(document.getElementById("gs").value);
      const trackDeg = parseFloat(document.getElementById("track").value);
      const dateStr = document.getElementById("date").value;
      const tz = parseFloat(document.getElementById("tz").value);

      const resultDiv = document.getElementById("result");
      resultDiv.innerHTML = "Calculating...";

      const date = new Date(dateStr + "T00:00:00Z");
      let sunriseUTC = calcSunTime(lat, lon, date, true);
      let sunsetUTC = calcSunTime(lat, lon, date, false);

      const altitudeCorrectionMin = getAltitudeCorrection(altitudeFt);

      // Adjust sunrise/sunset time for altitude dip angle
      let sunriseMinutes = timeStringToMinutes(sunriseUTC) - altitudeCorrectionMin;
      let sunsetMinutes = timeStringToMinutes(sunsetUTC) + altitudeCorrectionMin;

      // Compute time deltas in hours
      const sunriseHoursFromNow = (sunriseMinutes - timeStringToMinutes(currentUTCTime())) / 60;
      const sunsetHoursFromNow = (sunsetMinutes - timeStringToMinutes(currentUTCTime())) / 60;

      // Calculate new positions based on groundspeed and track for sunrise and sunset
      const sunrisePos = movePosition(lat, lon, gs, trackDeg, sunriseHoursFromNow);
      const sunsetPos = movePosition(lat, lon, gs, trackDeg, sunsetHoursFromNow);

      // Recalculate sunrise/sunset based on predicted positions
      sunriseUTC = calcSunTime(sunrisePos.lat, sunrisePos.lon, date, true);
      sunsetUTC = calcSunTime(sunsetPos.lat, sunsetPos.lon, date, false);

      // Reapply altitude corrections
      sunriseMinutes = timeStringToMinutes(sunriseUTC) - altitudeCorrectionMin;
      sunsetMinutes = timeStringToMinutes(sunsetUTC) + altitudeCorrectionMin;

      // Apply timezone adjustment
      const sunriseLocal = minutesToTimeString(sunriseMinutes + tz * 60);
      const sunsetLocal = minutesToTimeString(sunsetMinutes + tz * 60);

      resultDiv.innerHTML = `
        <strong>Future Position at Sunrise:</strong><br>
        Lat: ${sunrisePos.lat.toFixed(4)}°, Lon: ${sunrisePos.lon.toFixed(4)}°<br><br>
        <strong>Future Position at Sunset:</strong><br>
        Lat: ${sunsetPos.lat.toFixed(4)}°, Lon: ${sunsetPos.lon.toFixed(4)}°<br><br>
        <strong>Sunrise (Local Time):</strong> ${sunriseLocal}<br>
        <strong>Sunset (Local Time):</strong> ${sunsetLocal}
      `;
    }

    function calcSunTime(lat, lon, date, isSunrise) {
      const zenith = 90.833;
      const D2R = Math.PI / 180;
      const R2D = 180 / Math.PI;

      const N = Math.floor((date - new Date(date.getUTCFullYear(), 0, 0)) / 86400000);
      const lngHour = lon / 15;

      let t = isSunrise ? N + ((6 - lngHour) / 24) : N + ((18 - lngHour) / 24);

      const M = (0.9856 * t) - 3.289;
      let L = M + (1.916 * Math.sin(M * D2R)) + (0.020 * Math.sin(2 * M * D2R)) + 282.634;
      L = (L + 360) % 360;

      const RA = (R2D * Math.atan(0.91764 * Math.tan(L * D2R))) % 360;
      const Lquadrant = Math.floor(L / 90) * 90;
      const RAquadrant = Math.floor(RA / 90) * 90;
      const RAfinal = (RA + (Lquadrant - RAquadrant)) / 15;

      const sinDec = 0.39782 * Math.sin(L * D2R);
      const cosDec = Math.cos(Math.asin(sinDec));

      const cosH = (Math.cos(zenith * D2R) - (sinDec * Math.sin(lat * D2R))) / (cosDec * Math.cos(lat * D2R));

      if (cosH > 1 || cosH < -1) return "No event";

      let H = isSunrise ? 360 - R2D * Math.acos(cosH) : R2D * Math.acos(cosH);
      H = H / 15;

      const T = H + RAfinal - (0.06571 * t) - 6.622;
      let UT = (T - lngHour) % 24;
      if (UT < 0) UT += 24;

      const hours = Math.floor(UT);
      const minutes = Math.floor((UT - hours) * 60);
      return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`;
    }

    function movePosition(lat, lon, speedKnots, trackDeg, hoursAhead) {
      const earthRadiusNm = 3440.065;
      const distanceNm = speedKnots * hoursAhead;

      const trackRad = trackDeg * Math.PI / 180;
      const latRad = lat * Math.PI / 180;
      const lonRad = lon * Math.PI / 180;

      const newLatRad = Math.asin(Math.sin(latRad) * Math.cos(distanceNm / earthRadiusNm) +
        Math.cos(latRad) * Math.sin(distanceNm / earthRadiusNm) * Math.cos(trackRad));

      const newLonRad = lonRad + Math.atan2(
        Math.sin(trackRad) * Math.sin(distanceNm / earthRadiusNm) * Math.cos(latRad),
        Math.cos(distanceNm / earthRadiusNm) - Math.sin(latRad) * Math.sin(newLatRad)
      );

      return {
        lat: newLatRad * 180 / Math.PI,
        lon: ((newLonRad * 180 / Math.PI + 540) % 360) - 180
      };
    }

    function getAltitudeCorrection(altitudeFt) {
      const radiusEarthFt = 20925524.9;
      const dipAngleRad = Math.sqrt(2 * altitudeFt / radiusEarthFt);
      const dipDegrees = dipAngleRad * (180 / Math.PI);
      return dipDegrees * 4;
    }

    function timeStringToMinutes(timeStr) {
      if (timeStr === "No event") return 0;
      const [h, m] = timeStr.split(':').map(Number);
      return h * 60 + m;
    }

    function minutesToTimeString(totalMinutes) {
      totalMinutes = (totalMinutes + 1440) % 1440;
      const h = Math.floor(totalMinutes / 60);
      const m = Math.floor(totalMinutes % 60);
      return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}`;
    }

    function currentUTCTime() {
      const now = new Date();
      return `${String(now.getUTCHours()).padStart(2, '0')}:${String(now.getUTCMinutes()).padStart(2, '0')}`;
    }
  </script>
</body>
</html>
