<script lang="ts">
  import data from "./song.json";
  import midimap from "./midimapping.json";
  import Vex from "vexflow";
  import {
    Stave,
    StaveNote,
    StaveConnector,
    Dot,
    Factory,
    Formatter,
    Voice,
    Renderer,
  } from "vexflow";
  import { onMount } from "svelte";

  interface Note {
    eventType: string;
    delta: number;
    duration: number;
    midinote: number;
    velocity: number;
  }

  let containerRef;

  const BPM = data.bpm;
  const timeSig = data.timeSig;
  let STAVE_WIDTH = $state(250);
  let STAVE_HEIGHT = $state(250);
  let staveIter = 0;
  let lineIter = 0;
  // Create a mapping from MIDI note number to note name
  const midiNoteToName = new Map(
    midimap.map((item) => [
      item["MIDI note number"],
      item["Note names (English)"],
    ]),
  );
  const durationToTicks: Record<number, string> = {
    "1920": "w",
    "1440": "hd",
    "960": "h",
    "720": "qd",
    "480": "q",
    "360": "8d",
    "320": "8d",
    "240": "8",
    "180": "16d",
    "160": "16d",
    "120": "16",
    "90": "32d",
    "60": "32",
    "30": "64",
  };

  function convertNoteToVexFlow(noteName: string): string {
    // Handle cases like "F#7/Gb7" - just take the first one
    const primaryNote = noteName.split("/")[0];

    // Extract note letter and octave
    const match = primaryNote.match(/^([A-G])(#|b)?(\d+)$/);
    if (!match) {
      console.warn(`Could not parse note name: ${noteName}`);
      return ""; // Return an empty string or handle error appropriately
    }

    const noteLetter = match[1].toLowerCase();
    const accidental = match[2] || "";
    const octave = match[3];
    return `${noteLetter}${accidental}/${octave}`;
  }
  function dotted(staveNote: StaveNote, noteIndex = -1) {
      if (noteIndex < 0) {
        Dot.buildAndAttach([staveNote], {
          all: true,
        });
      } else {
        Dot.buildAndAttach([staveNote], {
          index: noteIndex,
        });
      }
      return staveNote;
    }
  onMount(() => {
    function drawNotes(
      context: any,
      notes: StaveNote[],
      staveIter: number,
      lineIter: number,
    ) {
      const chunkSize = 4; // Assuming 4 notes per bar/chunk
      for (let i = 0; i < notes.length; i += chunkSize) {
        // Create a new stave for each chunk
        const newStaveMeasure = new Stave(
          staveIter * STAVE_WIDTH,
          lineIter * STAVE_HEIGHT,
          STAVE_WIDTH,
        );
        const current_chunk = notes.slice(i, i + chunkSize);
        newStaveMeasure.setContext(context).draw();
        Formatter.FormatAndDraw(context, newStaveMeasure, current_chunk);
        staveIter++;
      }
    }
    const vf = new Factory({
      renderer: { elementId: "containerRef", width: 1000, height: 400 },
    });
    const renderer = new Renderer("containerRef", Renderer.Backends.SVG);
    // Configure the rendering context.
    renderer.resize(1000, 1000);
    const context = renderer.getContext();
    // Create a stave of width 400 at position 10, 40 on the canvas.
    const prevStaveMeasure = new Stave(0, 0, STAVE_WIDTH);

    // Add a clef and time signature.
    prevStaveMeasure.addClef("treble").addTimeSignature(timeSig);

    // Connect it to the rendering context and draw!
    prevStaveMeasure.setContext(context).draw();

    // Create the notess
    const notes2: StaveNote[] = [];
    function gatherNotes() {
      for (let i = 0; i < data.tracks[0].length; i++) {
        const note: Note = data.tracks[0][i];
        const noteName: string = midiNoteToName.get(note.midinote.toString());
        const vexFlowNote = convertNoteToVexFlow(noteName);
        const dur = durationToTicks[note.duration];
        if (dur.includes("d")) {
          notes2.push(
            dotted(new StaveNote({ keys: [vexFlowNote], duration: dur })),
          );
        } else {
          notes2.push(new StaveNote({ keys: [vexFlowNote], duration: dur }));
        }
        console.log(`Note: ${vexFlowNote}, Duration: ${dur}`);
      }
    }
    gatherNotes();
    drawNotes(context, notes2, staveIter, lineIter);
  });

</script>

<style>
  .h1 {
    color: black;
  }
</style>

<main>
  <div id="containerRef"></div>
  <h2>
    <ul>
      <li>{data.bpm}</li>
      <li>{data.timeSig}</li>
      
    </ul>
  </h2>
  
  <div>
    <!-- {#each data.tracks[0] as note}
    <ul>
        <li>{note.eventType}</li>
        <li>{note.delta}</li>
        <li>{note.duration}</li>
        <li>{midiNoteToName.get(note.midinote.toString())}</li>
        <li>{note.velocity}</li>
    </ul>
    {/each} -->
  </div>
</main>


