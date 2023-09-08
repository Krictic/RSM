<!--suppress ALL -->

<script>

  import { writable, derived } from "svelte/store";

  const credits = writable(1000000);
  const oreCargo = writable(0);
  const oreHold = writable(5000);

  const oreValue = writable(10);

  const ironite = writable({
      name : "Ironite",
      valueRange : [2.5, 7.5], // This represents the range of possible value for this ore
      currentValue : 2.5,
      rarity : 0.8, // Rarity goes from 0 to 1, 0 meaning impossible to find and 1 meaning can always find
      refiningYield : [55, 75], // This represents the range of possible yield values for this ore, the higher the player´s skill, the closer to the second value it is.
      description : "Ironite is ore rich in iron, can yield up to 75% metallic iron upon refinement. Very common in most asteroid fields.",
    })

    const copperite = writable({
      name : "Copperite",
      valueRange : [1.5, 5], // This represents the range of possible value for this ore
      currentValue : 1.5,
      rarity : 0.95, // Rarity goes from 0 to 1, 0 meaning impossible to find and 1 meaning can always find
      refiningYield : [72, 92], // This represents the range of possible yield values for this ore, the higher the player´s skill, the closer to the second value it is.
      description : "Copperite is ore rich in copper, can yield up to 92% metallic copper upon refinement. Very common in most asteroid fields.",
    })

  const asteroidField = writable({
    name : "Asteroid Field",
    description : "This is a place of high density of mineral-rich asteroids which can be mined by anyone willing to risk their lives.",
    ores : [$ironite, $copperite],
  })

  const cargoFull = derived(oreCargo, ($oreCargo) => $oreCargo >= $oreHold);

  const progress = writable(0);
  const message = writable("");

  const isAtField = writable(true);
  const isAtStation = writable(false);
  const mineButtonClicked = writable(false);
  const travelButtonClicked = writable(false);
  const hasDrones = writable(false);
  const areDronesMining = writable(false);
  const droneMineTime = writable(1);
  const dronesYield = writable([100, 150]);

  const ship = writable({
    name : "Small Mining Barge Type A",
    description : "This is a small mining barge which can be used to mine asteroids.",
    yieldMin : 250,
    yieldMax : 500,
    mineTime : 1,
    mineTimeReset : 1,
    travelTime : 1,
    yieldUpgradeCost : 20000,
    miningTimeUpgradeCost : 40000,
    oreCargoHoldUpgradeCost : 45000,
  })

  function asteroidSpawn() {

  }

  function dronesMining() {

  }

  function updateTick() {
    priceRandomizer();
  }

  function priceRandomizer() {
    $ironite.currentValue = Math.random() * ($ironite.valueRange[1] - $ironite.valueRange[0]) + $ironite.valueRange[0];
    $copperite.currentValue = Math.random() * ($copperite.valueRange[1] - $copperite.valueRange[0]) + $copperite.valueRange[0];
  }

  function mine() {
    if (!$isAtField) return;
    if ($cargoFull) return;

    const miningYield = Math.round(Math.random() * ($ship.yieldMax - $ship.yieldMin) + $ship.yieldMin);

    if ($oreCargo + miningYield >= $oreHold) {
      $oreCargo = $oreHold;
    } else {
      let liquidOreYield;
      if (Math.random() < 0.5) {
        liquidOreYield = (miningYield / 100) * $ironite.refiningYield[0];
        $oreCargo += liquidOreYield;
        $message = `You have mined ${Math.round(liquidOreYield)}m3 of ironite ore. (raw: ${miningYield})`;
        updateTick();
      } else {
        liquidOreYield = (miningYield / 100) * $copperite.refiningYield[0];
        $oreCargo += liquidOreYield;
        $message = `You have mined ${Math.round(liquidOreYield)}m3 of copperite ore. (raw: ${miningYield})`;
        updateTick();
      }
    }
  }

  function miningTimer() {
    if ($isAtField) {
      let timer;
      if (
        $mineButtonClicked ||
        $cargoFull ||
        $travelButtonClicked ||
        !$isAtField
      )
        return;

      $mineButtonClicked = true;
      startProgress();

      const deadline = Date.now() + $ship.mineTime * 1000;

      timer = setInterval(() => miningCallback(deadline, timer), 1000);

      return () => clearInterval(timer);
    }
  }

  /**
   * @param {number} deadline
   * @param {number} timer
   */
  function miningCallback(deadline, timer) {
    const newTimeLeft = Math.round((deadline - Date.now()) / 1000);
    $ship.mineTime = newTimeLeft > 0 ? newTimeLeft : 0;

    if (newTimeLeft <= 0) {
      clearInterval(timer);
      mine();
      $mineButtonClicked = false;
      $ship.mineTime = $ship.mineTimeReset;
    }
  }

  function travel() {
    $isAtField = !$isAtField;
    $isAtStation = !$isAtStation;
  }

  function travelTimer() {
    if ($mineButtonClicked || $travelButtonClicked) return;

    $travelButtonClicked = true;
    startProgress();

    $message = "Travelling...";

    const deadline = Date.now() + $ship.travelTime * 1000;

    const timer = setInterval(() => {
      const newTimeLeft = Math.round((deadline - Date.now()) / 1000);
      $ship.travelTime = newTimeLeft > 0 ? newTimeLeft : 0;

      if (newTimeLeft <= 0) {
        clearInterval(timer);
        travel();
        $travelButtonClicked = false;
        $ship.travelTime = 1;
        $message = "";
      }
    }, 1000);

    return () => clearInterval(timer);
  }

  function startProgress() {
    if (!$mineButtonClicked && !$travelButtonClicked) return;

    let timer;
    if ($mineButtonClicked) {
      timer = $ship.mineTime;
    } else {
      timer = $ship.travelTime;
    }

    const startTime = Date.now();

    let animationFrameId;

    const updateProgress = () => {
      const currentTime = Date.now();
      const elapsedTime = currentTime - startTime;
      const progressPercentage = (elapsedTime / (timer * 1000)) * 100;

      if (progressPercentage <= 100) {
        $progress = progressPercentage;
        animationFrameId = requestAnimationFrame(updateProgress);
      } else {
        $progress = 100;
        cancelAnimationFrame(animationFrameId);
        $progress = 0;
      }
    };

    updateProgress();
  }

  function sell() {
    if ($oreCargo === 0) {
      $message = "You have no ores to sell.";
      return;
    }

    $message = `${$oreCargo} Ores sold at ${$oreValue} credits/m3 for ${
      $oreCargo * $oreValue
    } credits.`;
    credits.update((prevCreds) => prevCreds + ($oreCargo * $oreValue))
    $oreCargo = 0;
  }

  /**
   * @param {string} upgradeType
   */
  function buyUpgrade(upgradeType) {
    if (upgradeType === "yield") {
      if ($credits >= $ship.yieldUpgradeCost) {
        $credits -= $ship.yieldUpgradeCost;
        $ship.yieldMin *= 1.05;
        $ship.yieldMax *= 1.05;
        $ship.yieldUpgradeCost *= 1.5;
        $message = "New yield: " + Math.round($ship.yieldMin) + " / " + Math.round($ship.yieldMax);
      } else {
        $message = "Cannot buy.";
      }
    } else if (upgradeType === "miningTime") {
      if ($credits >= $ship.miningTimeUpgradeCost) {
        $credits -= $ship.miningTimeUpgradeCost;
        $ship.mineTime *= 0.96;
        $ship.mineTimeReset *= 0.96;
        $ship.miningTimeUpgradeCost *= 1.5;
        $message = "New time: " + $ship.mineTime + "s";
      } else {
        $message = "Cannot buy.";
      }
    } else if (upgradeType === "oreCargo") {
      if ($credits >= $ship.oreCargoHoldUpgradeCost) {
        $credits -= $ship.oreCargoHoldUpgradeCost;
        $oreHold *= 1.05;
        $ship.oreCargoHoldUpgradeCost *= 1.4;
      } else {
        $message = "Cannot buy.";
      }
    }
  }

  $: startProgress();
</script>

<div id="game">
  <div id="info">
    <h1>Current Ship: {$ship.name}</h1>
    <h2>Credits: {Math.round($credits)}</h2>
    <h2>Ironite Yield: {$ironite.refiningYield[0]}%</h2>
    <h2>Ironite Yield: {$copperite.refiningYield[0]}%</h2>

    <p>
      Ore cargo: {Math.round($oreCargo)} / {Math.round($oreHold)} m<sup>3</sup>
    </p>

    {#if $message}
      <p id="message">{($message)}</p>
    {/if}

    <div id="progress-container">
      <div id="progress-bar" style="width: {Math.round($progress)}%" />
    </div>
  </div>

  <div id="controls">
    {#if $isAtField}
      <button on:click={miningTimer} title="Yield: {Math.round($ship.yieldMin)}/{Math.round($ship.yieldMax)} Time: {$ship.mineTime.toFixed(2)}">Mine Asteroid</button>
    {/if}

    <button on:click={travelTimer} class="to-station" title="Time to Destination: {$ship.travelTime}s">
      {#if $isAtField}
        Travel to Station
      {:else}
        Travel to Field
      {/if}
    </button>

    {#if $isAtStation}
      <button on:click={sell}>Sell Ore</button>
      <div id="upgrades">
        <button class="upgrade" on:click={() => buyUpgrade("yield")}>
          Increase Yield ({Math.round($ship.yieldUpgradeCost)} credits)
        </button>

        <button class="upgrade" on:click={() => buyUpgrade("miningTime")}>
          Decrease Mining Time ({Math.round($ship.miningTimeUpgradeCost)} credits)
        </button>

        <button class="upgrade" on:click={() => buyUpgrade("oreCargo")}>
          Increase Cargo Hold ({Math.round($ship.oreCargoHoldUpgradeCost)} credits)
        </button>
      </div>
    {/if}
  </div>
</div>

<style>
:root {
  --primary-color: #f47a20;
  --secondary-color: #ff8e3c;
  --background-color: #0a0e14;
  --text-color: #e1e5ea;
  --dark-color: #131a22;
  --accent-color: #b65c1c;
  --progress-bar-color: var(--secondary-color);
}

#game {
  max-width: 800px;
  margin: 5% auto;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
}

#info {
  flex: 1;
  text-align: center;
  padding: 2rem;
}

#controls {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
}

h1 {
  font-size: 2rem;
  margin-bottom: 1rem;
}

h2 {
  font-size: 1.8rem;
  margin-bottom: 0.5rem;
}

p {
  line-height: 1.6;
}

#progress-container {
  width: 100%;
  height: 2rem;
  background-color: var(--dark-color);
  border-radius: 0.5rem;
  margin-top: 1rem;
  overflow: hidden;
}

#progress-bar {
  width: 0;
  height: 100%;
  background-color: var(--progress-bar-color);
  transition: width ease-in-out;
}

#message {
  margin: 1rem 0;
  padding: 0.5rem;
  background-color: var(--dark-color);
  border-radius: 0.5rem;
}

button {
  background-color: var(--primary-color);
  color: var(--background-color);
  border: none;
  padding: 1rem 2rem;
  font-size: 1rem;
  border-radius: 0.5rem;
  transition: background-color 0.2s;
  margin-top: 1rem;
}

button:hover {
  background-color: var(--secondary-color);
}

button.to-station {
  background-color: var(--secondary-color);
}

#upgrades {
  margin-top: 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.upgrade {
  background-color: var(--accent-color);
  padding: 2rem;
  border-radius: 1rem;
  margin-bottom: 2rem;
  width: 10rem;
}

.upgrade:hover {
  background-color: var(--secondary-color);
}

</style>
