<script>
// @ts-nocheck

  import { writable, derived } from "svelte/store";

  const credits = writable(1000000);
  const oreCargo = writable(0);
  const oreHold = writable(5000);

  const oreValue = 10;

  const minYield = writable(250);
  const maxYield = writable(500);
  const mineTime = writable(1);
  const mineTimeReset = writable(1);
  const travelTime = writable(1);

  const isAtField = writable(true);
  const isAtStation = writable(false);

  const cargoFull = derived(oreCargo, ($oreCargo) => $oreCargo >= $oreHold);

  const mineButtonClicked = writable(false);
  const travelButtonClicked = writable(false);

  const progress = writable(0);
  const message = writable("");

  const yieldUpgradeCost = writable(20000);
  const miningTimeUpgradeCost = writable(40000);
  const oreCargoHoldUpgradeCost = writable(45000);

  const yieldUpgradeLevel = writable(0);
  const miningTimeUpgradeLevel = writable(0);
  const oreCargoHoldUpgradeLevel = writable(0);

  const ship = writable({
    name: "Mining Barge Type A",
  })

  function mine() {
    if (!$isAtField) return;
    if ($cargoFull) return;

    const miningYield =
      Math.floor(Math.random() * ($maxYield - $minYield + 1)) + $minYield;

    if ($oreCargo + miningYield >= $oreHold) {
      $oreCargo = $oreHold;
    } else {
      $oreCargo += miningYield;
      $message = `You have mined ${Math.round(miningYield)}m3 of ironite ore.`;
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

      const deadline = Date.now() + $mineTime * 1000;

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
    $mineTime = newTimeLeft > 0 ? newTimeLeft : 0;

    if (newTimeLeft <= 0) {
      clearInterval(timer);
      mine();
      $mineButtonClicked = false;
      $mineTime = $mineTimeReset;
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

    const deadline = Date.now() + $travelTime * 1000;

    const timer = setInterval(() => {
      const newTimeLeft = Math.round((deadline - Date.now()) / 1000);
      $travelTime = newTimeLeft > 0 ? newTimeLeft : 0;

      if (newTimeLeft <= 0) {
        clearInterval(timer);
        travel();
        $travelButtonClicked = false;
        $travelTime = 1;
        $message = "";
      }
    }, 1000);

    return () => clearInterval(timer);
  }

  function startProgress() {
    if (!$mineButtonClicked && !$travelButtonClicked) return;

    let timer;
    if ($mineButtonClicked) {
      timer = $mineTime;
    } else {
      timer = $travelTime;
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

    $message = `${$oreCargo} Ores sold at ${oreValue} credits/m3 for ${
      $oreCargo * oreValue
    } credits.`;
    $credits += $oreCargo * oreValue;
    $oreCargo = 0;
  }

  function buyUpgrade(upgradeType) {
    if (upgradeType === "yield") {
      if ($credits >= $yieldUpgradeCost) {
        $credits -= $yieldUpgradeCost;
        $minYield *= 1.05;
        $maxYield *= 1.05;
        $yieldUpgradeCost *= 1.5;
        $message = "New yield: " + Math.round($minYield) + " / " + Math.round($maxYield);
      } else {
        $message = "Cannot buy.";
      }
    } else if (upgradeType === "miningTime") {
      if ($credits >= $miningTimeUpgradeCost) {
        $credits -= $miningTimeUpgradeCost;
        $mineTime *= 0.96;
        $mineTimeReset *= 0.96;
        $mine
        $miningTimeUpgradeCost *= 1.5;
        $message = "New time: " + $mineTime + "s";
      } else {
        $message = "Cannot buy.";
      }
    } else if (upgradeType === "oreCargo") {
      if ($credits >= $oreCargoHoldUpgradeCost) {
        $credits -= $oreCargoHoldUpgradeCost;
        $oreHold *= 1.05;
        $oreCargoHoldUpgradeCost *= 1.4;
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

    <h2>Credits: {Math.round($credits.toFixed(2))}</h2>

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
      <button on:click={miningTimer} title="Yield: {Math.round($minYield)}/{Math.round($maxYield)} Time: {$mineTime.toFixed(2)}">Mine Asteroid</button>
    {/if}

    <button on:click={travelTimer} class="to-station" title="Time to Destination: {$travelTime}s">
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
          Increase Yield ({Math.round($yieldUpgradeCost)} credits)
        </button>

        <button class="upgrade" on:click={() => buyUpgrade("miningTime")}>
          Decrease Mining Time ({Math.round($miningTimeUpgradeCost)} credits)
        </button>

        <button class="upgrade" on:click={() => buyUpgrade("oreCargo")}>
          Increase Cargo Hold ({Math.round($oreCargoHoldUpgradeCost)} credits)
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
    margin: 50px auto;
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
  }

  #info {
    flex: 1;
    text-align: center;
    padding: 20px;
  }

  #controls {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 20px;
  }

  h1 {
    font-size: 32px;
    margin-bottom: 20px;
  }

  h2 {
    font-size: 28px;
    margin-bottom: 10px;
  }

  p {
    line-height: 1.6;
  }

  #progress-container {
    width: 100%;
    height: 20px;
    background-color: var(--dark-color);
    border-radius: 5px;
    margin-top: 20px;
    overflow: hidden;
  }

  #progress-bar {
    width: 0;
    height: 100%;
    background-color: var(--progress-bar-color);
    transition: width ease-in-out;
  }

  #message {
    margin: 20px 0;
    padding: 10px;
    background-color: var(--dark-color);
    border-radius: 5px;
  }

  button {
    background-color: var(--primary-color);
    color: var(--background-color);
    border: none;
    padding: 10px 20px;
    font-size: 16px;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.2s;
    margin-top: 10px;
  }

  button:hover {
    background-color: var(--secondary-color);
  }

  button.to-station {
    background-color: var(--secondary-color);
  }

  #upgrades {
    margin-top: 40px;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .upgrade {
    background-color: var(--accent-color);
    padding: 20px;
    border-radius: 10px;
    margin-bottom: 20px;
    width: 200px;
  }

  .upgrade:hover {
    background-color: var(--secondary-color);
  }
</style>
