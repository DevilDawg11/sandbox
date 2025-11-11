<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Ultimate Sandbox Game</title>
  <style>
    html,body { height:100%; margin:0; font-family: Arial, sans-serif; }
    body { position: relative; background: #eef; overflow: hidden; }
    #ui {
      position: fixed;
      left: 8px;
      top: 8px;
      padding: 8px 10px;
      background: rgba(255,255,255,0.95);
      border-radius: 6px;
      box-shadow: 0 2px 6px rgba(0,0,0,0.2);
      z-index: 2000;
      font-size: 13px;
      line-height: 1.4;
      pointer-events: none;
    }
    #ui * { pointer-events: none; }
    .ragdoll {
      width: 40px;
      height: 40px;
      pointer-events: auto;
      transition: transform 0.08s linear;
      position: absolute;
      user-select: none;
    }
    .ragdoll img.ragdoll-body { display:block; width:100%; height:100%; }
    .dead-ragdoll { filter: grayscale(80%); transform: rotate(16deg); }
    .health-bar {
      height: 5px;
      background: linear-gradient(90deg,#4caf50,#c62828);
      position: absolute;
      top: -10px;
      left: 0;
      border-radius: 2px;
      box-shadow: 0 1px 2px rgba(0,0,0,0.3);
      width: 40px;
    }
    .weapon {
      width:40px;
      height:40px;
      pointer-events: auto;
      cursor: pointer;
      position: absolute;
    }
    .blood {
      position: absolute;
      width: 20px;
      height: 20px;
      background-color: darkred;
      border-radius: 50%;
      opacity: 0.6;
      z-index: 1500;
      pointer-events: none;
      mix-blend-mode: multiply;
    }
  </style>
</head>
<body>
  <div id="ui">
    Controls: 1=Box | 2=Melon | 3=Ragdoll | 4=Axe | 5=Sword | 6=Hammer | 7=Bomb | 8=Gun<br/>
    Click=Spawn | Drag=Swing | Space=Explode Bomb | Right Click=Shoot | T=Stun Baton | Z=Taser Gun | F=Flamethrower<br/>
    R=Rocket | Y=Trap | H=Health Pack | 9=Shield | 0=Spear | L=Laser Gun | G=Grenade
  </div>

  <canvas id="game" style="display:none;"></canvas>

  <script>
  // Simple blink simulation used for scp173 behavior
  let blinked = false;
  setInterval(() => {
    blinked = true;
    setTimeout(() => { blinked = false; }, 500);
  }, 5000);

  function spawnBlood(x, y) {
    const blood = document.createElement("div");
    blood.className = "blood";
    blood.style.left = `${x - 10}px`;
    blood.style.top = `${y - 10}px`;
    document.body.appendChild(blood);
    setTimeout(() => { blood.remove(); }, 3000);
  }

  function createRagdoll(x, y) {
    const container = document.createElement("div");
    container.className = "ragdoll";
    container.style.left = `${x}px`;
    container.style.top = `${y}px`;

    const healthBar = document.createElement("div");
    healthBar.className = "health-bar";
    healthBar.style.width = "40px";
    container.appendChild(healthBar);

    const img = document.createElement("img");
    img.className = "ragdoll-body";
    img.src = "https://img.icons8.com/ios-filled/50/000000/visible.png";
    img.alt = "ragdoll";
    container.appendChild(img);

    container.dataset.health = "100";
    document.body.appendChild(container);
    return container;
  }

  function damageRagdoll(ragdoll, amount) {
    let health = parseInt(ragdoll.dataset.health, 10);
    health = Math.max(0, health - amount);
    ragdoll.dataset.health = String(health);

    const width = Math.round((health / 100) * 40);
    const hb = ragdoll.querySelector(".health-bar");
    if (hb) hb.style.width = `${width}px`;

    const rx = parseFloat(ragdoll.style.left) || 0;
    const ry = parseFloat(ragdoll.style.top) || 0;
    spawnBlood(rx + 20, ry + 20);

    if (health <= 0 && !ragdoll.classList.contains("dead-ragdoll")) {
      ragdoll.classList.add("dead-ragdoll");
      ragdoll.style.opacity = 0.6;
      const img = ragdoll.querySelector("img");
      if (img) img.src = "https://img.icons8.com/ios-filled/50/ff0000/skull.png";
    }
  }

  // Simple AI: chases nearby ragdolls based on scp type rules
  function chaseSCP(scp) {
    const type = scp.dataset.type;
    setInterval(() => {
      const ragdolls = document.querySelectorAll(".ragdoll");
      let target = null;
      const sx = parseFloat(scp.style.left) || 0;
      const sy = parseFloat(scp.style.top) || 0;

      ragdolls.forEach(r => {
        if (r.classList.contains("dead-ragdoll")) return;
        const rx = parseFloat(r.style.left) || 0;
        const ry = parseFloat(r.style.top) || 0;
        const dx = rx - sx;
        const dy = ry - sy;
        const dist = Math.sqrt(dx*dx + dy*dy);

        if (dist < 200) {
          if (type === "scp939" || type === "scp049") {
            target = r;
          } else if (type === "scp096" && dist < 100) {
            target = r;
          } else if (type === "scp173") {
            if (blinked || dist > 100) target = r;
          }
        }
      });

      if (target) {
        const tx = parseFloat(target.style.left) || 0;
        const ty = parseFloat(target.style.top) || 0;
        const dx = tx - sx;
        const dy = ty - sy;
        const dist = Math.sqrt(dx*dx + dy*dy);
        if (dist > 5) {
          const speed = 2;
          scp.style.left = `${sx + (dx / dist) * speed}px`;
          scp.style.top  = `${sy + (dy / dist) * speed}px`;
        } else {
          damageRagdoll(target, 20);
        }
      }
    }, 200);
  }

  function spawnSCP(type, x, y) {
    const img = document.createElement("img");
    img.className = "scp";
    img.dataset.type = type;
    img.style.position = "absolute";
    img.style.left = `${x}px`;
    img.style.top = `${y}px`;
    img.style.width = "40px";
    img.style.height = "40px";
    img.alt = type + " icon";
    img.src = "https://img.icons8.com/ios-filled/50/000000/ghost.png";
    document.body.appendChild(img);
    chaseSCP(img);
    return img;
  }

  function spawnWeapon(type, x, y) {
    const img = document.createElement("img");
    img.className = "weapon";
    img.dataset.type = type;
    img.style.left = `${x}px`;
    img.style.top = `${y}px`;
    img.style.width = "40px";
    img.style.height = "40px";
    img.alt = type + " icon";
    img.src = `https://img.icons8.com/ios-filled/50/000000/${type}.png`;
    img.onerror = () => { img.src = "https://img.icons8.com/ios-filled/50/000000/hand.png"; };

    img.addEventListener("click", () => {
      const wx = parseFloat(img.style.left) || 0;
      const wy = parseFloat(img.style.top) || 0;
      const weaponType = img.dataset.type;

      document.querySelectorAll(".ragdoll").forEach(r => {
        if (r.classList.contains("dead-ragdoll")) return;
        const rx = parseFloat(r.style.left) || 0;
        const ry = parseFloat(r.style.top) || 0;
        const dx = rx - wx;
        const dy = ry - wy;
        const dist = Math.sqrt(dx*dx + dy*dy);
        if (dist < 100) {
          const dmg = weaponType === "pistol" ? 30 : weaponType === "rifle" ? 40 : 20;
          damageRagdoll(r, dmg);
        }
      });
    });

    document.body.appendChild(img);
    return img;
  }

  // demo initialization
  window.addEventListener("load", () => {
    createRagdoll(100, 100);
    createRagdoll(160, 120);

    spawnSCP("scp939", 300, 300);
    spawnSCP("scp096", 400, 300);
    spawnSCP("scp173", 500, 300);
    spawnSCP("scp049", 600, 300);

    spawnWeapon("pistol", 200, 100);
    spawnWeapon("rifle", 250, 100);
    spawnWeapon("baseball-bat", 300, 100);
    spawnWeapon("knife", 350, 100);
  });
  </script>
</body>
</html>
