<script>
  import {
    keyMap,
    bassKeyMap,
    sleep,
    layout,
    bassLayout,
    buttonIdMap,
    rows,
    bassRows,
    rowTones,
    toggleBellows,
    scales,
  } from "./data.js";

  // Audio
  const audio = new (window.AudioContext || window.webkitAudioContext)();
  const gainNode = audio.createGain();
  gainNode.gain.value = 0.1;
  gainNode.connect(audio.destination);

  // State
  let direction = "pull";
  let tuning = "FBE";
  let activeButtonIdMap = {};

  // Min/max tone period
  const minTonePeriod = 300;
  const maxTonePeriod = 3000;
  let currentPeriod = 1650;
  let inverseCurrentPeriod = maxTonePeriod + minTonePeriod - currentPeriod;

  function handleChangeTuning(e) {
    tuning = e.target.value;
  }

  // Handlers
  function playTone(id) {
    const { frequency } = buttonIdMap[id];
    let oscillator;

    if (Array.isArray(frequency)) {
      oscillator = frequency.map((hz) => {
        const oscillator = audio.createOscillator();
        oscillator.type = "sawtooth";
        oscillator.connect(gainNode);
        oscillator.frequency.value = hz;
        oscillator.start();

        return oscillator;
      });
    } else {
      oscillator = audio.createOscillator();
      oscillator.type = "sawtooth";
      oscillator.connect(gainNode);
      oscillator.frequency.value = frequency;
      oscillator.start();
    }

    return { oscillator };
  }

  function stopTone(id) {
    const { oscillator } = activeButtonIdMap[id];

    if (Array.isArray(oscillator)) {
      oscillator.forEach((osc) => osc?.stop());
    } else {
      oscillator?.stop();
    }
  }

  function handleToggleBellows(newDirection) {
    if (direction !== newDirection) {
      direction = newDirection;

      const newActiveButtonIdMap = { ...activeButtonIdMap };
      let isBass = false;

      // When switching the bellows
      for (const [keyId, keyValues] of Object.entries(activeButtonIdMap)) {
        // Remove existing value

        if (Array.isArray(keyValues.oscillator)) {
          isBass = true;
          keyValues.oscillator.forEach((hz) => hz?.stop());
        } else {
          keyValues.oscillator?.stop();
        }

        // Must be reassigned in Svelte
        delete newActiveButtonIdMap[keyId];

        // Add the reverse value
        const reverseKeyId = `${keyId.split("-")[0]}-${
          keyId.split("-")[1]
        }-${newDirection}${isBass ? "-bass" : ""}`;
        const { oscillator } = playTone(reverseKeyId);

        newActiveButtonIdMap[reverseKeyId] = {
          oscillator,
          ...buttonIdMap[reverseKeyId],
        };
      }
      activeButtonIdMap = newActiveButtonIdMap;
    }
  }

  function updateActiveButtonMap(id) {
    if (!activeButtonIdMap[id]) {
      const { oscillator } = playTone(id);

      activeButtonIdMap[id] = { oscillator, ...buttonIdMap[id] };
    }
  }

  function handleKeyPressNote(e) {
    const key = `${e.key}`.toLowerCase() || e.key;

    if (key === toggleBellows) {
      handleToggleBellows("push");
      return;
    }

    const buttonMapData = keyMap[key];

    if (buttonMapData) {
      const { row, column } = buttonMapData;
      const id = `${row}-${column}-${direction}`;

      return updateActiveButtonMap(id);
    }

    const bassButtonMapData = bassKeyMap[key];
    if (bassButtonMapData) {
      const { row, column } = bassButtonMapData;
      const id = `${row}-${column}-${direction}-bass`;

      return updateActiveButtonMap(id);
    }
  }

  function handleKeyUpNote(e) {
    const key = `${e.key}`.toLowerCase() || e.key;

    if (key === toggleBellows) {
      handleToggleBellows("pull");
      return;
    }

    const buttonMapData = keyMap[key];

    if (buttonMapData) {
      const { row, column } = buttonMapData;
      const id = `${row}-${column}-${direction}`;

      if (activeButtonIdMap[id]) {
        stopTone(id);
        // Must be reassigned in Svelte
        const newActiveButtonIdMap = { ...activeButtonIdMap };
        delete newActiveButtonIdMap[id];
        activeButtonIdMap = newActiveButtonIdMap;
      }
    }

    const bassButtonMapData = bassKeyMap[key];

    if (bassButtonMapData) {
      const { row, column } = bassButtonMapData;
      const id = `${row}-${column}-${direction}-bass`;

      if (activeButtonIdMap[id]) {
        stopTone(id);
        // Must be reassigned in Svelte
        const newActiveButtonIdMap = { ...activeButtonIdMap };
        delete newActiveButtonIdMap[id];
        activeButtonIdMap = newActiveButtonIdMap;
      }
    }
  }

  const handleClickNote = (id) => {
    updateActiveButtonMap(id);
  };

  const handleClearAllNotes = () => {
    for (const [keyId, keyValues] of Object.entries(activeButtonIdMap)) {
      // Remove existing value
      if (Array.isArray(keyValues.oscillator)) {
        keyValues.oscillator.forEach((hz) => hz?.stop());
      } else {
        keyValues.oscillator?.stop();
      }
    }
    activeButtonIdMap = {};
  };

  async function playNotesInScale(idSet) {
    handleClearAllNotes();

    for (const id of idSet) {
      const [row, col, newDirection] = id.split("-");
      handleToggleBellows(newDirection);
      if (!activeButtonIdMap[id]) {
        const { oscillator } = playTone(id);

        activeButtonIdMap[id] = { oscillator, ...buttonIdMap[id] };
      }
    }

    await sleep(currentPeriod);

    for (const id of idSet) {
      stopTone(id);
      // Must be reassigned in Svelte
      const newActiveButtonIdMap = { ...activeButtonIdMap };
      delete newActiveButtonIdMap[id];
      activeButtonIdMap = newActiveButtonIdMap;
    }
  }

  const playScale = (scale, type) => async () => {
    console.dir(scale);
    const reverse = [...scales[scale][type]].reverse();
    reverse.shift();
    const scaleBackAndForth = [...scales[scale][type], ...reverse];

    for (const idSet of scaleBackAndForth) {
      await playNotesInScale(idSet);
    }

    direction = "pull"; // reset the direction to pull once done
  };

  const handlePeriodChange = (e) => {
    const val = e.target.value;
    currentPeriod = maxTonePeriod - val + minTonePeriod;
  };
</script>

<svelte:body
  on:keypress={handleKeyPressNote}
  on:keyup={handleKeyUpNote}
  on:mouseup={handleClearAllNotes} />

<main>
  <div class="mobile-only">
    <div class="banner">This app is only available on a desktop!</div>
  </div>

  <div class="layout">
    <div class="keyboard-side">
      <div class="desktop-only accordion-layout">
        {#each rows as row}
          <div class="row {row}">
            {#each layout[row].filter( ({ id }) => id.includes(direction) ) as button}
              <div
                class={`circle ${
                  activeButtonIdMap[button.id] ? "active" : ""
                } ${direction} `}
                id={button.id}
                on:mousedown={handleClickNote(button.id)}
              >
                {button.name}
              </div>
            {/each}
            <h4>{rowTones[tuning][row]}<br />{row}</h4>
          </div>
        {/each}
      </div>
    </div>

    <div class="information-side">
      <div class="information">
        <div class="title">
          <h1>Keyboard Accordion</h1>
          <h2>
            Play the diatonic button accordion with your computer keyboard!
          </h2>
        </div>
        <div>
          <h3>How to use</h3>
          <ul>
            <li>
              Each key on the keyboard corresponds to a button on the accordion.
            </li>
            <li>
              Hold down <kbd>q</kbd> to <strong>push</strong> the bellows.
              Default is
              <strong>pull</strong>.
            </li>
            <li>
              The treble side buttons begin with <kbd>z</kbd>, <kbd>a</kbd>, and
              <kbd>w</kbd>
            </li>
            <li>
              The twelve bass buttons use the number row from <kbd>1</kbd> to
              <kbd>=</kbd>
            </li>
          </ul>
        </div>

        <div class="flex">
          <div>
            <h3>Tuning</h3>
            <select on:click={handleChangeTuning}>
              <option value="FBE" selected><h2>FB♭E♭ (Fa)</h2></option>
              <option value="GCF" disabled><h2>GCF (Sol) Not yet</h2></option>
              <option value="EAD" disabled><h2>EAD (Mi) Not yet</h2></option>
            </select>
          </div>
        </div>

        <div>
          <h3>Major Scales</h3>
          <div class="scales">
            <div class="scale">
              <h4>A Major</h4>
              <div>
                <button on:click={playScale("A_", "notes")}>Lower Notes</button>
                <button on:click={playScale("A_", "thirds")}
                  >Lower Thirds</button
                >
              </div>
              <div>
                <button on:click={playScale("A", "notes")}>Notes</button>
                <button on:click={playScale("A", "thirds")}>Thirds</button>
              </div>
            </div>
            <div class="scale">
              <h4>B Major</h4>
              <div>
                <button on:click={playScale("B", "notes")}>Notes</button>
                <button on:click={playScale("B", "thirds")}>Thirds</button>
              </div>
            </div>
          </div>
          <div class="scales">
            <div class="scale">
              <h4>C Major</h4>
              <div>
                <button on:click={playScale("C", "notes")}>Notes</button>
                <button on:click={playScale("C", "thirds")}>Thirds</button>
              </div>
            </div>
            <div class="scale">
              <h4>D Major</h4>
              <div>
                <button on:click={playScale("D", "notes")}>Notes</button>
                <button on:click={playScale("D", "thirds")}>Thirds*</button>
              </div>
            </div>
            <div class="scale">
              <h4>E Major</h4>
              <div>
                <button on:click={playScale("E", "notes")}>Notes</button>
                <button on:click={playScale("E", "thirds")}>Thirds*</button>
              </div>
            </div>
          </div>
        </div>
        <div class="scales">
          <div class="scale">
            <h4>F Major</h4>
            <div>
              <button on:click={playScale("F", "notes")}>Notes</button>
              <button on:click={playScale("F", "thirds")}>Thirds</button>
            </div>
          </div>
          <div class="scale">
            <h4>G Major</h4>
            <div>
              <button on:click={playScale("G_", "notes")}>Lower Notes</button>
              <button on:click={playScale("G_", "thirds")}>Lower Thirds</button>
            </div>
            <div>
              <button on:click={playScale("G", "notes")}>Notes</button>
              <button on:click={playScale("G", "thirds")}>Thirds*</button>
            </div>
          </div>
          <div class="scale">
            <h4>B♭ Major</h4>
            <div>
              <button on:click={playScale("Bb", "notes")}>Notes</button>
              <button on:click={playScale("Bb", "thirds")}>Thirds</button>
            </div>
          </div>
          <div class="scale">
            <h4>E♭ Major</h4>
            <div>
              <button on:click={playScale("Eb", "notes")}>Notes</button>
              <button on:click={playScale("Eb", "thirds")}>Thirds</button>
            </div>
          </div>
        </div>
      </div>

      <div class="desktop-only">
        <div class="currently-playing">
          {#each Object.entries(activeButtonIdMap) as [id, value]}
            <div class="flex col">
              <div class="circle note">{value.name}</div>
              <div>
                <small
                  >Row: {id.split("-")[0]}<br /> Col: {id.split("-")[1]}</small
                >
              </div>
            </div>
          {/each}
        </div>
      </div>
    </div>
    <div class="bass-side">
      <div>
        <h3>Timing Options</h3>
        <p>Slow to fast</p>
        <input
          type="range"
          min={minTonePeriod}
          max={maxTonePeriod}
          value={inverseCurrentPeriod}
          on:change={handlePeriodChange}
        />
      </div>
      <div class="desktop-only accordion-layout">
        {#each bassRows as row}
          <div class="row {row}">
            {#each bassLayout[row].filter( ({ id }) => id.includes(direction) ) as button}
              <div
                class={`circle ${
                  activeButtonIdMap[button.id] ? "active" : ""
                } ${direction} `}
                id={button.id}
                on:mousedown={handleClickNote(button.id)}
              >
                {button.name}
              </div>
            {/each}
          </div>
        {/each}
      </div>
      <footer>
        <p>
          Made with 💾 by <a href="https://tania.dev" target="_blank">Tania</a>.
        </p>
        <p>
          <a href="https://github.com/taniarascia/accordion" target="_blank"
            >Open source</a
          >
        </p>
      </footer>
    </div>
  </div>
</main>
