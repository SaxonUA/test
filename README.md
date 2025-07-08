<!DOCTYPE html>
<html lang="uk">
<head>
  <meta charset="UTF-8">
  <title>Віджет бою</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css" />
  <style>
    :root {
      --bg-color: rgba(0, 0, 0, 0.0);
      --text-color: #ffffff;
      --glow-color: #ffa500;
    }

    body {
      background: var(--bg-color);
      color: var(--text-color);
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 1rem;
      display: none; /* Початково приховано */
      justify-content: flex-start;
      align-items: flex-start;
      height: 100vh;
      transition: opacity 0.5s ease-in-out;
    }

    .battle-widget {
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      gap: 0.3rem;
      font-size: 1.2rem;
      font-weight: bold;
    }

    .stat-block {
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }

    .stat-label {
      font-size: 1.4rem;
      text-shadow: 0 0 6px var(--glow-color);
      min-width: 30px;
      text-align: center;
    }

    .stat-value {
      font-size: 1.4rem;
      color: white;
      text-shadow: 0 0 6px var(--glow-color);
      min-width: 80px;
      text-align: left;
    }

    .hp-bar {
      width: 100px;
      height: 6px;
      background-color: rgba(255, 255, 255, 0.2);
      border-radius: 3px;
      overflow: hidden;
    }

    #hp-bar-fill {
      height: 100%;
      background-color: #ffa500;
      width: 0%;
      transition: width 0.3s ease-in-out;
    }
  </style>
</head>
<body>
  <div class="battle-widget">
    <div class="stat-block">
      <div class="stat-label"><i class="fas fa-location-dot"></i></div>
      <div id="map-name" class="stat-value">-</div>
    </div>
    <div class="stat-block">
      <div class="stat-label"><i class="fas fa-car"></i></div>
      <div id="tank-name" class="stat-value">-</div>
    </div>
    <div class="stat-block">
      <div class="stat-label"><i class="fas fa-location-crosshairs"></i></div>
      <div id="damage-value" class="stat-value">0</div>
    </div>
    <div class="stat-block">
      <div class="stat-label"><i class="fas fa-eye"></i></div>
      <div id="assist-value" class="stat-value">0</div>
    </div>
    <div class="stat-block">
      <div class="stat-label"><i class="fas fa-heart"></i></div>
      <div style="display: flex; flex-direction: column; gap: 0.2rem;">
        <div id="health-value" class="stat-value">0</div>
        <div class="hp-bar">
          <div id="hp-bar-fill"></div>
        </div>
      </div>
    </div>
  </div>

  <script>
    const MY_PLAYER_ID = 594760646;
    const damageEl = document.getElementById("damage-value");
    const assistEl = document.getElementById("assist-value");
    const healthEl = document.getElementById("health-value");
    const hpBarFill = document.getElementById("hp-bar-fill");
    const tankNameEl = document.getElementById("tank-name");
    const mapNameEl = document.getElementById("map-name");

    const ws = new WebSocket("ws://localhost:38200");

    let totalDamage = 0;
    let totalAssist = 0;
    let currentHealth = 0;
    let maxHealth = 0;

    function formatNum(num) {
      return Number(num).toLocaleString('uk-UA');
    }

    function updateHealthDisplay() {
      if (maxHealth > 0) {
        healthEl.textContent = `${formatNum(maxHealth)} / ${formatNum(currentHealth)}`;
        const percent = Math.max(0, Math.min(100, (currentHealth / maxHealth) * 100));
        hpBarFill.style.width = `${percent}%`;
      } else {
        healthEl.textContent = formatNum(currentHealth);
        hpBarFill.style.width = `0%`;
      }
    }

    ws.onopen = () => console.log("✅ WebSocket підключено");
    ws.onerror = (err) => console.error("❌ WebSocket помилка", err);
    ws.onclose = () => console.warn("🔌 WebSocket відключено");

    ws.addEventListener("message", (event) => {
      try {
        const payload = JSON.parse(event.data);
        console.log("📦 Усі вхідні дані:", payload);

        if (!payload || !payload.type || !payload.path) return;

        if (payload.type === 'state') {
          switch (payload.path) {
            case 'battle.health':
              currentHealth = payload.value;
              updateHealthDisplay();
              break;

            case 'battle.vehicle':
              if (payload.value?.localizedShortName) {
                tankNameEl.textContent = payload.value.localizedShortName;
              }
              if (payload.value?.health) {
                maxHealth = payload.value.health;
                updateHealthDisplay();
              }
              break;

            case 'battle.arena':
              if (payload.value?.localizedName) {
                mapNameEl.textContent = payload.value.localizedName;
              }
              break;

            case 'battle.efficiency.assist':
              totalAssist = payload.value;
              assistEl.textContent = formatNum(totalAssist);
              break;

            case 'hangar.isInHangar':
              const inHangar = payload.value;
              document.body.style.display = inHangar ? 'none' : 'flex';

              if (inHangar) {
                totalDamage = 0;
                totalAssist = 0;
                currentHealth = 0;
                maxHealth = 0;

                damageEl.textContent = '0';
                assistEl.textContent = '0';
                healthEl.textContent = '0';
                hpBarFill.style.width = '0%';
                tankNameEl.textContent = '-';
                mapNameEl.textContent = '-';
              }
              break;
          }
        }

        if (payload.type === 'trigger') {
          const data = payload.value;

          if (payload.path === 'battle.onDamage') {
            const attacker = data.attacker;
            if (attacker && attacker.playerId === MY_PLAYER_ID) {
              console.log("⬇️ Мій урон:", data);
              totalDamage += data.damage || 0;
              damageEl.textContent = formatNum(totalDamage);
            }
          }
        }
      } catch (e) {
        console.error("❌ Помилка при обробці повідомлення:", e);
      }
    });
  </script>
</body>
</html>
